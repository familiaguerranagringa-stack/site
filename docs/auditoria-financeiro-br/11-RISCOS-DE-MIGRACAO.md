# 11 — Riscos de Modernização/Migração

## O que pode ser alterado facilmente (baixo risco)
- **Estilo visual (CSS)**: cores, espaçamento, tipografia — isolado e sem efeito colateral em dados
- **Textos e rótulos da interface**: mudanças de copy não afetam a estrutura de dados
- **Adição de novos campos opcionais** aos formulários existentes (desde que com valor padrão seguro quando ausente em registros antigos)
- **Ferramentas de manutenção** (limpar cache, migrar anexos) — já isoladas e não destrutivas por design

## O que deve ser preservado
- **A convenção de chaves de dados** (`imoveisfg_<entidade>_<ano>_<mes>`) — qualquer migração de banco precisa mapear esse padrão para o novo modelo sem perder o vínculo com o período correto
- **Os 11 grupos fixos de imóveis e os 10 blocos fixos do Guerra Pessoal** — são regras de negócio reais da família (nomes de casas/pessoas específicas), não just "categorias genéricas" — uma modernização que tratar isso como dado normal de cadastro, sem preservar a ordem/nomes exatos, vai quebrar a experiência que a família já está acostumada
- **O vínculo automático Transferência→Entrada** entre os dois módulos — é o único ponto de integração e fácil de esquecer numa reescrita módulo-por-módulo
- **A lógica de status dinâmico** (A Pagar / A Vencer / Vencida / Pago, calculada por data, não travada no valor salvo) — foi corrigida via bug real durante o desenvolvimento; uma reescrita ingênua pode reintroduzir o mesmo bug

## O que apresenta risco real de quebrar o sistema
1. **Trocar o modelo de sincronização** (blob único → banco relacional/documentos individuais) **sem migrar os dados existentes primeiro**. Todo o histórico financeiro está hoje dentro de um único nó JSON no Firebase; qualquer nova arquitetura de banco precisa de um script de migração testado, não uma reescrita "do zero".
2. **Mudar o formato do `id`** dos registros. Diversos campos usam esse `id` como chave estrangeira lógica (`imovelId`, `categoriaId`, `rateioLancId`) espalhados por várias entidades — trocar o formato sem atualizar todas as referências quebra os vínculos silenciosamente (sem erro, os dados só "somem" de onde deveriam aparecer).
3. **Introduzir autenticação real (Firebase Auth ou outra) sem migrar as 3 sessões ativas com cuidado** — troca abrupta pode bloquear o acesso da própria família até a reconfiguração ser concluída.
4. **Alterar a lógica de agrupamento (`GRUPOS_ORDEM`/`BLOCOS_ORDEM_PESSOAL`)** sem testar com os dados reais — o sistema já teve múltiplos bugs de "imóvel sumiu do agrupamento" por causa de nomenclatura inconsistente; qualquer reescrita dessa lógica precisa rodar contra uma cópia dos dados reais antes de ir para produção.
5. **Assumir que os dados estão limpos.** Como não há validação forte na escrita, é praticamente certo que existam registros com campos ausentes, nomes inconsistentes ou tipos inesperados (string onde se espera número, etc.) — qualquer novo backend com tipagem estrita (ex: TypeScript + banco SQL com colunas `NOT NULL`) vai rejeitar parte dos dados existentes até que sejam limpos/normalizados.

## O que precisa de testes antes de qualquer alteração
- A geração automática de lançamentos (Contas Rateadas → Despesas + Contas; Transferência → Entrada Pessoal) — são os únicos pontos "mágicos" onde uma ação gera efeitos em múltiplas entidades
- O parser de PDF de relatório de imobiliária e o OCR de boleto — são heurísticas frágeis por natureza; qualquer mudança na lógica de extração deveria ser validada contra uma amostra real de PDFs já usados
- O cálculo de saldo/DRE (soma de líquido − despesas pagas, por imóvel e consolidado) — é o número mais sensível do sistema (dinheiro real); qualquer refatoração desse cálculo merece conferência manual linha a linha contra o resultado atual antes de substituir

## O que deveria ser refatorado (não necessariamente descartado)
- Separar o arquivo único em módulos organizados (mesmo mantendo vanilla JS, dá para dividir por pasta/arquivo com um bundler simples)
- Extrair a lógica de agrupamento fixo (Imóveis e Pessoal) para uma função genérica única, parametrizada pela lista de grupos — elimina a duplicação
- Criar uma camada de acesso a dados (repository pattern) em vez de manipular `localStorage`/Firebase diretamente espalhado por todas as telas

## O que deveria ser substituído
- **Login por string hardcoded** → autenticação real (Firebase Auth, ou backend próprio com hash de senha)
- **Sincronização "sobrescreve tudo"** → escrita granular por registro (ex: Firestore com documentos individuais, ou um banco relacional com um backend real)
- **Armazenamento de arquivos dentro do Realtime Database em base64** → Firebase Storage ou S3-compatível, com URLs assinadas
- **API "compat" do Firebase (v8 style)** → SDK modular v9+ (menor, mais rápido, oficialmente recomendado)

## Risco de concorrência (não é um bug de código, é uma limitação estrutural)
Como cada `save()` reenvia o **estado completo do localStorage** para o Firebase, se dois usuários estiverem com o sistema aberto ao mesmo tempo em abas/dispositivos diferentes e ambos fizerem uma alteração antes de sincronizar, **o último a salvar sobrescreve o trabalho do outro sem aviso nem merge**. Isso é uma limitação já presente hoje (não é introduzida pela modernização), mas deve ser resolvida em qualquer arquitetura nova.
