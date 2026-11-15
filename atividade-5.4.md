# 5.4 Atividade prática: evidências de replicação e sharding

**Aluna:** Michelle de Sales Silva
**Banco:** `lab5_rs_michelle_silva` (replica set `rs0`)
**Data da execução:** 15/11/2026

Legenda: **Observado** é o que apareceu no ambiente. **Esperado** é o que a teoria prevê. **Impedimento** é uma etapa que não pôde ser executada, registrada sem dados fictícios.

## 1. Diagnóstico e conexão

**1. Endereços e papéis**

Conexão ao conjunto pelo Compass, com os três endereços e `replicaSet=rs0`.

```javascript
db.hello()
```

Observado (campos relevantes):

```
setName: 'rs0'
hosts: [
  'srv1598762.hstgr.cloud:27017',
  'srv1598762.hstgr.cloud:27018',
  'srv1598762.hstgr.cloud:27019'
]
isWritablePrimary: true
primary: 'srv1598762.hstgr.cloud:27017'
electionId: ObjectId('7fffffff0000000000000009')
```

```javascript
rs.status().members.map(m => ({ nome: m.name, papel: m.stateStr, saude: m.health }))
```

Observado:

```
{ nome: 'srv1598762.hstgr.cloud:27017', papel: 'PRIMARY', saude: 1 }
{ nome: 'srv1598762.hstgr.cloud:27018', papel: 'SECONDARY', saude: 1 }
{ nome: 'srv1598762.hstgr.cloud:27019', papel: 'SECONDARY', saude: 1 }
```

Também foram criadas três conexões diretas (`directConnection=true`), uma por porta. Resultado de `db.hello().isWritablePrimary` em cada uma:

| Conexão direta | isWritablePrimary |
|----------------|-------------------|
| 27017 | true |
| 27018 | false |
| 27019 | false |

**2. Por que conectar a três portas não configura replicação?**

Conectar apenas abre conexões com três processos `mongod` independentes. A replicação precisa ser configurada no conjunto (nome do replica set e lista de membros, por exemplo com `rs.initiate()`), para que os membros se reconheçam, elejam um primary e repliquem os dados entre si.

**3. Conexão ao conjunto x conexão direta**

Na conexão ao conjunto (com `replicaSet=rs0`), o cliente descobre todos os membros e acompanha o primary atual. Se o primary mudar, as escritas passam a ir ao novo primary. Na conexão direta, o cliente fala só com aquele processo. Se o primary mudar, a conexão continua no mesmo membro, que pode ter virado secondary e passa a rejeitar escritas.

## Pt2. Implementar e observar replicação

**4. Dois chamados com confirmação por maioria**

Executado na conexão do conjunto:

```javascript
use lab5_rs_michelle_silva

db.chamados.insertMany(
  [
    { _id: "michelle_silva-CH001", protocolo: "michelle_silva-CH001", cliente: { id: 1001 }, assunto: "Falha no acesso", status: "aberto", abertoEm: new Date() },
    { _id: "michelle_silva-CH002", protocolo: "michelle_silva-CH002", cliente: { id: 1002 }, assunto: "Lentidão na rede", status: "aberto", abertoEm: new Date() }
  ],
  { writeConcern: { w: "majority" } }
)
```

Consulta com `db.chamados.find()` nas três conexões diretas. Observado:

```
{
  _id: 'michelle_silva-CH001',
  protocolo: 'michelle_silva-CH001',
  cliente: { id: 1001 },
  assunto: 'Falha no acesso',
  status: 'aberto',
  abertoEm: ISODate('2026-11-15T22:31:06.355Z')
}
{
  _id: 'michelle_silva-CH002',
  protocolo: 'michelle_silva-CH002',
  cliente: { id: 1002 },
  assunto: 'Lentidão na rede',
  status: 'aberto',
  abertoEm: ISODate('2026-11-15T22:31:06.355Z')
}
```

