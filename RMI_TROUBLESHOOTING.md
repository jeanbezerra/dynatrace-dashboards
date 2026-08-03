# Troubleshooting de RMI, threads e rede TCP por host

Consultas para **Dynatrace SaaS / Grail / DQL (Latest Dynatrace)**, revisadas em
2026-08-03. O fluxo recomendado para o dashboard e:

1. selecionar o host em `$Host`;
2. selecionar um dos processos encontrados nesse host em `$Process`;
3. analisar processo, chamadas Java RMI, threads JVM e conexoes TCP.

As consultas novas usam `dt.smartscape.host` e `dt.smartscape.process`. Os
campos `dt.entity.host` e `dt.entity.process_group_instance` sao legados e estao
em processo de descontinuacao.

## Pre-requisitos

- OneAgent com monitoramento profundo no processo Java para spans RMI e
  metricas JVM.
- Traces powered by Grail para as consultas `fetch spans`.
- Metrics on Grail para as consultas `timeseries`.
- Extended Network Monitoring/NetTracer para RTT, retransmissoes e metricas
  `dt.process.nettracer.*`.
- Network connection monitoring habilitado para os eventos do bucket
  `default_network_flows`.
- Permissoes de leitura para Smartscape, metricas, traces e eventos.

> Nem todos os indicadores opcionais existem em todos os hosts. Por exemplo,
> `dt.profiling.jvm.*` depende da captura de profiling, e
> `dt.process.nettracer.*` depende do NetTracer. Por isso as consultas de
> metricas usam `union:true`.

## Variaveis do dashboard

Configure as duas variaveis como DQL, selecao unica e nesta ordem. Cada consulta
retorna exatamente um campo, conforme exigido por variaveis DQL.

### Variavel `Host`

Retorna nomes legiveis dos hosts. Se houver nomes duplicados no tenant, use a
variante por ID descrita na secao "Ambientes com nomes duplicados".

```dql
smartscapeNodes HOST
| fields Host = name
| dedup Host
| sort Host asc
```

### Variavel dependente `Process`

A variavel e recalculada quando `$Host` muda e lista somente os processos que
executam no host selecionado.

```dql
smartscapeNodes PROCESS
| traverse runs_on, HOST, direction:forward, fieldsKeep:name
| filter name == $Host
| fields Process = dt.traverse.history[0][name]
| dedup Process
| sort Process asc
```

## 1. Inventario de processos do host

Use como tabela para confirmar quais processos o Smartscape relaciona ao host.

```dql
smartscapeNodes PROCESS
| traverse runs_on, HOST, direction:forward, fieldsKeep:name
| filter name == $Host
| fields
    Host = name,
    Process = dt.traverse.history[0][name],
    ProcessId = dt.traverse.history[0][id]
| sort Process asc
```

## 2. Saude e consumo do processo selecionado

Visao temporal de disponibilidade, CPU e memoria. CPU e memoria estao em
percentual; `WorkingSet` esta em bytes.

```dql
timeseries {
    Availability = avg(dt.process.availability),
    CpuAvg = avg(dt.process.cpu.usage),
    CpuMax = max(dt.process.cpu.usage),
    MemoryAvg = avg(dt.process.memory.usage),
    WorkingSet = max(dt.process.memory.working_set_size)
  },
  by:{dt.smartscape.host, dt.smartscape.process},
  filter:{
    getNodeName(dt.smartscape.host) == $Host and
    getNodeName(dt.smartscape.process) == $Process
  },
  union:true
| fieldsAdd
    Host = getNodeName(dt.smartscape.host),
    Process = getNodeName(dt.smartscape.process)
| fieldsRemove dt.smartscape.host, dt.smartscape.process
```

### Resumo escalar do processo

Use como tabela ou conjunto de single values.

```dql
timeseries {
    CpuAvg = avg(dt.process.cpu.usage, scalar:true),
    CpuMax = max(dt.process.cpu.usage, scalar:true),
    MemoryAvg = avg(dt.process.memory.usage, scalar:true),
    WorkingSetMax = max(dt.process.memory.working_set_size, scalar:true),
    FileDescriptorsMaxPct = max(dt.process.handles.file_descriptors_percent_used, scalar:true),
    ThreadExhaustionEvents = sum(dt.process.threads_exhausted, scalar:true)
  },
  by:{dt.smartscape.host, dt.smartscape.process},
  filter:{
    getNodeName(dt.smartscape.host) == $Host and
    getNodeName(dt.smartscape.process) == $Process
  },
  union:true
| fields
    Host = getNodeName(dt.smartscape.host),
    Process = getNodeName(dt.smartscape.process),
    CpuAvg,
    CpuMax,
    MemoryAvg,
    WorkingSetMax,
    FileDescriptorsMaxPct,
    ThreadExhaustionEvents
```

