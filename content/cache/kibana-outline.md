+++
title = 'Kibana Outline'
date = 2022-04-08T12:00:00-07:00
draft = false
type = 'post'
showTableOfContents = true
tags = [ 'elastic', 'kibana', 'jq' ]
+++

> This Kibana outline is purposely narrowed to topics/APIs _most_ relevant to performance/outage 
> troubleshooting. 

---

[doc](https://www.elastic.co/guide/en/kibana/current/index.html)
, [api](https://www.elastic.co/guide/en/kibana/current/api.html)
, [code](https://github.com/elastic/kibana)
, [diagnostic](https://github.com/elastic/support-diagnostics#usage-examples) ([code](https://github.com/elastic/support-diagnostics/blob/main/src/main/resources/kibana-rest.yml))
, [logs](https://www.elastic.co/guide/en/kibana/current/logging-settings.html) ([configure](https://www.elastic.co/guide/en/kibana/current/logging-configuration.html), [examples](https://www.elastic.co/guide/en/kibana/current/log-settings-examples.html)) 
, [Event Log](https://www.elastic.co/guide/en/kibana/current/event-log-index.html) ([code](https://github.com/elastic/kibana/tree/main/x-pack/plugins/event_log), `.kibana-event-log-*`)



**Problem Box**
- [doc](https://www.elastic.co/guide/en/kibana/current/logging-configuration.html): debug logs ( `logging.verbose:true` in [7.x](https://github.com/elastic/kibana/blob/7.16/config/kibana.yml#L90-L107) ;  `logging.root.level:debug` in [8.x](https://github.com/elastic/kibana/blob/main/config/kibana.yml#L97-L121) )
- [support-diagnostics#578](https://github.com/elastic/support-diagnostics/issues/578#issuecomment-1501198281): diagnostic only pulls `space:default` 
- [blog](https://www.elastic.co/blog/troubleshooting-kibana-health): health, performance issues
    > Most features run from the Elasticsearch service; however a handful of features run from Kibana: [Telemetry](https://www.elastic.co/guide/en/kibana/current/telemetry-settings-kbn.html), [Rules](https://www.elastic.co/guide/en/kibana/current/alerting-getting-started.html) (w/associated [Actions](https://www.elastic.co/guide/en/kibana/current/action-types.html)), [Reporting](https://www.elastic.co/guide/en/kibana/current/reporting-getting-started.html), & [Sessions](https://www.elastic.co/guide/en/kibana/current/xpack-security-session-management.html). 
    > 
    > Most of these only require metadata to process so are discoverable via searching `.kibana*`. However, Reporting and Sessions contain PII so are stored separately (respectively `.reporting*` & `.kibana_security_session*`). These latter indices are redacted from Elastic's view in [ESS](https://www.elastic.co/cloud/) and all mentioned [system indices](https://www.elastic.co/guide/en/elasticsearch/reference/current/api-conventions.html#system-indices) are redacted from [Diagnostic](https://github.com/elastic/support-diagnostics#usage-examples) view. 
    > 
    > [Task Manager doc](https://www.elastic.co/guide/en/kibana/current/task-manager-production-considerations.html) Kibana Task Manager is leveraged by features such as [Rules](app://obsidian.md/elastic%20kibana%20rule), [Actions](app://obsidian.md/elastic%20kibana%20connector%20action), and [Reporting](app://obsidian.md/elastic%20kibana%20reporting) to run mission critical work as persistent background tasks. These background tasks distribute work across multiple Kibana instances.


[Spaces](https://www.elastic.co/guide/en/kibana/current/xpack-spaces.html) ([api](https://www.elastic.co/guide/en/kibana/current/spaces-api.html))
, [Discover](https://www.elastic.co/guide/en/kibana/current/discover.html) ([resolve](https://www.elastic.co/blog/troubleshooting-guide-common-issues-kibana-discover-load))
, [Dev Tools](https://www.elastic.co/guide/en/kibana/current/console-kibana.html)
, [Stack Monitoring](https://www.elastic.co/guide/en/kibana/current/xpack-monitoring.html) ([alerts](https://www.elastic.co/guide/en/kibana/current/kibana-alerts.html))
, [Stack Management](https://www.elastic.co/guide/en/kibana/current/management.html) ([Index Management](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-mgmt.html), [Advanced Settings](https://www.elastic.co/guide/en/kibana/current/advanced-options.html))
, [Task Manager Health](https://www.elastic.co/guide/en/kibana/current/task-manager-api-health.html) ([resolve](https://www.elastic.co/guide/en/kibana/current/task-manager-troubleshooting.html))

```bash
# internal, undocumented
> GET KIBANA_URL/status
$ cat kibana_status.json | jq -rc '.status|{elasticsearch:.core.elasticsearch.level, savedObjects:.core.savedObjects.level, plugins:.overall.level}'

# task manager
> GET kbn:api/task_manager/_health
$ cat kibana_task_manager_health.json | jq -rc '{overall:.status, capacity:.stats.capacity_estimation.status, configuration:.stats.configuration.status, runtime:.stats.runtime.status, workload:.stats.workload.status}'

# spaces
> GET .kibana/_search?q=type:space
> GET .kibana/_search?q=space.name.keyword:*

# non-default advanced settings
> GET kbn:/api/kibana/settings
> GET .kibana/_search?q=type:config&sort=_index:desc
```


# Setup

[doc](https://www.elastic.co/guide/en/kibana/current/setup.html)

**Problem Box**
- [doc](https://www.elastic.co/guide/en/kibana/current/access.html#not-ready): `Kibana Server is not Ready yet`
- `Unable to retrieve version information from Elasticsearch nodes.` either a) need `ELASTICSEARCH_HOSTS`, b) wrong username/password
- `server.publicBaseUrl is missing and should be configured when running in a production environment. Some features may not behave correctly. See the documentation.` v7.14.0 [kibana#109970](https://github.com/elastic/kibana/issues/109970) 

# Session

[doc](https://www.elastic.co/guide/en/kibana/current/xpack-security-session-management.html)

**Problem Box**
- [resolve](https://discuss.elastic.co/t/kibana-session-issues/289299/10), [kibana#83914](https://github.com/elastic/kibana/issues/83914): Auth troubleshooting guide 
- Login erring 500
    - `.kibana*` indices unhealthy
    - ESS/ECE > Kibana built-in `role_mapping` disconnected
- Login erring 403
    - Traffic Filters & user IP doesn’t qualify 
    - username does not have sufficient permissions
    - when SSO realm login errors HTTP 403, need to sequentially check
      1. Pull log of SSO authentication + redirect to Kibana via SAML Tracer ([Chrome](https://chrome.google.com/webstore/detail/saml-tracer/mpdajninpobndbfcldcmbpnnbhibjmch), [Firefox](https://addons.mozilla.org/en-US/firefox/addon/saml-tracer/)) (works regardless of SSO realm, alt [Wireshark](https://www.wireshark.org/) if you want to dig). 
      2. Update Kibana URL to `KIBANA_URL/internal/security/me` to check metadata returned by IDP, `role_mapping` line-up, and therefore granted `roles`. 
- Login erring 401
    - Invalid username/password

# Reports
[doc](https://www.elastic.co/guide/en/kibana/current/reporting-getting-started.html) , [diagnostic](https://www.elastic.co/guide/en/kibana/current/reporting-troubleshooting.html#reporting-diagnostics)

```bash
> GET .reporting*/_search?q=!status:completed
```

**Problem Box**
- `Reporting task runner has not been initialized!` (Fix Task Manager Health then) Once `KIBANA_URL/api/status` claims reporting is unhealthy but task manager is healthy, then kindly navigate Kibana > Stack Management > Reporting and run the [Diagnostic](https://www.elastic.co/guide/en/kibana/current/reporting-troubleshooting.html#reporting-diagnostics). If insufficient, enable debug logging on `log.logger:plugins.reporting`. 
- CSV
    - 7.5.0≤v≤7.13.0 `Expected _scroll_id in the following Elasticsearch response` CSV export not supported against Frozen. Resolved [kibana#88303](https://github.com/elastic/kibana/pull/88303). 
    - `token expired` / `Cannot read properties of undefined (reading 'convert')` short-term increase xpack.security.authc.token.timeout, long-term [kibana#71866](https://github.com/elastic/kibana/issues/71866) 
    - `Array buffer allocation failed` Upgrade ≥v7.15.0 for chunked export via [kibana#108485](https://github.com/elastic/kibana/pull/108485)
    - timeouts 8.6.0≤v<8.8.2 for throttled PIT requests, [elasticsearch#96775](https://github.com/elastic/elasticsearch/issues/96775)

# Saved Objects
[doc](https://www.elastic.co/guide/en/kibana/current/managing-saved-objects.html) ([export/import](https://www.elastic.co/guide/en/kibana/current/managing-saved-objects.html#managing-saved-objects-export-objects)), ([service doc](https://www.elastic.co/guide/en/kibana/8.0/saved-objects-service.html)), [api](https://www.elastic.co/guide/en/kibana/current/saved-objects-api.html), [version compatibilities](https://www.elastic.co/guide/en/kibana/current/managing-saved-objects.html#_compatibility_across_versions) 

comments
- I believe everything BUT spaces
- v8.8.0 `.kibana` split into sub-indices

**Problem Box**
- Drilldown weak links, embedded imports break [kibana#71409](https://github.com/elastic/kibana/issues/71409), [kibana#123550](https://github.com/elastic/kibana/issues/123550) 
    - Lens Edit save issues [kibana#147646](https://github.com/elastic/kibana/issues/147646) 
    - Not imported when reference another Dashboard [kibana#170340](https://github.com/elastic/kibana/issues/170340) 
    - Invalid Drilldown hidden for table row [kibana#163642](https://github.com/elastic/kibana/issues/163642) 

## Actions
[doc](https://www.elastic.co/guide/en/kibana/current/action-types.html), [api](https://www.elastic.co/guide/en/kibana/current/actions-and-connectors-api.html) (aka. Connectors)

```bash
> GET .kibana/_search?q=type:action&size=20
$ cat actions.json | jq ['.hits.hits[]|._source.action as $a|{id:._id,space:._source.namespaces, type: $a.actionTypeId, name: $a.name}']
```

## Dashboards
[doc](https://www.elastic.co/guide/en/kibana/current/dashboard.html), [api](https://www.elastic.co/guide/en/kibana/current/dashboard-api.html)

```bash
> GET .kibana/_search?size=100&q=type:dashboard
# search
> GET .kibana/_search 
{"query": { "bool": { "must" : { "query_string": { "analyze_wildcard": true, "query": "type:\"dashboard\" AND \"DASHBOARD_NAME_SEARCH_HERE\"" }}}},"size": 100}
```

## Visualizations

[doc](https://www.elastic.co/guide/en/kibana/current/aggregation-reference.html)

```bash
> GET .kibana/_search?size=100&q=type:visualization&filter_path=hits.hits._id,hits.hits._source.visualization.title,hits.hits._source.references,hits.hits._source.updated_at,hits.hits._source.namespace
# search
> GET .kibana/_search 
> {"query": { "bool": { "must" : { "query_string": { "analyze_wildcard": True, "query": f"type:\"visualization\" AND \"VISUALIZATION_SEARCH_HERE\"" }}}},"size": 100}
```

## Data Views
[doc](https://www.elastic.co/guide/en/kibana/current/data-views.html), [api](https://www.elastic.co/guide/en/kibana/current/data-views-api.html) (aka "Index Patterns")

```bash
> GET .kibana/_search?size=100&q=type:index-pattern

# search
> GET .kibana/_search 
{"query": { "bool": { "must" : { "query_string": { "analyze_wildcard": True, "query": f"type:\"index-pattern\" AND \"INDEX_PATTERN_NAME_SEARCH\"" }}}},"size": 100}
```

## Rules
[doc](https://www.elastic.co/guide/en/kibana/current/create-and-manage-rules.html), [api](https://www.elastic.co/guide/en/kibana/current/alerting-apis.html), [detections](https://www.elastic.co/guide/en/security/current/rules-ui-management.html) ([api](https://www.elastic.co/guide/en/security/current/rule-api-overview.html), [prerequistes/permissions](https://www.elastic.co/guide/en/security/current/detections-permissions-section.html))

```bash
# Alerting-level
> GET .kibana*/_search?q=type:alert&size=100
## enabled
> GET .kibana*/_count?q=alert.enabled:true&filter_path=count
> GET .kibana*/_search?q=alert.enabled:true&size=1000
## enabled.errors
> GET .kibana*/_search?q=alert:*%20AND%20alert.executionStatus.status:error%20AND%20alert.enabled:true
## search
> GET .kibana*/_search?sort=updated_at  
{"query": { "bool": { "must" : { "query_string": { "analyze_wildcard": true,"query": "type:alert AND alert.enabled:true AND RULE_NAME_WORDS_HERE" }}}},"size": 100}

# count
$ cat rules.json | jq '.hits.total.value'
# errors
$ cat rules.json | jq '.hits.hits[]|._source.alert.executionStatus.status as $s|select($s!="ok" and $s!="active")'
# summary table
$ cat rules.json | jq -rc '.hits.hits[]._source.alert |[ .enabled, .schedule.interval, .params.from, .params.to, (.params.index[]? // "NULL"), .legacyId, .params.ruleId, .name ]'
# rules.scheduledIntervals
$ cat rules.json | jq '.hits.hits[]|._source.alert.schedule.interval' | sort | uniq -c
# rules.createdAt
$ cat rules.json | jq -rc '.hits.hits[]|._source.alert as $a|$a.createdAt[:10]' | sort | uniq -c

# SIEM-level, https://github.com/elastic/support-diagnostics/issues/578#issuecomment-1501198281
## >8.x
> GET .kibana*/_search?q=type:"siem-detection-engine-rule-execution-info"&size=2000
## ≤8.0
> GET .kibana*/_search?q=type:"siem-detection-engine-rule-status"&size=2000
```

**Problem Box**
- [doc](https://www.elastic.co/guide/en/kibana/current/alerting-setup.html#alerting-authorization): permissions of last editing user
- Rules searching Frozen
    - Rule w/low `params.from` inducing `partial-*` searches seen via CAT Tasks. If confirmed, 1) [Disable Rule](https://www.elastic.co/guide/en/kibana/current/disable-rule-api.html) , 2) fix Frozen dates by either a) wait for these Frozen indices to ILM roll off / [delete](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-delete-index.html) , b) [Reindex](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html) to non-Frozen then [Update by Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-update-by-query.html) or [Delete by Query](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-delete-by-query.html) to remove the incorrect dates.
        ```bash
        ## symptoms
        > GET _cat/thread_pool/search?v=true&s=n,nn&h=n,nn,q,a,r,c # search queue on frozen nodes
        > GET _tasks?pretty&human&detailed=true&actions=indices:data/read/search

        ## confirm, increasing expensiveness
        # only available later versions, I think ≥v8.3
        > GET _cluster/state?filter_path=metadata.indices.*.timestamp_range
        $ cat cluster_state.json | jq -cMr '.metadata.indices| to_entries| sort_by(.key)| .[]| .value.timestamp_range as $ts| select($ts.max)| select(($ts.max/1000.0)>now)| {min:($ts.min/1000.0|todate), max:($ts.max/1000.0|todate), index:.key}'
        # 👻 look for future docs, may be expensive
        > GET */_search
        { "size": 0, "aggs": { "0": {"terms": {"field": "_index", "order": { "_count": "desc" }, "size": 200 }} }, "query": {"bool": {"must": [], "filter": [ { "range": { "@timestamp": { "gte": "now" }}}] } } }
        # 👻 👻 time ranges of all indices, will likely be expensive
        > GET */_search?filter_path=aggregations
        { "size": 0, "aggs": { "2": {"aggs": {"first_time": {"min": {"field": "@timestamp"} }, "last_time": {"max": {"field": "@timestamp"} } }, "terms": {"field": "_index", "order": {"_key": "asc"}, "size": 500 } }} }
        ```

- [blog](https://www.elastic.co/blog/troubleshooting-kibana-health), [doc](https://www.elastic.co/guide/en/kibana/current/alerting-common-issues.html#rules-long-execution-time): duration, expensive
    
    Kibana [SIEM](https://www.elastic.co/guide/en/security/current/es-overview.html) (aka. Security "Detections Engine") is a specific type of the [Rule](https://www.elastic.co/guide/en/kibana/current/alerting-getting-started.html) task which is a sub-set of Kibana [Tasks](https://www.elastic.co/guide/en/kibana/current/task-manager-production-considerations.html) ([PDF](https://github.com/stefnestor/elastic/blob/main/Kibana/Security/security_cheatsheet.pdf)). Kibana Task processing stores [Event Logs](https://github.com/elastic/kibana/tree/main/x-pack/plugins/event_log) into space-agnostic .kibana-event-log* (and by association (SIEM) Rule processing info). Therefore, you can use [this query](https://www.elastic.co/guide/en/kibana/current/alerting-common-issues.html#rules-long-execution-time) to view top expensive Rules by `_source.alert.legacyID`.
    ```bash
    # event.duration in nanoseconds
    > GET .kibana-event-log-*/_search
    {"aggs": {"rule_id": {"aggs": {"avg": {"avg": {"field": "event.duration"} }, "rule_name": {"terms": {"field": "rule.name", "size": 1 } } }, "terms": {"field": "rule.id", "order": {"avg": "desc"}, 
      "size": 20 } } }, "query": {"bool": {"filter": [{"range": {"@timestamp": {
      "gte": "now-24h", "lte": "now"} } } ], "must":[{"query_string": {"analyze_wildcard": true, "query": "event.provider:alerting AND event.action:execute"} } ] } }, "size": 0 }
    
    $ cat k/rules_most_expensive.json | jq -rc '.aggregations.rule_id.buckets[]|[.doc_count, (.avg.value*1e-9|round), .key, .rule_name.buckets[].key]'
    
    > GET /.kibana-event-log*/_search
    {"size": 0, "query": {"bool": {"filter": [{"range": {"@timestamp": {"gte": "now-1d", "lte": "now"} } }, {"term": {"event.action": {"value": "execute"} } }, {"term": {"event.provider": {"value": "alerting"} } } ] } }, "runtime_mappings": {"event.duration_in_seconds": {"type": "double", "script": {"source": "emit(doc['event.duration'].value / 1E9)"} } }, "aggs": {"ruleIdsByExecutionDuration": {"histogram": {"field": "event.duration_in_seconds", "min_doc_count": 1, "interval": 1 }, "aggs": {"ruleId": {"nested": {"path": "kibana.saved_objects"}, "aggs": {"ruleId": {"terms": {"field": "kibana.saved_objects.id", "size": 10 } } } } } } } }
    ```
- [kibana#117471](https://github.com/elastic/kibana/issues/117471): backfill not possible (will attempt to cover missed windows but not erring windows)
