# 15 — Especificação Funcional Completa do Sistema Legado (`imoveis/index.html`)

> **Para quem está lendo isto sem contexto**: este documento descreve, campo por campo e regra por regra, um sistema de gestão financeira de locações de imóveis + finanças pessoais da Família Guerra ("IMÓVEIS — Família Guerra Holding"), atualmente rodando em produção como uma página HTML única (`imoveis/index.html`, ~2.754 linhas, ~139 funções JavaScript), sem framework, sem backend próprio, usando Firebase Realtime Database como armazenamento. Seu objetivo, se você está lendo isto para reconstruir o sistema (projeto **FINANCEIROBR**), é reproduzir **todo o comportamento funcional aqui descrito**, decidindo conscientemente (não por acidente) o que preservar, o que melhorar e o que descartar — seções específicas no final deste documento já fazem essa triagem.
>
> Nenhuma senha real é reproduzida neste documento. Onde uma credencial existe no sistema legado, isso é indicado sem o valor.
>
> Este documento **não altera** o sistema legado, o Firebase ou qualquer dado. É extração de conhecimento apenas.

---

## SUMÁRIO

1. Visão geral e navegação
2. Módulo IMÓVEIS — telas, campos, regras (detalhado)
3. Módulo GUERRA PESSOAL — telas, campos, regras (detalhado) + comparação com Imóveis
4. Todos os cálculos financeiros (fórmulas exatas)
5. Fluxos passo a passo
6. Árvore completa de dados no Firebase + mapeamento TELA→CAMPO→FIREBASE
7. Funcionalidades ocultas / menos óbvias
8. Preservar / Melhorar / Não Reproduzir
9. Mapa de migração (DADO ANTIGO → SIGNIFICADO → NOVA ENTIDADE)
10. Checklist de paridade

---

## 1. VISÃO GERAL E NAVEGAÇÃO

### 1.1 Estrutura de tela
- **Sidebar (esquerda, fixa, 230px)**: logo "Família Guerra Holding" → nome do módulo atual (IMÓVEIS ou GUERRA PESSOAL, dentro de uma caixa de altura fixa 38px para não pular de posição) → subtítulo ("Gestão de Locações" / "Finanças Pessoais") → **chave (switch) de alternância de módulo** com legenda → menu de navegação (muda conforme o módulo) → rodapé fixo (tema, sincronizar, corrigir anexos, limpar cache, sair).
- **Topbar (superior, dentro da área principal)**: título da aba atual (à esquerda) → seletor de Mês + seletor de Ano (dispara `changePeriod()`) → botão "⎙ Imprimir" (`window.print()` nativo do navegador, sem geração de PDF customizada) → indicador de status de sincronização (texto + cor, ex: "✓ Salvo", "⟳ Salvando...", "• Online").
- **Área de conteúdo**: renderizada inteiramente por uma função `renderX()` que devolve uma string HTML, injetada via `innerHTML`. Não há transição de página — é sempre a mesma página, só o conteúdo interno muda.
- **Modal genérico**: um único `<div id="modal-root">` reaproveitado por todas as janelas modais do sistema (edição, confirmação, cadastro rápido) — só um modal pode estar aberto por vez.
- **Notificações (toasts)**: um único `<div id="notif">` no topo da página, onde `notify(msg, tipo)` insere um elemento por 3,2 segundos e depois remove.

### 1.2 Seletor de módulo (chave/switch)
- Elemento: `<input type="checkbox" id="toggleModulo" onchange="alternarModulo(this.checked)">` estilizado como uma chave (trilho + bolinha), com brilho verde (`#10B981`) quando desmarcado (módulo Imóveis) e azul (`#3B82F6`) quando marcado (módulo Guerra Pessoal).
- Ao alternar, a função `trocarModulo(modulo)`:
  1. Atualiza `ST.modulo`
  2. Mostra/esconde os dois blocos de menu (`#menu-imoveis-only` / `#menu-pessoal-only`)
  3. Troca o texto do título da marca (`IMÓVEIS` ↔ `GUERRA PESSOAL`) e subtítulo
  4. Troca a legenda ao lado da chave
  5. Adiciona/remove classes CSS `modulo-pessoal`/`tema-pessoal` no `<body>` (usadas para eventual estilização condicional — hoje sem efeito visual adicional relevante detectado além do que já é feito diretamente)
  6. Zera a navegação para a primeira aba do módulo escolhido (`dashboard` ou `gp_dashboard`) e re-renderiza

### 1.3 Menu — Módulo IMÓVEIS
| Seção do menu | Item | `data-tab` | Ícone |
|---|---|---|---|
| Visão Geral | Dashboard | `dashboard` | 📊 |
| Lançamentos | Aluguéis | `alugueis` | 🏠 |
| Lançamentos | Contas | `repasses` | 📄 |
| Lançamentos | Despesas Diretas | `despesas` | 📉 |
| Lançamentos | Condomínio Adiantado | `condominio` | 🔁 |
| Lançamentos | Contas Rateadas | `rateio` | ➗ |
| Gestão | Cadastro de Imóveis | `imoveis` | 🏢 |
| Gestão | Transferência p/ Pessoal | `pessoal` | ↪ |
| Gestão | DRE | `dre` | 📊 |

### 1.4 Menu — Módulo GUERRA PESSOAL
| Seção do menu | Item | `data-tab` | Ícone |
|---|---|---|---|
| Visão Geral | Dashboard (Pessoal) | `gp_dashboard` | 📊 |
| Lançamentos | Entradas | `gp_entradas` | 💰 |
| Lançamentos | Contas | `gp_contas` | 📄 |
| Gestão | Cadastro | `gp_cadastro` | 📝 |
| Gestão | DRE | `gp_dre` | 📈 |

### 1.5 Rodapé da sidebar (comum aos dois módulos)
| Botão | Ação |
|---|---|
| 🌙/⚔️/🛡️ (alterna rótulo) "Darkness"/"Brightness" | `alternarTema()` — troca tema claro/escuro |
| ☁ Sincronizar | `syncManual()` — força sincronização com o Firebase |
| 🩹 Corrigir anexos antigos | `migrarAnexosAntigos()` — ferramenta de manutenção (seção 7) |
| 🧹 Limpar cache local | `limparCacheLocal()` — ferramenta de manutenção (seção 7) |
| ↩ Sair | `doLogout()` — encerra a sessão |

### 1.6 Autenticação
- Tela de login separada (`#login-screen`), com campos **Usuário** (`#l-user`, texto) e **Senha** (`#l-pass`, senha), botão **Entrar** (`doLogin()`).
- Comparação: usuário+senha digitados são comparados contra uma constante `CREDS` (1 usuário) e uma lista `CREDS_EXTRA` (2 usuários adicionais). Comparação de string simples — sem hash.
- **Três contas existem hoje**: uma delas com nome de usuário "Guerra"; as outras duas nomeadas "TERRA" e "PAIVA" (com um campo `nome` de exibição "Terra"/"Paiva" que não é usado em nenhuma tela atualmente). As senhas reais não são reproduzidas aqui.
- Login bem-sucedido grava em `sessionStorage`: `imoveisfg_auth='1'` e `imoveisfg_user=<usuario digitado>`. Chama `initApp()` e `sincronizarTemaComNuvem()`.
- Falha: exibe mensagem fixa "Usuário ou senha incorretos." (`#l-err`).
- Logout: remove `imoveisfg_auth` do `sessionStorage` e volta pra tela de login (não limpa `localStorage`, então os dados carregados continuam em cache até o próximo carregamento).
- **Não existe** recuperação de senha, criação de conta pela interface, nem qualquer indicação de "usuário logado" nas telas além da preferência de tema.

---

## 2. MÓDULO IMÓVEIS — TELAS DETALHADAS

### 2.1 Regra de agrupamento fixo (usada em quase todas as telas do módulo)

Todo imóvel cadastrado é automaticamente encaixado em um de **11 grupos fixos e ordenados**, definidos na constante `GRUPOS_ORDEM`. A ordem abaixo é a ordem exata de exibição em todas as telas (Dashboard, Aluguéis, Contas):

| # | Título do grupo | Palavras-chave de correspondência (case/acento-insensível) |
|---|---|---|
| 1 | Paul Harris | "paul harris" |
| 2 | CHAVEIRO | "getulio vargas" / "getúlio vargas" |
| 3 | Rua Rudy Alberto | "rudy alberto" |
| 4 | RESIDENCIAL FAMILIA GUERRA | "erineia" / "erinéia" / "residencial familia guerra" / "residencial família guerra" |
| 5 | Travessa Gomes | "travessa gomes" |
| 6 | SALAS COMERCIAIS ENGRACIA | "engracia" / "engrácia" |
| 7 | James Mendonça | "james mendonca" / "james mendonça" / "mendonca" / "mendonça" |
| 8 | Recanto Família Guerra | "recanto" |
| 9 | Rua Carioca | "rua carioca" / "carioca" |
| 10 | Rua Ouro Preto | "ouro preto" |
| 11 | Rua Liliane | "liliane" |

**Regra especial embutida**: se o nome do imóvel contém "SALA 603" ou "SALA 609" (verificação textual, não importa qual endereço/grupo mais), ele **sempre** vai para o grupo "SALAS COMERCIAIS ENGRACIA", mesmo que o restante do texto sugira outro logradouro. Essa regra é verificada **antes** de qualquer outra correspondência.

**Algoritmo de decisão do grupo de um imóvel** (`getGrupoTituloImovel`):
1. Se o campo `grupo` do imóvel foi preenchido manualmente, compara (normalizado: maiúsculas, sem acento) contra os títulos e palavras-chave de `GRUPOS_ORDEM`. Se bater, usa esse grupo.
2. Senão, aplica a mesma lógica de correspondência de palavra-chave sobre `nome + " " + endereco` combinados (cobre tanto "TRAVESSA GOMES CASA 105" quanto "TRAVESSA GOMES, 50, CASA 105" — pontuação e números de rua não atrapalham, pois a checagem é "o texto contém a palavra-chave", não igualdade exata).
3. Se nada corresponder, o imóvel é considerado **órfão**: vira seu **próprio grupo isolado** (título = o próprio nome do imóvel), sempre exibido **depois** dos 11 grupos fixos, na ordem em que aparece no cadastro.

Dentro de cada grupo, os imóveis (e, nas telas de lançamento, as linhas) são ordenados por **ordenação natural** (`ordemNatural`): comparação alfabética que entende números corretamente (ex.: "Casa 2" vem antes de "Casa 10", não depois, ao contrário de uma ordenação puramente textual).

### 2.2 Tela: Cadastro de Imóveis (`imoveis`)

**Finalidade**: CRUD do cadastro mestre de imóveis, base de todo o resto do sistema.

**Acesso**: menu lateral → Gestão → Cadastro de Imóveis.

**Estrutura visual**:
- Cabeçalho com dois botões: **"+ Novo Imóvel"** (abre/fecha painel retrátil de cadastro) e **"🔧 Corrigir Grupos Automaticamente"** (ferramenta de manutenção em massa, seção 7)
- Painel retrátil "Novo Imóvel" (fechado por padrão, animação de altura ao abrir/fechar)
- Tabela "Imóveis Cadastrados" (sempre visível, não paginada, mostra **todos** os imóveis já cadastrados, na ordem dos grupos fixos)

**Formulário "Novo Imóvel"** — campo por campo:

| Rótulo exibido | ID técnico | Tipo | Obrigatório? | Padrão | Validação | Observação |
|---|---|---|---|---|---|---|
| Grupo / Empreendimento | `im-grupo` | texto (`input-uppercase` — visual apenas, CSS `text-transform:uppercase`, o valor salvo **não** é forçado a maiúsculas no JS) | Não | vazio | nenhuma | Se vazio, o sistema tenta detectar pelo nome/endereço na hora de exibir (não grava o valor detectado de volta automaticamente, a menos que se use "Corrigir Grupos") |
| Nome / Identificação | `im-nome` | texto | **Sim** | vazio | `if(!nome) notify('Informe o nome do imóvel','err')` — bloqueia o salvamento | É o campo usado como rótulo em todas as tabelas e seletores |
| Endereço | `im-end` | texto | Não | vazio | nenhuma | Usado no matching automático de PDF |
| Inquilino | `im-inq` | texto | Não | vazio | nenhuma | Só informativo — não gera nenhum cálculo |
| Imobiliária / Corretor | `im-corr` | texto | Não | vazio | nenhuma | Só informativo |
| Valor Aluguel Contratado | `im-val` | número (`step=0.01`) | Não | vazio → `0` | `+valor||0` (conversão para número, `NaN`/vazio vira 0) | Valor de referência do contrato — **não** é usado em nenhum cálculo de saldo; é só exibido na listagem |

