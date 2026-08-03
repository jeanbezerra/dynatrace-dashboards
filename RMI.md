## Variable Environment

```dql
smartscapeNodes HOST
| filter isNotNull(primary_tags.environment)
| fields Environment = primary_tags.environment
| dedup Environment
| sort Environment asc
```

## Variable Host

```dql
smartscapeNodes HOST
| filter primary_tags.environment == $Environment
| fields Host = name
| dedup Host
| sort Host asc
```

## Variable WASProcess

```dql
smartscapeNodes PROCESS
| filter primary_tags.environment == $Environment
| filter process.software_technologies.os ~ "WEBSPHERE"
| traverse runs_on, HOST, fieldsKeep:name
| filter name == $Host
| fields WASProcess = dt.traverse.history[0][name]
| dedup WASProcess
| sort WASProcess asc
```

## Variable RmiPort

```dql
fetch events, bucket:{"default_network_flows"}
| filter primary_tags.environment == $Environment
| filter getNodeName(dt.smartscape.host) == $Host
| filter getNodeName(dt.smartscape.process) == $WASProcess
| filter network_flow.network.transport == "TCP"
| summarize FlowRecords = count(),
    by:{network_flow.destination.port}
| sort FlowRecords desc
| fields RmiPort = network_flow.destination.port
```

## Variable RemoteIp

```dql
fetch events, bucket:{"default_network_flows"}
| filter primary_tags.environment == $Environment
| filter getNodeName(dt.smartscape.host) == $Host
| filter getNodeName(dt.smartscape.process) == $WASProcess
| filter network_flow.network.transport == "TCP"
| filter network_flow.destination.port == $RmiPort:noquote
| fieldsAdd RemoteIp = if(
    network_flow.process_is_server == true,
    toString(network_flow.source.address),
    else: toString(network_flow.destination.address)
  )
| fields RemoteIp
| dedup RemoteIp
| sort RemoteIp asc
```

