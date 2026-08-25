# 09 — Situação Atual do GitHub

## Estrutura do repositório
- **Repositório**: `familiaguerranagringa-stack/site`
- **Visibilidade**: **Público** (qualquer pessoa na internet pode ver todo o código-fonte, incluindo as senhas — ver `07-SEGURANCA.md`)
- **Criado em**: 09/06/2026
- **Último push**: 20/08/2026
- **Tamanho do repositório**: ~4,3 MB
- **Total de commits**: 287 (todos na mesma branch)

## Branches
- **Apenas 1 branch existe**: `main` (que também é a branch padrão)
- Não há branches de desenvolvimento, feature branches ou branches de release
- Todo commit em `main` é publicado automaticamente em produção via GitHub Pages

## Organização dos commits
Analisando o histórico recente, os commits seguem um padrão de mensagem descritiva (em português, geralmente começando com "Fix:", "Add:" ou "feat:"), um commit por alteração pontual — não há esmagamento (squash) nem organização por Pull Request. Não há uso de Pull Requests neste fluxo — tudo é commitado direto na branch principal.

## Arquivos ignorados / `.gitignore`
**Não existe arquivo `.gitignore` no repositório.** Isso significa que não há proteção automática contra o versionamento acidental de arquivos sensíveis (`.env`, chaves privadas, `node_modules/`, etc.) — embora, como não há build/dependências Node neste projeto específico, o risco prático hoje seja baixo, é uma prática ausente que deveria ser corrigida antes de qualquer modernização com backend.

## Arquivos que não deveriam estar versionados
Não foi encontrado nenhum arquivo de segredo dedicado (`.env`, `credentials.json`, chave privada) no repositório. O problema não é "arquivo errado versionado" — é que os segredos foram escritos **diretamente dentro do código-fonte da aplicação** (`imoveis/index.html`), o que é pior: não há como simplesmente adicionar ao `.gitignore` e remover, pois os dados fazem parte da lógica de login da aplicação em produção. Corrigir isso exige mudança de arquitetura (mover autenticação para fora do client-side), não apenas um ajuste de versionamento.

## Estrutura de pastas no repositório
```
/
├── CNAME
├── index.html                    ← site institucional
├── dados/
│   └── srcapas_bacaxa.json
├── dre/index.html                ← sistema separado
├── financeirousa/index.html      ← sistema separado
├── gestao/index.html             ← sistema separado (SR. CAPAS)
├── imoveis/index.html            ← ESTE sistema (Financeiro BR)
├── painel/index.html             ← sistema separado
├── scannerimoveis/index.html     ← sistema separado
└── termos/index.html             ← sistema separado
```
Cada pasta é um sistema HTML autônomo e desacoplado, sem código compartilhado entre eles (cada um tem seu próprio CSS/JS embutido, inclusive duplicando lógica semelhante em vários lugares — ver `10-PROBLEMAS-ENCONTRADOS.md`).

## Workflows / GitHub Actions
**Nenhum workflow configurado.** Não existe a pasta `.github/workflows/`. Não há testes, lint, build ou deploy automatizado além do comportamento nativo do GitHub Pages (publicar o conteúdo estático da branch `main`).

## Estratégia de deploy identificável
GitHub Pages configurado para servir a partir da branch `main`, raiz do repositório (`/`), com domínio customizado via `CNAME`. Não há indicação de uso de Actions para deploy — é o comportamento padrão e automático do GitHub Pages.
