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
