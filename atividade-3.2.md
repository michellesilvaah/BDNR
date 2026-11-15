# 3.2 Otimização das consultas de um sistema de chamados

Banco: `aula3_atividade` (MongoDB / mongosh)

## Parte 1: preparação da base

```javascript
use aula3_atividade
```

```javascript
const sortear = (arr) => arr[Math.floor(Math.random() * arr.length)]
const inteiro = (min, max) => Math.floor(Math.random() * (max - min + 1)) + min
const estados = ["SP", "RJ", "MG", "PR", "RS", "SC", "BA"]
const categorias = ["rede", "hardware", "software", "acesso", "financeiro"]
const prioridades = ["baixa", "media", "alta", "critica"]
const statusLista = ["aberto", "em_andamento", "aguardando_cliente", "resolvido", "fechado"]
const tagsLista = ["vpn", "senha", "impressora", "email", "backup", "wifi", "lentidao"]
const DIA_MS = 24 * 60 * 60 * 1000

function gerarChamado(numero) {
  const clienteId = inteiro(1, 20000)
  const t1 = sortear(tagsLista)
  let t2 = sortear(tagsLista)
  while (t2 === t1) t2 = sortear(tagsLista)

  return {
    protocolo: "CH" + numero,
    cliente: {
      id: clienteId,
      nome: "Cliente " + clienteId,
      estado: sortear(estados)
    },
    atendente: "atendente" + inteiro(1, 40),
    categoria: sortear(categorias),
    prioridade: sortear(prioridades),
    status: sortear(statusLista),
    tags: [t1, t2],
    abertoEm: new Date(Date.now() - Math.floor(Math.random() * 365 * DIA_MS))
  }
}
```

```javascript
for (let i = 1; i <= 300000; i += 10000) {
  const lote = []
  for (let n = i; n < i + 10000 && n <= 300000; n++) {
    lote.push(gerarChamado(n))
  }
  db.chamados.insertMany(lote)
}
```

Conferência:

```javascript
db.chamados.countDocuments()
```

Resultado: (preencher)

```javascript
db.chamados.findOne()
```

Resultado: (preencher)

```javascript
db.chamados.getIndexes()
```

Resultado: (preencher)

## Parte 2: consultas

```javascript
db.chamados.find({ protocolo: "CH250000" })
```

```javascript
db.chamados.find({ "cliente.id": 1234 }).sort({ abertoEm: -1 })
```

```javascript
db.chamados.find({ atendente: "atendente7", status: "em_andamento" })
```

```javascript
db.chamados.find({ status: "aberto", prioridade: "critica" }).sort({ abertoEm: -1 }).limit(20)
```

```javascript
db.chamados.find({ abertoEm: { $gte: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) } })
```

```javascript
db.chamados.find({ tags: "vpn", "cliente.estado": "RS" })
```

```javascript
db.chamados.find({ status: "fechado" })
```

Correspondência: C1 busca por protocolo, C2 histórico do cliente, C3 fila do atendente, C4 painel de urgências, C5 relatório semanal, C6 análise regional, C7 arquivo.

## Parte 3 e Parte 5: script de medição

Este script executa as 7 consultas, mostra o plano, `nReturned`, `totalDocsExamined` e o tempo médio de 3 execuções. Foi executado antes e depois da criação dos índices.

```javascript
const consultas = {
  C1: () => db.chamados.find({ protocolo: "CH250000" }),
  C2: () => db.chamados.find({ "cliente.id": 1234 }).sort({ abertoEm: -1 }),
  C3: () => db.chamados.find({ atendente: "atendente7", status: "em_andamento" }),
  C4: () => db.chamados.find({ status: "aberto", prioridade: "critica" }).sort({ abertoEm: -1 }).limit(20),
  C5: () => db.chamados.find({ abertoEm: { $gte: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) } }),
  C6: () => db.chamados.find({ tags: "vpn", "cliente.estado": "RS" }),
  C7: () => db.chamados.find({ status: "fechado" })
}

function estagios(plano) {
  const lista = []
  let p = plano.queryPlan || plano
  while (p) {
    lista.push(p.stage)
    p = p.inputStage
  }
  return lista.join(" > ")
}

function medir(codigo) {
  const ex = consultas[codigo]().explain("executionStats")
  let soma = 0
  for (let i = 0; i < 3; i++) {
    const ini = Date.now()
    consultas[codigo]().toArray()
    soma += Date.now() - ini
  }
  print(codigo + " | plano: " + estagios(ex.queryPlanner.winningPlan) +
    " | nReturned: " + ex.executionStats.nReturned +
    " | docsExaminados: " + ex.executionStats.totalDocsExamined +
    " | tempoMedioMs: " + (soma / 3).toFixed(1))
}
```

```javascript
Object.keys(consultas).forEach(medir)
```

Medição do custo de escrita (insere 10.000 chamados e remove em seguida):

```javascript
function medirEscrita() {
  const lote = []
  for (let n = 1000001; n <= 1010000; n++) lote.push(gerarChamado(n))
  const ini = Date.now()
  db.chamados.insertMany(lote)
  const tempo = Date.now() - ini
  db.chamados.deleteMany({ protocolo: { $regex: /^CH\d{7}$/ } })
  print("Inserir 10.000 chamados: " + tempo + " ms")
}
```

```javascript
medirEscrita()
```

### Resultados sem índices

| Consulta | Plano | nReturned | Docs examinados | Tempo médio (ms) |
|----------|-------|-----------|-----------------|------------------|
| C1 | (preencher) | (preencher) | (preencher) | (preencher) |
| C2 | (preencher) | (preencher) | (preencher) | (preencher) |
| C3 | (preencher) | (preencher) | (preencher) | (preencher) |
| C4 | (preencher) | (preencher) | (preencher) | (preencher) |
| C5 | (preencher) | (preencher) | (preencher) | (preencher) |
| C6 | (preencher) | (preencher) | (preencher) | (preencher) |
| C7 | (preencher) | (preencher) | (preencher) | (preencher) |

