# 13 — Plano de Modernização (proposta faseada — nada foi executado)

## Fase 0 — Contenção de risco (fazer antes de qualquer coisa, sem esperar o resto do plano)
1. **Verificar manualmente as Regras de Segurança do Firebase Realtime Database no console** — confirmar que não estão abertas ao público (ver `07-SEGURANCA.md`). Isso não exige nenhuma mudança de código, só uma verificação/ajuste de configuração, e é o item de maior risco potencial hoje.
2. Trocar as 3 senhas atuais (mesmo continuando com o modelo atual por enquanto) — reduz o dano de a senha já ter circulado em texto plano publicamente
3. Confirmar se existe alguma exportação/backup do Firebase configurada; se não, habilitar uma rotina básica de exportação periódica

## Fase 1 — Backup e ambiente de staging (sem mudar produção)
1. Criar um segundo projeto Firebase, exclusivo para testes
2. Exportar uma cópia dos dados reais (anonimizada, se possível) para popular o ambiente de staging
3. Criar branch `staging` no GitHub, publicada num subdomínio de teste
4. Configurar GitHub Actions básico: nada além de "publicar staging quando a branch `staging` receber push"

## Fase 2 — Autenticação real
1. Habilitar Firebase Auth no projeto
2. Criar as 3 contas de usuário no Firebase Auth (mesmos nomes, senhas novas)
3. Trocar a tela de login para usar `firebase.auth()` em vez da comparação de string hardcoded
4. Atualizar as Regras de Segurança do banco para exigir autenticação (`auth != null`)
5. Testar em staging antes de ir para produção

## Fase 3 — Migração de dados para modelo granular
1. Escrever um script único de migração: ler o blob atual do Realtime Database e gravar cada entidade como documentos individuais no Firestore (ou manter Realtime Database, mas reestruturado por sub-nós em vez de um blob único — decisão a tomar com a família)
2. Rodar a migração primeiro em staging, validar visualmente que todos os dados aparecem corretos em todas as telas
3. Ajustar o código de `save`/`load` para gravar/ler granularmente, em vez do padrão atual "sobrescreve tudo"
4. Só então migrar produção, num horário de baixo uso, com backup imediatamente antes

## Fase 4 — Higienização de segurança do frontend
1. Adicionar função de escape de HTML e aplicá-la em todos os pontos que hoje inserem texto do usuário via `innerHTML`
2. Validar tamanho/tipo de arquivo nos uploads
3. Revisar e remover código morto (`handlePDFFile` e função relacionadas — ver `10-PROBLEMAS-ENCONTRADOS.md`)

## Fase 5 — Redução de duplicação
1. Extrair a lógica de agrupamento fixo (hoje duplicada entre Imóveis e Guerra Pessoal) para uma função/módulo único e genérico
2. Unificar os fluxos de "marcar como pago com comprovante obrigatório" entre os dois módulos

## Fase 6 — Observabilidade
1. Adicionar captura de erros client-side (ex: Sentry)
2. Configurar monitoramento básico de disponibilidade do domínio

## Fase 7 — Domínio final
1. Configurar `financeiro-br.familiaguerra.com.br` (ou a opção escolhida pela família — ver `12-ARQUITETURA-RECOMENDADA.md`)
2. Publicar produção final somente após todas as fases anteriores estarem validadas em staging

## O que este plano deliberadamente NÃO inclui
- Reescrita do frontend num framework novo — é uma decisão separada, de menor urgência que os itens de segurança/dados acima, e pode ser feita depois, com o sistema já mais seguro
- Mudança de hospedagem (GitHub Pages continua adequado para este tipo de aplicação)
- Qualquer alteração das regras de negócio existentes (grupos fixos, cálculo de saldo, etc.) — a modernização é de infraestrutura, não de funcionalidade
