# 01 — Visão Geral do Sistema (Financeiro BR / "IMÓVEIS — Família Guerra Holding")

## O que é
Sistema de gestão financeira de locações de imóveis da Família Guerra, operando no Brasil (BRL, formato de data brasileiro, PDFs de imobiliárias brasileiras). Módulo irmão "Guerra Pessoal" (finanças pessoais) foi adicionado dentro do mesmo arquivo.

Nome interno usado neste relatório: **Financeiro BR**, correspondendo à pasta `imoveis/` do repositório.

## Onde vive
- **Repositório GitHub:** `familiaguerranagringa-stack/site` (público, branch única `main`)
- **Arquivo:** `imoveis/index.html` — um único arquivo de ~174.000 caracteres / 2.754 linhas contendo HTML + CSS + JavaScript
- **Hospedagem:** GitHub Pages, servido via domínio customizado `grupofamiliaguerra.com.br` (arquivo `CNAME` na raiz do repo)
- **URL de produção:** `https://grupofamiliaguerra.com.br/imoveis/`
- **Banco de dados:** Firebase Realtime Database (projeto `grupofamiliaguerra-9d13e`), caminho raiz `imoveisfg`

## Classificação arquitetural
Isto **não é** uma aplicação com backend tradicional (sem Node/PHP/Python server, sem ORM, sem API REST própria, sem tabelas SQL). É uma **SPA client-side pura**:
- Todo o código roda no navegador do usuário
- O "banco de dados" é o Firebase Realtime Database, acessado diretamente do navegador via SDK JS
- Não há servidor de aplicação, não há autenticação real (Firebase Auth não é usado — ver `07-SEGURANCA.md`)
- O "deploy" é literalmente um `git commit` + `git push` para o arquivo HTML; o GitHub Pages serve o arquivo estático

## Módulos dentro do mesmo arquivo
1. **IMÓVEIS** — gestão de locações (Dashboard, Aluguéis, Contas, Despesas Diretas, Condomínio Adiantado, Contas Rateadas, Cadastro de Imóveis, Transferência p/ Pessoal, DRE)
2. **GUERRA PESSOAL** — finanças pessoais (Dashboard, Entradas, Contas, Cadastro de Categorias, DRE), alternável por uma chave (switch) no topo da sidebar
3. Os dois módulos compartilham o mesmo arquivo, mesmo login, mesmo projeto Firebase (caminhos de dados separados por prefixo de chave: `imoveisfg_*` e `pessoalfg_*`)

## Outros sistemas no mesmo repositório (fora do escopo desta auditoria)
O repositório contém outros sistemas independentes, cada um em sua própria pasta e completamente desacoplado do `imoveis/`:
- `gestao/` — "SR. CAPAS — Gestão Financeira" (varejo)
- `financeirousa/` — gestão financeira das LLCs americanas
- `dre/` — ferramenta de DRE mensal separada
- `painel/` — painel interno / hub
- `scannerimoveis/`, `termos/`, `index.html` (site institucional)

Nenhum desses foi analisado neste relatório — o escopo é exclusivamente `imoveis/index.html`.

## Usuários
Três usuários com credenciais fixas no código-fonte (ver `06-USUARIOS-E-PERMISSOES.md` e `07-SEGURANCA.md`): Guerra, TERRA, PAIVA. Todos têm acesso total e idêntico ao sistema — não há distinção de papéis/permissões.

## Estágio de maturidade
Sistema em desenvolvimento contínuo e ativo (últimos commits datados de agosto de 2026), construído incrementalmente por meio de pedidos avulsos, sem plano de arquitetura formal, sem testes automatizados, sem ambiente de staging.