Escrita sem índices: (preencher) ms

## Parte 4: índices

```javascript
db.chamados.createIndex({ protocolo: 1 }, { unique: true })
```

C1: a busca é por igualdade em um único campo. O índice único leva direto ao documento e ainda impede protocolos repetidos.

```javascript
db.chamados.createIndex({ "cliente.id": 1, abertoEm: -1 })
```

C2: o filtro é por igualdade em `cliente.id` e a ordenação é por `abertoEm` decrescente. O índice composto já entrega os documentos na ordem pedida, sem etapa de `SORT`.

```javascript
db.chamados.createIndex({ atendente: 1, status: 1 })
```

C3: dois filtros de igualdade. O índice composto cobre os dois campos.

```javascript
db.chamados.createIndex({ status: 1, prioridade: 1, abertoEm: -1 })
```

C4: dois filtros de igualdade seguidos da ordenação por `abertoEm`. Com essa ordem de campos, a consulta lê só os 20 primeiros do índice, já ordenados.

```javascript
db.chamados.createIndex({ abertoEm: -1 })
```

C5: filtro por intervalo de datas. O índice em `abertoEm` permite ler apenas a faixa dos últimos 7 dias.

```javascript
db.chamados.createIndex({ "cliente.estado": 1, tags: 1 })
```

C6: um filtro de igualdade no estado e outro em um elemento do array `tags`. O índice composto (multikey, por causa de `tags`) atende os dois filtros.

## Parte 5: medição com índices

Mesmo script da Parte 3, repetido após a criação dos índices de C1 a C6:

```javascript
Object.keys(consultas).forEach(medir)
```

```javascript
medirEscrita()
```

Índice para C7:

```javascript
db.chamados.createIndex({ status: 1 })
```

```javascript
medir("C7")
```

## Parte 6: tabela comparativa

| Consulta | Índice criado | Plano antes | Plano depois | Docs examinados antes | Docs examinados depois | Tempo antes (ms) | Tempo depois (ms) | nReturned |
|----------|---------------|-------------|--------------|-----------------------|------------------------|------------------|-------------------|-----------|
| C1 | `{ protocolo: 1 }` único | | | | | | | |
| C2 | `{ "cliente.id": 1, abertoEm: -1 }` | | | | | | | |
| C3 | `{ atendente: 1, status: 1 }` | | | | | | | |
| C4 | `{ status: 1, prioridade: 1, abertoEm: -1 }` | | | | | | | |
| C5 | `{ abertoEm: -1 }` | | | | | | | |
| C6 | `{ "cliente.estado": 1, tags: 1 }` | | | | | | | |
| C7 | `{ status: 1 }` | | | | | | | |

| Fase | Tempo para inserir 10.000 chamados (ms) |
|------|-----------------------------------------|
| Sem índices | |
| Com índices | |

## Parte 7: análise crítica

**1. Em qual consulta o ganho de tempo foi maior?**

Maior ganho em: (preencher com a consulta, provavelmente C1). Antes do índice, `totalDocsExamined` foi (preencher) enquanto `nReturned` foi (preencher). Quanto maior a distância entre os dois números, mais trabalho foi desperdiçado lendo documentos que não pertencem ao resultado. Com o índice, os dois valores ficaram próximos.

**2. O que aconteceu com C7 após a criação do índice?**

Resultado observado: (preencher). A consulta devolve todos os chamados com status `fechado`, que é cerca de um quinto da coleção. Como o resultado é grande, o índice poupa pouco trabalho: o MongoDB ainda precisa buscar um número enorme de documentos. Por isso o ganho é pequeno ou inexistente, e o índice é pouco seletivo (poucos valores distintos em `status`).

**3. O estágio SORT apareceu em C2 e C4?**

Antes do índice: (preencher, esperado: sim, `SORT` em memória após `COLLSCAN`). Depois do índice: (preencher, esperado: não). Isso significa que, sem índice, o MongoDB lê os documentos e os ordena em memória. Com o índice composto, a ordem de `abertoEm` já está no próprio índice e a ordenação deixa de ser necessária.

**4. Quanto aumentou o tempo de inserção após os índices?**

Sem índices: (preencher) ms. Com índices: (preencher) ms. Aumento de (preencher)%. Cada inserção precisa atualizar todos os índices, então o custo de escrita cresce a cada índice criado. A decisão precisa equilibrar o ganho nas leituras com esse custo, criando índices apenas para as consultas realmente frequentes.

**5. Algum índice pode garantir uma regra de integridade?**

Sim. O índice de C1, `{ protocolo: 1 }` com `unique: true`, impede dois chamados com o mesmo protocolo. Em um banco sem chaves estrangeiras e sem restrições rígidas, o índice único é uma forma de manter a unicidade direto no banco.

**6. Como ficaria a consulta C6 se o cliente fosse apenas referenciado por `clienteId`?**

O estado deixaria de estar no chamado e ficaria na coleção de clientes. A consulta precisaria de duas etapas: buscar os clientes do RS e depois os chamados com a tag `vpn` desses clientes, ou usar `$lookup` em um `aggregate`.

```javascript
db.chamados.aggregate([
  { $match: { tags: "vpn" } },
  { $lookup: { from: "clientes", localField: "clienteId", foreignField: "_id", as: "cliente" } },
  { $unwind: "$cliente" },
  { $match: { "cliente.estado": "RS" } }
])
```

Fica mais lenta e mais complexa do que com o cliente embutido, porque exige cruzar duas coleções.