**Botão**: "+ ADICIONAR IMÓVEL" → `addImovel()`.

**Destino no banco**: novo objeto `{id: uuid(), nome, grupo, endereco, inquilino, corretor, valor}` é adicionado ao array em `imoveisfg_imoveis` (chave sem período — é cadastro permanente, não mensal).

**Tabela "Imóveis Cadastrados"** — colunas: Imóvel (nome + endereço em cinza abaixo, se houver), Grupo, Inquilino, Corretor, Valor Contratado, Ações (editar/excluir).

**Editar** (`abrirEdicaoImovel`/`salvarEdicaoImovel`): modal com os mesmos 6 campos (prefixo `ed-`), texto de aviso: *"O valor do aluguel muda com o tempo — atualize aqui sempre que reajustar ou trocar de inquilino. O Grupo define em qual bloco o imóvel aparece no Dashboard, Aluguéis e Contas."* Ao salvar, sobrescreve o objeto in-place no array.

**Excluir** (`delImovel`): `confirm()` nativo do navegador com o texto *"Excluir este imóvel? Lançamentos vinculados continuam existindo mas ficarão sem referência."* — **atenção**: isso é literal. Excluir um imóvel **não** exclui nem avisa sobre os lançamentos (aluguéis, contas, despesas) que referenciam esse `imovelId` — eles ficam órfãos, e `imovelNome(id)` passa a retornar `'—'` para eles.

### 2.3 Ferramenta: Corrigir Grupos Automaticamente

Botão na tela de Cadastro de Imóveis. Função `corrigirGruposAutomaticamente()`:
1. Para cada imóvel cadastrado, calcula o grupo que **seria** detectado hoje via `getGrupoTitulo(nome + " " + endereco)` (nota: usa a versão que **não** prioriza o campo `grupo` manual — recalcula do zero pelo texto).
2. Se o grupo detectado é diferente do que está salvo no campo `grupo` do imóvel (incluindo o caso de estar vazio), acumula numa lista de mudanças.
3. Se não há nada para corrigir: notificação "Nenhum imóvel precisava de correção — todos os grupos já estavam certos."
4. Se há mudanças: `confirm()` nativo mostrando a lista completa no formato `Nome do Imóvel: "grupo antigo" → "grupo novo"`, uma por linha.
5. Se confirmado: sobrescreve o campo `grupo` de todos os imóveis afetados de uma vez e salva.

### 2.4 Tela: Aluguéis (`alugueis`)

**Finalidade**: registrar os recebimentos mensais de aluguel líquido por imóvel — manual ou por leitura automática de um relatório PDF de administração de imobiliária (Aprisco/Alude).

**Estrutura**: painel retrátil "Novo Recebimento (Manual ou Relatório)" no topo, seguido de um bloco de tabela por grupo fixo de imóveis (mesmo agrupamento da seção 2.1), cada um com subtotal, e um card de "Total Líquido do Mês — Todos os Módulos" no final.

**Texto de ajuda fixo no topo do painel**: *"Lance aqui o valor líquido que a imobiliária de fato repassou — já descontado o que ela paga direto (IPTU/água, quando for o caso). O bruto é só referência. Se anexar um relatório da imobiliária (Aprisco/Alude) em PDF no rodapé, os campos abaixo são preenchidos automaticamente pra você conferir."*

**Formulário "Novo Recebimento"** — campo por campo:

| Rótulo | ID | Tipo | Obrigatório | Padrão | Validação | Preenchimento automático via PDF? |
|---|---|---|---|---|---|---|
| Imóvel | `al-im` | select (agrupado por `<optgroup>` = os 11 grupos + órfãos) | **Sim** | primeira opção / vazio se não houver imóveis | `if(!imovelId) notify('Cadastre e selecione um imóvel','err')` | Sim — ver seção 5.3 (fluxo de importação) |
| Valor Boleto (bruto) | `al-bruto` | número | Não | vazio → 0 | conversão numérica | Sim, com o "Total Cobrado" extraído do relatório |
| Valor Líquido Recebido | `al-liquido` | número | Não | vazio → 0 | conversão numérica | Sim, com o "Valor Repassado" extraído |
| Data do Repasse | `al-data` | data | Não | hoje | — | Sim, com a data de repasse extraída |
| Data de Vencimento | `al-vencimento` | data | Não | vazio | — | Sim, com a data de vencimento da cobrança extraída |
| Observação | `al-obs` | texto | Não | vazio | — | Não — **nunca** preenchido automaticamente (decisão explícita de produto, documentada no código como correção proposital) |

Abaixo do campo Imóvel existe um elemento dinâmico (`#wrapperAvisoImovel`) que muda de conteúdo conforme o resultado da leitura do PDF (detalhado na seção 5.3), sempre com um link para cadastrar um novo imóvel na hora, sem sair da tela.

**Rodapé do formulário**: campo de arquivo "Anexar Relatório / Comprovante (PDF)" (`al-comprovante`, aceita `.pdf,image/*`, dispara `handleAluguelAnexo(this)` no `onchange`) + botão **"+ REGISTRAR RECEBIMENTO"** (`addAluguel(this)`).

**Ao registrar** (`addAluguel`):
1. Valida que um imóvel foi selecionado.
2. Monta o objeto `{id, imovelId, bruto, liquido, data, vencimento, obs}`.
3. Desabilita o botão e muda o texto para "Enviando..." (feedback visual, evita duplo clique).
4. Processa o arquivo anexado (se houver) via `lerArquivo` — nome automático gerado como `{IMOVEL}_{RECIBO}_{MESANO}.{ext}` (função `nomeAutomatico`, ver seção 7.2).
5. Salva no array `imoveisfg_alugueis_<ano>_<mes>`.
6. Reabilita o botão, notifica "Recebimento registrado", re-renderiza a tela inteira.

**Tabela por grupo** — colunas: Subunidade, Bruto, Líquido, Data de Repasse, Vencimento, Obs (truncada com reticências, tooltip com o texto completo), Comprovante (botão compacto "📎 ANEXO"), Ações (editar/excluir). Rodapé de cada bloco: "Subtotal — {Nome do Grupo}" somando a coluna Líquido.

**Editar** (`abrirEdicaoAluguel`/`salvarEdicaoAluguel`): modal com Imóvel, Valor Boleto, Valor Líquido, Data do Repasse, Data de Vencimento, Observação, e campo para **substituir** o comprovante (mantém o antigo se nada for enviado). Texto de ajuda: *"Corrija valores lidos errado (ex: juros/multa que não entraram) sem precisar excluir e relançar."*

**Excluir** (`delItem('alugueis', id)`): remove sem confirmação adicional (usa a função genérica de exclusão, que não pede `confirm()` para esta entidade especificamente — diferente de Imóvel e Rateio, que pedem).

### 2.5 Tela: Contas (`repasses`)

**Finalidade**: registrar contas que o proprietário envia para o corretor pagar usando o próprio dinheiro do aluguel (IPTU, água, condomínio, etc.) — **este valor NÃO entra no cálculo de saldo do DRE/`calcMes`** porque já foi descontado antes de chegar como "líquido" em Aluguéis. É um registro de conferência/comprovação, não uma despesa contábil adicional (ver seção 4 sobre a divergência com o Dashboard).

**Aviso condicional no topo**: se existir, em qualquer mês, algum lançamento com um campo legado `relatorioOrigem` (resquício de uma versão antiga do sistema que anexava relatórios automaticamente por engano), aparece uma faixa de aviso com botão **"🧹 Remover relatórios antigos"** (`limparRelatoriosAntigos()` — remove só o campo `relatorioOrigem` de todos os lançamentos afetados em todos os meses, não exclui o lançamento).

**Formulário "Nova Conta Enviada ao Corretor"**:

| Rótulo | ID | Tipo | Obrigatório | Padrão | Opções |
|---|---|---|---|---|---|
| Imóvel | `rp-im` | select (agrupado) | Sim | — | — |
| Tipo | `rp-tipo` | select | Sim (sempre tem valor, é select) | "IPTU" (primeira opção) | IPTU, Água, Condomínio, Luz, Internet, Manutenção, Outro |
| Valor | `rp-val` | número | Não (mas sem validação de valor > 0) | vazio → 0 | — |
| Data de Vencimento | `rp-envio` | data | Não | hoje | — |
| Forma de Pagamento | `rp-forma` | select | Sim | "Boleto" | Boleto, Pix |
| Status | `rp-status` | select | Sim | "A Pagar" (`a_pagar`) | ver Status Dinâmico abaixo |
| Observações | `rp-obs` | texto | Não | vazio | — |

**Anexos no rodapé**: Boleto (`rp-file-boleto`), Comprovante 1 (`rp-file-comp`), Comprovante 2 (`rp-file-comp2`) — todos opcionais no momento do cadastro inicial (a obrigatoriedade de comprovante só se aplica quando o **Status** é definido como Pago).

**Status Dinâmico (regra de negócio central, compartilhada com o módulo Guerra Pessoal)** — `statusBadge()`:
- Se `status === 'pago'` → exibe **Pago** (badge azul) — **imutável pela data**, uma vez marcado como pago fica pago até alguém mudar manualmente.
- Senão, calcula `dias = (data de vencimento) - (hoje)`, em dias inteiros, com horas zeradas em ambas as datas:
  - `dias === null` (sem data de vencimento cadastrada) → **A Pagar**
  - `dias < 0` (já venceu) → **Vencida** (badge vermelho)
  - `dias === 0` (vence hoje) → **A Pagar**
  - `0 < dias <= 5` → **A Vencer** (badge laranja/amber)
  - `dias > 5` → **A Pagar** (badge amarelo claro)
- **Importante**: os status "A Vencer" e "Vencida" **nunca são valores armazenados permanentemente** de forma significativa — são recalculados a cada renderização a partir da data. Só "A Pagar" (padrão/fallback) e "Pago" (definido explicitamente pelo usuário) são os dois estados reais persistidos com efeito duradouro.

**Marcar como Pago pela tabela** (botão ✓, só aparece se `status !== 'pago'`) → `marcarRepassePago(id)`:
1. Abre modal: Data do Pagamento (padrão hoje), Boleto (opcional), **Comprovante de Pagamento 1 (obrigatório)**, Comprovante de Pagamento 2 (opcional).
2. `confirmarRepassePago`: bloqueia se não houver `mp-comp` selecionado **e** o registro ainda não tiver `comprovantePagamento` salvo de antes. Ao confirmar: processa os até 3 arquivos em cadeia (boleto → comp1 → comp2), define `status='pago'`, `dataPagamento` = data escolhida, toca um som de sucesso (`playSuccessSound`, dois beeps ascendentes via Web Audio API, 660Hz e 880Hz).

**Editar** (`abrirEdicaoRepasse`/`salvarEdicaoRepasse`): modal completo com Imóvel, Tipo, Valor, Data de Vencimento, Forma de Pagamento, Status (mudar para "Pago" aqui também exige comprovante 1, com a mesma trava), Data de Pagamento (editável manualmente, sobrepõe o auto-preenchimento), Comprovante 1/2 (substituíveis), Observações.

**Tabela por grupo** — colunas: Imóvel, Tipo, Valor, Vencimento, Data Pagto, Forma Pgto, Status, Obs, Boleto, Comprov. 1, Comprov. 2, Ações. Rodapé: "Subtotal — {Grupo}" somando Valor.

### 2.6 Tela: Despesas Diretas (`despesas`)

**Finalidade**: gastos que o proprietário paga diretamente (manutenção, reparos) — **estes SIM entram no cálculo de saldo do DRE** (`calcMes`), mas **não** no cálculo de saldo exibido no Dashboard (que usa "Contas" pagas — ver seção 4).

**Formulário "Nova Despesa Direta"**:

| Rótulo | ID | Tipo | Obrigatório | Padrão |
|---|---|---|---|---|
| Imóvel | `ds-im` | select | Sim | — |
| Descrição | `ds-desc` | texto | Não (mas se vazio vira "Despesa" automaticamente) | vazio |
| Valor | `ds-val` | número | Não | 0 |
| Forma de Pagamento | `ds-forma` | select | Sim | "PIX" | Opções: PIX, Boleto, Dinheiro, Cartão |
| Data | `ds-data` | data | Não | hoje |

**Anexo**: um único campo de arquivo no rodapé (`ds-file`, opcional no cadastro).

