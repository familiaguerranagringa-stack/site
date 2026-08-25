# 07 — Segurança

**Nota sobre este relatório**: onde uma senha, token ou chave foi encontrado, o valor **não é reproduzido** aqui — apenas confirmo a existência e a localização (arquivo/linha), conforme solicitado.

## 🔴 CRÍTICO — Credenciais em texto plano no código-fonte público
**Localização**: `imoveis/index.html`, linhas ~241–244 (constantes `CREDS` e `CREDS_EXTRA`).
As 3 senhas de acesso ao sistema estão escritas em texto plano, direto no JavaScript que é enviado ao navegador de qualquer visitante. Como o repositório GitHub é **público** e o arquivo é servido sem nenhuma proteção, **qualquer pessoa pode**:
- Ver o código-fonte da página (Ctrl+U / "Exibir código-fonte") e ler as 3 senhas
- Ou simplesmente abrir o repositório no GitHub e ler o arquivo diretamente
- Com usuário+senha em mãos, logar no sistema e ver/editar todos os dados financeiros

Isso equivale, na prática, a não ter senha nenhuma — é apenas uma barreira visual, não uma barreira de segurança real.

## 🔴 CRÍTICO — Sem autenticação de verdade (Firebase Auth não é usado)
O SDK do Firebase está presente, mas **apenas o Realtime Database é usado** — não há `firebase-auth`. Isso significa que o acesso ao banco de dados depende inteiramente das **Regras de Segurança do Realtime Database**, configuradas no console do Firebase (fora deste repositório, portanto **não pude auditar o conteúdo dessas regras**).

Isso é um ponto que precisa de verificação urgente e manual no console do Firebase: se as regras estiverem no modo padrão "de teste" (`.read: true, .write: true` sem expiração, ou expiradas e "abertas" por padrão), **qualquer pessoa na internet** que descubra a URL do banco (`https://grupofamiliaguerra-9d13e-default-rtdb.firebaseio.com`, que também está exposta em texto plano no código-fonte) pode ler e escrever em todos os dados financeiros da família, sem precisar sequer logar no site — o "login" da aplicação é só uma tela, não uma barreira no banco.

**Ação recomendada antes de qualquer outra coisa**: acessar o Console do Firebase → Realtime Database → Regras, e confirmar que o acesso está de fato restrito.

## 🔴 CRÍTICO — XSS armazenado (Stored XSS)
**Localização**: múltiplos pontos, ex. linhas ~690, ~1149, ~2059 (renderização de campos "Observação" e outros campos de texto livre).
O sistema insere texto digitado pelo usuário (observações, nomes, descrições) diretamente em `innerHTML` sem nenhuma função de escape de HTML (`escapeHtml`/`esc()` — **não existe no código**). Em alguns lugares há um `.replace(/"/g,'&quot;')` isolado (só protege contra quebra de atributo `value="..."`, não contra tags HTML), mas na maioria das exibições em tabela o campo é inserido cru.

Na prática: se alguém digitar `<img src=x onerror="...">` num campo de Observação, esse código HTML/JS será executado no navegador de **qualquer outro usuário** que visualizar aquele lançamento depois (inclusive após sincronizar de outro dispositivo). O risco prático é baixo hoje (apenas 3 usuários de confiança, todos da mesma família), mas é uma vulnerabilidade real que se agrava caso o sistema seja aberto a mais pessoas no futuro.

## 🟠 ALTO — Chave de API do Firebase exposta
**Localização**: linha ~285 (`apiKey:'AIzaSy...'`).
Isso é **esperado e normal** para aplicações client-side com Firebase (a `apiKey` do Firebase não é secreta por natureza — a Google documenta isso). A segurança real depende 100% das Regras do Realtime Database (ver item acima), não de esconder essa chave. Incluído aqui apenas como confirmação de que foi encontrada e para reforçar que a mitigação correta é a configuração de regras, não a remoção da chave.

## 🟡 MÉDIO — Geração de ID fraca
**Localização**: função `uuid()` no código.
Os identificadores dos registros (`id`) não usam `crypto.randomUUID()` nem uma biblioteca de UUID v4 real — são gerados por uma função simples baseada em `Math.random()`. O risco de colisão é baixo no volume atual de uso, mas não segue prática recomendada e não é criptograficamente seguro (irrelevante para IDs de registro, mas ruim como hábito caso reaproveitado em outro contexto sensível).

## 🟡 MÉDIO — Sem limite de tamanho/tipo de arquivo no upload
Não há validação de tamanho máximo nem de tipo MIME real (apenas o atributo `accept=".pdf,image/*"` do `<input>`, que é só uma sugestão de UI e pode ser burlado facilmente). Um arquivo grande ou malicioso pode ser enviado.

## 🟡 MÉDIO — Sem HTTPS enforcement explícito no código
O domínio customizado via GitHub Pages normalmente serve com HTTPS por padrão, mas isso depende da configuração do domínio no painel do GitHub (não visível neste repositório) e não há nenhum redirecionamento HTTP→HTTPS feito pela própria aplicação.

## 🟢 BAIXO / Não encontrado
- **SQL Injection**: não aplicável — não há banco SQL.
- **CSRF clássico**: baixo risco — não há formulários submetidos a um backend próprio com cookies de sessão; a "sessão" é local (`sessionStorage`) e as escritas vão direto para o Firebase autenticado (ou não) pela regra do banco.
- **Tokens de API/segredos de terceiros no código**: não encontrado nenhum token do GitHub, AWS, Stripe, etc. embutido no `imoveis/index.html`. (Nota: durante o desenvolvimento deste sistema, um token de acesso do GitHub foi usado **fora do repositório**, apenas nas ferramentas de linha de comando usadas para publicar as alterações — não faz parte do código publicado e não está commitado no histórico do arquivo analisado.)
- **Dados pessoais sensíveis versionados**: nomes de inquilinos, valores de aluguel e dados de contato aparecem apenas como **dados de uso em runtime** (armazenados no Firebase, não no Git) — não há dados de produção commitados diretamente no repositório GitHub.

## Resumo de prioridade de correção
1. Trocar a URL do banco/regras do Firebase para modo autenticado de verdade (Firebase Auth com e-mail/senha, ou no mínimo regras `.read`/`.write` restritas por token) — **antes de qualquer outra mudança**
2. Remover senhas em texto plano do JS público — mover autenticação para o backend/Firebase Auth
3. Adicionar uma função de escape de HTML e aplicá-la em todo campo de texto livre renderizado
4. Validar tamanho/tipo de arquivo no upload
