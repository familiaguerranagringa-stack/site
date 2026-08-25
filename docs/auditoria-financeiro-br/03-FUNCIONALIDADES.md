# 03 — Funcionalidades (do ponto de vista do usuário)

Levantado diretamente do código (funções `render*`, `add*`, `salvar*`), não presumido.

## Autenticação
- **Login** (`doLogin`): tela com usuário/senha, 3 contas fixas aceitas (ver `06-USUARIOS-E-PERMISSOES.md`)
- **Logout** (`doLogout`): limpa `sessionStorage`, volta pra tela de login
- Sessão dura enquanto a aba do navegador estiver aberta (`sessionStorage`, não `localStorage` — expira ao fechar a aba)

## Módulo IMÓVEIS

| Tela | O que faz | Função principal |
|---|---|---|
| **Dashboard** | KPIs do mês (líquido recebido, despesas pagas, saldo, transferido) + saldo por imóvel agrupado em 11 blocos fixos (Paul Harris, Chaveiro, etc.) | `renderDash` |
| **Aluguéis** | Registro de recebimentos de aluguel, manual ou por importação automática de relatório PDF da imobiliária (Aprisco/Alude) com leitura de valores e correspondência automática do imóvel | `renderAlugueis`, `handleAluguelAnexo`, `addAluguel` |
| **Contas** | Contas enviadas para o corretor pagar (IPTU, água, condomínio...), com status dinâmico (A Pagar / A Vencer / Vencida / Pago), 2 comprovantes de pagamento + boleto | `renderRepasses`, `addRepasse` |
| **Despesas Diretas** | Gastos pagos diretamente pelo proprietário (manutenção, reparos) | `renderDespesas`, `addDespesa` |
| **Condomínio Adiantado** | Condomínio pago adiantado pelo proprietário, aguardando devolução do inquilino | `renderCondominio`, `addCondominio` |
| **Contas Rateadas** | Uma conta única dividida entre vários imóveis por percentual, gerando despesa + lançamento em Contas automaticamente para cada um | `renderRateio`, `confirmarRateio` |
| **Cadastro de Imóveis** | CRUD de imóveis (nome, grupo, endereço, inquilino, corretor, valor) + ferramenta de correção em massa do campo Grupo | `renderImoveis`, `corrigirGruposAutomaticamente` |
| **Transferência p/ Pessoal** | Registra valor transferido do caixa dos imóveis para uso pessoal — **gera automaticamente uma Entrada espelho no módulo Guerra Pessoal** | `renderPessoal`, `addTransferencia` |
| **DRE** | Demonstrativo de resultado do mês, com filtro por imóvel | `renderDRE` |

## Módulo GUERRA PESSOAL

| Tela | O que faz | Função principal |
|---|---|---|
| **Dashboard (Pessoal)** | Total de entradas, gastos pagos, gastos a pagar, saldo líquido; gastos por categoria | `renderDashPessoal` |
| **Entradas** | Dinheiro recebido (repasse automático dos imóveis ou manual), com campo Origem (Repasse Imóveis / Arena / Software / Leandro Pessoal) | `renderEntradasPessoal`, `addEntradaPessoal` |
| **Contas** | Gastos pessoais agrupados em 10 blocos fixos (casas + pessoas da família), com tipo/especificação, status, 2 comprovantes | `renderContasPessoal`, `addContaPessoal` |
| **Cadastro** | CRUD de categorias (casa, pessoa, veículo) | `renderCadastroPessoal` |
| **DRE** | Entradas − gastos pagos = saldo líquido pessoal | `renderDREPessoal` |

## Funcionalidades transversais
- **Sincronização com a nuvem**: manual (botão "Sincronizar") e automática a cada gravação
- **Modo escuro/claro**: alternável, persistido por usuário (localStorage + Firebase)
- **Painéis retráteis**: formulários de lançamento ficam escondidos por padrão, expandem ao clicar
- **Leitura automática de PDF/OCR**: extrai valores, datas e imóvel de relatórios de imobiliária e boletos
- **Anexos**: upload de comprovantes/boletos em qualquer lançamento, com preview/download
- **Ferramentas de manutenção**: limpar cache local, migrar anexos antigos, corrigir grupos automaticamente, remover relatórios antigos anexados incorretamente

## O que NÃO existe (confirmado por ausência no código)
- Não há tela de "Usuários" (gestão de usuários é só a lista fixa `CREDS_EXTRA`)
- Não há tela de "Permissões" (todo usuário tem acesso total)
- Não há exportação de relatórios (PDF/Excel) além da geração de PDF de leitura (importação)
- Não há histórico/auditoria de alterações (quem mudou o quê e quando)
- Não há notificações por e-mail/push
- Não há app mobile nativo (é responsivo via CSS, mas é a mesma SPA web)
