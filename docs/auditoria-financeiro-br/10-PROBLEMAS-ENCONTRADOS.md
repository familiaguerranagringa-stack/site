# 10 — Problemas Encontrados

(Segurança está detalhada em `07-SEGURANCA.md` — este documento cobre qualidade de código, dependências e dívida técnica.)

## Código morto confirmado
As funções abaixo **não são mais chamadas por nenhum elemento ativo da interface** (confirmado por busca: os `id`/`onclick` que as acionavam não existem mais no HTML atual), mas continuam no arquivo, ocupando espaço e criando confusão para manutenção futura:
- `handlePDFFile`
- `abrirPreviewImportPDF`
- `confirmarImportPDF`
- `criarImovelInlineImport`
- `toggleNovoImovelImport`

Esse era o fluxo antigo de importação de PDF na aba Aluguéis, substituído por um fluxo unificado (`handleAluguelAnexo`) durante o desenvolvimento, mas o código antigo nunca foi removido — ~90 linhas de função morta.

## Código duplicado / lógica paralela
O módulo **Guerra Pessoal** foi construído replicando manualmente boa parte da lógica já existente no módulo **Imóveis**, em vez de reaproveitar componentes genéricos:
- `montarModulos()` (Imóveis) e `montarModulosPessoal()` (Pessoal) — mesmo algoritmo de agrupamento fixo, escrito duas vezes com nomes de variável diferentes
- `getGrupoTituloImovel()` / `getBlocoTituloPessoal()` — mesma ideia, duas implementações
- Renderização de tabela de "Contas" (Imóveis `renderRepasses` vs Pessoal `renderContasPessoal`) — estrutura de colunas, badges de status e botões de ação quase idênticos, copiados e levemente adaptados
- Fluxo de "marcar como pago com comprovante obrigatório" duplicado entre `marcarRepassePago`/`confirmarRepassePago` (Imóveis) e `marcarContaPessoalPaga`/`confirmarContaPessoalPaga` (Pessoal)

Isso não quebra nada hoje, mas significa que **todo bug corrigido ou melhoria feita em um módulo precisa ser manualmente replicada no outro** — já aconteceu mais de uma vez ao longo do desenvolvimento.

## Ausência total de testes automatizados
Não existe nenhum teste unitário, de integração ou end-to-end **dentro do repositório**. (Durante o próprio processo de desenvolvimento iterativo deste sistema, testes ad-hoc foram escritos em Node.js fora do repositório para validar mudanças antes de publicar, mas eles não fazem parte do projeto versionado — não há suíte de testes reexecutável.)

## Dependências e versões
| Biblioteca | Versão usada | Observação |
|---|---|---|
| Firebase JS SDK | 10.12.0 (compat) | Usa a API "compat" (estilo antigo v8), não a API modular v9+ recomendada atualmente pelo Google — mais pesada e já considerada legada pela própria documentação do Firebase |
| pdf.js | 3.11.174 | Fixada via CDN, sem processo de atualização — não há como saber, sem checar manualmente, se há versões mais novas com correções de segurança |
| tesseract.js | 5.0.4 | Mesma observação |

Nenhuma dessas dependências é gerenciada por `package.json`/lockfile — atualização, quando necessária, exige edição manual da URL no HTML.

## Falta de escape de HTML (repetido aqui por ser também problema de qualidade, não só segurança)
Não existe uma função utilitária de escape (`escapeHtml`), o que gera os riscos de XSS descritos em `07-SEGURANCA.md`, além de bugs de exibição sempre que um texto do usuário contém `<`, `>`, `&` ou aspas.

## Inconsistência de nomenclatura de dados históricos
Vários bugs corrigidos ao longo do desenvolvimento vieram do mesmo padrão: dados cadastrados em momentos diferentes com formatos ligeiramente diferentes (ex.: nome de imóvel às vezes limpo, às vezes com endereço completo colado; campo "Grupo" às vezes preenchido, às vezes vazio). Isso indica **falta de validação de entrada consistente** no momento do cadastro — o sistema tenta compensar isso em tempo de leitura (com heurísticas de correspondência por palavra-chave) em vez de garantir consistência na escrita.

## Arquivo único gigante
`imoveis/index.html` tem ~174 KB / 2.754 linhas em um único arquivo, misturando HTML, CSS e ~139 funções JavaScript sem nenhuma separação de módulos, sem organização em pastas por funcionalidade. Isso dificulta:
- Localizar código relacionado a uma funcionalidade específica
- Revisão de código (todo PR seria "o arquivo inteiro mudou")
- Reuso entre os diferentes sistemas do repositório (cada pasta duplica sua própria cópia de CSS/lógica de UI parecida)

## Uso de `console.log`/`console.error` como único mecanismo de diagnóstico
24 ocorrências de `console.log`/`console.error` espalhadas pelo código, usadas como principal (e único) mecanismo de depuração em produção — úteis durante o desenvolvimento, mas isso significa que **hoje, se algo dá errado para o usuário final, não há visibilidade alguma** a menos que alguém abra manualmente o console do navegador dele no exato momento do erro.

## Falta de validação de formulário consistente
Alguns formulários validam campos obrigatórios (ex: exigir comprovante quando status = Pago), outros não validam nada (ex: é possível salvar um lançamento com valor zero ou negativo em vários pontos, sem aviso).