## 3. Indicadores de Java RMI

No modelo semantico atual, RMI e identificado por `rpc.framework` e
`rpc.protocol` iguais a `java-rmi`. O fallback em `network.protocol.name`
permite encontrar spans produzidos com o identificador `java_rmi`.

As consultas abaixo mantem apenas spans `server`, isto e, chamadas recebidas e
executadas pelo processo selecionado. Troque para `client` para investigar as
chamadas RMI feitas por esse processo a servidores remotos.

### RMI: volume, falhas e latencia ao longo do tempo

O calculo de `Calls` e `Failures` considera sampling, agregacao de spans e o
sampling de leitura aplicado pelo Grail.

```dql
fetch spans
| filter getNodeName(dt.smartscape.host) == $Host
| filter getNodeName(dt.smartscape.process) == $Process
| filter span.kind == "server"
| filter
    rpc.framework == "java-rmi" or
    rpc.protocol == "java-rmi" or
    network.protocol.name == "java_rmi"
| fieldsAdd SamplingProbability =
    (power(2, 56) - coalesce(sampling.threshold, 0)) * power(2, -56)
| fieldsAdd Multiplicity =
    coalesce(1 / SamplingProbability, 1) *
    coalesce(aggregation.count, 1) *
    dt.system.sampling_ratio
| fieldsAdd Failed = request.is_failed == true or span.status_code == "error"
| makeTimeseries {
    Calls = sum(Multiplicity),
    Failures = sum(if(Failed, Multiplicity, else:0.0)),
    ResponseTimeAvg = avg(duration),
    ResponseTimeP95 = percentile(duration, 95),
    ResponseTimeP99 = percentile(duration, 99)
  }, bins:120
| fieldsAdd FailureRate = if(
    Calls[] > 0,
    Failures[] * 100.0 / Calls[],
    else:0.0
  )
```

> Os percentis refletem os spans lidos. Contagens e taxa de falha sao
> extrapoladas; percentis nao podem ser extrapolados da mesma forma.

### RMI: metodos mais chamados e mais lentos

Use como tabela. Ela mostra classe/servico, metodo, registry, porta, chamadas,
falhas e latencias.

```dql
fetch spans
| filter getNodeName(dt.smartscape.host) == $Host
| filter getNodeName(dt.smartscape.process) == $Process
| filter span.kind == "server"
| filter
    rpc.framework == "java-rmi" or
    rpc.protocol == "java-rmi" or
    network.protocol.name == "java_rmi"
| fieldsAdd SamplingProbability =
    (power(2, 56) - coalesce(sampling.threshold, 0)) * power(2, -56)
| fieldsAdd Multiplicity =
    coalesce(1 / SamplingProbability, 1) *
    coalesce(aggregation.count, 1) *
    dt.system.sampling_ratio
| fieldsAdd Failed = request.is_failed == true or span.status_code == "error"
| summarize {
    Calls = sum(Multiplicity),
    Failures = sum(if(Failed, Multiplicity, else:0.0)),
    ResponseTimeAvg = avg(duration),
    ResponseTimeP95 = percentile(duration, 95),
    ResponseTimeMax = max(duration)
  }, by:{
    Service = coalesce(rpc.service, code.namespace),
    Method = coalesce(rpc.method, code.function, endpoint.name),
    Registry = rpc.rmi.registry,
    ServerAddress = server.address,
    ServerPort = server.port
  }
| fieldsAdd FailureRate = if(
    Calls > 0,
    round(Failures * 100.0 / Calls, decimals:2),
    else:0.0
  )
| sort ResponseTimeP95 desc
| limit 100
```

### RMI: chamadas com falha ou mais lentas

Use como tabela de investigacao e abra `trace.id` para navegar ate o trace.
O limite de `1s` e apenas um ponto de partida e pode ser ajustado.