**Particularidade — múltiplos comprovantes por parcela**: diferente de Contas/Aluguéis (que têm no máximo 2 comprovantes fixos), Despesas Diretas usa um **array `comprovantes[]` de tamanho livre**. Botão "+ anexar" na tabela abre um modal simples que adiciona mais um arquivo ao array, sem limite, sem substituir os anteriores — pensado explicitamente para pagamentos parcelados (ex: sinal + final de uma manutenção). Texto de ajuda: *"Se o pagamento foi parcelado (ex: metade no início, metade no final), lance o valor total uma vez e anexe cada comprovante separadamente."*

**Tabela**: NÃO é agrupada por grupo fixo de imóveis (diferente de Aluguéis e Contas) — é uma lista única, plana, ordenada pela ordem de inserção. Colunas: Imóvel, Descrição, Valor, Forma, Data, Comprovantes (lista de chips numerados "🧾 COMPROV. 1", "🧾 COMPROV. 2"...), Ações (+ anexar / excluir — **sem** opção de editar os demais campos, só o valor de anexar mais comprovante). Rodapé: "Total Despesas Diretas".

### 2.7 Tela: Condomínio Adiantado (`condominio`)

**Finalidade**: registrar condomínio que o proprietário paga adiantado, aguardando devolução do inquilino no dia do aluguel. Enquanto não devolvido, o valor **é descontado do saldo do imóvel** no cálculo do DRE (`calcMes`), mas **não** no Dashboard.

**Formulário "Novo Adiantamento"**:
| Rótulo | ID | Tipo | Obrigatório | Padrão |
|---|---|---|---|---|
| Imóvel | `cd-im` | select | Sim | — |
| Valor Adiantado | `cd-val` | número | Não | 0 |
| Data do Adiantamento | `cd-data` | data | Não | hoje |

Sem campo de anexo nesta tela.

**Tabela** (não agrupada por grupo fixo): Imóvel, Valor, Adiantado em, Status (badge "Pendente" amarelo / "Devolvido" verde), Ações. Se pendente: botão "✓ marcar devolvido" (`marcarDevolvido(id)` — define `devolvido: true`, sem exigir nenhuma confirmação ou anexo). Se devolvido: só botão excluir. Rodapé: "Total Pendente de Devolução" somando apenas os `!devolvido`.

### 2.8 Tela: Contas Rateadas (`rateio`)

**Finalidade**: dividir uma única conta (ex: uma fatura de água que cobre várias casas do mesmo condomínio) entre vários imóveis por percentual, gerando automaticamente um registro de Despesa Direta **e** um registro de Conta por imóvel participante.

**Estrutura**: botão "+ Novo Lançamento" (painel retrátil com animação de altura/opacidade CSS, controlado por `toggleRateioForm()` que manipula a classe CSS diretamente, sem re-renderizar a tela inteira, para permitir a transição visual) → formulário → tabela "Rateios Lançados no Mês".

**Formulário "Lançar Rateio"**:

| Rótulo | ID | Tipo | Obrigatório | Padrão | Observação |
|---|---|---|---|---|---|
| Valor Total da Conta | `rt-valor-total` | número | Sim (> 0) | vazio | Recalcula a prévia de cada linha ao digitar (`oninput`) |
| Data | `rt-data` | data | Sim (implícito) | hoje | — |
| Dividir em quantas partes? | `rt-qtd` | select (1 a 10) | Sim | 3 | Regera dinamicamente N linhas de imóvel+percentual |
| Tipo | `rt-tipo` | select | Sim | "IPTU" | IPTU, Água, Condomínio, Luz, Internet, Manutenção, Outro |
| Forma de Pagamento | `rt-forma` | select | Sim | "Boleto" | Boleto, Pix |
| Status | `rt-status` | select | Sim | "A Pagar" | mesmas 4 opções do Status Dinâmico |
| Observações | `rt-obs` | texto | Não | vazio | — |
| (dinâmico, 1 a 10 linhas) Imóvel N | `.rt-linha-im` (classe, não id único) | select | Sim para linhas usadas | — | Linhas com imóvel vazio OU percentual 0 são ignoradas na hora de confirmar |
| (dinâmico) % | `.rt-linha-pct` | número | Sim para linhas usadas | vazio → 0 | Soma de todas deve ser exatamente 100% (tolerância de 0,05) |
| (dinâmico, somente leitura) Valor (R$) | `.rt-linha-valor` | texto readonly | — | calculado | `= Valor Total × percentual / 100`, atualizado em tempo real |

**Campo condicional — Comprovante de Pagamento**: só aparece (`#rt-comprovante-pgto-wrap`) quando Status = "Pago", e nesse caso é **obrigatório** para confirmar.

**Anexo "Anexar Conta/Comprovante"** (rodapé do formulário, `rt-file-input`): serve para anexar a própria conta original (fatura), e **dispara automaticamente a leitura do valor e vencimento** — ver seção 5.4 (fluxo detalhado de leitura de PDF/OCR/linha digitável).

**Validação ao confirmar** (`confirmarRateio`):
1. `total > 0`
2. Ao menos uma linha com imóvel selecionado e percentual > 0
3. Soma dos percentuais das linhas válidas deve estar entre 99,95% e 100,05% (tolerância de arredondamento)
4. Se Status = Pago, exige o arquivo de comprovante de pagamento

**Cálculo de cota por imóvel** (`calcularCotasAjustadas`) — ver fórmula exata na seção 4.

**Efeito ao confirmar** (gera 3 registros diferentes a partir de 1 só ação do usuário):
1. Um registro em `rateio_lanc` (o lançamento "pai")
2. **Uma Despesa Direta por imóvel participante** (`desc: "Conta rateada ({pct}%)"`, `forma: "Rateio"`), cada uma com `rateioLancId` apontando para o pai
3. **Uma Conta por imóvel participante** (`tipo`, `valor` da cota, `envio` = data do rateio, `boleto` = mesmo arquivo anexado, `comprovantePagamento` = mesmo comprovante de pagamento se Status=Pago), cada uma também com `rateioLancId`

**Editar rateio** (`abrirEdicaoRateioLanc`/`salvarEdicaoRateio`): reabre o mesmo formulário preenchido, botão vira "SALVAR ALTERAÇÕES" + "CANCELAR EDIÇÃO". Ao salvar: **todas as Despesas e Contas geradas anteriormente por este rateio são excluídas e recriadas do zero** com os novos valores (não é um ajuste incremental — é "apaga tudo e refaz").

**Excluir rateio** (`delRateioLanc`): `confirm()` explícito avisando que despesas e contas vinculadas também serão excluídas. Remove o lançamento pai e **todos** os registros (Despesas e Contas) que tenham aquele `rateioLancId`.

**+ anexar comprovante** (na tabela de rateios já lançados): adiciona mais um arquivo ao array `comprovantes[]` do rateio pai, sem afetar as Despesas/Contas já geradas.

**Tabela "Rateios Lançados no Mês"**: Data, Valor Total, Divisão (texto formatado "Nome do Imóvel: R$ X (Y%)" separado por " · " para cada participante — ou uma mensagem de "registro antigo, sem detalhamento" para rateios muito antigos salvos antes de existir o array `partes`), Comprovantes, Ações (editar/+anexar/excluir).

### 2.9 Tela: Transferência p/ Pessoal (`pessoal`)

**Finalidade**: registrar explicitamente quanto do saldo dos imóveis foi transferido para uso pessoal — **é o único ponto de acoplamento entre os módulos Imóveis e Guerra Pessoal.**

**Formulário "Nova Transferência"**:
| Rótulo | ID | Tipo | Obrigatório | Padrão |
|---|---|---|---|---|
| Valor | `ps-val` | número | Não (sem bloqueio se 0) | vazio → 0 |
| Data | `ps-data` | data | Não | hoje |
| Observação | `ps-obs` | texto | Não | vazio |

Sem campo de anexo.

**Ao registrar** (`addTransferencia`) — **efeito colateral crítico**:
1. Salva o registro em `imoveisfg_transferencias_<ano>_<mes>`.
2. **Cria automaticamente** um segundo registro em `pessoalfg_entradas_<mesmo ano>_<mesmo mes>` com: `valor` idêntico, `data` idêntica, `obs` idêntica (ou vazia), e `origem: 'REPASSE IMÓVEIS'` (string fixa, usada depois para exibir um badge diferenciado na tela de Entradas do módulo Pessoal).
3. Notificação explícita: "Transferência registrada — entrada criada automaticamente no Guerra Pessoal".

Esse acoplamento é **unidirecional** (Imóveis → Pessoal) e **não há edição sincronizada**: se você editar ou excluir a Transferência depois, a Entrada espelho no Pessoal **não** é automaticamente atualizada ou removida — são dois registros independentes após a criação inicial.

**Tabela**: Data, Valor, Observação, Ações (excluir apenas — sem edição). Rodapé: "Total Transferido para Pessoal".

### 2.10 Tela: DRE (`dre`)

**Finalidade**: demonstrativo de resultado do mês, consolidado ou filtrado por um imóvel específico.

**Filtro**: um único select "Ver resultado de" (`dre-filtro-im`), com opção "Todos os Imóveis (consolidado)" (valor vazio) + um item por imóvel cadastrado (usa a lista simples `imoveisOrdenados()`, sem agrupamento visual por optgroup nesta tela específica). Ao mudar, grava em `ST.dreImovelId` (estado em memória, não persistido) e re-renderiza.

**Estrutura do relatório** (usa `calcMes(filtro)` — ver fórmula exata na seção 4):
```
Faturamento [— Nome do Imóvel, se filtrado]
  Aluguéis (líquido recebido)              R$ totalLiquido
  ─────────────────────────────────────────────────
  Total Entradas                            R$ totalLiquido

Despesas
  Despesas diretas (manutenção etc.)        R$ totalDespesas
  Condomínio pendente de devolução          R$ totalCondPend
  ─────────────────────────────────────────────────
  Total Despesas                            R$ totalDespesas + totalCondPend

Resultado
  Saldo Imóveis do Mês (ou "Lucro/Prejuízo — {Imóvel}" se filtrado)   R$ totalSaldo
  (–) Transferido para Pessoal [só aparece se NÃO filtrado por imóvel]    R$ totalTransf
  ─────────────────────────────────────────────────
  Disponível Após Pessoal [só aparece se NÃO filtrado]     R$ disponivelAposPessoal
```
- Se filtrado por imóvel, aparece a nota: *"A transferência para Pessoal é um controle geral, não por imóvel — por isso não aparece neste recorte."*
- Se `totalIptuAPagar > 0` (há IPTU que a imobiliária repassou mas ainda não foi pago ao órgão público), aparece um aviso vermelho de atenção.
- Nota fixa sempre visível: *"IPTU e água descontados diretamente pela imobiliária não entram neste DRE porque nunca chegaram a compor o valor líquido recebido — evita contar a mesma despesa duas vezes."*

**Sem exportação, sem impressão dedicada** além do botão genérico "Imprimir" da topbar (que chama `window.print()` do navegador, imprimindo a tela como está visualmente, sem formatação de relatório específica).

---

## 3. MÓDULO GUERRA PESSOAL — TELAS DETALHADAS

### 3.1 Propósito e relação com Imóveis

Controle de finanças pessoais da família, **separado** do controle dos imóveis, mas com **um ponto de integração automática** (a Transferência → Entrada, seção 2.9). Foi construído **depois** do módulo Imóveis, replicando deliberadamente boa parte de seus padrões visuais e de código (ver seção 3.6, "O que é compartilhado / duplicado").

### 3.2 Regra de agrupamento fixo — Guerra Pessoal

Análogo aos 11 grupos de Imóveis, mas com **10 blocos fixos diferentes**, aplicados a **Categorias** (não a Imóveis):

| # | Título do bloco | Palavras-chave |
|---|---|---|
| 1 | Casa Conceição – Rua Coaraci Nunes | "conceicao" / "conceição" |
| 2 | Casa Marli – Rua Afonso Pena Jr | "marli" |
| 3 | Casa Jenny – Rua Helena | "jenny" |
| 4 | Outros IPTUs | "outros iptu" / "iptu" |
| 5 | Veículos Pessoais | "veiculo" / "veículo" |
| 6 | Maya Guerra | "maya" |
| 7 | Miguel Guerra | "miguel" |
| 8 | Julyana Guerra | "julyana" |
| 9 | Leandro Guerra | "leandro" |
| 10 | Julio Cesar Guerra | "julio cesar" / "júlio césar" |

**Diferença importante em relação ao algoritmo de Imóveis**: aqui, a correspondência primeiro tenta o campo **Tipo** da categoria (Casa/Pessoa/Veículo/Outro) contra os títulos/palavras-chave — o que raramente bate, já que "Tipo" é um valor genérico, não o nome específico do bloco. Na prática, quase sempre cai no segundo critério: nome da categoria por palavra-chave. Categorias sem correspondência viram blocos órfãos isolados, mesma lógica de Imóveis.

