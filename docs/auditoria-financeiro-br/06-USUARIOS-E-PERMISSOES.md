# 06 — Usuários e Permissões

## Modelo de usuários (confirmado no código, arquivo `imoveis/index.html`)

```javascript
const CREDS={u:'Guerra',p:'[senha em texto plano — ver 07-SEGURANCA.md]'};
const CREDS_EXTRA=[
  {u:'TERRA',p:'[senha em texto plano]',nome:'Terra'},
  {u:'PAIVA',p:'[senha em texto plano]',nome:'Paiva'},
];
```

- **3 contas fixas**, definidas diretamente no código-fonte JavaScript, que é público (qualquer pessoa pode ver o código-fonte da página).
- Login verifica se usuário+senha digitados batem **exatamente** com uma dessas entradas (comparação de string simples, sem hash, sem salt).
- Ao logar com sucesso: grava `sessionStorage.imoveisfg_auth='1'` e `sessionStorage.imoveisfg_user=<usuario>`.
- **Não existe cadastro de novos usuários pela interface** — adicionar/remover usuário exige editar o código-fonte e publicar.

## Perfis e permissões
**Não existem.** Todos os 3 usuários têm exatamente o mesmo nível de acesso a tudo: ambos os módulos (Imóveis e Guerra Pessoal), todas as telas, todos os dados financeiros, todos os anexos. Não há:
- Distinção entre administrador e usuário comum
- Permissão por módulo, tela ou ação
- Restrição de valores/dados por usuário
- Aprovação em múltiplas etapas para nenhuma operação

## Controle de sessão
- Baseado em `sessionStorage` (expira ao fechar a aba/navegador — não é "lembrar-me" persistente)
- Sem expiração por tempo (token não expira enquanto a aba ficar aberta)
- Sem invalidação remota (não há como "deslogar" um usuário à força de outro dispositivo)
- Sem proteção contra múltiplos logins simultâneos

## O uso do campo "usuário" para outras finalidades
O identificador de usuário logado (`imoveisfg_user`) é reaproveitado apenas para:
- Personalizar a preferência de tema (claro/escuro) por usuário
- Nada além disso — não há atribuição de lançamentos a um usuário específico (não se sabe, olhando um lançamento, quem o criou)

## Recomendação de leitura cruzada
Ver `07-SEGURANCA.md` para os riscos concretos desse modelo (senha exposta no código-fonte, sem Firebase Auth, sem regras de banco visíveis/auditáveis por este relatório).
