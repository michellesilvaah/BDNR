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

```
300000
```

```javascript
db.chamados.findOne()
```

```
{
  _id: ObjectId('6afa2243614b09846b2f0553'),
  protocolo: 'CH1',
  cliente: {
    id: 2918,
    nome: 'Cliente 2918',
    estado: 'BA'
  },
  atendente: 'atendente14',
  categoria: 'software',
  prioridade: 'critica',
  status: 'fechado',
  tags: [
    'vpn',
    'email'
  ],
  abertoEm: ISODate('2026-02-09T13:28:36.062Z')
}
```

```javascript
db.chamados.getIndexes()
```

```
[ { v: 2, key: { _id: 1 }, name: '_id_' } ]
```

## Parte 2: consultas

C1, busca rápida por protocolo:

```javascript
db.chamados.find({ protocolo: "CH250000" })
```

C2, histórico do cliente:

```javascript
db.chamados.find({ "cliente.id": 1234 }).sort({ abertoEm: -1 })
```

C3, fila do atendente:

```javascript
db.chamados.find({ atendente: "atendente7", status: "em_andamento" })
```

C4, painel de urgências:

```javascript
db.chamados.find({ status: "aberto", prioridade: "critica" }).sort({ abertoEm: -1 }).limit(20)
```

C5, relatório semanal:

```javascript
db.chamados.find({ abertoEm: { $gte: new Date(Date.now() - 7 * 24 * 60 * 60 * 1000) } })
```

C6, análise regional:

```javascript
db.chamados.find({ tags: "vpn", "cliente.estado": "RS" })
```

C7, arquivo:

```javascript
db.chamados.find({ status: "fechado" })
```

## Parte 3: medição sem índices

Script de medição. Para cada consulta ele mostra o plano, `nReturned`, `totalDocsExamined` e o tempo médio de 3 execuções.

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

Custo de escrita (insere 10.000 chamados e remove em seguida):

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

Medição executada com a coleção apenas com o índice do `_id`:

```javascript
Object.keys(consultas).forEach(medir)
```

```
C1 | plano: COLLSCAN | nReturned: 1 | docsExaminados: 300000 | tempoMedioMs: 126.0
C2 | plano: SORT > COLLSCAN | nReturned: 13 | docsExaminados: 300000 | tempoMedioMs: 153.7
C3 | plano: COLLSCAN | nReturned: 1512 | docsExaminados: 300000 | tempoMedioMs: 156.3
C4 | plano: SORT > COLLSCAN | nReturned: 20 | docsExaminados: 300000 | tempoMedioMs: 141.0
C5 | plano: COLLSCAN | nReturned: 5854 | docsExaminados: 300000 | tempoMedioMs: 257.7
C6 | plano: COLLSCAN | nReturned: 12195 | docsExaminados: 300000 | tempoMedioMs: 400.0
C7 | plano: COLLSCAN | nReturned: 59759 | docsExaminados: 300000 | tempoMedioMs: 1410.3
```

```javascript
medirEscrita()
```

```
Inserir 10.000 chamados: 248 ms
```

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

C4: dois filtros de igualdade seguidos da ordenação por `abertoEm`. Com essa ordem de campos, a consulta lê apenas os 20 primeiros itens do índice, já ordenados.

```javascript
db.chamados.createIndex({ abertoEm: -1 })
```

C5: filtro por intervalo de datas. O índice em `abertoEm` permite ler só a faixa dos últimos 7 dias.

```javascript
db.chamados.createIndex({ "cliente.estado": 1, tags: 1 })
```

C6: um filtro de igualdade no estado e outro em um elemento do array `tags`. O índice composto (multikey, por causa de `tags`) atende os dois filtros.

## Parte 5: medição com índices

Depois de criar os índices de C1 a C6:

```javascript
Object.keys(consultas).forEach(medir)
```

```
C1 | plano: EXPRESS_IXSCAN | nReturned: 1 | docsExaminados: 1 | tempoMedioMs: 1.3
C2 | plano: FETCH > IXSCAN | nReturned: 13 | docsExaminados: 13 | tempoMedioMs: 2.0
C3 | plano: FETCH > IXSCAN | nReturned: 1512 | docsExaminados: 1512 | tempoMedioMs: 28.0
C4 | plano: LIMIT > FETCH > IXSCAN | nReturned: 20 | docsExaminados: 20 | tempoMedioMs: 4.7
C5 | plano: FETCH > IXSCAN | nReturned: 5854 | docsExaminados: 5854 | tempoMedioMs: 101.3
C6 | plano: FETCH > IXSCAN | nReturned: 12195 | docsExaminados: 12195 | tempoMedioMs: 306.3
C7 | plano: FETCH > IXSCAN | nReturned: 59759 | docsExaminados: 59759 | tempoMedioMs: 834.0
```

```javascript
medirEscrita()
```

```
Inserir 10.000 chamados: 544 ms
```

Observação: nenhum índice foi criado para C7 nesta etapa, mas o plano de C7 já aparece com `IXSCAN`. Isso acontece porque o índice criado para C4, `{ status: 1, prioridade: 1, abertoEm: -1 }`, começa pelo campo `status`, e o MongoDB pode usá-lo para filtrar só por `status`.

Índice específico para C7:

```javascript
db.chamados.createIndex({ status: 1 })
```

```javascript
medir("C7")
```

```
C7 | plano: FETCH > IXSCAN | nReturned: 59759 | docsExaminados: 59759 | tempoMedioMs: 861.3
```

## Parte 6: tabela comparativa