```dql
fetch spans
| filter getNodeName(dt.smartscape.host) == $Host
| filter getNodeName(dt.smartscape.process) == $Process
| filter span.kind == "server"
| filter
    rpc.framework == "java-rmi" or
    rpc.protocol == "java-rmi" or
    network.protocol.name == "java_rmi"
| filter request.is_failed == true or span.status_code == "error" or duration >= 1s
| fields
    start_time,
    trace.id,
    span.id,
    Failed = coalesce(request.is_failed, span.status_code == "error", false),
    ResponseTime = duration,
    Service = coalesce(rpc.service, code.namespace),
    Method = coalesce(rpc.method, code.function, endpoint.name),
    Registry = rpc.rmi.registry,
    ServerAddress = server.address,
    ServerPort = server.port,
    Status = span.status_code,
    StatusMessage = span.status_message
| sort ResponseTime desc
| limit 100
```

### RMI: dependencias de saida do processo

Esta consulta troca o ponto de vista para `span.kind == "client"` e revela os
servidores RMI chamados pelo processo selecionado.

```dql
fetch spans
| filter getNodeName(dt.smartscape.host) == $Host
| filter getNodeName(dt.smartscape.process) == $Process
| filter span.kind == "client"
| filter
    rpc.framework == "java-rmi" or
    rpc.protocol == "java-rmi" or
    network.protocol.name == "java_rmi"
| summarize {
    CallsObserved = count(),
    FailuresObserved = countIf(request.is_failed == true or span.status_code == "error"),
    ResponseTimeAvg = avg(duration),
    ResponseTimeP95 = percentile(duration, 95),
    ResponseTimeMax = max(duration)
  }, by:{
    ServerAddress = server.address,
    ServerPort = server.port,
    Registry = rpc.rmi.registry,
    Service = rpc.service,
    Method = rpc.method
  }
| sort CallsObserved desc
| limit 100
```

## 4. Indicadores de threads JVM

### Threads vivas, ativas e inativas

`LiveThreads` vem da JVM. `ActiveThreads` e `InactiveThreads` sao metricas de
profiling e podem ficar vazias quando esse recurso nao estiver disponivel.

```dql
timeseries {
    LiveThreads = avg(dt.runtime.jvm.threads.count),
    ActiveThreads = avg(dt.profiling.jvm.threads.active),
    InactiveThreads = avg(dt.profiling.jvm.threads.inactive),
    ThreadCpuTime = sum(dt.profiling.jvm.threads.cpu_time),
    ThreadExhaustionEvents = sum(dt.process.threads_exhausted)
  },
  by:{dt.smartscape.host, dt.smartscape.process},
  filter:{
    getNodeName(dt.smartscape.host) == $Host and
    getNodeName(dt.smartscape.process) == $Process
  },
  union:true
| fieldsAdd
    Host = getNodeName(dt.smartscape.host),
    Process = getNodeName(dt.smartscape.process)
| fieldsRemove dt.smartscape.host, dt.smartscape.process
```

### Resumo escalar de threads

```dql
timeseries {
    LiveThreadsAvg = avg(dt.runtime.jvm.threads.count, scalar:true),
    LiveThreadsMax = max(dt.runtime.jvm.threads.count, scalar:true),
    ActiveThreadsMax = max(dt.profiling.jvm.threads.active, scalar:true),
    InactiveThreadsMax = max(dt.profiling.jvm.threads.inactive, scalar:true),
    ThreadExhaustionEvents = sum(dt.process.threads_exhausted, scalar:true)
  },
  by:{dt.smartscape.host, dt.smartscape.process},
  filter:{
    getNodeName(dt.smartscape.host) == $Host and
    getNodeName(dt.smartscape.process) == $Process
  },
  union:true
| fields
    Host = getNodeName(dt.smartscape.host),
    Process = getNodeName(dt.smartscape.process),
    LiveThreadsAvg,
    LiveThreadsMax,
    ActiveThreadsMax,
    InactiveThreadsMax,
    ThreadExhaustionEvents
```

## 5. Indicadores de rede TCP

### Trafego TCP do processo

Bytes recebidos/enviados e throughput do processo.

```dql
timeseries {
    BytesRx = avg(dt.process.network.bytes_rx),
    BytesTx = avg(dt.process.network.bytes_tx),
    Throughput = avg(dt.process.network.throughput),
    PacketsRx = avg(dt.process.network.packets.rx),
    PacketsTx = avg(dt.process.network.packets.tx)
  },
  by:{dt.smartscape.host, dt.smartscape.process},
  filter:{
    getNodeName(dt.smartscape.host) == $Host and
    getNodeName(dt.smartscape.process) == $Process
  },
  union:true
| fieldsAdd
    Host = getNodeName(dt.smartscape.host),
    Process = getNodeName(dt.smartscape.process)
| fieldsRemove dt.smartscape.host, dt.smartscape.process
```

