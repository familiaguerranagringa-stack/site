# 08 — Infraestrutura e Deploy

## Hospedagem atual
- **Plataforma**: GitHub Pages (gratuito)
- **Repositório**: `familiaguerranagringa-stack/site` (público)
- **Branch servida**: `main` (única branch existente no repositório)
- **Domínio customizado**: `grupofamiliaguerra.com.br` (arquivo `CNAME` na raiz)
- **URL do sistema**: `https://grupofamiliaguerra.com.br/imoveis/`
- **Certificado HTTPS**: gerenciado automaticamente pelo GitHub Pages (Let's Encrypt), não requer ação manual — mas não pude confirmar o status exato de "Enforce HTTPS" no painel do GitHub a partir deste repositório.

## Processo de deploy atual
**Não há pipeline de CI/CD.** O processo é:
1. Alteração é feita diretamente no arquivo `imoveis/index.html`
2. `git commit` + `git push` direto para a branch `main`
3. GitHub Pages detecta o push e publica automaticamente em produção, tipicamente em menos de 1 minuto

**Isso significa que todo commit na branch `main` vai direto para produção**, sem etapa de revisão, sem ambiente de teste, sem aprovação. Não existe branch de desenvolvimento nem staging.

## Ambientes
| Ambiente | Existe? |
|---|---|
| Desenvolvimento | ❌ Não — alterações são feitas e publicadas diretamente |
| Staging/Homologação | ❌ Não existe |
| Produção | ✅ `grupofamiliaguerra.com.br/imoveis/` (é o único ambiente) |

## Docker
Não existe — não há `Dockerfile`, `docker-compose.yml` nem qualquer configuração de containerização. Como o sistema é um único arquivo HTML estático, não há necessidade técnica de container para rodar o frontend (mas seria útil para um eventual ambiente de desenvolvimento local com backend real, caso a modernização crie um).

## Scripts / Build
Não existe `package.json`, não existe processo de build, não existe minificação, não existe bundling. O arquivo publicado é exatamente o mesmo arquivo editado — não há diferença entre "código fonte" e "código publicado".

## CI/CD e GitHub Actions
Não foi encontrado nenhum workflow (`.github/workflows/`) no repositório. Não há testes automatizados rodando em nenhum momento do processo.

## Logs e Monitoramento
- **Logs de aplicação**: apenas `console.log`/`console.error` no navegador do usuário — nada é centralizado, coletado ou persistido em nenhum serviço (não há Sentry, LogRocket, Datadog, etc.)
- **Monitoramento de disponibilidade**: nenhum (não há uptime monitor configurado neste projeto)
- **Analytics**: nenhum encontrado no código (sem Google Analytics, sem Firebase Analytics)

## Backups
- **Do código**: o histórico de commits do Git funciona como backup de versões do código-fonte (isso é adequado).
- **Dos dados**: **não existe rotina de backup dos dados financeiros.** O Firebase Realtime Database não tem exportação automática configurada neste projeto (até onde é visível pelo código). Como toda gravação **sobrescreve o nó inteiro do banco**, um erro de sincronização, bug ou ação acidental pode sobrescrever/perder dados sem possibilidade de recuperação simples, a menos que o Firebase tenha backups próprios habilitados no console (não verificável a partir deste repositório).

## Escalabilidade
Como não há backend nem processamento server-side, a "escalabilidade" de tráfego não é uma preocupação relevante (GitHub Pages/CDN aguenta tranquilamente). O gargalo real é:
- Crescimento ilimitado dos arrays de dados carregados inteiros no navegador a cada acesso (sem paginação)
- O modelo de "sobrescrever o banco inteiro a cada gravação" não escala para mais usuários simultâneos