Os dois chamados foram encontrados nos três membros.

**5. Atualização de status e atraso**

Executado na conexão do conjunto:

```javascript
db.chamados.updateOne(
  { _id: "michelle_silva-CH001" },
  { $set: { status: "em_andamento" } },
  { writeConcern: { w: "majority" } }
)
```

Consulta imediata nas conexões diretas:

```javascript
db.chamados.find({ _id: "michelle_silva-CH001" }, { status: 1 })
```

Observado: `{ _id: 'michelle_silva-CH001', status: 'em_andamento' }`. A alteração apareceu já na primeira consulta. **Nenhum atraso de replicação foi percebido.**

**6. Atualização em um secondary**

Papel confirmado antes do teste, na conexão direta à porta 27018:

```javascript
db.hello().isWritablePrimary
```

Observado: `false`.

```javascript
db.chamados.updateOne({ _id: "michelle_silva-CH001" }, { $set: { status: "fechado" } })
```

Observado:

```
MongoServerError[NotWritablePrimary]: not primary
```

O secondary recusou a escrita, porque somente o primary aceita escritas. Os secondaries apenas aplicam o que recebem do primary. O chamado permaneceu `em_andamento`.

**7. Eleição coordenada pelo professor**

**Impedimento:** a eleição não foi realizada até o momento deste registro, portanto não há dados de primary atual nem de retomada de escrita após a troca. Para não apresentar dados fictícios, registro apenas a situação anterior:

- Primary anterior: `srv1598762.hstgr.cloud:27017`
- `electionId` anterior: `ObjectId('7fffffff0000000000000009')`

Esperado: após a eleição, `db.hello()` mostraria outro primary e um `electionId` diferente. O conjunto voltaria a aceitar escritas pelo novo primary, e os dados continuariam presentes nos três membros.

**8. `rs.stepDown()` x interromper o serviço**

Com `rs.stepDown()`, o primary entrega o papel de forma controlada: o processo continua ativo e a eleição é rápida. Ao interromper o serviço, o membro simplesmente desaparece, os demais só percebem pela falta de heartbeats (cerca de 10 segundos) e só então elegem outro primary, com escritas indisponíveis nesse intervalo.

A experiência não tolera a falha da VPS inteira: os três processos estão na mesma máquina e cairiam juntos.

## Pt3. Implementar sharding

**Impedimento (itens 9 a 14):** o ambiente de sharding (roteador e lista de dois shards fornecida pelo professor) não foi acessado nesta execução. Não há evidências observadas para esses itens e nenhum dado foi inventado. Seguem apenas os pontos conceituais e o que se espera ver.

**11. Por que a contagem de 10.000 documentos pelo roteador, sozinha, não comprova a distribuição?**

O roteador soma o resultado de todos os shards. O total seria o mesmo se todos os documentos estivessem em um único shard. A distribuição se comprova com `db.chamados.getShardDistribution()`, que mostra quantos documentos há em cada shard.

**12. Consultas por cliente, por status e por cliente mais status** (resultado esperado, não observado)

| Consulta | Esperado |
|----------|----------|
| Por cliente (igualdade em `cliente.id`) | Direcionada a um único shard |
| Por status | Todos os shards (o filtro não contém a chave) |
| Por cliente e status | Direcionada a um único shard (contém a chave) |

**13. Índice de status** (esperado, não observado)

O índice reduz o trabalho local em cada shard (menos documentos examinados), mas não muda o roteamento: a consulta por status continua indo a todos os shards, porque o filtro não contém a chave de distribuição.

## Pt4. Implementação do grupo

Função de registro, a ser usada na coleção particionada do grupo:

