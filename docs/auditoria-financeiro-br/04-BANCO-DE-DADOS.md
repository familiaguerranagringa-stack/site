# 04 — Banco de Dados

## Atenção conceitual
Não existe um banco de dados relacional. O que segue é o mapeamento das **"entidades"** (coleções de registros JSON) dentro do Firebase Realtime Database, adaptado ao formato pedido (nome / finalidade / campos / chaves) para fins de planejamento de migração — mas nenhuma dessas é uma tabela SQL real, não há `PRIMARY KEY`/`FOREIGN KEY` impostas pelo banco, e nenhuma integridade referencial é garantida pelo sistema (tudo é responsabilidade do código JS).

Todos os registros usam `id` gerado no cliente via função `uuid()` (short random string, **não é UUID v4 padrão** — ver `07-SEGURANCA.md`/`10-PROBLEMAS-ENCONTRADOS.md`).

## Nó raiz no Firebase
```
imoveisfg/                     ← nó único, sobrescrito inteiro a cada gravação
  ├─ _ts                       ← timestamp da última escrita
  ├─ imoveisfg_imoveis         ← array de imóveis (não paginado por mês)
  ├─ imoveisfg_alugueis_AAAA_MM
  ├─ imoveisfg_repasses_AAAA_MM
  ├─ imoveisfg_despesas_AAAA_MM
  ├─ imoveisfg_condominio_AAAA_MM
  ├─ imoveisfg_rateio_lanc_AAAA_MM
  ├─ imoveisfg_transferencias_AAAA_MM
  ├─ imoveisfg_files/{fileId}  ← arquivos em base64
  ├─ imoveisfg_theme / imoveisfg_theme_<usuario>
  ├─ pessoalfg_categorias
  ├─ pessoalfg_entradas_AAAA_MM
  ├─ pessoalfg_contas_AAAA_MM
  └─ pessoalfg_theme_<usuario>
```

## Entidades — Módulo Imóveis

### `imoveis` (chave: `imoveisfg_imoveis`)
Cadastro mestre de imóveis.
- **Campos**: `id`, `nome`, `grupo`, `endereco`, `inquilino`, `corretor`, `valor`
- **Chave primária**: `id` (string, gerada no cliente)
- **Referenciada por**: `imovelId` em `alugueis`, `repasses`, `despesas`, `condominio`, e (indiretamente, via `partes[].imovelId`) em `rateio_lanc`
- **Sem chave estrangeira imposta**: nada impede um registro com `imovelId` apontando para um imóvel excluído (registro órfão)

### `alugueis` (chave: `imoveisfg_alugueis_<ano>_<mes>`)
Recebimentos de aluguel do mês.
- **Campos**: `id`, `imovelId`, `bruto`, `liquido`, `data`, `vencimento`, `obs`, `comprovante`
- **FK lógica**: `imovelId` → `imoveis.id`

### `repasses` (chave: `imoveisfg_repasses_<ano>_<mes>`) — aba "Contas"
- **Campos**: `id`, `imovelId`, `tipo`, `valor`, `envio` (vencimento), `formaPagamento`, `status`, `obs`, `boleto`, `comprovantePagamento`, `comprovantePagamento2`, `dataPagamento`, opcionalmente `rateioLancId`
- **FK lógica**: `imovelId` → `imoveis.id`; `rateioLancId` → `rateio_lanc.id` (quando gerado por rateio)

### `despesas` (chave: `imoveisfg_despesas_<ano>_<mes>`)
- **Campos**: `id`, `imovelId`, `desc`, `valor`, `forma`, `data`, `comprovantes[]`, opcionalmente `rateioLancId`

### `condominio` (chave: `imoveisfg_condominio_<ano>_<mes>`)
- **Campos**: `id`, `imovelId`, `valor`, `data`, `devolvido` (boolean)

### `rateio_lanc` (chave: `imoveisfg_rateio_lanc_<ano>_<mes>`)
Divisão de uma conta entre vários imóveis.
- **Campos**: `id`, `valor`, `data`, `partes` (array de `{imovelId, pct}`), `tipo`, `formaPagamento`, `status`, `obs`, `comprovantes[]`
- **Efeito colateral no salvamento**: gera automaticamente N registros em `despesas` e N em `repasses` (um por imóvel da divisão), todos marcados com `rateioLancId` apontando de volta

### `transferencias` (chave: `imoveisfg_transferencias_<ano>_<mes>`)
- **Campos**: `id`, `valor`, `data`, `obs`
- **Efeito colateral no salvamento**: cria automaticamente um registro espelho em `pessoalfg_entradas_<mesmo_periodo>` com `origem:'REPASSE IMÓVEIS'` — **este é o único ponto de acoplamento entre os dois módulos**

## Entidades — Módulo Guerra Pessoal

### `categorias` (chave: `pessoalfg_categorias`)
- **Campos**: `id`, `nome`, `tipo` (Casa/Pessoa/Veículo/Outro)

### `entradas` (chave: `pessoalfg_entradas_<ano>_<mes>`)
- **Campos**: `id`, `valor`, `data`, `obs`, `origem`, `comprovante`

### `contas` (Pessoal) (chave: `pessoalfg_contas_<ano>_<mes>`)
- **Campos**: `id`, `categoriaId`, `desc` (tipo/especificação), `valor`, `forma`, `vencimento`, `status`, `obs`, `boleto`, `comprovantePagamento`, `comprovantePagamento2`, `dataPagamento`
- **FK lógica**: `categoriaId` → `categorias.id`

## Diagrama de relacionamento (textual)

```
imoveis (1) ──< (N) alugueis
imoveis (1) ──< (N) repasses
imoveis (1) ──< (N) despesas
imoveis (1) ──< (N) condominio
imoveis (N) ──< via rateio_lanc.partes[] >── rateio_lanc (1)
rateio_lanc (1) ──< (N) despesas (geradas)
rateio_lanc (1) ──< (N) repasses (gerados)

transferencias (1) ──── gera 1:1 ──> pessoalfg.entradas (mesmo período)

pessoalfg.categorias (1) ──< (N) pessoalfg.contas
```

## Observações críticas sobre o modelo de dados
1. **Não há particionamento por mês nas entidades cadastrais** (`imoveis`, `categorias`) — crescem indefinidamente num único array.
2. **Entidades por mês crescem sem limite de retenção** — não há arquivamento/expurgo de dados antigos.
3. **Sem paginação**: toda tela carrega o array inteiro do mês/cadastro na memória do navegador.
4. **Sem transações**: as operações de rateio (que escrevem em 3 entidades diferentes) não são atômicas — uma falha de rede no meio do processo pode deixar dados inconsistentes (ex: despesa criada mas conta não).
5. **Duplicação de conceito de "status"**: `STATUS_REPASSE` e a lógica de badge são compartilhadas entre os dois módulos, mas o "status Pago" trava a edição de forma diferente em cada tela.
