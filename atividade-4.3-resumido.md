# 4.3 Replicação e sharding no sistema de chamados

## Parte 1: o que os índices resolveram?

1. **C1:** de `COLLSCAN` com 300.000 docs examinados (126,0 ms) para `EXPRESS_IXSCAN` com 1 doc (1,3 ms). **C4:** de `SORT > COLLSCAN` com 300.000 docs (141,0 ms) para `LIMIT > FETCH > IXSCAN` com 20 docs (4,7 ms), sem ordenação em memória.
2. Não. Os índices ficam na mesma máquina dos dados. Se ela cair, tudo fica indisponível.
3. Não. Mais índices ocupam mais espaço. A solução é distribuir os dados (sharding).
4. Sharding resolve capacidade, não filtro ruim ou modelagem. A C7, por exemplo, devolve ~20% da coleção e continuaria consultando todos os shards.

| Necessidade do sistema | Medida proposta | Justificativa |
|---|---|---|
| Reduzir a leitura de documentos em C1 | Índice único em `protocolo` | Vai direto ao documento (1 doc examinado). |
| Atender ao filtro e à ordenação de C2 | Índice `{ "cliente.id": 1, abertoEm: -1 }` | Filtra e já entrega ordenado, sem `SORT`. |
| Tolerar a falha de uma máquina | Replicação (replica set) | Cópias completas em outros membros e eleição de novo primary. |
| Distribuir dados além de uma máquina | Sharding | Divide a coleção entre vários shards. |
| Recuperar chamados apagados | Backup | A replicação propaga a exclusão. |

## Parte 2: disponibilidade

| Situação | Comunicação | Existe maioria? | Efeito sobre as escritas |
|---|---|---|---|
| C indisponível | A e B | Sim (2 de 3) | A segue como primary e as escritas continuam. |
| A indisponível | B e C | Sim (2 de 3) | B e C elegem novo primary e as escritas voltam. |
| B e C indisponíveis | Só A | Não (1 de 3) | A vira secondary e as escritas são rejeitadas. |
| A isolado pela rede | A; B e C | A: não. B e C: sim | B e C elegem primary e seguem escrevendo. A não aceita escritas. |

1. A maioria é 2 de 3. Ela é calculada sobre os membros configurados para impedir que dois lados de uma partição elejam primaries ao mesmo tempo.
2. Os membros só percebem a falha após o tempo dos heartbeats (cerca de 10 s) e depois fazem a eleição. Nesse intervalo não há primary.
3. Não. A volta como secondary e se ressincroniza.
4. Não. Se a máquina cair, os três membros caem juntos.
5. Cada membro guarda os 300.000 chamados completos. Dividir os dados seria sharding.

## Parte 3: leitura e confirmação

1. Não são incompatíveis. A replicação tem atraso: um membro já aplicou a alteração e o outro ainda não.
2. Não. `w: "majority"` garante que 2 de 3 membros gravaram. O terceiro pode estar atrasado.
3. Read preference define qual membro atende a leitura (primary ou secondary). No primary o dado é o mais recente, no secondary pode estar atrasado.
4. Em C3 o atendente pode ver um chamado já fechado ou não ver um novo. Em C4 um chamado crítico pode demorar a aparecer. Essas telas exigem dados atuais e devem ler do primary.
5. Só com backup. A exclusão é replicada aos três membros. Também ajudam um membro com atraso configurado ou exclusão lógica.

## Parte 4: distribuição

| Consulta | Informa a chave? | Shards consultados | Justificativa |
|---|---|---|---|
| C1 | Não | S1, S2, S3 | Filtra por `protocolo`. |
| C2 (cliente 1234) | Sim | S1 | 1234 está na faixa de 1 a 7.000. |
| C3 | Não | S1, S2, S3 | Filtra por `atendente` e `status`. |
| C4 | Não | S1, S2, S3 | Filtra por `status` e `prioridade`. |
| C5 | Não | S1, S2, S3 | Filtra por data. |
| C6 | Não | S1, S2, S3 | Estado do cliente não é `cliente.id`. |
| C7 | Não | S1, S2, S3 | Filtra por `status`. |

1. S3 (16000 está na faixa de 14.001 a 20.000).
2. S1 (cliente 1234) e S3 (cliente 16000).
3. Não. O índice só acelera a busca dentro de cada shard. Sem `cliente.id`, o roteador consulta todos.
4. Não. Clientes do RS estão em todas as faixas de `cliente.id`.
5. Não. O índice acelera cada shard, mas o filtro não tem a chave, então todos continuam sendo consultados.
6. Cada shard devolve seus 20 mais recentes. O roteador reúne os resultados, reordena por `abertoEm` e devolve os 20 mais recentes.

## Parte 5: chave de distribuição

| Chave | Valores distintos | Consultas favorecidas | Risco ou limitação |
|---|---|---|---|
| `cliente.id` | ~20.000 | C2 | Demais consultas vão a todos os shards. Um cliente muito ativo concentra dados em um shard. |
| `atendente` | 40 | C3 | Poucos valores e carga desigual entre atendentes. |
| `status` | 5 | C7 | Poucos valores, distribuição desigual, e o documento migraria de shard a cada mudança de status. |
| `abertoEm` | ~300.000 (quase únicos) | C5 | Datas crescentes concentram as inserções no último intervalo (ponto quente). |

1. Sim. A igualdade permite calcular o hash e ir direto ao shard.
2. Não. O hash não preserva faixas: ids próximos podem ficar em shards diferentes.
3. Não. O mesmo `cliente.id` gera sempre o mesmo hash, então todos os chamados desse cliente ficam em um shard.
4. Escolho `cliente.id` com hash. Favorece a C2 (direcionada a um shard) e distribui bem os clientes. Não favorece a C1: a busca por protocolo vai a todos os shards. O mesmo ocorre com C3 a C7.
