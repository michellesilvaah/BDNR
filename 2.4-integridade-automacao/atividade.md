# 2.4 Integridade e automação

Banco: `atividade2` (MongoDB / mongosh)

## Código

### criarPedido (exercícios 1, 2 e 3)

```javascript
function criarPedido(clienteId, itens) {
  if (!Array.isArray(itens) || itens.length === 0) {
    print("Pedido inválido: é necessário informar ao menos um item.")
    return
  }

  for (const item of itens) {
    if (typeof item.quantidade !== "number" || item.quantidade <= 0) {
      print("Pedido inválido: a quantidade deve ser maior que zero.")
      return
    }
  }

  const cliente = db.clientes.findOne({ _id: clienteId, ativo: true })
  if (!cliente) {
    print("Cliente inexistente ou inativo.")
    return
  }

  let valorTotal = 0
  const itensPedido = []

  for (const item of itens) {
    const produto = db.produtos.findOne({
      _id: item.produtoId,
      ativo: true,
      estoque: { $gte: item.quantidade }
    })

    if (!produto) {
      print("Produto inválido ou estoque insuficiente: " + item.produtoId)
      return
    }

    valorTotal += produto.preco * item.quantidade
    itensPedido.push({
      produtoId: produto._id,
      nome: produto.nome,
      quantidade: item.quantidade,
      precoUnitario: produto.preco
    })
  }

  db.pedidos.insertOne({
    clienteId: cliente._id,
    data: new Date(),
    status: "finalizado",
    cancelado: false,
    itens: itensPedido,
    valorTotal: valorTotal
  })

  for (const item of itensPedido) {
    db.produtos.updateOne(
      { _id: item.produtoId },
      { $inc: { estoque: -item.quantidade } }
    )
  }

  print("Pedido criado com sucesso.")
}
```

### cancelarPedido (exercícios 3 e 4)

```javascript
function cancelarPedido(pedidoId) {
  const pedido = db.pedidos.findOne({ _id: pedidoId })
  if (!pedido) {
    print("Pedido inexistente.")
    return
  }

  const r = db.pedidos.updateOne(
    { _id: pedidoId, cancelado: { $ne: true } },
    { $set: { cancelado: true, status: "cancelado", canceladoEm: new Date() } }
  )

  if (r.modifiedCount === 0) {
    print("Pedido já estava cancelado.")
    return
  }

  for (const item of pedido.itens) {
    db.produtos.updateOne(
      { _id: item.produtoId },
      { $inc: { estoque: item.quantidade } }
    )
  }

  print("Pedido cancelado e estoque devolvido.")
}
```

## Exercício 1: impedir pedidos com quantidade menor ou igual a zero

A função percorre os itens e recusa o pedido se algum tiver quantidade que não seja número ou seja menor ou igual a zero.

```javascript
criarPedido(ObjectId("650000000000000000000101"), [{ produtoId: ObjectId("650000000000000000000201"), quantidade: 0 }])
```

Resultado:

```
Pedido inválido: a quantidade deve ser maior que zero.
```

```javascript
criarPedido(ObjectId("650000000000000000000101"), [{ produtoId: ObjectId("650000000000000000000201"), quantidade: -2 }])
```

Resultado:

```
Pedido inválido: a quantidade deve ser maior que zero.
```

## Exercício 2: impedir pedidos sem itens

A função recusa o pedido se `itens` não for um array ou estiver vazio.

```javascript
criarPedido(ObjectId("650000000000000000000101"), [])
```

Resultado:

```
Pedido inválido: é necessário informar ao menos um item.
```

## Exercício 3: campo `cancelado` e simulação de cancelamento

Todo pedido novo é gravado com `cancelado: false`. A função `cancelarPedido` muda o campo para `true`, altera o `status` para `"cancelado"` e registra `canceladoEm`.

Pedido válido criado antes do cancelamento (2 teclados e 1 mouse):

```javascript
criarPedido(ObjectId("650000000000000000000101"), [
  { produtoId: ObjectId("650000000000000000000201"), quantidade: 2 },
  { produtoId: ObjectId("650000000000000000000202"), quantidade: 1 }
])
```

Resultado:

```
Pedido criado com sucesso.
```

Cancelamento:

```javascript
var pedido = db.pedidos.findOne({ cancelado: false })
cancelarPedido(pedido._id)
```

Resultado:

```
(COLE AQUI a mensagem que apareceu após cancelarPedido)
```

Pedido após o cancelamento:

```javascript
db.pedidos.find()
```

Resultado:

```
(COLE AQUI o resultado do db.pedidos.find())
```

## Exercício 4: devolver a quantidade ao estoque ao cancelar

O `cancelarPedido` percorre os itens do pedido e devolve cada quantidade ao produto com `$inc`. O `updateOne` do cancelamento só atua em pedidos ainda não cancelados (`cancelado: { $ne: true }`), o que evita devolver o estoque duas vezes.

Estoque depois da venda e antes do cancelamento:

```javascript
db.produtos.find({}, { nome: 1, estoque: 1, _id: 0 })
```

```
{ nome: 'Teclado', estoque: 8 }
{ nome: 'Mouse', estoque: 2 }
```

Estoque depois do cancelamento:

```
{ nome: 'Teclado', estoque: 10 }
{ nome: 'Mouse', estoque: 3 }
```

Tentativa de cancelar o mesmo pedido de novo:

```javascript
cancelarPedido(pedido._id)
```

Resultado:

```
(COLE AQUI a mensagem que apareceu na segunda tentativa de cancelar)
```

## Exercício 5: total vendido por produto

A consulta ignora pedidos cancelados, separa os itens com `$unwind` e agrupa por produto somando a quantidade e o valor (`quantidade × precoUnitario`). Foi executada antes do cancelamento.

```javascript
db.pedidos.aggregate([
  { $match: { cancelado: { $ne: true } } },
  { $unwind: "$itens" },
  { $group: {
      _id: "$itens.produtoId",
      nome: { $first: "$itens.nome" },
      quantidadeVendida: { $sum: "$itens.quantidade" },
      totalVendido: { $sum: { $multiply: ["$itens.quantidade", "$itens.precoUnitario"] } }
  } },
  { $sort: { totalVendido: -1 } }
])
```

Resultado:

```
{
  _id: ObjectId('650000000000000000000201'),
  nome: 'Teclado',
  quantidadeVendida: 2,
  totalVendido: 240
}
{
  _id: ObjectId('650000000000000000000202'),
  nome: 'Mouse',
  quantidadeVendida: 1,
  totalVendido: 60
}
```
