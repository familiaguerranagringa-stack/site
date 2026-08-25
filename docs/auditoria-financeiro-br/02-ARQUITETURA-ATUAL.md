# 02 — Arquitetura Atual

## Diagrama textual

```
┌──────────────────────────────────────────────────────────┐
│  Navegador do usuário                                     │
│  ┌────────────────────────────────────────────────────┐  │
│  │  imoveis/index.html (2.754 linhas)                  │  │
│  │  - HTML + CSS embutido (<style>)                     │  │
│  │  - JavaScript embutido (<script>), ~139 funções      │  │
│  │  - Estado em memória: objeto global ST               │  │
│  │  - Cache local: localStorage (chaves imoveisfg_*/    │  │
│  │    pessoalfg_*)                                      │  │
│  └────────────────────────────────────────────────────┘  │
│         │ fetch/CDN                    │ Firebase SDK      │
└─────────┼──────────────────────────────┼───────────────────┘
          ▼                              ▼
  CDNs externos (sem build):    Firebase Realtime Database
  - firebase-app-compat.js       (projeto grupofamiliaguerra-9d13e)
  - firebase-database-compat.js  Caminho: /imoveisfg (blob único JSON)
  - pdf.js (extração de texto de PDF)
  - tesseract.js (OCR client-side)
```

## Stack real (não presumida — confirmada no código)
| Camada | Tecnologia | Observação |
|---|---|---|
| Frontend | HTML + CSS + JavaScript vanilla (ES6+) | Sem framework (não é React/Vue/Angular) |
| Build | **Nenhum** | Sem bundler, sem npm, sem `package.json`, sem transpilação |
| Backend/API | **Nenhum** | Não existe servidor de aplicação próprio |
| Banco de dados | Firebase Realtime Database | NoSQL, um único blob JSON por sincronização |
| Autenticação | **Nenhuma (Firebase Auth não usado)** | Login é uma checagem de string hardcoded no JS |
| Armazenamento de arquivos | Firebase Realtime Database (`imoveisfg_files/{fileId}`) | Arquivos em base64 dentro do próprio RTDB, não Firebase Storage |
| Hospedagem | GitHub Pages | Domínio customizado via CNAME |
| CDN de bibliotecas | cdnjs.cloudflare.com, gstatic.com | Sem versionamento local, dependência direta de terceiros |

## Modelo de dados e sincronização (o "banco" na prática)
Não existem tabelas, linhas ou relacionamentos no sentido tradicional. O modelo é:

1. **localStorage** guarda um conjunto de chaves (`imoveisfg_alugueis_2026_08`, `imoveisfg_repasses_2026_08`, `imoveisfg_imoveis`, etc.), cada uma contendo um array JSON de "registros" (objetos JS simples).
2. Toda alteração (`save(k,d)`) grava a chave localmente e depois dispara `pushToFB()`, que:
   - Varre **todo** o localStorage com prefixo `imoveisfg_`/`pessoalfg_`
   - Monta um único objeto JSON gigante com todas as chaves
   - Sobrescreve o nó inteiro `imoveisfg` no Firebase Realtime Database via `.set()`
3. Ao carregar a página / sincronizar manualmente, `syncFromFB()` lê esse nó inteiro e sobrescreve o localStorage local com o que veio da nuvem.

**Implicação crítica:** não há sincronização granular por registro. Cada gravação reescreve o banco inteiro. Dois usuários editando em abas/dispositivos diferentes ao mesmo tempo podem se sobrescrever mutuamente sem aviso (ver `11-RISCOS-DE-MIGRACAO.md`).

## Convenção de chaves de dados
- `imoveisfg_<entidade>` — dados não vinculados a um mês (ex: `imoveisfg_imoveis`)
- `imoveisfg_<entidade>_<ano>_<mes>` — dados do período selecionado na tela (ex: `imoveisfg_alugueis_2026_08`)
- Mesmo padrão com prefixo `pessoalfg_` para o módulo Guerra Pessoal

## Padrão de renderização
Não há Virtual DOM nem componentização. Cada "aba" é uma função `renderX()` que retorna uma string HTML gigante (template literals), atribuída diretamente a `document.getElementById('content').innerHTML`. Toda a tela é redesenhada do zero a cada interação relevante (`render()`), exceto onde otimizações pontuais foram feitas manualmente (ex: painel retrátil de "Novo Lançamento" que alterna uma classe CSS em vez de re-renderizar, para permitir animação).

## Arquivos e upload
- Arquivos (comprovantes, boletos) são lidos no navegador via `FileReader`, convertidos para base64 (`data:...`) e enviados **diretamente para o Firebase Realtime Database** sob `imoveisfg_files/{fileId}`, não para Firebase Storage.
- O registro do lançamento guarda apenas a referência leve (`{fileId, nomeFinal, mime, tamanho}`), não o arquivo em si — desde uma correção aplicada durante o desenvolvimento (antes disso, arquivos ficavam embutidos diretamente no registro, causando estouro de cota do localStorage; existe inclusive uma ferramenta de migração manual, `migrarAnexosAntigos()`, para converter registros antigos).
- **Não há limite de tamanho de arquivo validado no código.** Arquivos grandes em base64 podem ser pesados demais para o Realtime Database (que não foi desenhado para isso).

## Leitura de PDF / OCR
- `pdf.js` extrai texto nativo de PDFs (relatórios de administração de imobiliária)
- Se não houver texto (PDF escaneado), o sistema tenta OCR via `tesseract.js`, com timeout de 5s
- Fallback adicional: varredura de "linha digitável" (código de barras de boleto) diretamente nos bytes crus do arquivo, com validação de faixa de valor razoável
- Toda essa lógica roda 100% no navegador do usuário — nenhum processamento server-side