### 3.3 Tela: Cadastro (`gp_cadastro`)

**Finalidade**: CRUD de Categorias (equivalente pessoal ao Cadastro de Imóveis).

**Formulário "Nova Categoria"**:
| Rótulo | ID | Tipo | Obrigatório | Opções |
|---|---|---|---|---|
| Nome | `pc-nome` | texto | Sim | — |
| Tipo | `pc-tipo` | select | Sim | Casa, Pessoa, Veículo, Outro |

**Muito mais simples que o Cadastro de Imóveis** — não tem Endereço, Inquilino, Corretor, Valor nem Grupo explícito (o "Tipo" faz um papel parecido mas com opções fixas genéricas, não o nome do bloco).

**Destino**: array em `pessoalfg_categorias` (sem período — cadastro permanente).

**Tabela**: Nome, Tipo, Ações (editar/excluir). Sem agrupamento visual nesta tela específica (a lista de categorias aparece plana; o agrupamento só é aplicado nas telas de Entradas/Contas/Dashboard que consomem essas categorias).

### 3.4 Tela: Entradas (`gp_entradas`)

**Finalidade**: registrar dinheiro recebido no âmbito pessoal — manual ou automático (via Transferência do módulo Imóveis).

**Formulário "Nova Entrada"**:
| Rótulo | ID | Tipo | Obrigatório | Padrão | Opções |
|---|---|---|---|---|---|
| Origem | `pe-origem` | select | Sim | "REPASSE IMÓVEIS" (primeira opção) | REPASSE IMÓVEIS, ARENA, SOFTWARE, LEANDRO PESSOAL |
| Valor | `pe-val` | número | **Sim, > 0** | vazio | `if(valor<=0) notify('Informe o valor da entrada','err')` |
| Data | `pe-data` | data | Não | hoje | — |
| Observação | `pe-obs` | texto | Não | vazio | — |

**Anexo**: um campo de comprovante no rodapé, opcional.

**Badge de Origem na tabela**: se `origem === 'REPASSE IMÓVEIS'` (correspondência exata de string, sensível a maiúsculas/acentos) → badge azul destacado; qualquer outra origem preenchida → badge amarelo/neutro; sem origem → travessão cinza.

**Tabela** (plana, não agrupada — Entradas não usa os 10 blocos, só Contas usa): Data, Valor, Origem, Observação, Comprovante, Ações (excluir apenas). Rodapé: "Total Entradas".

**Nota**: registros criados automaticamente pela Transferência (seção 2.9) aparecem aqui misturados com os manuais, indistinguíveis exceto pelo badge de Origem.

### 3.5 Tela: Contas (`gp_contas`)

**Finalidade**: gastos pessoais recorrentes ou pontuais, agrupados pelos 10 blocos fixos.

**Formulário "Nova Conta"**:
| Rótulo | ID | Tipo | Obrigatório | Padrão | Opções/Comportamento |
|---|---|---|---|---|---|
| Categoria | `pcn-cat` | select | **Sim** | — | Lista simples das categorias cadastradas (sem agrupamento no select) |
| Tipo / Especificação | `pcn-desc` | select | Sim | "Aluguel" (primeira opção) | Aluguel, IPTU, Água, Condomínio, Luz, Internet, Manutenção, Mesada, Telefone, Cartão, Transporte, Consórcio, Outros |
| Especifique (condicional) | `pcn-desc-outro` | texto | Só aparece/obrigatório funcionalmente se Tipo = "Outros" | vazio → cai para "Outros" se deixado vazio mesmo assim | Aparece/some via `toggleOutroTipoPessoal()` |
| Valor | `pcn-val` | número | Não (sem bloqueio) | vazio → 0 | — |
| Forma de Pagamento | `pcn-forma` | select | Sim | "Boleto" | Boleto, Pix, Dinheiro, Contra Cheque, Cartão |
| Data de Vencimento | `pcn-venc` | data | Não | hoje | — |
| Status | `pcn-status` | select | Sim | "A Pagar" | mesmas 4 opções do Status Dinâmico (compartilhado com Imóveis) |
| Observações | `pcn-obs` | texto | Não | vazio | — |

**Campo condicional — Comprovante de Pagamento**: mesma regra de Contas (Imóveis) — só aparece e só é obrigatório quando Status = Pago.

**Anexos no rodapé**: Boleto (`pcn-boleto`) e Comprovante 2 (`pcn-comprovante2`), ambos opcionais no cadastro inicial (Comprovante 1 tem seu próprio campo condicional descrito acima, distinto dos dois do rodapé).

**Tabela agrupada pelos 10 blocos** — colunas: Tipo, Valor, Vencimento, Data Pagto, Forma Pgto, Status, Obs, Boleto, Comprov. 1, Comprov. 2, Ações. **Note a ausência da coluna "Categoria/Unidade"** — foi removida deliberadamente (já que a categoria já está implícita pelo bloco em que a linha aparece). Rodapé de cada bloco: apenas a palavra **"Subtotal"** (sem repetir o nome do bloco, diferente de Imóveis que repete "Subtotal — {Grupo}").

**Marcar como Pago / Editar**: mesmíssimo padrão de Contas em Imóveis (modal com comprovante obrigatório, som de sucesso, `dataPagamento` automática ou editável).

### 3.6 Tela: Dashboard (Pessoal) (`gp_dashboard`)

**Cards (KPIs)**:
| Card | Fórmula | Cor |
|---|---|---|
| Total Entradas | soma de `valor` de todas as Entradas do mês | verde |
| Gastos Pessoais (Pago) | soma de `valor` das Contas do mês com `status === 'pago'` | vermelho |
| Gastos a Pagar | soma de `valor` das Contas do mês com `status !== 'pago'` (inclui A Vencer/Vencida, calculadas dinamicamente) | âmbar |
| Saldo Líquido Pessoal | Total Entradas − Gastos Pessoais (Pago) | verde/vermelho conforme sinal |

**Tabela "Gastos por Categoria"** (não pelos 10 blocos — por categoria individual): Categoria, Pago, A Pagar. Só lista categorias que têm pelo menos um lançamento (pago ou a pagar) no mês — categorias sem movimento não aparecem.

### 3.7 Tela: DRE (Guerra Pessoal) (`gp_dre`)

Muito mais simples que o DRE de Imóveis — sem filtro por categoria/pessoa, sem seção de avisos:
```
Entradas
  Total de Entradas                          R$ totalEntradas
Gastos
  Gastos Pagos                               R$ gastosPago
  Gastos a Pagar (não abate do saldo ainda)  R$ gastosAPagar
  ─────────────────────────────────────
  Saldo Líquido Pessoal                       R$ (totalEntradas − gastosPago)
```
Nota fixa: *"Saldo Líquido = Total de Entradas − Gastos já Pagos. Gastos 'a pagar' ainda não saíram do seu caixa, por isso não abatem o saldo."*

### 3.8 O que é compartilhado / duplicado entre Imóveis e Guerra Pessoal

**Compartilhado de verdade (mesma função usada por ambos)**:
- `STATUS_REPASSE` / `statusBadge()` / `diasParaVencimento()` — o sistema de status dinâmico é literalmente a mesma função para as duas telas de "Contas"
- `ordemNatural()`, `normTxt()` — utilitários de texto/ordenação
- `comprovanteChip`/`comprovanteChipRotulo`/`lerArquivo`/`nomeAutomatico` — toda a infraestrutura de upload de arquivo
- `toggleJanela()` — mecanismo genérico de painel retrátil
- `notify()`, `fecharModal()`, `playSuccessSound()` — utilitários de UI

**Duplicado (mesma ideia, código escrito duas vezes com nomes diferentes)**:
| Conceito | Versão Imóveis | Versão Guerra Pessoal |
|---|---|---|
| Lista de grupos fixos | `GRUPOS_ORDEM` (11 itens) | `BLOCOS_ORDEM_PESSOAL` (10 itens) |
| Detectar grupo de um item | `getGrupoTitulo()` / `getGrupoTituloImovel()` | `getBlocoTituloPessoal()` |
| Montar módulos agrupados | `montarModulos()` | `montarModulosPessoal()` |
| Renderizar tabela de "Contas" agrupada | dentro de `renderRepasses()` | dentro de `renderContasPessoal()` |
| Marcar como pago com comprovante obrigatório | `marcarRepassePago`/`confirmarRepassePago` | `marcarContaPessoalPaga`/`confirmarContaPessoalPaga` |
| Editar lançamento de conta | `abrirEdicaoRepasse`/`salvarEdicaoRepasse` | `abrirEdicaoContaPessoal`/`salvarEdicaoContaPessoal` |
| Calcular totais do mês | `calcMes()` | `calcMesPessoal()` |

**Estrutura de dados — diferenças**:
- Imóveis tem um cadastro rico (nome, grupo, endereço, inquilino, corretor, valor); Categorias (Pessoal) é minimalista (nome, tipo).
- "Contas" em Imóveis referencia `imovelId`; "Contas" em Pessoal referencia `categoriaId` — campos com nomes diferentes para o mesmo papel estrutural.
- Imóveis tem uma tela de leitura de PDF de relatório de imobiliária (Aprisco/Alude) super específica; Pessoal não tem nada equivalente — só usa o leitor genérico de valor/vencimento (o mesmo do Rateio) indiretamente **não**, na verdade **nem isso**: Pessoal não tem nenhum recurso de leitura automática de PDF em lugar nenhum — todo lançamento é 100% manual.
- Imóveis tem Despesas Diretas, Condomínio Adiantado e Contas Rateadas como telas próprias; **Pessoal não tem equivalente para nenhuma das três** — não há "despesa direta" separada de "conta", não há conceito de adiantamento, não há rateio entre categorias.

---

## 4. TODOS OS CÁLCULOS FINANCEIROS (fórmulas exatas, sem simplificação)

### 4.1 `calcMes(imovelIdFiltro)` — usado pelo DRE de Imóveis e (parcialmente) pelo Dashboard

Para cada imóvel (ou só o filtrado, se `imovelIdFiltro` for passado):
```
liquido    = Σ (r.liquido)  para todo r em alugueis do mês onde r.imovelId === imóvel.id
despesas   = Σ (r.valor)    para todo r em despesas do mês onde r.imovelId === imóvel.id
condPend   = Σ (r.valor)    para todo r em condominio do mês onde r.imovelId === imóvel.id E r.devolvido === false
iptuAPagar = Σ (r.valor)    para todo r em repasses (Contas) do mês onde r.imovelId === imóvel.id E r.status === 'recebido_a_pagar'
saldo      = liquido - despesas - condPend
```
**Nota sobre `iptuAPagar`**: o status `'recebido_a_pagar'` **não existe mais** como opção selecionável em nenhum formulário atual (era um valor legado de uma versão anterior do `STATUS_REPASSE`). Na prática, com os dados atuais, `iptuAPagar` tende a ser sempre 0, a menos que existam registros muito antigos com esse status ainda salvo. O comentário no próprio código confirma: esse valor é só informativo, nunca foi somado como receita nem descontado do saldo.

Totais consolidados (somando todos os imóveis, ou só o filtrado):
```
totalLiquido        = Σ liquido de todos os imóveis
totalDespesas       = Σ despesas de todos os imóveis
totalCondPend       = Σ condPend de todos os imóveis
totalIptuAPagar     = Σ iptuAPagar de todos os imóveis
totalSaldo          = Σ saldo de todos os imóveis (equivalente a totalLiquido - totalDespesas - totalCondPend)
totalTransf         = Σ (r.valor) de TODAS as transferências do mês (não filtra por imóvel — é um valor único, geral)
disponivelAposPessoal = totalSaldo - totalTransf
```
**Arredondamento**: nenhum arredondamento explícito é aplicado nessas somas — usa aritmética de ponto flutuante JavaScript padrão (`+`), sem `toFixed()` intermediário. A formatação para exibição (`fmt()`) usa `Intl.NumberFormat('pt-BR', {style:'currency', currency:'BRL'})`, que arredonda apenas na hora de exibir (2 casas decimais), não altera o valor armazenado.
**Tratamento de valores vazios**: todo campo numérico é lido como `+campo.value || 0` no momento do cadastro (JavaScript: `+""` vira `0`, `+null` vira `0`) — ou seja, campos vazios já entram como `0` no banco, não como `null`/`undefined`. Nas somas, `+r.valor || 0` protege contra registros antigos que porventura tenham o campo ausente ou como string não numérica.
**Quando é recalculado**: toda vez que a tela do DRE (ou Dashboard) é renderizada — não há cache, é sempre "ao vivo" a partir dos arrays carregados do mês selecionado.

