# 12 — Arquitetura Recomendada (proposta — nada foi executado)

## Princípio geral
Modernizar em etapas, sem "big bang". O sistema atual funciona e está em uso ativo pela família — qualquer proposta que exija parar tudo para reescrever do zero é um risco desnecessário. A recomendação é migrar por camadas, mantendo o sistema no ar durante toda a transição.

## Stack recomendada

| Camada | Recomendação | Por quê |
|---|---|---|
| Frontend | Manter abordagem simples (HTML/CSS/JS) **ou** migrar para um framework leve (ex: um SPA com componentização real), dependendo do apetite da família por complexidade de manutenção | O sistema atual já funciona bem para o uso real; a prioridade não é "modernizar por modernizar" o frontend, e sim resolver segurança e dados |
| Autenticação | Firebase Auth (e-mail/senha ou magic link) — reaproveita o mesmo projeto Firebase já existente, migração mais suave | Elimina senhas em texto plano sem trocar toda a infraestrutura |
| Banco de dados | Manter Firebase, mas migrar de Realtime Database (blob único) para **Firestore** (documentos individuais, granularidade real, regras de segurança por coleção) | Resolve o problema de "sobrescrever tudo" e melhora a granularidade sem sair do ecossistema Firebase (menor esforço de migração) |
| Armazenamento de arquivos | Firebase Storage | Arquivos binários não deveriam estar dentro de um banco de dados JSON |
| Deploy | Manter GitHub Pages **com** um passo de build simples (mesmo que seja só um script que empacota/minifica) | Já funciona bem para um app estático; não há necessidade de trocar de hospedagem |
| CI/CD | GitHub Actions com: lint → testes → build → deploy automático apenas quando mergeado em `main` | Introduz um portão de qualidade sem trocar a plataforma de hospedagem |

## Ambientes propostos

```
DESENVOLVIMENTO           STAGING                    PRODUÇÃO
(local, no computador     (branch "staging",         (branch "main",
 do desenvolvedor)         projeto Firebase           projeto Firebase
                            de teste separado)         de produção)
     │                          │                          │
     └── PR ──────────────────►│                          │
                          revisão/teste                    │
                                └── merge aprovado ────────►│
```
- **Desenvolvimento**: cada alteração feita localmente, testada manualmente antes de subir
- **Staging**: uma branch separada (`staging`), publicada num subdomínio de teste (ex: `staging.familiaguerra.com.br`), conectada a um **projeto Firebase separado** (dados de teste, não os dados reais da família) — assim erros de desenvolvimento nunca tocam nos dados financeiros de verdade
- **Produção**: só recebe código que já passou por staging e foi aprovado manualmente (merge para `main`)

## Backups
- Habilitar exportação automática periódica do Firestore/Realtime Database para um bucket de armazenamento (ex: rotina diária), mantendo pelo menos 30 dias de histórico
- Isso é configurável dentro do próprio Firebase/Google Cloud, sem necessidade de infraestrutura adicional

## Logs e monitoramento
- Adicionar um serviço de captura de erro client-side (ex: Sentry, gratuito até certo volume) para saber quando algo quebra na tela de um usuário real, sem depender de "abrir o console do navegador dele"
- Habilitar os logs nativos do Firebase (Cloud Logging) para auditoria de leituras/escritas no banco

## Domínio: `familiaguerra.com.br/financeiro-br` vs `financeiro-br.familiaguerra.com.br`

| Critério | `familiaguerra.com.br/financeiro-br` (subpasta) | `financeiro-br.familiaguerra.com.br` (subdomínio) |
|---|---|---|
| Configuração de DNS | Nenhuma DNS extra — mesmo domínio raiz | Exige registro DNS adicional (CNAME/A) apontando para a hospedagem |
| Isolamento de cookies/sessão | Compartilha o mesmo "site" do domínio raiz (menos isolamento) | Isolamento total — cookies/sessão de um subdomínio não vazam para outro |
| Hospedar em serviços diferentes | Mais difícil — normalmente exige reverse proxy ou rewrite rules no mesmo servidor que serve a raiz | Mais fácil — cada subdomínio pode apontar para uma hospedagem/projeto totalmente diferente (útil se o Financeiro BR crescer e precisar de um backend próprio no futuro) |
| Compatibilidade com GitHub Pages | Requer que o domínio raiz já esteja no GitHub Pages e que o roteamento de subpasta seja resolvido pelo próprio servidor estático (funciona bem se for tudo estático, como hoje) | Requer configurar um novo `CNAME` específico para o subdomínio, apontando pro GitHub Pages (ou outro host) — processo simples, mas é um passo a mais de configuração |
| Percepção do usuário | Parece "parte do mesmo site institucional" | Parece um "sistema separado", o que reflete melhor a realidade (é um sistema interno, não uma página do site público) |

**Recomendação técnica**: `financeiro-br.familiaguerra.com.br` (subdomínio), pelos seguintes motivos práticos:
1. Isolamento real de sessão/cookies em relação ao site institucional público (`familiaguerra.com.br`), que é importante para um sistema com dados financeiros sensíveis
2. Flexibilidade para no futuro hospedar o Financeiro BR num ambiente diferente (ex: com backend próprio) sem depender do mesmo host do site institucional
3. Mais simples de replicar o padrão que já funciona hoje (`grupofamiliaguerra.com.br` → `grupofamiliaguerra.com.br/imoveis/` já é subpasta; migrar para subdomínio dedicado é uma boa oportunidade de separar de vez sistemas internos do site público)

Esta é uma recomendação técnica, não uma decisão — fica para a família validar considerando também fatores não-técnicos (ex: preferência de branding, facilidade de lembrar o endereço).
