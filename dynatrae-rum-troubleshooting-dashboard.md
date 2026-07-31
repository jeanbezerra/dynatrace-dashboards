
Crie duas variáveis de seleção única no Dashboard:

Variável frontend
- Nome: frontend
- Tipo: DQL
- Multi-select: desativado

```sh
fetch user.events, from: now() - 30d
| filter isNotNull(frontend.name)
| summarize by: {
    frontend.name
  }
| sort frontend.name asc
| fields frontend.name
```

Variável backendService
Esta versão retorna o ID exato SERVICE-..., compatível com a query principal.
- Nome: backendService
- Tipo: DQL
- Multi-select: desativado

```sh
timeseries request_count = sum(dt.service.request.count),
  by: {
    dt.entity.service
  },
  from: now() - 30d

| fields backendService = toString(dt.entity.service)

| sort backendService asc
```

```sh
timeseries backend_requests = sum(dt.service.request.count, default: 0),
  filter: {
    dt.entity.service == $backend_service
  },
  interval: 5m,
  nonempty: true
| join [
    timeseries {
      sessions = sum(
        dt.frontend.session.active.estimated_count,
        default: 0
      ),

      actions = sum(
        dt.frontend.user_action.count,
        default: 0
      ),

      browser_requests = sum(
        dt.frontend.request.count,
        default: 0
      )
    },
    filter: {
      frontend.name == $frontend
    },
    interval: 5m,
    nonempty: true
  ],
  kind: leftOuter,
  on: { timeframe },
  prefix: "rum."

| fieldsAdd collection_status = iCollectArray(
    if(
      backend_requests[] == 0,
      0,

      else: if(
        rum.sessions[] == 0
        and rum.actions[] == 0
        and rum.browser_requests[] == 0,
        3,

        else: if(
          rum.actions[] == 0
          or rum.browser_requests[] == 0,
          2,

          else: 1
        )
      )
    )
  )

| fieldsAdd rum_request_index = iCollectArray(
    if(
      backend_requests[] > 0,
      100.0 * rum.browser_requests[] / backend_requests[],
      else: null
    )
  )

| fields
    timeframe,
    interval,
    backend_requests,
    rum.sessions,
    rum.actions,
    rum.browser_requests,
    rum_request_index,
    collection_status
```


```sh
fetch user.events

| filter frontend.name == $frontend

| makeTimeseries {
    total_events = count(default: 0),

    sessions = countDistinctApprox(
      dt.rum.session.id,
      default: 0
    ),

    actions = countIf(
      characteristics.has_user_action == true,
      default: 0
    ),

    interactions = countIf(
      characteristics.has_user_interaction == true,
      default: 0
    ),

    navigations = countIf(
      characteristics.has_navigation == true,
      default: 0
    ),

    request_events = countIf(
      characteristics.has_request == true,
      default: 0
    ),

    csp_violations = countIf(
      characteristics.has_csp_violation == true,
      default: 0
    )
  },
  interval: 5m,
  nonempty: true
```