### 4.2 Cálculo do Dashboard (`renderDash`) — **DIFERENTE do DRE, mesmo mês, mesmos dados**

```
despesasContasPorImovel(imovelId) = Σ (r.valor) de "repasses" (Contas) do mês, filtrado por imovelId E status === 'pago'
saldo (Dashboard) = liquido - despesasContasPorImovel(imovel)
```
Totais do Dashboard:
```
totalDespesasPagas      = Σ despesasContasPorImovel de todos os imóveis
totalDespesasPendentes  = Σ (r.valor) de "repasses" (Contas) do mês com status !== 'pago' (TODOS os imóveis, sem distinção)
totalSaldoImoveis       = Σ saldo(Dashboard) de todos os imóveis  [= totalLiquido - totalDespesasPagas]
disponivelAposPessoalNovo = totalSaldoImoveis - totalTransf  [totalTransf vem do calcMes(), reaproveitado]
```

**🔴 DIVERGÊNCIA REAL E IMPORTANTE (documentar, decidir conscientemente no FINANCEIROBR)**: o "Saldo" e o "Disponível Após Pessoal" mostrados no **Dashboard** usam uma fórmula **diferente** da usada no **DRE**, para o mesmo mês:
- **DRE**: `saldo = líquido − despesas diretas − condomínio pendente` (ignora completamente o status das Contas — soma TODAS independente de pagas ou não, e nem soma o valor de Contas, só Despesas Diretas)
- **Dashboard**: `saldo = líquido − Contas já pagas` (ignora completamente Despesas Diretas e Condomínio Adiantado)

Ou seja: **dependendo de qual tela você olha, o mesmo mês do mesmo imóvel pode mostrar um "saldo" diferente**, porque cada tela soma uma despesa diferente (Despesas Diretas no DRE vs. Contas Pagas no Dashboard). Isso não é um bug de digitação — é uma decisão de produto tomada em momentos diferentes do desenvolvimento, documentada com uma nota explicativa apenas no Dashboard (*"Despesas aqui = soma dos lançamentos já pagos (status Pago) da aba Contas de cada imóvel neste mês..."*), mas **sem** nenhum aviso equivalente no DRE alertando que ele usa uma base diferente. **O FINANCEIROBR precisa decidir**: unificar numa fórmula só (recomendado) ou manter as duas visões com nomes claramente diferentes (ex: "Saldo de Caixa" vs "Resultado Contábil") para não confundir o usuário achando que é o mesmo número.

### 4.3 `calcMesPessoal()` — Guerra Pessoal

```
totalEntradas = Σ (r.valor) de todas as Entradas do mês
gastosPago    = Σ (c.valor) de Contas do mês onde status === 'pago'
gastosAPagar  = Σ (c.valor) de Contas do mês onde status !== 'pago'
saldoLiquido  = totalEntradas - gastosPago
```
Consistente entre Dashboard Pessoal e DRE Pessoal (aqui **não há** a divergência do módulo Imóveis — só existe uma fórmula de saldo no módulo Pessoal).

### 4.4 Rateio — cálculo de cotas ajustadas (`calcularCotasAjustadas`)

Dado um `valor` total e um array `partes` de `{imovelId, pct}`:
```
para cada parte i:
  bruto[i] = round(valor × pct[i]) / 100   // arredonda em CENTAVOS (multiplica por valor inteiro, divide por 100 no final)
somaArredondada = Σ bruto[i]
diferenca = round((valor - somaArredondada) × 100) / 100
cota[i] = bruto[i] para todo i, EXCETO:
cota[última parte] += diferenca   // a sobra/falta do arredondamento é sempre jogada na ÚLTIMA parte da lista
```
**Por que existe esse ajuste**: se você divide R$ 100,00 em 3 partes de 33,33%, cada parte "pura" daria R$ 33,33 (soma R$ 99,99, faltando 1 centavo). O sistema corrige essa sobra/falta de centavo(s) **inteiramente na última linha da divisão**, garantindo que a soma das cotas seja **sempre exatamente igual** ao valor total original, centavo a centavo.
**Quando é recalculado**: toda vez que o formulário de rateio é preenchido (prévia em tempo real) e novamente no momento de confirmar (para gerar os valores finais salvos).

### 4.5 Validação de soma de percentuais do rateio
```
soma = Σ pct de todas as linhas com imóvel selecionado e pct > 0
válido se |soma - 100| < 0.05
```
Tolerância de 0,05 pontos percentuais para absorver imprecisão de ponto flutuante — **não** é uma tolerância de negócio (não permite ratear "quase 100%" de propósito, é só uma margem técnica).

### 4.6 Status Dinâmico — `diasParaVencimento` (fórmula de datas)
```
hoje = data de hoje, à meia-noite local (00:00:00)
venc = data de vencimento do registro, à meia-noite local
dias = round((venc - hoje) em milissegundos / 86.400.000)  // 86.400.000 ms = 1 dia
```
Ambas as datas são construídas como `new Date(dataISO + 'T00:00:00')` — zera as horas explicitamente para evitar que fusos/horário do dia distorçam a contagem de dias inteiros.

### 4.7 Geração automática de nome de arquivo (`nomeAutomatico`)
```
nome = SLUG(nomeDoImovelOuCategoria) + "_" + SLUG(tipoDaConta) + "_" + mesComDoisDigitos + ano
```
Onde `SLUG(texto)`:
1. Maiúsculas
2. Remove acentos (`normalize('NFD')` + remove marcas diacríticas)
3. Remove tudo que não é letra ou número (`[^A-Z0-9]+` vira nada, sem espaços/hífens/pontuação)
4. Corta em 20 caracteres
5. Se resultado vazio, usa `"ARQ"` como fallback
A extensão do arquivo final é sempre a extensão original do arquivo enviado (ex: `.pdf`, `.jpg`), não fixada em `.pdf` apesar do comentário do código sugerir isso — confirmado lendo `lerArquivoCore`, que usa `f.name.split('.').pop()`.

### 4.8 Parser de valor de PDF genérico (Contas Rateadas) — `parseValorTotalGenerico`
Ordem de tentativas, a primeira que encontrar um valor vence:
1. Padrão de texto explícito, nesta ordem de prioridade: `"TOTAL A PAGAR (R$)"` → `"TOTAL A PAGAR"` → `"VALOR DO DOCUMENTO"` ou `"VALOR COBRADO"` → `"VALOR A PAGAR"` → `"TOTAL (R$)"` → bloco de texto perto de `"PAGUE SUA CONTA COM PIX"` → qualquer `"VALOR"` seguido de número (mais genérico, por último)
2. **Trava de sanidade**: se o valor capturado por qualquer um desses padrões for maior que `VALOR_MAXIMO_RAZOAVEL = 50.000`, é descartado e tenta o próximo padrão (evita capturar CNPJ, chave PIX ou código de barras por engano)
3. Se nada bateu: linha digitável (código de barras) — ver 4.9
4. Se ainda nada: pega o **maior** valor no formato "R$ X" encontrado em qualquer lugar do texto (excluindo valores acima do limite de sanidade)
5. Se nada: retorna `0` (campo fica em branco, preenchimento manual)

### 4.9 Extração de valor pela linha digitável (`parseLinhaDigitavelValor`)
Busca sequências de 35 a 70 caracteres compostas só por dígitos/pontos/espaços, extrai só os dígitos de cada candidata:
- **47 dígitos** (boleto bancário): valor = `parseInt(últimos 10 dígitos) / 100`
- **48 ou 44 dígitos** (conta de concessionária/IPTU): valor = `parseInt(dígitos do índice 4 ao 15) / 100`
- Mesma trava de sanidade (`≤ 50.000`) aplicada ao resultado antes de aceitar

### 4.10 OCR (Contas Rateadas) — quando é acionado e limites
- Só é tentado se: (a) o texto nativo do PDF não rendeu um valor válido, E (b) a varredura de linha digitável nos bytes crus também não encontrou nada
- Timeout rígido de **3 segundos** (`Promise.race` contra um timer) — se o OCR (Tesseract.js) não responder nesse prazo, desiste e libera o campo pra digitação manual, focando automaticamente o campo de valor
- Pré-processamento de imagem antes do OCR: conversão para escala de cinza + realce de contraste (fator 1,35) para reduzir o efeito de marcas d'água/fundo colorido em contas de concessionária
- Esse fluxo de OCR **só existe na tela de Contas Rateadas** — não existe em nenhuma outra tela (Aluguéis usa um parser de texto dedicado ao relatório Aprisco/Alude, sem OCR)

### 4.11 Parser específico de relatório de imobiliária (Aluguéis) — `parseRelatorioAlude`
Campos extraídos por expressão regular, todos opcionais/podem vir `null` se o padrão não bater:
| Campo | Padrão buscado (resumo) |
|---|---|
| `valorRepassado` | `"Valor repassado ... R$ X"` |
| `dataRepasse` | uma data `dd/mm/aaaa` seguida de "às"/"as" (hora) |
| `dataVencimento` | `"Data de vencimento da cobrança dd/mm/aaaa"` |
| `referente` | `"Referente a Mês/Ano"` |
| `imovelAddr` | texto entre `"Imóvel"` e `"Locatário"` |
| `locatario` | texto entre `"Locatário"` e o primeiro `(` (que normalmente antecede CPF) |
| `imobiliaria` | texto entre `"Imobiliária"` e o primeiro `(` |
| `alugCobrado`, `iptuCobrado`, `aguaCobrado`, `jurosCobrado`, `taxaManutCobrado`, `totalCobrado` | procurados dentro da seção de texto entre "O QUE FOI COBRADO" e "O QUE FOI TRANSFERIDO" |
| `alugTransf`, `iptuTransf`, `aguaTransf`, `jurosTransf`, `condoTransf` | procurados dentro de TODAS as seções de texto que vêm depois de "O QUE FOI TRANSFERIDO" (relatórios de 2+ páginas repetem esse cabeçalho — o código junta todas as ocorrências, não só até a próxima) |

**Bruto vs Líquido, definição exata usada no sistema**: "Bruto" = `totalCobrado` (tudo que foi cobrado do inquilino: aluguel + IPTU + água + condomínio + juros/multa + taxa de manutenção, quando aplicável, somados pela própria imobiliária no relatório). "Líquido" = `valorRepassado` (o que efetivamente caiu na conta do proprietário).

### 4.12 Correspondência automática de imóvel a partir do endereço do PDF (`matchImovel`)
Ordem de tentativas (para quando o relatório de aluguel identifica o endereço do imóvel):
1. **Match exato**: endereço do relatório (normalizado: maiúsculo, sem acento, pontuação virada espaço) === endereço cadastrado de algum imóvel, também normalizado
2. **Todas as palavras-chave presentes**: separa o endereço do relatório em palavras (só as com mais de 2 caracteres — inclui números como "105"), filtra os imóveis cujo `nome + endereço` combinados contém **todas** essas palavras. Se só 1 imóvel bate, usa ele. Se mais de 1 bate (ambíguo), escolhe o de nome+endereço **mais curto** (mais específico). Esta etapa existe especificamente para não confundir "Casa 101" com "Casa 105" só porque a rua é a mesma — o número da unidade entra como palavra-chave obrigatória.
3. **Maioria das palavras** (só no campo Endereço cadastrado, não no Nome): calcula a fração de palavras-chave do relatório que aparecem no endereço de cada imóvel; aceita o de maior pontuação, desde que ≥ 70%.
4. **Fallback final**: conta simples de palavras em comum entre o endereço do relatório e `nome + endereço` de cada imóvel (sem normalizar por fração) — aceita o de maior contagem, mesmo que baixa, desde que > 0.
5. Se nada bater em nenhuma etapa: retorna `null` (nenhum imóvel selecionado automaticamente; o usuário escolhe manualmente ou cadastra um novo).

---

## 5. FLUXOS PASSO A PASSO

### 5.1 Fluxo: Login
```
Usuário abre a URL
  → sistema verifica sessionStorage.imoveisfg_auth
    → se '1': pula direto pra tela principal, chama initApp() e sincronizarTemaComNuvem()
    → senão: mostra tela de login
        → usuário digita usuário+senha, clica Entrar (ou aperta Enter no campo senha)
        → doLogin() compara contra CREDS/CREDS_EXTRA
            → se bate: grava sessionStorage (auth + user), esconde login, chama initApp()
            → se não bate: mostra "Usuário ou senha incorretos."
```