### Sessoes TCP novas, resets e timeouts

Resets e timeouts diferentes de zero sao sinais diretos para correlacionar com
falhas ou picos de latencia RMI.

```dql
timeseries {
    NewSessions = avg(dt.process.network.sessions.new),
    SessionResets = avg(dt.process.network.sessions.reset),
    SessionTimeouts = avg(dt.process.network.sessions.timeout),
    RetransmittedRx = avg(dt.process.network.packets.re_rx),
    RetransmittedTx = avg(dt.process.network.packets.re_tx)
  },
  by:{dt.smartscape.host, dt.smartscape.process},
  filter:{
    getNodeName(dt.smartscape.host) == $Host and
    getNodeName(dt.smartscape.process) == $Process
  },
  union:true
| fieldsAdd
    Host = getNodeName(dt.smartscape.host),
    Process = getNodeName(dt.smartscape.process)
| fieldsRemove dt.smartscape.host, dt.smartscape.process
```

### RTT, latencia de ACK e retransmissoes do NetTracer

`RoundTrip` mede o handshake TCP; `AckLatency` mede o tempo entre dados enviados
e ACK; `NetTracerRtt` e `NetTracerRetransmissions` exigem NetTracer.

```dql
timeseries {
    RoundTrip = avg(dt.process.network.round_trip),
    AckLatency = avg(dt.process.network.latency),
    NetTracerRtt = avg(dt.process.nettracer.rtt),
    NetTracerRetransmissions = sum(dt.process.nettracer.pkts_retr)
  },
  by:{dt.smartscape.host, dt.smartscape.process},
  filter:{
    getNodeName(dt.smartscape.host) == $Host and
    getNodeName(dt.smartscape.process) == $Process
  },
  union:true
| fieldsAdd
    Host = getNodeName(dt.smartscape.host),
    Process = getNodeName(dt.smartscape.process)
| fieldsRemove dt.smartscape.host, dt.smartscape.process
```

### Fluxos TCP por endereco e porta

Use como tabela para identificar pares, portas RMI, volume, RTT, resets,
timeouts e retransmissoes. Essa consulta depende do Network connection
monitoring e do bucket `default_network_flows`.

```dql
fetch events, bucket:{"default_network_flows"}
| filter getNodeName(dt.smartscape.host) == $Host
| filter getNodeName(dt.smartscape.process) == $Process
| filter matchesValue(network_flow.network.transport, "TCP")
| summarize {
    FlowRecords = count(),
    BytesRx = sum(network_flow.bytes.rx),
    BytesTx = sum(network_flow.bytes.tx),
    PacketsRx = sum(network_flow.packets.rx),
    PacketsTx = sum(network_flow.packets.tx),
    RetransmittedRx = sum(network_flow.packets.retransmitted.rx),
    RetransmittedTx = sum(network_flow.packets.retransmitted.tx),
    NewSessions = sum(network_flow.tcp.sessions.new),
    SessionResets = sum(network_flow.tcp.sessions.reset),
    SessionTimeouts = sum(network_flow.tcp.sessions.timeout),
    RttAvg = avg(network_flow.tcp.rtt),
    RttMax = max(network_flow.tcp.rtt)
  }, by:{
    Direction = network_flow.direction,
    SourceAddress = network_flow.source.address,
    DestinationAddress = network_flow.destination.address,
    DestinationPort = network_flow.destination.port
  }
| fieldsAdd TotalBytes = coalesce(BytesRx, 0) + coalesce(BytesTx, 0)
| sort TotalBytes desc
| limit 200
```

### Portas TCP mais utilizadas pelo processo

```dql
fetch events, bucket:{"default_network_flows"}
| filter getNodeName(dt.smartscape.host) == $Host
| filter getNodeName(dt.smartscape.process) == $Process
| filter matchesValue(network_flow.network.transport, "TCP")
| summarize {
    FlowRecords = count(),
    BytesRx = sum(network_flow.bytes.rx),
    BytesTx = sum(network_flow.bytes.tx),
    Resets = sum(network_flow.tcp.sessions.reset),
    Timeouts = sum(network_flow.tcp.sessions.timeout),
    RttAvg = avg(network_flow.tcp.rtt)
  }, by:{DestinationPort = network_flow.destination.port}
| fieldsAdd TotalBytes = coalesce(BytesRx, 0) + coalesce(BytesTx, 0)
| sort FlowRecords desc
| limit 100
```