| Consulta | Índice criado | Plano antes | Plano depois | Docs examinados antes | Docs examinados depois | Tempo antes (ms) | Tempo depois (ms) | nReturned |
|----------|---------------|-------------|--------------|-----------------------|------------------------|------------------|-------------------|-----------|
| C1 | `{ protocolo: 1 }` único | COLLSCAN | EXPRESS_IXSCAN | 300000 | 1 | 126.0 | 1.3 | 1 |
| C2 | `{ "cliente.id": 1, abertoEm: -1 }` | SORT > COLLSCAN | FETCH > IXSCAN | 300000 | 13 | 153.7 | 2.0 | 13 |
| C3 | `{ atendente: 1, status: 1 }` | COLLSCAN | FETCH > IXSCAN | 300000 | 1512 | 156.3 | 28.0 | 1512 |
| C4 | `{ status: 1, prioridade: 1, abertoEm: -1 }` | SORT > COLLSCAN | LIMIT > FETCH > IXSCAN | 300000 | 20 | 141.0 | 4.7 | 20 |
| C5 | `{ abertoEm: -1 }` | COLLSCAN | FETCH > IXSCAN | 300000 | 5854 | 257.7 | 101.3 | 5854 |
| C6 | `{ "cliente.estado": 1, tags: 1 }` | COLLSCAN | FETCH > IXSCAN | 300000 | 12195 | 400.0 | 306.3 | 12195 |
| C7 | `{ status: 1 }` | COLLSCAN | FETCH > IXSCAN | 300000 | 59759 | 1410.3 | 861.3 | 59759 |

| Fase | Tempo para inserir 10.000 chamados (ms) |
|------|-----------------------------------------|
| Sem índices | 248 |
| Com índices (6 índices, além do `_id`) | 544 |

## Parte 7: análise crítica

**1. Em qual consulta o ganho de tempo foi maior? Relacione com a diferença entre `totalDocsExamined` e `nReturned` antes do índice.**

O maior ganho relativo foi em C1: de 126,0 ms para 1,3 ms, cerca de 97 vezes mais rápida. Antes do índice, a consulta examinou 300.000 documentos para devolver apenas 1. Quanto maior a distância entre `totalDocsExamined` e `nReturned`, mais trabalho foi desperdiçado lendo documentos que não pertencem ao resultado. C2 (300.000 examinados para 13 retornados, cerca de 77 vezes mais rápida) e C4 (300.000 para 20, cerca de 30 vezes mais rápida) seguem o mesmo padrão. Depois do índice, `totalDocsExamined` ficou igual a `nReturned` em todas as consultas.

**2. O que aconteceu com C7 após a criação do índice?**

O ganho foi pequeno: de 1410,3 ms para 861,3 ms, cerca de 1,6 vez mais rápida. A consulta devolve 59.759 documentos, aproximadamente 20% da coleção. Mesmo com o índice, o MongoDB precisa buscar quase 60 mil documentos (`FETCH`), e esse custo domina o tempo. Além disso, o campo `status` tem poucos valores distintos (5), então o índice é pouco seletivo.

Há um segundo ponto: antes de criar o índice `{ status: 1 }`, C7 já usava o índice de C4 (`status` é o primeiro campo dele) e levava 834,0 ms. O índice novo não trouxe melhora (861,3 ms, diferença dentro da variação normal entre execuções) e se tornou praticamente redundante.

**3. Em C2 e C4, o estágio `SORT` apareceu antes do índice? E depois? O que isso significa?**

Antes do índice, o plano de C2 e de C4 foi `SORT > COLLSCAN`: o MongoDB leu a coleção inteira e ordenou o resultado em memória. Depois do índice, o `SORT` desapareceu (`FETCH > IXSCAN` em C2 e `LIMIT > FETCH > IXSCAN` em C4). Isso significa que o índice já guarda as entradas na ordem de `abertoEm`, então os documentos saem ordenados sem etapa extra. Em C4, o `LIMIT` ainda permite parar após ler 20 entradas.

**4. Quanto aumentou o tempo de inserção após os índices?**

Foi de 248 ms para 544 ms, um aumento de 296 ms (cerca de 119%, isto é, mais que o dobro). Cada inserção precisa atualizar todos os índices, então o custo de escrita cresce a cada índice criado. A decisão deve equilibrar o ganho nas leituras com esse custo: vale criar índices para consultas frequentes e lentas, como C1, C2 e C4, e evitar índices que pouco ajudam, como o de C7.

**5. Algum dos índices criados pode garantir uma regra de integridade?**

Sim. O índice `{ protocolo: 1 }` criado com `unique: true` impede dois chamados com o mesmo protocolo. Em um banco sem chaves estrangeiras e sem restrições rígidas como as dos bancos relacionais, o índice único é uma forma de garantir unicidade no próprio banco.

**6. Como ficaria a consulta C6 se o cliente fosse apenas referenciado por `clienteId`?**

O estado deixaria de estar dentro do chamado e passaria a existir só na coleção de clientes. A consulta precisaria cruzar duas coleções, por exemplo com `$lookup` em um `aggregate`:

```javascript
db.chamados.aggregate([
  { $match: { tags: "vpn" } },
  { $lookup: { from: "clientes", localField: "clienteId", foreignField: "_id", as: "cliente" } },
  { $unwind: "$cliente" },
  { $match: { "cliente.estado": "RS" } }
])
```

Ficaria mais complexa e mais lenta do que com o cliente embutido, porque o filtro por estado só poderia ser aplicado depois do cruzamento com a coleção de clientes.