### 5.2 Fluxo: Registrar um recebimento de aluguel manual (sem PDF)
```
Usuário → aba Aluguéis → clica "+ Novo Recebimento" (painel abre)
  → seleciona Imóvel no select
  → digita Valor Bruto e Valor Líquido
  → ajusta Data do Repasse (padrão hoje) e Data de Vencimento
  → opcionalmente digita Observação
  → opcionalmente anexa um comprovante qualquer (não PDF de relatório, então handleAluguelAnexo só marca "arquivo anexado como comprovante" sem tentar ler nada)
  → clica "+ REGISTRAR RECEBIMENTO"
      → addAluguel() valida imóvel selecionado
      → processa o arquivo (se houver) → gera nome automático → envia ao Firebase Storage (imoveisfg_files)
      → salva o registro no array do mês
      → notifica sucesso, re-renderiza a tela (o card "Total Líquido do Mês" e o subtotal do grupo correspondente já reflete o novo valor)
```

### 5.3 Fluxo: Registrar aluguel via importação automática de relatório PDF
```
Usuário → aba Aluguéis → abre o painel → anexa um arquivo PDF no campo do rodapé
  → handleAluguelAnexo() detecta que é PDF, mostra "⟳ Lendo PDF..."
  → extrai texto via pdf.js (todas as páginas)
  → parseRelatorioAlude() extrai os campos (seção 4.11)
  → guarda o resultado em memória (_pdfParsedAtual, variável global)
  → matchImovel() tenta achar o imóvel correspondente (seção 4.12)
      CASO ENCONTROU:
        → seleciona automaticamente o imóvel no select (dispara evento 'change' manualmente)
        → mostra abaixo do select: "✓ Imóvel identificado: {nome} — Não é este? Cadastrar novo" (link sempre clicável)
      CASO NÃO ENCONTROU (mas havia um endereço no PDF):
        → limpa a seleção do imóvel
        → mostra: "⚠️ Não encontramos '{endereço}'. Selecione na lista ou: + Cadastrar este imóvel"
  → preenche automaticamente: Valor Bruto (totalCobrado), Valor Líquido (valorRepassado), Data do Repasse, Data de Vencimento
  → Observação NUNCA é preenchida automaticamente (decisão de produto)
  → mostra mensagem de status final: sucesso (dados preenchidos) ou aviso (não identificou os valores, mas o arquivo será anexado mesmo assim)
  → usuário confere/ajusta os campos e clica "+ REGISTRAR RECEBIMENTO" normalmente (mesmo fluxo do 5.2 a partir daqui)
```

### 5.4 Fluxo: Cadastrar um imóvel novo "na hora", a partir do relatório
```
Usuário clica no link "+ Cadastrar novo imóvel" (ou variantes "Não é este?"/"Cadastrar este imóvel")
  → abrirCadastroImovelInline() abre um modal
      → se havia dados de um PDF recém-lido (_pdfParsedAtual): pré-preenche
          Grupo    = detectado automaticamente pelo endereço do PDF (getGrupoTitulo)
          Nome     = "{GRUPO} {UNIDADE}" (ex: "TRAVESSA GOMES CASA 105") — limparNomeImovel() extrai o padrão CASA/LOJA/SALA/APTO + número
          Inquilino = locatário extraído do PDF
          Valor    = totalCobrado ou valorRepassado
          Endereço = endereço completo do PDF
          Dia de Vencimento = dia extraído da data de vencimento do PDF
  → usuário confere/ajusta e clica "+ SALVAR E SELECIONAR"
      → confirmarCadastroImovelInline() cria o imóvel (nome, grupo, endereço, inquilino em MAIÚSCULAS forçadas; corretor sempre vazio nesse fluxo específico)
      → fecha o modal
      → atualiza o select de Imóvel do formulário de Aluguéis SEM re-renderizar a tela inteira (evita perder o que já estava preenchido no formulário)
      → seleciona automaticamente o imóvel recém-criado
      → atualiza a mensagem abaixo do select para "✓ {nome} cadastrado e selecionado."
```

### 5.5 Fluxo: Marcar uma Conta (Imóveis ou Pessoal) como Paga
```
Usuário → tabela de Contas → clica no ✓ da linha
  → abre modal pedindo Data do Pagamento + Comprovante(s)
      Imóveis: Boleto (opcional) + Comprovante 1 (OBRIGATÓRIO) + Comprovante 2 (opcional)
      Pessoal: só Comprovante (OBRIGATÓRIO) — modal mais simples
  → se tentar confirmar sem o comprovante obrigatório: bloqueia com mensagem de erro, não fecha o modal
  → se confirmar com o(s) arquivo(s):
      → processa os arquivos em cadeia (boleto → comp1 → comp2, cada upload espera o anterior)
      → define status = 'pago', dataPagamento = data escolhida
      → toca o som de confirmação (dois beeps)
      → fecha modal, re-renderiza — a linha agora mostra badge "Pago" e some o botão ✓
```

### 5.6 Fluxo: Lançar uma Contas Rateada completa
```
Usuário → aba Contas Rateadas → "+ Novo Lançamento"
  → (opcional) anexa a conta original no campo "Anexar Conta/Comprovante"
      → sistema tenta ler o valor e vencimento automaticamente (cadeia de tentativas: texto → linha digitável nos bytes crus → OCR com timeout de 3s)
      → se achar, preenche Valor Total e Data automaticamente; senão, foca o campo Valor pra digitação manual
  → usuário ajusta/confirma Valor Total, Data, Tipo, Forma de Pagamento, Status, Observações
  → escolhe "Dividir em quantas partes?" (1 a 10) → sistema gera N linhas de Imóvel + %
  → para cada linha, seleciona o imóvel e digita o percentual
      → o campo "Valor (R$)" de cada linha atualiza em tempo real (Valor Total × % / 100)
      → indicador de soma dos percentuais fica verde quando soma = 100%, vermelho caso contrário
  → se Status = Pago, aparece campo extra de Comprovante de Pagamento (obrigatório)
  → clica "CONFIRMAR RATEIO"
      → valida: valor > 0, ao menos 1 linha válida, soma = 100% (±0,05), comprovante se pago
      → calcula as cotas ajustadas (seção 4.4)
      → cria 1 registro em Contas Rateadas + N Despesas Diretas + N Contas (uma de cada por imóvel participante)
      → limpa o formulário, notifica sucesso
```

### 5.7 Fluxo: Transferência para Pessoal → efeito no outro módulo
```
Usuário → módulo Imóveis → Transferência p/ Pessoal → preenche Valor/Data/Observação → "+ REGISTRAR TRANSFERÊNCIA"
  → addTransferencia() salva o registro em imoveisfg_transferencias
  → IMEDIATAMENTE, sem intervenção do usuário, cria um registro espelho em pessoalfg_entradas do MESMO mês/ano, com origem='REPASSE IMÓVEIS'
  → notifica "Transferência registrada — entrada criada automaticamente no Guerra Pessoal"
  → se o usuário alternar a chave pro módulo Guerra Pessoal e for em Entradas, verá essa entrada lá, com o badge azul "REPASSE IMÓVEIS"
```

---

## 6. ÁRVORE COMPLETA DE DADOS NO FIREBASE + MAPEAMENTO TELA → CAMPO → FIREBASE

### 6.1 Árvore completa

```
imoveisfg  (nó raiz — TUDO fica sob este único nó, sobrescrito inteiro a cada gravação)
│
├─ _ts                                    ← timestamp da última escrita (Date.now())
│
├─ imoveisfg_imoveis                      ← array de imóveis (cadastro permanente)
│   └─ [ {id, nome, grupo, endereco, inquilino, corretor, valor, diaVencimento?} ]
│
├─ imoveisfg_alugueis_<AAAA>_<MM>          ← um array por mês
│   └─ [ {id, imovelId, bruto, liquido, data, vencimento, obs, comprovante} ]
│
├─ imoveisfg_repasses_<AAAA>_<MM>          ← "Contas" — um array por mês
│   └─ [ {id, imovelId, tipo, valor, envio, formaPagamento, status, obs,
│         boleto, comprovantePagamento, comprovantePagamento2, dataPagamento,
│         rateioLancId?, relatorioOrigem?(legado)} ]
│
├─ imoveisfg_despesas_<AAAA>_<MM>          ← um array por mês
│   └─ [ {id, imovelId, desc, valor, forma, data, comprovantes:[...], rateioLancId?} ]
│
├─ imoveisfg_condominio_<AAAA>_<MM>        ← um array por mês
│   └─ [ {id, imovelId, valor, data, devolvido} ]
│
├─ imoveisfg_rateio_lanc_<AAAA>_<MM>       ← um array por mês
│   └─ [ {id, valor, data, partes:[{imovelId,pct}], tipo, formaPagamento,
│         status, obs, comprovantes:[...]} ]
│
├─ imoveisfg_transferencias_<AAAA>_<MM>    ← um array por mês
│   └─ [ {id, valor, data, obs} ]
│
├─ imoveisfg_files                         ← arquivos, fora do padrão de array — é um objeto/dicionário
│   └─ {fileId}: "data:<mime>;base64,<conteúdo>"   (string longa, um nó por arquivo)
│
├─ imoveisfg_theme                         ← preferência de tema "global" (fallback)
├─ imoveisfg_theme_<usuario>               ← preferência de tema POR usuário (ex: imoveisfg_theme_Guerra)
│
├─ pessoalfg_categorias                    ← array (cadastro permanente)
│   └─ [ {id, nome, tipo} ]
│
├─ pessoalfg_entradas_<AAAA>_<MM>          ← um array por mês
│   └─ [ {id, valor, data, obs, origem, comprovante} ]
│
└─ pessoalfg_contas_<AAAA>_<MM>            ← um array por mês
    └─ [ {id, categoriaId, desc, valor, forma, vencimento, status, obs,
          boleto, comprovantePagamento, comprovantePagamento2, dataPagamento} ]
```

**Atenção**: apesar do nome sugerir hierarquia, `imoveisfg_theme` **não fica sob** `imoveisfg_files` nem nada assim — todas essas chaves são **irmãs diretas**, todas soltas dentro do nó raiz `imoveisfg`, porque o mecanismo de sincronização (`_coletaDados`) simplesmente pega cada chave do `localStorage` (que já tem esses nomes completos, com o prefixo) e a usa como uma propriedade direta do objeto gigante que sobrescreve o nó `imoveisfg` inteiro no Firebase — não há nenhuma estrutura de sub-nó real sendo construída deliberadamente, é achatado.

### 6.2 Mapeamento TELA → CAMPO → FIREBASE (para toda entidade que tem formulário)

**Cadastro de Imóveis** (`imoveisfg_imoveis[]`):
| Campo na tela | Propriedade no objeto Firebase |
|---|---|
| Grupo / Empreendimento | `grupo` |
| Nome / Identificação | `nome` |
| Endereço | `endereco` |
| Inquilino | `inquilino` |
| Imobiliária / Corretor | `corretor` |
| Valor Aluguel Contratado | `valor` |

**Aluguéis** (`imoveisfg_alugueis_<ano>_<mes>[]`):
| Campo na tela | Propriedade |
|---|---|
| Imóvel (select) | `imovelId` |
| Valor Boleto (bruto) | `bruto` |
| Valor Líquido Recebido | `liquido` |
| Data do Repasse | `data` |
| Data de Vencimento | `vencimento` |
| Observação | `obs` |
| Anexo | `comprovante` (objeto `{nomeOriginal, nomeFinal, mime, tamanho, fileId}`) |

**Contas (Imóveis)** (`imoveisfg_repasses_<ano>_<mes>[]`):
| Campo na tela | Propriedade |
|---|---|
| Imóvel | `imovelId` |
| Tipo | `tipo` |
| Valor | `valor` |
| Data de Vencimento | `envio` (nome de campo histórico — na tela chama "Vencimento", no banco chama `envio`) |
| Forma de Pagamento | `formaPagamento` |
| Status | `status` |
| Observações | `obs` |
| Boleto | `boleto` |
| Comprovante de Pagamento 1 | `comprovantePagamento` |
| Comprovante de Pagamento 2 | `comprovantePagamento2` |
| Data de Pagamento | `dataPagamento` |
| (interno, não editável) | `rateioLancId` — só existe se este registro foi gerado por um Rateio |

**Despesas Diretas** (`imoveisfg_despesas_<ano>_<mes>[]`):
| Campo na tela | Propriedade |
|---|---|
| Imóvel | `imovelId` |
| Descrição | `desc` |
| Valor | `valor` |
| Forma de Pagamento | `forma` |
| Data | `data` |
| Anexos (+ anexar, múltiplos) | `comprovantes` (array de objetos de arquivo) |
| (interno) | `rateioLancId` — se gerado por Rateio |