## 6. Correlacao RMI x threads x TCP

Para o dashboard, coloque os graficos temporais na mesma linha e use o mesmo
timeframe. Correlacione principalmente:

- aumento de `ResponseTimeP95/P99` RMI com aumento de `LiveThreads`;
- falhas RMI com `ThreadExhaustionEvents`;
- falhas RMI com `SessionResets` ou `SessionTimeouts`;
- latencia RMI com `RoundTrip`, `AckLatency` ou `NetTracerRtt`;
- throughput alto com retransmissoes e degradacao de RTT.

## 7. Descoberta e diagnostico de disponibilidade dos dados

### Metricas disponiveis para o host e processo

Execute primeiro quando algum grafico retornar vazio. A consulta mostra apenas
as chaves realmente presentes no tenant e no timeframe selecionado.

```dql
metrics
| filter getNodeName(dt.smartscape.host) == $Host
| filter getNodeName(dt.smartscape.process) == $Process
| filter
    startsWith(metric.key, "dt.process.") or
    startsWith(metric.key, "dt.runtime.jvm.") or
    startsWith(metric.key, "dt.profiling.jvm.")
| fields metric.key
| dedup metric.key
| sort metric.key asc
```

### Confirmar os campos RMI recebidos em spans

```dql
fetch spans
| filter getNodeName(dt.smartscape.host) == $Host
| filter getNodeName(dt.smartscape.process) == $Process
| filter
    rpc.framework == "java-rmi" or
    rpc.protocol == "java-rmi" or
    network.protocol.name == "java_rmi"
| fieldsSummary
    span.kind,
    rpc.framework,
    rpc.protocol,
    network.protocol.name,
    rpc.rmi.registry,
    rpc.service,
    rpc.method,
    server.address,
    server.port
```

### Confirmar campos dos eventos de rede

```dql
fetch events, bucket:{"default_network_flows"}
| filter getNodeName(dt.smartscape.host) == $Host
| filter getNodeName(dt.smartscape.process) == $Process
| fieldsSummary
    network_flow.network.transport,
    network_flow.direction,
    network_flow.source.address,
    network_flow.destination.address,
    network_flow.destination.port,
    network_flow.tcp.rtt,
    network_flow.tcp.sessions.reset,
    network_flow.tcp.sessions.timeout
```

## Ambientes com nomes duplicados

Nomes sao mais amigaveis no seletor, mas nao sao identificadores unicos. Em um
tenant com hosts ou processos de mesmo nome, retorne `id` nas variaveis e filtre
com `toSmartscapeId()`.

### Variavel `HostId`

```dql
smartscapeNodes HOST
| fields HostId = toString(id)
| sort HostId asc
```

### Variavel dependente `ProcessId`

```dql
smartscapeNodes PROCESS
| traverse runs_on, HOST, direction:forward, fieldsKeep:id
| filter id == toSmartscapeId($HostId)
| fields ProcessId = toString(dt.traverse.history[0][id])
| sort ProcessId asc
```

Substitua os filtros por nome das consultas por:

```dql
| filter dt.smartscape.host == toSmartscapeId($HostId)
| filter dt.smartscape.process == toSmartscapeId($ProcessId)
```

Nos comandos `timeseries`, use:

```dql
filter:{
  dt.smartscape.host == toSmartscapeId($HostId) and
  dt.smartscape.process == toSmartscapeId($ProcessId)
}
```

## Referencias oficiais

- [Built-in Metrics on Grail](https://docs.dynatrace.com/docs/analyze-explore-automate/metrics/built-in-metrics-on-grail)
- [DQL metric commands](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language/commands/metric-commands)
- [DQL Smartscape commands](https://docs.dynatrace.com/docs/platform/grail/dynatrace-query-language/commands/smartscape-commands)
- [Semantic Dictionary: traces, RPC e RMI](https://docs.dynatrace.com/docs/semantic-dictionary/model/trace)
- [Dashboard variables](https://docs.dynatrace.com/docs/analyze-explore-automate/dashboards-and-notebooks/dashboards-new/components/dashboard-component-variable)
- [Extended network monitoring / NetTracer](https://docs.dynatrace.com/docs/observe/infrastructure-monitoring/networks/network-monitoring-with-nettracer)
