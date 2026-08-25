# RESUMO PARA CHATGPT — Sistema "Financeiro BR" (Família Guerra)

> Este documento condensa tudo que é necessário para entender o sistema e ajudar a planejar a modernização. Baseado em auditoria técnica real do código-fonte (arquivo `imoveis/index.html`, repositório GitHub `familiaguerranagringa-stack/site`), feita em agosto de 2026. Nada foi presumido — tudo aqui foi confirmado lendo o código.

## 1. Stack completa
- **Frontend**: HTML + CSS + JavaScript vanilla (ES6+), sem framework, sem build, tudo em um único arquivo (`imoveis/index.html`, ~174 KB, 2.754 linhas, ~139 funções)
- **Backend**: nenhum backend próprio. Firebase Realtime Database acessado diretamente do navegador via SDK JS (versão "compat"/legada, 10.12.0)
- **Banco de dados**: Firebase Realtime Database (NoSQL), projeto `grupofamiliaguerra-9d13e`, caminho raiz `imoveisfg`
- **Armazenamento de arquivos**: base64 dentro do próprio Realtime Database (`imoveisfg_files/{fileId}`), não Firebase Storage
- **Hospedagem**: GitHub Pages, domínio customizado `grupofamiliaguerra.com.br`, servindo `imoveis/index.html` em `/imoveis/`
- **Bibliotecas externas via CDN**: pdf.js 3.11.174 (leitura de PDF), tesseract.js 5.0.4 (OCR)
- **Sem** ORM, sem servidor de aplicação, sem API REST própria, sem CI/CD, sem testes automatizados versionados, sem Docker

## 2. Estrutura do projeto
Repositório GitHub público com vários sistemas independentes em pastas separadas; **o Financeiro BR corresponde exclusivamente à pasta `imoveis/`**. Outras pastas (`gestao/`, `financeirousa/`, `dre/`, `painel/`) são sistemas distintos, fora de escopo.

## 3. Arquitetura
SPA client-side pura. Todo o estado vive em `localStorage` do navegador e é sincronizado com o Firebase por meio de um padrão "sobrescreve tudo": cada gravação lê o localStorage inteiro e sobrescreve o nó `imoveisfg` completo no Firebase. Não há sincronização granular por registro, não há transações, não há resolução de conflito além de "o último a salvar vence".

## 4. Banco (modelo real, não SQL)
Coleções (arrays JSON) por chave, sem paginação nem limite de retenção. Ver `04-BANCO-DE-DADOS.md` para o mapeamento completo. Chaves seguem o padrão `imoveisfg_<entidade>` (cadastro) ou `imoveisfg_<entidade>_<ano>_<mes>` (lançamentos mensais).

## 5. Principais "tabelas" (entidades)
- `imoveis` — cadastro mestre de propriedades (nome, grupo, endereço, inquilino, corretor, valor)
- `alugueis` — recebimentos mensais de aluguel
- `repasses` (aba "Contas") — contas pagas via corretor (IPTU/água/condomínio), com status dinâmico
- `despesas` — gastos diretos do proprietário
- `condominio` — condomínio adiantado, aguardando devolução
- `rateio_lanc` — divisão de conta entre vários imóveis (gera despesas+repasses automaticamente)
- `transferencias` — dinheiro movido dos imóveis para uso pessoal (gera entrada espelho no módulo Pessoal)
- Módulo paralelo "Guerra Pessoal": `categorias`, `entradas`, `contas` (mesma estrutura de conceito, dados separados)

## 6. Principais módulos (visão do usuário)
**Imóveis**: Dashboard, Aluguéis, Contas, Despesas Diretas, Condomínio Adiantado, Contas Rateadas, Cadastro de Imóveis, Transferência p/ Pessoal, DRE.
**Guerra Pessoal**: Dashboard, Entradas, Contas, Cadastro (categorias), DRE. Alternável por uma chave na sidebar.

## 7. Fluxos financeiros principais
- Aluguel recebido → líquido soma no saldo do imóvel
- Conta paga (status "Pago") → abate do saldo do imóvel (contas "a pagar"/"a vencer" não abatem)
- Saldo do imóvel = líquido recebido − despesas pagas
- Rateio: uma conta única gera automaticamente uma despesa + um lançamento de Conta por imóvel participante, proporcional ao percentual definido
- Transferência p/ Pessoal → gera automaticamente uma Entrada no módulo Guerra Pessoal (único ponto de acoplamento entre os módulos)
- DRE Pessoal = total de entradas − gastos já pagos

