# 05 — Integrações

## Integrações externas reais (confirmadas no código)

| Serviço | Uso | Tipo |
|---|---|---|
| **Firebase Realtime Database** | Único "backend" — armazenamento de todos os dados e arquivos | SDK JS client-side, sem servidor intermediário |
| **cdnjs.cloudflare.com** | Hospeda `pdf.js` e `tesseract.js` | CDN público, sem fallback local |
| **gstatic.com** | Hospeda o SDK do Firebase (`firebase-app-compat.js`, `firebase-database-compat.js`) | CDN oficial do Google |
| **Google Fonts** | Tipografia (Syne, DM Sans) | CDN público |

## O que NÃO existe
- Nenhuma integração com gateway de pagamento
- Nenhuma integração com banco (Open Finance, extrato automático)
- Nenhum webhook, próprio ou de terceiros
- Nenhuma API própria exposta (não há endpoints REST/GraphQL criados por este sistema)
- Nenhuma integração com WhatsApp, e-mail (SMTP) ou SMS
- Nenhuma integração com Receita Federal / SEFAZ / emissão de nota fiscal

## "Leitura de PDF de imobiliária" não é uma integração de API
Apesar de o sistema ler relatórios da Aprisco/Alude (imobiliárias), **não há integração de API com essas empresas**. O fluxo é: o usuário baixa manualmente o PDF do relatório (fora do sistema) e faz upload; o `pdf.js` extrai o texto no navegador e uma função de regex (`parseRelatorioAlude`) tenta reconhecer os campos pelo padrão textual do relatório dessas duas imobiliárias especificamente. Qualquer mudança no layout do PDF dessas empresas quebra silenciosamente o parser.

## Dependências de terceiros e risco de disponibilidade
Como não há bundling/build, toda vez que a página carrega ela busca ao vivo:
```html
<script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-app-compat.js"></script>
<script src="https://www.gstatic.com/firebasejs/10.12.0/firebase-database-compat.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/tesseract.js/5.0.4/tesseract.min.js"></script>
```
Se qualquer um desses CDNs estiver fora do ar, o sistema **quebra completamente** (Firebase é indispensável para login/dados) ou perde funcionalidade parcial (leitura de PDF/OCR). Não há versionamento local nem SRI (Subresource Integrity hash) nessas tags — se o CDN for comprometido, código malicioso poderia ser injetado sem que o repositório mude uma linha.