```javascript
const COLECAO = "chamados"

function registrarChamado(clienteId, protocolo, assunto) {
  if (!Number.isInteger(clienteId) || clienteId <= 0) {
    throw new Error("cliente.id deve ser um inteiro positivo")
  }
  if (typeof protocolo !== "string" || protocolo.trim() === "") {
    throw new Error("protocolo não pode ser vazio")
  }
  if (typeof assunto !== "string" || assunto.trim() === "") {
    throw new Error("assunto não pode ser vazio")
  }

  const doc = {
    _id: new ObjectId(),
    protocolo: protocolo.trim(),
    cliente: { id: clienteId },
    assunto: assunto.trim(),
    status: "aberto",
    abertoEm: new Date()
  }

  const r = db.getCollection(COLECAO).insertOne(doc, { writeConcern: { w: "majority" } })
  return r.insertedId
}
```

Casos de teste previstos (um válido e dois inválidos):

```javascript
registrarChamado(50001, "michelle_silva-CH100", "Falha no acesso")
registrarChamado(-5, "michelle_silva-CH101", "Falha no acesso")
registrarChamado(50001, "", "Falha no acesso")
```

**Impedimento:** a função não foi executada, porque a coleção particionada do grupo não estava disponível. Esperado: o primeiro caso retorna o `_id` inserido, e os outros dois lançam erro sem inserir nada.

**Falha ou repetição da operação:** se a operação falhar por erro de rede, não se deve repetir a inserção às cegas, porque o documento pode ter sido gravado. Antes de repetir, consultaria por `cliente.id` e `protocolo` (consulta direcionada, porque contém a chave) para confirmar se o chamado já existe.

**Índice para o histórico do cliente por data:**

```javascript
db.chamados.createIndex({ "cliente.id": 1, abertoEm: -1 })
```

- **Roteamento:** a igualdade em `cliente.id` (chave hashed) já direciona a consulta a um único shard. O índice não altera o destino.
- **Ordenação:** dentro do shard, o índice composto entrega os chamados já ordenados por `abertoEm` decrescente, sem etapa de ordenação em memória.

## Entrega

| Evidência | Registro |
|-----------|----------|
| Banco e identificador do grupo | Banco `lab5_rs_michelle_silva`, replica set `rs0` |
| Papéis antes e depois da eleição | Antes: 27017 PRIMARY, 27018 e 27019 SECONDARY. Depois: impedimento (eleição não realizada) |
| Mesmo documento observado nos três membros | Sim: `CH001` e `CH002` nos três membros (Parte 2, item 4) |
| Resposta à escrita em secondary | `MongoServerError[NotWritablePrimary]: not primary` |
| Retomada da escrita após eleição | Impedimento (eleição não realizada) |
| Identificação do roteador e dos shards | Impedimento (ambiente de sharding não acessado) |
| Distribuição real dos documentos | Impedimento |
| Shards participantes nas três consultas | Impedimento (apenas esperado, na Parte 3) |
| Comparação antes/depois do índice | Impedimento (apenas esperado, na Parte 3) |
| Função implementada e casos de teste | Função implementada (Parte 4). Execução: impedimento |

## Discussão final

**Por que três cópias não dividem a coleção em três partes?** Na replicação, cada membro guarda a coleção completa. Isso foi observado: os mesmos chamados `CH001` e `CH002` apareceram nos três membros. As cópias existem para disponibilidade e redundância. Dividir os dados entre máquinas é função do sharding.

**Por que dois shards não garantem cópia de segurança um do outro?** Cada shard guarda uma parte diferente dos dados, e não uma cópia da parte do outro. Se um shard for perdido, os documentos que só estavam nele deixam de existir. Para ter cópia, cada shard precisa ser, por sua vez, um replica set.

**Por que dois processos na mesma VPS não oferecem tolerância à perda dessa VPS?** Se a VPS falhar, todos os processos que estão nela caem juntos. No ambiente usado, os três membros do `rs0` estão no mesmo servidor (`srv1598762.hstgr.cloud`, portas 27017, 27018 e 27019), então a falha dessa máquina derrubaria os três. A tolerância exige membros em máquinas ou domínios de falha diferentes.