## 8. Usuários
3 contas fixas com credenciais em texto plano **dentro do código-fonte público** (Guerra / TERRA / PAIVA). Todas com acesso idêntico e total.

## 9. Permissões
**Não existem.** Sem papéis, sem restrição por tela/módulo/ação. Qualquer um dos 3 usuários vê e edita tudo.

## 10. Integrações
Apenas Firebase (banco+arquivos) e CDNs de bibliotecas (pdf.js, tesseract.js, Google Fonts). Nenhuma API de terceiros de verdade (a "leitura de relatório de imobiliária" é um parser local de texto de PDF, não uma integração de API com a imobiliária).

## 11. Infraestrutura
GitHub Pages + domínio customizado. Sem staging, sem ambiente de desenvolvimento separado, sem CI/CD, sem monitoramento, sem backup automatizado de dados confirmado.

## 12. Deploy
`git push` para a branch `main` publica direto em produção, sem revisão, sem testes, em menos de 1 minuto. Não há Pull Requests neste fluxo hoje.

## 13. GitHub
Repositório público, 1 branch (`main`), 287 commits, sem `.gitignore`, sem Actions/workflows.

## 14. Problemas encontrados (resumo — completo em `07-SEGURANCA.md` e `10-PROBLEMAS-ENCONTRADOS.md`)
- 🔴 Senhas em texto plano no código-fonte público
- 🔴 Sem Firebase Auth — segurança depende inteiramente das Regras do Realtime Database, não verificáveis a partir do código (precisa checar no console)
- 🔴 XSS armazenado — textos do usuário inseridos sem escape de HTML
- 🟠 Chave de API do Firebase exposta (esperado/normal para apps client-side, mas reforça a necessidade de regras de banco corretas)
- 🟡 Geração de ID fraca (não é UUID real)
- 🟡 Sem validação de tamanho/tipo de arquivo no upload
- Código morto (~90 linhas de fluxo de importação de PDF substituído e nunca removido)
- Lógica duplicada entre os módulos Imóveis e Guerra Pessoal (agrupamento fixo, tabelas de contas, fluxo de marcar como pago)
- Zero testes automatizados
- Zero observabilidade (sem logs centralizados, sem monitoramento)
- Backup de dados não confirmado

## 15. Riscos
Ver `11-RISCOS-DE-MIGRACAO.md` para a lista completa. Os mais importantes: qualquer migração de banco precisa preservar os `id`s como chave estrangeira lógica; os grupos fixos (11 no Imóveis, 10 no Pessoal) são regras de negócio reais da família, não dados genéricos; o vínculo automático Transferência→Entrada é fácil de perder numa reescrita; concorrência entre 2 sessões abertas ao mesmo tempo pode sobrescrever dados silenciosamente hoje.

## 16. Débitos técnicos
Arquivo único gigante sem modularização; Firebase SDK na API legada "compat"; sem paginação de dados (tudo carregado na memória do navegador); sem particionamento/expurgo de dados antigos; heurísticas frágeis de parsing de PDF (quebram se o layout do relatório da imobiliária mudar).

## 17. Recomendações (detalhe em `12-ARQUITETURA-RECOMENDADA.md`)
Migrar por fases, sem big-bang: (0) verificar/travar as regras do Firebase AGORA, (1) criar staging com projeto Firebase separado, (2) Firebase Auth real, (3) migrar de Realtime Database blob-único para Firestore granular, (4) sanitização de HTML no frontend, (5) eliminar duplicação entre módulos, (6) observabilidade, (7) domínio final. Domínio recomendado: `financeiro-br.familiaguerra.com.br` (subdomínio, por isolamento de sessão e flexibilidade futura) em vez de subpasta.

## 18. Decisões que precisam ser tomadas pela família
1. **Confirmar/travar as Regras de Segurança do Firebase Realtime Database imediatamente** (isso não depende de nenhuma decisão de arquitetura — é urgente e independente)
2. Aprovar o modelo de banco de destino: continuar no ecossistema Firebase (Firestore) ou migrar para um banco relacional com backend próprio?
3. Definir se o frontend será reescrito num framework moderno ou se a base atual (vanilla JS) será mantida e apenas reorganizada em módulos
4. Escolher entre subpasta (`familiaguerra.com.br/financeiro-br`) e subdomínio (`financeiro-br.familiaguerra.com.br`)
5. Decidir se vale a pena, nesta modernização, também eliminar a duplicação entre Imóveis e Guerra Pessoal criando uma base de código compartilhada entre os dois módulos
6. Definir apetite de investimento/tempo para as fases propostas (algumas são rápidas — ex: Fase 0 — outras são maiores esforços — ex: Fase 3)
