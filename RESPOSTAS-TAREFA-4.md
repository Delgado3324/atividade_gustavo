# Tarefa 4 — Exclusão (DELETE)

Backend usado nos testes: `https://backend-gustavoweb.vercel.app`
Front rodando em `http://localhost:5173` (`npm run dev`).
Contas descartáveis usadas: `claude.ui.teste1@exemplo-teste.com` e `claude.ui.teste2@exemplo-teste.com` (as duas já foram desativadas no fim do teste).

---

## 1. O código final (`src/services/api.js`)

```js
export async function desativarConta(token) {
  const resposta = await fetch(`${API_URL}/api/usuarios/desativar`, {
    method: "DELETE",
    headers: {
      Authorization: `Bearer ${token}`,
    },
  });

  const dados = await resposta.json();

  if (!resposta.ok) {
    throw new Error(dados.mensagem || "Não foi possível desativar a conta.");
  }

  return dados;
}
```

Sem `body`, sem `Content-Type` — só o método e o crachá. Quem é o dono da conta,
o servidor descobre pelo próprio token.

---

## 2. O que estava incorreto (e foi corrigido)

### Bug real: `dados.message` em vez de `dados.mensagem`

```diff
-    throw new Error(dados.message || "Erro ao desativar a conta");
+    throw new Error(dados.mensagem || "Não foi possível desativar a conta.");
```

O backend **sempre** responde com a chave `mensagem` (em português):

```json
{ "sucesso": false, "mensagem": "Token inválido ou expirado. Faça login novamente." }
```

`dados.message` (em inglês) é sempre `undefined`, então o `||` caía direto no texto
genérico. O erro não quebrava a tela, mas **engolia a explicação do servidor**: em vez
de "Token inválido ou expirado", o card vermelho mostrava só "Erro ao desativar a conta".
Era justamente o teste do Passo 2 que ficava sem sentido.

Depois da correção, o teste do Passo 2 passa a mostrar a mensagem real do backend
(comprovado no navegador — ver seção 4).

### Ajuste menor: mensagem de fallback do `editarPerfil`

```diff
-    throw new Error(dados.mensagem || "Não deu certo👌.");
+    throw new Error(dados.mensagem || "Não foi possível salvar as alterações.");
```

Não é bug, mas é texto que o usuário lê na tela — vale escrever como as outras quatro.

### O que já estava certo

`login`, `cadastrar`, `listarUsuarios` e `editarPerfil` seguem a receita corretamente
(método, headers, `.json()`, `resposta.ok`, `return`). `listarUsuarios` devolve
`dados.usuarios` (só o array), como pedido. Nenhum componente precisou de mudança.

---

## 3. Passo 3 — a investigação

### Experiência 1 — entrar com a conta desativada

```
POST /api/usuarios/login
→ 401  {"sucesso":false,"mensagem":"E-mail ou senha inválidos."}
```

Confirma a hipótese: parece que a conta sumiu.

### Experiência 2 — cadastrar o mesmo e-mail de novo

```
POST /api/usuarios/cadastrar
→ 400  {"sucesso":false,"mensagem":"Este e-mail já está cadastrado."}
```

**Aqui a hipótese cai.** Se a conta tivesse sido apagada, o e-mail estaria livre.

### A explicação: soft delete

O `DELETE` não apaga nada. No backend ele é um UPDATE disfarçado:

```js
await Usuario.findByIdAndUpdate(req.usuario._id, { ativo: false });
```

O documento continua inteiro no MongoDB — só trocou um campo. Quem some é só quem
**filtra por `ativo`**:

| Rota | Consulta que ela faz | Por isso… |
|---|---|---|
| Listagem | `find({ ativo: true })` | você some do mural |
| Login | `findOne({ email, ativo: true })` | Experiência 1 deu **401** |
| Cadastro | checa só o e-mail, **sem** filtrar `ativo` | Experiência 2 deu **400** |

**Por que empresas fazem assim:** arrependimento ("reative sua conta"), histórico
(pedidos antigos não podem virar pedidos de ninguém), lei (nota fiscal tem prazo de
guarda) e erro humano (DELETE de verdade não tem Ctrl+Z).

**O preço:** o e-mail fica preso. O campo é único no banco inteiro, inclusive para os
inativos. Não é bug — é a conta que essa decisão cobra. Toda escolha técnica resolve um
problema e cria outro em outro lugar.

---

## 4. Checklist "antes de entregar" — tudo verificado

| Item | Resultado |
|---|---|
| Desativei uma conta e fui deslogado | ✅ voltou para a tela de login na hora |
| Entrei com outra conta e a desativada sumiu do mural | ✅ mural foi de 3 para 2 pessoas |
| No console de rede: `DELETE` com **200** e **sem body** no pedido | ✅ `{"sucesso":true,"mensagem":"Conta desativada com sucesso."}` |
| Fiz as duas experiências do Passo 3 e sei explicar | ✅ seção 3 |
| Estraguei o token de propósito, vi a mensagem no card — e desfiz | ✅ apareceu "Token inválido ou expirado. Faça login novamente." dentro do card vermelho, **sem** deslogar; depois desfeito |

Testes extras feitos direto na API (curl), todos batendo com o esperado:

| Pedido | Status | Resposta |
|---|---|---|
| `POST /api/usuarios/cadastrar` | 201 | "Usuário cadastrado com sucesso!" |
| `POST /api/usuarios/login` | 200 | "Login realizado com sucesso!" |
| `GET /api/usuarios` | 200 | lista com a conta presente |
| `PUT /api/usuarios/editar` | 200 | "Dados atualizados com sucesso!" |
| `DELETE /api/usuarios/desativar` (token estragado) | 401 | "Token inválido ou expirado…" |
| `DELETE /api/usuarios/desativar` (sem token nenhum) | 401 | "Acesso negado. Faça login para continuar." |
| `DELETE /api/usuarios/desativar` (token válido) | 200 | "Conta desativada com sucesso." |
| `GET /api/usuarios` (por outra conta) | 200 | a conta desativada **não** aparece |

---

## 5. O CRUD completo, em uma tabela

| Função | Método | Rota | Body? | Token? | Status |
|---|---|---|---|---|---|
| `cadastrar` | POST | `/api/usuarios/cadastrar` | sim | não | 201 — criei |
| `login` | POST | `/api/usuarios/login` | sim | não | 200 |
| `listarUsuarios` | GET | `/api/usuarios` | não | sim | 200 — li |
| `editarPerfil` | PUT | `/api/usuarios/editar` | sim | sim | 200 — atualizei |
| `desativarConta` | DELETE | `/api/usuarios/desativar` | não | sim | 200 — removi |

Os quatro passos são sempre os mesmos: **mandar** (`fetch`) → **traduzir** (`.json()`)
→ **checar** (`resposta.ok`) → **entregar** (`return`). O que muda de uma para outra é
só o método, ter ou não body, e ter ou não token.