**Condomínio Adiantado** (`imoveisfg_condominio_<ano>_<mes>[]`):
| Campo na tela | Propriedade |
|---|---|
| Imóvel | `imovelId` |
| Valor Adiantado | `valor` |
| Data do Adiantamento | `data` |
| (marcar devolvido) | `devolvido` (boolean) |

**Contas Rateadas** (`imoveisfg_rateio_lanc_<ano>_<mes>[]`):
| Campo na tela | Propriedade |
|---|---|
| Valor Total da Conta | `valor` |
| Data | `data` |
| Linhas dinâmicas Imóvel+% | `partes` (array de `{imovelId, pct}`) |
| Tipo | `tipo` |
| Forma de Pagamento | `formaPagamento` |
| Status | `status` |
| Observações | `obs` |
| Anexo(s) da conta original | `comprovantes` (array) |

**Transferência p/ Pessoal** (`imoveisfg_transferencias_<ano>_<mes>[]`):
| Campo na tela | Propriedade |
|---|---|
| Valor | `valor` |
| Data | `data` |
| Observação | `obs` |
| *(gerado, não é campo da tela)* | dispara criação de `pessoalfg_entradas_<mesmo ano>_<mesmo mes>[]` com `{valor, data, obs, origem:'REPASSE IMÓVEIS'}` |

**Cadastro (Categorias Pessoal)** (`pessoalfg_categorias[]`):
| Campo na tela | Propriedade |
|---|---|
| Nome | `nome` |
| Tipo | `tipo` |

**Entradas (Pessoal)** (`pessoalfg_entradas_<ano>_<mes>[]`):
| Campo na tela | Propriedade |
|---|---|
| Origem | `origem` |
| Valor | `valor` |
| Data | `data` |
| Observação | `obs` |
| Anexo | `comprovante` |

**Contas (Pessoal)** (`pessoalfg_contas_<ano>_<mes>[]`):
| Campo na tela | Propriedade |
|---|---|
| Categoria | `categoriaId` |
| Tipo / Especificação (ou texto customizado se "Outros") | `desc` |
| Valor | `valor` |
| Forma de Pagamento | `forma` |
| Data de Vencimento | `vencimento` |
| Status | `status` |
| Observações | `obs` |
| Boleto | `boleto` |
| Comprovante 1 | `comprovantePagamento` |
| Comprovante 2 | `comprovantePagamento2` |
| Data de Pagamento | `dataPagamento` |

### 6.3 Formato de um objeto de arquivo anexado (padrão em todo o sistema)
```json
{
  "nomeOriginal": "nome_que_o_usuario_enviou.pdf",
  "nomeFinal": "PAULHARRISLOJA01_IPTU-BOLETO_082026.pdf",
  "mime": "application/pdf",
  "tamanho": 123456,
  "fileId": "abc123xyz456"
}
```
O conteúdo real do arquivo (base64) **não fica dentro deste objeto** — fica em `imoveisfg_files/{fileId}`, referenciado só pelo `fileId`. **Exceção histórica**: registros muito antigos (anteriores a uma correção feita durante o desenvolvimento) podem ainda ter um campo `data` diretamente no objeto do anexo, com o conteúdo base64 embutido — a função `obterDadosArquivo()` verifica primeiro se existe `comp.data` (formato antigo) antes de tentar buscar por `comp.fileId` (formato atual). A ferramenta "Corrigir anexos antigos" (seção 7) converte registros do formato antigo para o atual.

---

## 7. FUNCIONALIDADES OCULTAS OU MENOS ÓBVIAS

### 7.1 Ferramenta: Corrigir Grupos Automaticamente
Já detalhada na seção 2.3. Não é óbvia porque fica escondida atrás de um botão secundário na tela de Cadastro de Imóveis, mas resolve um problema real e recorrente (imóveis "sumindo" do agrupamento por inconsistência de nome/endereço).

### 7.2 Geração automática de nome de arquivo (renomeação silenciosa)
Todo arquivo enviado é **renomeado automaticamente** no padrão `{IMOVEL_OU_CATEGORIA}_{TIPO}_{MESANO}.{extensão original}` antes de ser salvo — o usuário nunca vê nem escolhe esse nome, só percebe quando abre/baixa o arquivo depois. Isso não é anunciado em lugar nenhum da interface, é um comportamento silencioso.

### 7.3 Limite de tamanho de arquivo (4 MB) — validado silenciosamente
`lerArquivoCore()` rejeita qualquer arquivo maior que 4.194.304 bytes (4×1024×1024) com uma notificação de erro. Esse limite **não aparece escrito em lugar nenhum da interface antes do usuário tentar** — só descobre ao tentar enviar um arquivo grande demais.

### 7.4 Ferramenta: Migrar Anexos Antigos (`migrarAnexosAntigos`)
Botão no rodapé da sidebar. Varre **recursivamente** todos os objetos salvos em qualquer entidade (por mês, por cadastro) procurando qualquer campo `.data` que comece com `"data:"` (indicando um anexo salvo no formato antigo, pesado, embutido). Para cada um encontrado: envia o conteúdo para `imoveisfg_files/{novoFileId}`, remove o campo `.data` do registro original e adiciona `.fileId`. Ao final, salva tudo de volta e sincroniza. Só reporta sucesso quantitativo ("N anexo(s) migrado(s)"), não lista quais.

### 7.5 Ferramenta: Limpar Cache Local (`limparCacheLocal`)
Remove **todas** as chaves `imoveisfg_*`/`pessoalfg_*` do `localStorage` do navegador (não mexe em nada na nuvem) e força um recarregamento completo a partir do Firebase. Existe porque o `localStorage` do navegador tem um limite de tamanho (~5-10MB dependendo do navegador) e pode encher com o uso prolongado — quando isso acontece, o sistema já avisa (`save()` captura o erro de `localStorage.setItem` e notifica "Aviso: cache local cheio, mas os dados vão para a nuvem normalmente"), mas continua funcionando (grava só na nuvem, sem cache local daquela chave específica).

### 7.6 Ferramenta: Remover Relatórios Antigos (`limparRelatoriosAntigos`)
Aparece só condicionalmente, no topo da aba Contas, se existir algum registro (em qualquer mês) com um campo legado `relatorioOrigem`. Remove esse campo especificamente de todos os registros afetados, em todos os meses — não exclui o lançamento em si.

### 7.7 Tema claro/escuro persistido por usuário, sincronizado via Firebase
Não é óbvio que a preferência de tema (`alternarTema`) é salva de duas formas ao mesmo tempo: `localStorage` local (pra aplicar instantaneamente e evitar "flash" de tela clara antes de escurecer) **e** um nó específico no Firebase (`imoveisfg_theme/{usuario}`), sincronizado automaticamente ao logar (`sincronizarTemaComNuvem`) — permitindo que o mesmo usuário logando em outro dispositivo/navegador já veja o tema que escolheu da última vez.

### 7.8 Som de confirmação ao marcar uma conta como paga
`playSuccessSound()` — gera dois tons ascendentes (660Hz → 880Hz) via Web Audio API nativa do navegador (não é um arquivo de áudio), tocado **apenas** na transição de "não pago" para "pago" (não toca em outras ações). Silenciosamente ignora qualquer erro (ex: navegador sem suporte a `AudioContext`).

### 7.9 Botão "Imprimir" é o `window.print()` nativo do navegador
Não gera um PDF customizado nem um layout de impressão dedicado — literalmente aciona a caixa de diálogo de impressão do navegador sobre a tela atual, como ela está renderizada na hora (incluindo sidebar e topbar, a menos que exista alguma regra `@media print` no CSS que eu não tenha visto isolada — não há nenhuma regra `@media print` encontrada no código, então a impressão sai "como está na tela", sidebar incluída).

### 7.10 Validação de arquivo: `accept=".pdf,image/*"` é só sugestão visual
O atributo `accept` do `<input type="file">` em todos os campos de upload restringe apenas o que aparece pré-filtrado no seletor de arquivos do sistema operacional — não impede, por si só, que um usuário arraste ou selecione manualmente (em navegadores que permitem) qualquer outro tipo de arquivo. Não há verificação de tipo MIME real no código depois do upload.

### 7.11 O parser de relatório de imobiliária é específico da Aprisco/Alude
Os regex do `parseRelatorioAlude` foram desenhados em cima do formato específico de relatório dessas duas imobiliárias mencionadas nos comentários do código. Um relatório de qualquer outra imobiliária/formato **não será reconhecido corretamente** — o sistema não tem um parser genérico de relatório de aluguel, só o dedicado a esse formato (diferente do parser de contas/boletos da tela de Rateio, que é intencionalmente genérico).

### 7.12 `handleAluguelAnexo` reseta a leitura anterior ao trocar de arquivo
Se você anexar um arquivo que não é PDF depois de já ter lido um PDF (`_pdfParsedAtual` preenchido), o sistema zera essa variável (`_pdfParsedAtual = null`) — então o botão "+ Cadastrar novo imóvel" que aparece depois não vai mais sugerir os dados do PDF anterior.

### 7.13 Código morto ainda presente no arquivo (não afeta a interface atual, mas ocupa espaço)
As funções `handlePDFFile`, `abrirPreviewImportPDF`, `confirmarImportPDF`, `criarImovelInlineImport`, `toggleNovoImovelImport` implementam um fluxo completo alternativo de importação de PDF (com modal de conferência dedicado, campo de novo imóvel embutido no modal) que **foi substituído** pelo fluxo atual (`handleAluguelAnexo`, direto no formulário principal) mas nunca foi removido do arquivo. Nenhum elemento visível da interface aciona mais esse código — confirmado por busca: não existe mais nenhum `<input id="pdf-import-input">` no HTML atual (as funções mortas referenciam esse ID, que não existe). **Não deve ser reproduzido no FINANCEIROBR.**

### 7.14 `imoveisfg` como parte do caminho, mesmo para dados do Guerra Pessoal
Apesar do prefixo de chave ser `pessoalfg_*`, o nó raiz no Firebase onde tudo é salvo continua sendo `imoveisfg` (a referência `_dbRef = _db.ref('imoveisfg')` é única para o arquivo inteiro) — ou seja, o nome "imoveisfg" no Firebase não corresponde exclusivamente ao módulo Imóveis; é o nome do "banco" do sistema como um todo, herdado de antes do módulo Pessoal existir.

### 7.15 Diferença de comportamento entre `delItem` das diferentes telas
Algumas exclusões pedem `confirm()` do navegador antes (Imóvel, Rateio, Categoria); a maioria das outras (Aluguéis, Contas, Despesas, Condomínio, Transferência, Entradas/Contas do Pessoal) usa a função genérica `delItem`/`delItemPessoal`, que **exclui imediatamente, sem nenhuma confirmação**. Isso é inconsistente e não documentado — vale a pena decidir um padrão único no FINANCEIROBR (ex: sempre confirmar, ou nunca, mas ser consistente).

---

## 8. PRESERVAR / MELHORAR / NÃO REPRODUZIR

### 8.1 PRESERVAR FUNCIONALMENTE (o FINANCEIROBR precisa fazer tudo isto)
- Os 9 módulos/telas do módulo Imóveis e as 5 telas do Guerra Pessoal, com todos os campos listados nas seções 2 e 3
- O sistema de agrupamento fixo e ordenado (11 grupos de Imóveis, 10 blocos de Pessoal), incluindo a regra especial de Sala 603/609
- O Status Dinâmico calculado por data (A Pagar / A Vencer / Vencida / Pago), nunca travado no valor salvo exceto "Pago"
- A obrigatoriedade de comprovante ao marcar qualquer conta como Paga (em ambos os módulos)
- O fluxo completo de leitura de PDF/OCR em Contas Rateadas (texto → linha digitável nos bytes crus → OCR com timeout → fallback manual), incluindo a trava de sanidade de valor máximo
- O parser dedicado de relatório Aprisco/Alude em Aluguéis, com a correspondência automática de imóvel (`matchImovel`, incluindo a etapa "todas as palavras-chave" que evita confundir unidades vizinhas)
- O cálculo de cotas ajustadas do rateio (arredondamento absorvido na última parte, garantindo soma exata)
- O único ponto de integração automática Transferência → Entrada Pessoal
- A geração automática de Despesa + Conta por imóvel a partir de um Rateio
- O sistema de renomeação automática de arquivos e o armazenamento de arquivo separado do registro (arquivo pesado fora do documento/linha, só uma referência)
- Tema claro/escuro persistido por usuário
- A distinção entre Despesas Diretas (paga direto), Contas (paga via corretor, não conta no DRE) e Condomínio Adiantado (pendente de devolução) — são três conceitos financeiros genuinamente diferentes na cabeça da família, não devem ser fundidos numa coisa só

### 8.2 MELHORAR (a ideia é boa, a implementação tem problema)
| Funcionalidade | Problema atual | Melhoria sugerida |
|---|---|---|
| Cálculo de saldo | Dashboard e DRE usam fórmulas diferentes pro "mesmo" saldo (seção 4.2) | Unificar numa única fonte de verdade, ou nomear claramente como dois relatórios diferentes com propósitos diferentes |
| Autenticação | Senha em texto plano no código-fonte público | Autenticação real de backend/Firebase Auth com hash de senha |
| Sincronização | Sobrescreve o banco inteiro a cada gravação (sem granularidade, sem controle de concorrência) | Persistência granular por registro/documento |
| Exclusão de registros | Inconsistente: algumas pedem confirmação, outras não | Padronizar (ex: sempre confirmar exclusões irreversíveis) |
| Exclusão de Imóvel/Categoria | Deixa lançamentos vinculados órfãos silenciosamente | Bloquear a exclusão se houver vínculos, ou oferecer explicitamente "excluir e desvincular" vs "cancelar" |
| Limite de arquivo (4MB) | Não é comunicado antes do usuário tentar enviar | Mostrar o limite no próprio campo de upload |
| Parser de relatório de imobiliária | Funciona só para o formato Aprisco/Alude | Generalizar para múltiplos formatos, ou permitir configurar novos padrões sem editar código |
| Edição de Rateio | Apaga e recria todos os registros vinculados a cada edição | Atualizar incrementalmente os registros existentes, preservando histórico de quem pagou o quê antes da edição |
| Mensagens de erro técnico expostas ao usuário | O card de erro do `render()` mostra `e.message` cru (mensagem de erro JavaScript) | Mensagens amigáveis para o usuário final, log técnico só no console/observabilidade |
| Uploads sem validação real de tipo | `accept` é só sugestão de UI | Validar tipo MIME de verdade no momento do upload |

### 8.3 NÃO REPRODUZIR (descartar conscientemente)
- Código morto: `handlePDFFile`, `abrirPreviewImportPDF`, `confirmarImportPDF`, `criarImovelInlineImport`, `toggleNovoImovelImport` (seção 7.13) — fluxo substituído, nunca chamado
- Credenciais hardcoded em texto plano no código-fonte cliente
- Inserção de texto de usuário sem escape de HTML (`innerHTML` direto — risco de XSS armazenado)
- IDs gerados por `Math.random()` em vez de UUID real
- O status legado `'recebido_a_pagar'` (seção 4.1) — não é mais usado por nenhum formulário, só existe como resíduo de leitura
- O campo legado `relatorioOrigem` em registros de Contas (já tem inclusive uma ferramenta dedicada só pra limpar isso — seção 7.6)
- O formato antigo de anexo com `data` embutido diretamente no registro (mantenha só o padrão atual: registro leve + arquivo referenciado à parte)
- Duplicação de lógica entre os módulos Imóveis/Pessoal (seção 3.8) — construir uma vez, parametrizada, e reaproveitar nos dois módulos

---

## 9. MAPA DE MIGRAÇÃO — DADO ANTIGO → SIGNIFICADO → NOVA ENTIDADE SUGERIDA

**Nenhuma migração é executada aqui — só a proposta de mapeamento.**

| Dado antigo (chave/campo Firebase) | Significado real | Nova entidade sugerida (FINANCEIROBR) |
|---|---|---|
| `imoveisfg_imoveis[]` | Cadastro mestre de imóveis | Tabela/coleção `imoveis` |
| `imoveisfg_imoveis[].grupo` | Nome do bloco fixo ao qual o imóvel pertence (texto livre, comparado por heurística) | Chave estrangeira `grupo_id` apontando para uma tabela `grupos_imoveis` com os 11 grupos como registros reais (não mais texto livre comparado por regex) |
| `imoveisfg_alugueis_<ano>_<mes>[]` | Recebimentos de aluguel | Tabela `recebimentos_aluguel`, com coluna de data real (não mais uma chave separada por mês/ano — usar filtro por intervalo de datas numa tabela única, contínua) |
| `imoveisfg_repasses_<ano>_<mes>[].envio` | Data de vencimento da conta | Renomear para `data_vencimento` (nome mais claro que "envio") |
| `imoveisfg_repasses_<ano>_<mes>[].status === 'recebido_a_pagar'` | Status legado, quase não usado | Não migrar como status válido — se existir algum registro assim, tratar como `'a_pagar'` na migração |
| `imoveisfg_repasses_<ano>_<mes>[].relatorioOrigem` | Campo legado de anexo automático indevido | Descartar na migração (não tem valor de negócio) |
| `imoveisfg_despesas_<ano>_<mes>[]` | Despesas diretas pagas pelo proprietário | Tabela `despesas_diretas` |
| `imoveisfg_condominio_<ano>_<mes>[]` | Condomínio adiantado | Tabela `condominio_adiantado` |
| `imoveisfg_rateio_lanc_<ano>_<mes>[]` | Lançamento "pai" de rateio | Tabela `rateios`, com uma tabela filha `rateio_participantes` (imovelId, percentual, cota) em vez do array `partes` embutido |
| `imoveisfg_transferencias_<ano>_<mes>[]` | Transferência para uso pessoal | Tabela `transferencias_pessoal`, com uma referência explícita (FK) para a `entrada_pessoal` gerada, em vez da duplicação silenciosa atual |
| `imoveisfg_files/{fileId}` | Conteúdo de arquivo em base64 | Migrar para armazenamento de objeto binário real (S3-compatível ou Firebase Storage), mantendo só a referência/URL na tabela de anexos |
| `pessoalfg_categorias[]` | Cadastro de categorias pessoais | Tabela `categorias_pessoais`, com `bloco_id` explícito (equivalente ao `grupo_id` sugerido acima) em vez de `tipo` genérico |
| `pessoalfg_entradas_<ano>_<mes>[]` | Entradas pessoais | Tabela `entradas_pessoais` |
| `pessoalfg_entradas_<ano>_<mes>[].origem` | Texto livre (REPASSE IMÓVEIS/ARENA/SOFTWARE/LEANDRO PESSOAL) | Tabela de referência `origens_entrada` (enum ou tabela pequena), evitando strings soltas comparadas por igualdade exata |
| `pessoalfg_contas_<ano>_<mes>[]` | Contas/gastos pessoais | Tabela `contas_pessoais` |
| `pessoalfg_contas_<ano>_<mes>[].desc` | Tipo de gasto (Aluguel/IPTU/Água/.../Outros + texto livre se Outros) | Coluna `tipo_id` (FK pra tabela de tipos fixos) + coluna `descricao_livre` separada para o caso "Outros" |
| `STATUS_REPASSE` (constante no código) | As 4 opções de status usadas em Imóveis e Pessoal | Tabela/enum `status_conta` compartilhada pelos dois módulos desde o início (não duplicada) |
| `GRUPOS_ORDEM` / `BLOCOS_ORDEM_PESSOAL` (constantes no código) | As listas fixas de agrupamento | Tabelas `grupos_imoveis` e `blocos_pessoais`, com uma coluna de **ordem explícita** (`ordem_exibicao INT`) em vez de depender da ordem do array no código |
| `imoveisfg_theme` / `imoveisfg_theme_<usuario>` | Preferência de tema | Coluna `tema_preferido` na tabela de usuários (uma vez que exista autenticação real) |
| `CREDS` / `CREDS_EXTRA` (constantes no código) | As 3 contas de usuário | Tabela `usuarios` de verdade, com senha com hash, gerenciada por um sistema de autenticação real |

---

## 10. CHECKLIST DE PARIDADE

*Cada item deve poder ser respondido com Sim/Não ao testar o FINANCEIROBR contra o comportamento aqui documentado.*

### Módulo Imóveis
- [ ] Login com 3 contas de usuário (nomes preservados, senhas novas) funciona
- [ ] Cadastro de Imóveis: criar, editar, excluir, com todos os 6 campos (Grupo, Nome, Endereço, Inquilino, Corretor, Valor)
- [ ] Ferramenta "Corrigir Grupos Automaticamente" existe e funciona como descrito na seção 2.3
- [ ] Os 11 grupos fixos aparecem na ordem exata da seção 2.1, com a regra especial de Sala 603/609
- [ ] Imóveis sem grupo correspondente aparecem como blocos órfãos, no final
- [ ] Aluguéis: cadastro manual completo (todos os campos da seção 2.4)
- [ ] Aluguéis: importação de PDF de relatório Aprisco/Alude preenche os campos automaticamente (seção 5.3)
- [ ] Aluguéis: correspondência automática de imóvel funciona mesmo com nome "sujo" (vírgulas, número de rua) — seção 4.12
- [ ] Aluguéis: cadastro rápido de imóvel novo a partir do PDF (seção 5.4)
- [ ] Aluguéis: campo Observação nunca é preenchido automaticamente
- [ ] Aluguéis: edição e exclusão de lançamentos
- [ ] Contas: cadastro com Boleto + 2 Comprovantes
- [ ] Contas: Status Dinâmico calculado corretamente (A Pagar/A Vencer/Vencida/Pago) conforme seção 4.6
- [ ] Contas: obrigatoriedade de comprovante ao marcar como Pago (bloqueia sem)
- [ ] Contas: NÃO entra no cálculo do DRE (seção 2.5, confirmado)
- [ ] Despesas Diretas: múltiplos comprovantes por lançamento (array, sem limite)
- [ ] Despesas Diretas: entra no cálculo do DRE
- [ ] Condomínio Adiantado: marcar como devolvido, entra como pendência no DRE enquanto não devolvido
- [ ] Contas Rateadas: divisão de 1 a 10 partes, soma deve ser 100%
- [ ] Contas Rateadas: leitura automática de valor/vencimento (texto → linha digitável → OCR, seção 4.8-4.10)
- [ ] Contas Rateadas: gera Despesa + Conta por imóvel participante, com cota ajustada (seção 4.4)
- [ ] Contas Rateadas: editar recalcula tudo; excluir remove os gerados também
- [ ] Transferência p/ Pessoal: cria automaticamente uma Entrada espelho no módulo Pessoal
- [ ] DRE: fórmula exata da seção 4.1, com e sem filtro por imóvel
- [ ] Dashboard: KPIs e blocos por grupo, com a fórmula específica da seção 4.2 (ou a fórmula unificada, se essa decisão for tomada — mas documentada explicitamente)

### Módulo Guerra Pessoal
- [ ] Chave de alternância entre os dois módulos, preservando o estado de cada um
- [ ] Cadastro de Categorias: Nome + Tipo
- [ ] Os 10 blocos fixos aparecem na ordem exata da seção 3.2
- [ ] Entradas: Origem com as 4 opções fixas, badge diferenciado para "REPASSE IMÓVEIS"
- [ ] Contas: Tipo/Especificação com as 13 opções fixas + campo condicional "Outros"
- [ ] Contas: mesma trava de comprovante obrigatório ao marcar como Pago
- [ ] Dashboard Pessoal: 4 KPIs + tabela por categoria (seção 3.6)
- [ ] DRE Pessoal: fórmula da seção 4.3
- [ ] Confirmar que Pessoal usa `categoriaId`, não reaproveita `imovelId`

### Transversal
- [ ] Sincronização de dados entre sessões/dispositivos (decidir arquitetura nova, mas o resultado observável — dados aparecem atualizados ao trocar de dispositivo — deve se manter)
- [ ] Upload de arquivo com limite de tamanho comunicado ao usuário (4MB ou o novo limite definido)
- [ ] Renomeação automática de arquivo no padrão observável (ou um padrão novo, mas definido e documentado)
- [ ] Tema claro/escuro persistido por usuário
- [ ] Painéis retráteis "Novo Lançamento" fechados por padrão em todas as telas de cadastro
- [ ] Impressão (ou substituída por exportação de relatório de verdade — melhoria sugerida)
- [ ] Mensagens de erro amigáveis, sem stack trace técnico exposto ao usuário final

---

## FIM DO DOCUMENTO

Este documento foi gerado por leitura completa e sequencial de `imoveis/index.html` (2.754 linhas), sem execução de nenhum código, sem alteração de nenhum arquivo, sem qualquer escrita no Firebase. Toda tabela, fórmula e fluxo aqui descrito corresponde a uma citação direta ou paráfrase fiel de trechos reais do código-fonte, não a suposições sobre o que o sistema "provavelmente" faz.
