+++
title = 'Elasticsearch Outline'
date = 2022-04-03T12:00:00-07:00
draft = false
type = 'post'
showTableOfContents = true
tags = [ 'elastic', 'elasticsearch', 'jq' ]
+++


> This Elasticsearch outline is purposely narrowed to topics/APIs _most_ relevant to performance/outage 
> troubleshooting. 
> 
> It correlates to [USE Method for Elasticsearch](https://medium.com/@stefnestor/use-method-for-elasticsearch-d976802d8ba6). See [related PDF](/use_method_elasticsearch.pdf).

---

[doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
, [api](https://www.elastic.co/guide/en/elasticsearch/reference/current/rest-apis.html)
, [code](https://github.com/elastic/elasticsearch/)
, [diagnostic](https://github.com/elastic/support-diagnostics#usage-examples) ([code](https://github.com/elastic/support-diagnostics/blob/main/src/main/resources/elastic-rest.yml))
, [logs](https://www.elastic.co/guide/en/elasticsearch/reference/current/logging.html)
([loggers](https://gist.github.com/stefnestor/c508c23f305258723d49b915d684456d), [audit](https://www.elastic.co/guide/en/elasticsearch/reference/current/audit-event-types.html), [slow logs](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-slowlog.html))

**Problem Box**
- diagnostic errors
  - `Error: Could not find or load main class com.elastic.support.diagnostics.DiagnosticApp` downloaded source instead of zip from releases
  - `Could not retrieve the Elasticsearch version due to a system or network error - unable to continue.` Is generated during the first operation of the diagnostic where it determines what calls to run by first checking what version of Elasticsearch is the target. If this fails it is usually due to incorrect information supplied in the parameter list. Such as an invalid host, [doc](https://github.com/elastic/support-diagnostics/issues/389#issuecomment-605308128)
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/logging.html#configuring-logging-levels): debug logs ( `logger.org.elasticsearch: DEBUG` )


# Nodes
[doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html)
( [node roles](https://medium.com/@stefnestor/elasticsearch-node-roles-d2c7ad08b4f8), [data tiers](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-tiers.html) ([resource usage per](https://www.elastic.co/pdf/elasticsearch-sizing-and-capacity-planning.pdf)), [attributes](https://www.elastic.co/guide/en/elasticsearch/reference/current/shard-allocation-filtering.html) )


[CAT Nodes](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-nodes.html) (`cat_nodes.txt`), [Hot Threads](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-nodes-hot-threads.html) (`node_hot_threads.txt`; [Usage](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-nodes-usage.html) (`nodes_usage.json`), [Stats](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-nodes-stats.html) (`nodes_stats.json`), [Info](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-nodes-info.html) (`nodes.json`), [CAT NodeAttrs](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-nodeattrs.html) (`cat_nodeattrs.txt`)

```bash
# not container level, will report host’s stats
> GET _cat/nodes?v&s=master,name&h=name,id,master,node.role,heap.percent,disk.used_percent,cpu,uptime
$ jq '.nodes|input.master_node as $master|input.nodes as $ns|to_entries[]|{id:.key, ip:.value.ip, host:.value.host, name:.value.name, roles:.value.roles, is_master:(if .key == $master then true else false end), disk_avail:$ns[.key].fs.total.free, heap_percent:$ns[.key].jvm.mem.heap_used_percent, cpu:$ns[.key].os.cpu.percent, loads:$ns[.key].os.cpu.load_average}' nodes.json cluster_state.json nodes_stats.json

> GET _nodes
# node version
$ cat nodes.json | jq '[.nodes[]|{id:.name,v:.version}]'
# max heap
$ cat nodes.json | jq -r '.nodes|to_entries[]|[.value.jvm.mem.heap_max, .key[:4]]|@tsv' | column -t | sort

> GET _nodes/hot_threads # default only shows active threads  
# sum by thread pool stats
$ cat nodes_hot_threads.txt | grep -Ei ':::|cpu|%' | grep -v '0.0%' | grep -Eo '\]\[+[^,]*\]\[T' | sed 's/\]\[T//g' | sed 's/\]\[//g' | stats
# simplified
$ cat nodes_hot_threads.txt | grep -Ei ':::|cpu|%' | grep -v '0.0%'

> GET _nodes/usage
# node uptime
$ cat nodes_usage.json | jq -cr '.nodes|to_entries[]|[(.value.since|tostring|.[:10]|tonumber|todate), .key[:4]]|@tsv' | column -t | sort -r

> GET _cat/nodeattrs
```

**Problem Box**
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/8.11/_onerror_and_onoutofmemoryerror_checks.html): OOM, `OutOfMemoryError` 
  - [elasticsearch#14065](https://github.com/elastic/elasticsearch/issues/14065#issuecomment-148734756): guarantee node restarted after OOM
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/hotspotting.html): hot spot, hotspotting - when nodes not balanced (by resource usage, available resources, or throughput) 
    - [compare two ES diagnostic's ingest node*index](https://gist.github.com/octavioranieri/e75cf7252928a321a418a528ed506c22)
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cluster.html#disk-based-shard-allocation) ([resolve](https://www.elastic.co/guide/en/elasticsearch/reference/current/fix-watermark-errors.html)): watermark, full disk, disk full
  ```bash
  > GET _cluster/settings?include_defaults=true&filter_path=*.cluster.routing.allocation.disk.watermark
  ```
  - (default) levels : effect
    - 75%: UI warns but no system action
    - 85% `low`: stop allocating more shards
    - 90% `high`: moves shards off node
    - 95% `flood-stage`: sets indices to read-only (auto-unsets after node recovers off `high`)
  - if suspect "auto-unset" not take effect, run once (usually comes up for `.kibana` indices; if system re-applies, root problem not yet resolved):
    ```bash
    > PUT _all/_settings
    { "index.blocks.read_only_allow_delete": null }
    ```
- IOStat, IOWait, I/O, Input/Output
  - "I/O blocking" or "IO Wait" is when CPU needs to write to (sync) disk and is caught waiting on write acknowledgement from data storage.
    - high CPU w/o explanation
    - use iostat tool (checking for persistently high avgqu-sz values) OR `bash linux sar`
    - high flush duration almost always indicates disk problems
- [doc](https://medium.com/@stefnestor/elasticsearch-ingest-rejections-ce97e2e9da00): Ingest Protections/Rejections, HTTP 429
  - Coordinating Rejections
    ```bash
    > GET _nodes/stats?filter_path=*.*.name,*.*.indexing_pressure.memory.total.coordinating_rejections
    $ cat nodes_stats.json | jq -c ['.nodes|to_entries[]|{node_name:.key,coordinating_rejections: .value.indexing_pressure.memory.total.coordinating_rejections}|select(.coordinating_rejections>0)']
    ```
  - [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker.html) ([resolve](https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker-errors.html)), [blog](https://www.elastic.co/blog/improving-node-resiliency-with-the-real-memory-circuit-breaker): Circuit Breakers, `Data too large`
    ```bash
    $ cat *.log | grep -i 'CircuitBreakingException'
    > GET _nodes/stats?filter_path=nodes.*.breakers
    $ cat nodes_stats.json | jq -c '.nodes[]|.name as $node|.breakers|to_entries[]|{node:$node, circuitBreaker:.key, tripped_count:.value.tripped}|select(.tripped_count>0)'
    ```
    - types
      - [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker.html#accounting-circuit-breaker): accounting 
        - used for lucene elastic elasticsearch index shard segment (and overhead, `GET _nodes/stats/indices?filter_path=**.segments` )
      - [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker.html#fielddata-circuit-breaker): fielddata used for global ordinals, fielddata
      - [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker.html#parent-circuit-breaker): parent
      - [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/eql.html#eql-circuit-breaker): eql_sequence
      - [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker.html#request-circuit-breaker): request
        - used for request bodies (e.g. Bulk API) 
      - [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker.html#circuit-breakers-page-model-inference): model_inference
      - [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker.html#in-flight-circuit-breaker): inflight_requests
        - vs request: [elasticsearch#27116](https://github.com/elastic/elasticsearch/pull/27116) (used for the network bytes which are being serialized) 
      - [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker.html#script-compilation-circuit-breaker): script
      - [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker.html#regex-circuit-breaker): regex


## Install/Upgrade
[doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/install-elasticsearch.html) ( [plugins](https://www.elastic.co/guide/en/elasticsearch/plugins/current/intro.html) ([list](https://www.elastic.co/guide/en/elasticsearch/plugins/current/listing-removing-updating.html)), [monitoring](https://www.elastic.co/guide/en/elasticsearch/reference/current/monitoring-overview.html) )

[CAT Plugins](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-plugins.html) (`plugins.json`)
```bash
> GET _cat/plugins?v&h=c,t,n,v,d&s=c,t
$ cat nodes.json | jq -c '.nodes[].plugins[]|{name:.name, version:.version, classname:.classname, description:.description}'
```

**Problem Box**

- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/high-availability.html) ([alt.ESS](https://www.elastic.co/guide/en/cloud/current/ec-planning.html)): high availablility, highly available, HA 
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-node.html#_node_data_path_settings): no security/vulnerability scanners
- [doc](https://www.elastic.co/guide/en/elasticsearch/guide/current/heap-sizing.html#compressed_oops): not >32GB ram
- [doc](https://www.elastic.co/guide/en/cloud/current/ec-configure-deployment-settings.html#ec_version): can’t downgrade; would need to snapshot-restore to reset to previous version
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup.html#dedicated-host), [alt](https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html): dedicated host
- [doc](https://www.elastic.co/blog/performance-considerations-elasticsearch-indexing#storage-devices) ([doc ≤v8.1](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/tune-for-indexing-speed.html#_use_faster_hardware); [elasticsearch#85060](https://github.com/elastic/elasticsearch/pull/85060) > [doc ≥8.2](https://www.elastic.co/guide/en/elasticsearch/reference/8.3/tune-for-indexing-speed.html#_use_faster_hardware)): storage devices (SSDs, NOT NFS/SMB/CIFS, beware EBS)
  ```
  $ df -hT
  $ mount
  ```
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-configuration-memory.html) swap, swapping

  > [A] Confirm taking effect :: If using `bootstrap.memory_lock:true `, confirm
  > 
  > ```bash
  > $ cat nodes.json | jq -r '.nodes|to_entries[]|[.value.settings.bootstrap.memory_lock, .value.process.mlockall, .key[:4]]|@tsv' | column -t
  > > GET _nodes?filter_path=**.mlockall,**.memory_lock
  > ```
  > 
  > Any method final effect
  > 
  > ```bash
  > > GET _nodes/stats?filter_path=**.swap
  > $ cat nodes_stats.json | jq -rc '.nodes[]|[.os.swap.used, .name]|@tsv' | column -t | sort -r
  > $ cat nodes_stats.json | jq -r '.nodes|to_entries[]|[.value.os.swap.used,.value.os.swap.total,.key[:4]]|@tsv' | column -t
  > ```
- sizing: [blog](https://www.elastic.co/blog/how-to-design-your-elasticsearch-data-storage-architecture-for-scale) ([partitioning](https://www.elastic.co/blog/found-sizing-elasticsearch)), [webinar](https://community.elastic.co/events/details/elastic-emea-virtual-presents-sizing-the-elastic-stack-for-security-use-cases/#/), [kibana](https://www.elastic.co/guide/en/kibana/current/task-manager-production-considerations.html#task-manager-scaling-guidance)

  > Total data GB  raw data GB/day) *# days retained* # (replicas  1
  > Total storage GB  total data GB * 1  0.15 disk watermark threshold  0.1 margin of error)
  > Total data nodes  ROUNDUP total storage GB / memory per data node GB / Memory:Data ratio)

  > JVM_FROM_DISK
  > 
  > ```
  > hot_nodes = X * Y * Z / A / B
  > where
  > - X: gb raw daily ingested size
  > - Y: days retained in hot nodes
  > - Z: number_of_replicas
  > - A: ram-to-disk; recommended = 30 - B: ram per host
  > ```
  > 
  > CPU_FROM_DISK
  > 
  > ```
  > hot_nodes = X / C / B
  > where
  > - X: gb raw daily ingested size - C: ingest per gb/ram; defaults:
  > - logging: 2 - B: ram per host
  > ```

## Snapshot Repository

[doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-register-repository.html) ( [how works](https://www.elastic.co/blog/how-do-incremental-snapshots-work), [restore](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-restore.html), ([ES tutorial](https://www.elastic.co/guide/en/elasticsearch/reference/master/getting-started-snapshot-lifecycle-management.html), [KB tutorial](https://www.elastic.co/guide/en/kibana/current/snapshot-restore-tutorial.html)), [versionCompatibility](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-restore.html#snapshot-restore-version-compatibility) )
- hosts (with default keystore):
  - [FileSystem](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshots-register-repository.html#snapshots-filesystem-repository) (FS)
  - [Azure](https://www.elastic.co/guide/en/cloud/current/ec-azure-snapshotting.html)  (  `azure.client.secondary.account`, `azure.client.secondary.key` )
  - [GCP](https://www.elastic.co/guide/en/cloud/current/ec-gcs-snapshotting.html) ( `gcs.client.default.credentials_file ` )
  - [AWS S3](https://www.elastic.co/guide/en/cloud/current/ec-aws-custom-repository.html), [doc](https://medium.com/@stefnestor/elasticsearch-snapshot-repositories-in-aws-s3-8e6e853eb6c3) ( `s3.client.default.access_key`, `s3.client.default.secret_key` )
      ```
      PUT _cluster/settings 
      {"transient": {"logger.org.elasticsearch.repositories.s3": "TRACE", "logger.com.amazonaws" : "DEBUG"}}
      ```
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/create-snapshot-api.html): `include_global_state`
  - 👍 : persistent cluster settings, (legacy) index templates, ingest pipelines, ILM policies, stored scripts, feature states (≥7.12)
  - 👎 : transient cluster settings, snapshot repositories, node configuration files (e.g. metadata XML, plugins, security), index settings/mappings
  - 🤷‍♀️? : license, search template, script, rollup job, cluster remote info, autoscaling, persistent tasks


[CAT Repositories](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-repositories.html) (`cat_repositories.txt`), [Read Repository](https://www.elastic.co/guide/en/elasticsearch/reference/current/get-snapshot-repo-api.html), [Put Repository](https://www.elastic.co/guide/en/elasticsearch/reference/current/put-snapshot-repo-api.html), [Read Metering](https://www.elastic.co/guide/en/elasticsearch/reference/current/get-repositories-metering-api.html), [Verify](https://www.elastic.co/guide/en/elasticsearch/reference/current/verify-snapshot-repo-api.html), [CAT Snapshot](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-snapshots.html), [Read Snapshot](https://www.elastic.co/guide/en/elasticsearch/reference/current/get-snapshot-api.html), [Restore](https://www.elastic.co/guide/en/elasticsearch/reference/current/restore-snapshot-api.html)

```bash
# repositories
> GET _cat/repositories?v&s=t
> GET _snapshot/*
## verify
> POST _snapshot/my_repository/_verify

# snapshots
> GET _cat/snapshots?v&s=eti:desc&h=id,re,fs,ts,r,sti,eti,dur
> GET _snapshot/_all/*
## current
GET _snapshot/_all/_current?filter_path=total,snapshots.state,snapshots.start_time,snapshots.shards
## ESS SLM
> GET _snapshot/found-snapshots/_all?index_names=false&sort=start_time&size=150&order=desc&slm_policy_filter=cloud-snapshot-policy
```

**Problem Box**
- "all current data"? No, all committed (not currently being written) segments
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-restore.html#snapshot-restore-warnings): other backup methods (e.g. VM) not supported
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/snapshot-restore.html#snapshot-restore-version-compatibility): can only restore older versions into newer
- [doc](https://underthehood.meltwater.com/blog/2023/05/11/promoting-replica-shards-to-primary-in-elasticsearch-and-how-it-saves-us-12k-during-rolling-restarts/): primary rotating requires fully syncing new segments

## Security

[doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html)
( [roles](https://www.elastic.co/guide/en/elasticsearch/reference/current/defining-roles.html) ([alt](https://www.elastic.co/guide/en/elasticsearch/reference/current/authorization.html#roles))
, [role mappings](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-roles.html)
, [api key](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-settings.html#api-key-service-settings)
, [service account](https://www.elastic.co/guide/en/elasticsearch/reference/current/service-accounts.html); token: [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/service-accounts.html#service-accounts-tokens), [CLI](https://www.elastic.co/guide/en/elasticsearch/reference/7.17/service-tokens-command.html)
, [transport](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-settings.html#transport-tls-ssl-settings) ([profile](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-network.html#transport-profiles), [settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-settings.html#ssl-tls-profile-settings)) )

```bash
> GET _cluster/settings?include_defaults=true&filter_path=*.xpack.security
```

realms: [ES doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/realms.html), [KB doc](https://www.elastic.co/guide/en/kibana/current/kibana-authentication.html), [ESS doc](https://www.elastic.co/guide/en/cloud/current/ec-securing-clusters-SAML.html)); restrictions: [ESS](https://www.elastic.co/guide/en/cloud/current/ec-restrictions.html#ec-restrictions-security), [ECE](https://www.elastic.co/guide/en/cloud-enterprise/current/ece-limitations.html#ece_security_2); [blog](https://www.elastic.co/blog/demystifying-authentication-and-authorization-in-elasticsearch)

[File-based](https://www.elastic.co/guide/en/elasticsearch/reference/current/file-realm.html) ([resolve](https://medium.com/@stefnestor/elasticsearch-file-based-realm-2508ff0e5de2))
, [Native](https://www.elastic.co/guide/en/elasticsearch/reference/current/native-realm.html) ([built-in](https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-users.html))
, [SAML](https://www.elastic.co/guide/en/elasticsearch/reference/current/saml-realm.html) ( [blog](https://www.elastic.co/guide/en/cloud/current/ec-securing-clusters-SAML.html); ADFS ([blog](https://www.elastic.co/blog/how-to-configure-elasticsearch-saml-authentication-with-adfs)), AWS, AzureAD ([blog](https://www.elastic.co/blog/saml-based-single-sign-on-with-elasticsearch-and-azure-active-directory)), GCP, Okta ([blog](https://www.elastic.co/blog/setting-up-saml-for-elastic-enterprise-search-okta-edition)) )
, [LDAP](https://www.elastic.co/guide/en/elasticsearch/reference/current/ldap-realm.html)
, [Kerberos](https://www.elastic.co/guide/en/elasticsearch/reference/current/kerberos-realm.html) ([blog](https://www.elastic.co/blog/how-to-secure-your-elasticsearch-clusters-using-kerberos))
, [JWT](https://www.elastic.co/guide/en/elasticsearch/reference/current/jwt-auth-realm.html)
, [OIDC](https://www.elastic.co/guide/en/elasticsearch/reference/current/oidc-guide.html) ([blog](https://www.elastic.co/blog/how-to-set-up-openid-connect-on-elastic-cloud-with-azure-google-okta))
, [PKI](https://www.elastic.co/guide/en/elasticsearch/reference/current/pki-realm.html)


[Create Native User](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api-put-user.html), [Update Native User Password](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api-change-password.html), [Enable Native User](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api-enable-user.html), [Disable Native User](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api-disable-user.html), [Read Native User](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api-get-user.html), [Delete Native User](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-api-delete-user.html)

```bash
# realms
> GET _cluster/settings?include_defaults=true&filter_path=*.xpack.security.authc.realms
$ cat cluster_settings_defaults.json | jq '.defaults|to_entries[]|select(.key|contains("xpack.security.authc.realms"))'
$ cat cluster_settings_defaults.json | jq . | grep xpack.security.authc.realms
# Lucene Logs
realm OR metadata OR authentic OR saml OR oidc or ldap or pki or kerberos

# realms > native users
> GET _security/user

> POST _security/user/tmp_superuser
{"password" : "changeme", "roles" :["superuser"]}

##
# superuser+system_indices
> POST _security/role/tmp_system_superuser_role
{ "applications": [{"application": "*", "privileges": ["*"], "resources": ["*"] } ], "run_as": ["*"], "cluster": ["all"], "indices": [{"privileges": ["all"], "allow_restricted_indices": true, "names": ["*"] } ] }

> POST _security/user/tmp_system_superuser
{"password" : "MY_RANDOM_PASSWORD", "roles" :["tmp_system_superuser_role"]}
## 
```

**Problem Box**
- usually find `elasticsearch.yml` issues by searching node start-up logs for `IllegalStateException: security initialization failed`
- when SSO login errors HTTP 403, need to sequentially check
  1. Pull log of SSO authentication + redirect to Kibana via SAML Tracer ([Chrome](https://chrome.google.com/webstore/detail/saml-tracer/mpdajninpobndbfcldcmbpnnbhibjmch), [Firefox](https://addons.mozilla.org/en-US/firefox/addon/saml-tracer/)) (works regardless of SSO realm, alt [Wireshark](https://www.wireshark.org/) if you want to dig). 
  2. Update Kibana URL to `KIBANA_URL/internal/security/me` to check metadata returned by IDP, `role_mapping` line-up, and therefore granted `roles`. 
    

# Cluster
[doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-state-publishing.html)

[Read Cluster State](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-state.html) (`cluster_state.json`), [Read Cluster Settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-get-settings.html) (`cluster_settings_defaults.json`), [Update Settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-update-settings.html)

```bash
> GET _cluster/state?human
# paths
$ cat cluster_state.json | jq 'paths' | pbcopy
# persistent tasks
> GET _cluster/state?filter_path=**.persistent_tasks
$ cat cluster_state.json | jq -c '.metadata.persistent_tasks.tasks'

> GET _cluster/settings?include_defaults&flat_settings
```

**Problem Box**
- Any cluster state who's JSON output is greater than 100MB is clearly in danger zone for disrupting cluster operation.


## Discovery/Master

[doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery.html) (aka. Bootstrap,Quorum,Voting) 

[CAT Master](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-master.html) (`cat_master.txt`), [Edit Voting Exclusions](https://www.elastic.co/guide/en/elasticsearch/reference/current/voting-config-exclusions.html)

```bash
> GET _cat/master
> GET _cluster/state?filter_path=master_node
$ jq '.nodes|input.master_node as $master|.[$master] as $value|{id:$master,ip:$value.ip, host:$value.host, name:$value.name}' nodes.json cluster_state.json
```

**Problem Box**
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-bootstrap-cluster.html): remove `cluster.initial_master_nodes` from `elasticsearch.yml` after bootstrapping
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-discovery-quorums.html): split brain, nodes within one cluster disagree on which node is elected master
  - even voting node count can lead to two nodes deciding they’re elected and not consolidating into one
  - detect: check `GET _cluster/state/master_node?local` against *every* node. Each node will report their master node. If any nodes disagree, then that represents a split brain.
  - (advanced) [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/node-tool.html#node-tool-detach-cluster): related logs `trying to join a different cluster with UUID`
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-fault-detection.html), [alt](https://medium.com/@stefnestor/es-cluster-fault-detection-6dfc10cd0d8d): cluster default detection
  ```
  $ find *.log | xargs grep -he 'node-join\|node-left\|disconnect\|NodeDisconnectedException\|master node changed\|lagging\|followers check retry count exceeded|elected-as-master|exitcode' | sed -e 's/, term:.*//' | sort
  # chronological order of nodes joining and leaving
  $ find *.log | xargs grep -he 'MasterService.*node-join\|node-left' | sed -e 's/, term:.*//' | sort 

  # Lucene
  "node-join" OR "node-left" OR disconnect OR disconnected OR NodeDisconnectedException OR "master node changed" OR lagging OR "followers check retry count exceeded" OR "CODE\=" OR "elected-as-master" OR "StackOverflowError" OR "OutOfMemory" OR exitcode
  ```
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/important-settings.html#path-settings): `environment is not locked`/`The process cannot access the file because it is being used by another process` occurs when ES file/folder modified outside ES services and undermines prerequisites. Common causes: admin, antivirus, other service touching NFS files.


## Allocation

[doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cluster.html#shards-rebalancing-settings)
( [alt](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-allocation.html)
, [blog](https://www.elastic.co/blog/red-elasticsearch-cluster-panic-no-longer)
, [deciders](https://mincong.io/2020/09/27/shard-allocation/) )
, [recovery](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-recovery.html) ([settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/recovery.html))

[CAT Allocation](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-allocation.html) (`cat_allocation.txt`), [Allocation Explain](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-allocation-explain.html) (`allocation_explain.json`); [Health](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-health.html) (`cluster_health.json`)
, [CAT Recovery](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-recovery.html) (cat_recovery.txt), [Read Recovery](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-recovery.html) (recovery.json), [Cluster Reroute](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-reroute.html)

```bash
> GET _cat/allocation?v&s=node&h=node,disk.*,shards
$ cat nodes_stats.json | jq '.nodes[]|{shards:.indices.shard_stats.total_count, disk_indices:.indices.store.size, disk_avail:.fs.total.free, disk_avail_bytes: .fs.total.free_in_bytes, disk_total:.fs.total.total, disk_total_bytes: .fs.total.total_in_bytes, host:.host, ip:.ip, node:.name}|.+{disk_used_bytes: (.disk_total_bytes-.disk_avail_bytes)}|.+{disk_percent: (100*.disk_used_bytes/.disk_total_bytes|round)}'

> GET _cluster/allocation/explain  
{ "index": "INDEX_NAME", "shard": NUMBER, "primary": BOOL }

> GET _cluster/health

> GET _recovery
> GET _cat/recovery?v&active_only=true&h=idx,sh,ty,st,time,bp,fp,top,snode,tnode&s=time:desc
$ cat recovery.json | jq -r --sort-keys 'to_entries[]|.key as $k|[.value.shards[]]|map_values(.+{index_name:$k})|.[]|{time:.total_time, index_name:.index_name, shard:.shard, primary:.primary, type:.type, stage:.stage, repository:.source.repository?, source_node:.source.name?, target_node:.target.name?, translog_percent:.translog.percent, bytes_percent:.index.percent}'

> POST _cluster/reroute?retry_failed=true
> POST _cluster/reroute
{"commands":[{"move":{ "index" : "INDEX_NAME", "shard" : NUMBER, "from_node" : "SOURCE_NODE", "to_node" : "TARGET_NODE" }}]}

```

**Problem Box**
- [doc](https://medium.com/@stefnestor/elasticsearch-data-health-5e760d303b49): troubleshoot `status:red/yellow` 
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cluster.html#shards-rebalancing-settings): cluster is "balanced" when nodes have even shard counts per data tier ≤8.6.0; however, [8.6.0 introduces](https://www.elastic.co/blog/whats-new-elasticsearch-kibana-cloud-8-6-0) a [desired balance](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cluster.html#shards-rebalancing-heuristics) (adding factors: ingest rate, shard size) where "even shard counts" goes from literal to weighted goal
  - WIP: [elasticsearch#41543](https://github.com/elastic/elasticsearch/issues/41543), [elasticsearch#17213](https://github.com/elastic/elasticsearch/issues/17213)
- common errors
  - `failed to obtain in-memory shard lock` / `translog from source [...] is corrupted` / `failed engine (reason: [corrupt file (source: [start])]) (resource=preexisting_corruption)` / `obtaining shard lock timed out after 5000ms` : Cluster Reroute Retry Failed (to get healthy data copy from other node) else Close Index and Snapshot Restore
  - CAT Allocation's `disk.indices <<< disk.used` 
    - Disk indices (assigned shards) vs disk used (+ translog, unassigned shards, soft-deletes, ~recovering/migrating for plans/ILM)
    - When moving shard, only deletes transient data on source node once index status:green, otherwise status:yellow leaves shard transient data on every node it touches
  - `no_attempt` > Cluster Reroute
  - `no_valid_shard_copy` > all nodes in cluster? snapshot restore
- slow allocation
  - [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-cluster.html#cluster-shard-allocation-settings): `cluster.routing.allocation.node_concurrent_recoveries`
    ```bash
    # set
    > PUT _cluster/settings
    { "transient": { "cluster.routing.allocation.node_concurrent_recoveries": 4 }}

    # reset
    > PUT _cluster/settings
    { "transient": { "cluster.routing.allocation.node_concurrent_recoveries": null }}
    ```
  - [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/recovery.html#recovery-settings): `indices.recovery.max_bytes_per_sec`, temporarily bump to 100-300MB for large clusters
    ```bash
    # set
    > PUT _cluster/settings
    { "transient": { "indices.recovery.max_bytes_per_sec": "300mb"}}  

    # reset
    > PUT _cluster/settings
    { "transient": { "indices.recovery.max_bytes_per_sec": null }}  
    ```


# Tasks

[doc](https://medium.com/@stefnestor/elasticsearch-tasks-a77f6b0cb558)

## Cluster

[CAT Cluster Tasks](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-pending-tasks.html) (`cat_pending_tasks.txt`), [Pending Cluster Tasks](https://www.elastic.co/guide/en/elasticsearch/reference/8.11/cluster-pending.html) (`cluster_pending_tasks.json`)

```bash
> GET _cat/pending_tasks?v&s=timeInQueue:desc
$ cat cluster_pending_tasks.json | jq -c '.tasks|sort_by(.insert_order)|.[]|{order:.insert_order, queued_time:.time_in_queue_millis, priority:.priority, source:.source}'
```

## ThreadPools

[doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/modules-threadpool.html)

[CAT ThreadPool](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-thread-pool.html) (`cat_thread_pool.txt`)

```bash
> GET _cat/thread_pool?v=true&s=n,nn&h=n,nn,q,a,r,c
> GET _nodes/stats?filter_path=nodes.*.thread_pool
$ cat nodes_stats.json | jq -rc '.nodes|to_entries[]|.key as $k|.value.thread_pool|to_entries[]|.key as $a|.value|[$k[:4], $a, .queue, .active, .completed, .rejected]|@tsv' | column -t
# has queue
$ cat cat_thread_pool.txt | awk 'NR==1 || ($4>0)'
# sum by thread
$ cat cat_thread_pool.txt | awk 'NR!=1 && $3>0' | print '$2"\t"$3' | awk '{a[$1] += $2} END{for (i in a) print a[i]"\t"i}' | sort

# threadpools
> GET _cluster/settings?include_defaults=true&filter_path=*.thread_pool
```

**Problem Box**
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/hotspotting.html): hot spot, hotspotting
  - Ingest Pipeline's processors run on `write` thread pool
- `es_rejected_execution_exception` > `QueueResizingEsThreadPoolExecutor` : queue full


## Node

[CAT Node Tasks](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-tasks.html) (`cat_tasks.txt`), [Read Node Tasks](https://www.elastic.co/guide/en/elasticsearch/reference/current/tasks.html) (`tasks.json`)

```bash
> GET _cat/tasks?v&s=time:desc&h=ty,ac,time,n,ti,pti
> GET _tasks?pretty&human&detailed
$ cat tasks.json | jq -r '.nodes | .[] | .tasks | .[] | [ ([(.running_time_in_nanos/1000000|round),"ms"]|join("")), .cancellable, .node[:5], .action, .id ] | @tsv' | sort -nr | head -10  
```

**Problem Box**
- pile-up of tasks of certain action
  ```bash
  $ cat tasks.json | jq '.nodes[].tasks[].action' | sort | uniq -c | sort -r
  ```
- longest running tasks
  ```bash
  $ cat tasks.json | jq -r '.nodes[].tasks[]|select(.action|contains("[s]")|not)| [.running_time_in_nanos, .running_time, .cancellable, .node[:5], .action]|@tsv' | sort -nr | head -10
  ```

# Index

[doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/documents-indices.html) ( [System](https://www.elastic.co/guide/en/elasticsearch/reference/current/api-conventions.html#system-indices) ([list](https://github.com/elastic/elasticsearch/issues/50251#issuecomment-1107637953)), [Aliases](https://www.elastic.co/guide/en/elasticsearch/reference/current/alias.html), [Mappings](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-mapping.html) ([types](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html)) (aka. field data types, columns), [Data Streams](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-streams.html) ([blog](https://spinscale.de/posts/2021-07-07-elasticsearch-data-streams-explained.html)) )

[Create Index](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-create-index.html), [Delete Index](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-delete-index.html), [Close Index](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-close.html)

[CAT Indices](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-indices.html) (`cat_indices.txt`), [Index Stats](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-stats.html) (`indices_stats.json`)
, [Read Index Settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-get-settings.html), [Update Index Settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-update-settings.html)
, [CAT Aliases](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-alias.html) (`cat_aliases.txt`), [Read Alias](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-get-alias.html) (`alias.json`) , [Update Alias](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-add-alias.html) 
, [Read Mappings](https://www.elastic.co/guide/en/elasticsearch/reference/8.7/indices-get-mapping.html) (`mapping.json`), [CAT FieldData](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-fielddata.html) (`cat_fielddata.txt`)
, [Read DataStream](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-get-data-stream.html) (`data_stream.json`), [Data Stream Stats](https://www.elastic.co/guide/en/elasticsearch/reference/current/data-stream-stats-api.html)

```bash
> GET _cat/indices?v=true&s=index&expand_wildcards=all&h=index,status,health,pri,rep,docs*
$ cat cluster_state.json | jq '.metadata.indices|to_entries[]|.value.settings.index as $set|{index:.key,status:.value.state, uuid:$set.uuid, pri:$set.number_of_shards, rep:$set.number_of_replicas, system:.value.system}' && cat indices_stats.json | jq '.indices|to_entries[]|.value as $v|{index:.key,docs_count:$v.primaries.docs.count, docs_deleted:$v.primaries.docs.deleted, store_size:$v.total.store.size, pri_store_size:$v.primaries.store.size}'

> GET _settings?human&expand_wildcards=all

> PUT INDEX_NAME/_settings
{"index": {"number_of_replicas": 0 }}


> GET _cat/aliases?v=true&s=alias,index&h=alias,index,is_write_index
$ cat aliases.json | jq -r 'to_entries[]|select(.value.aliases!={})|{i:.key, a:(.value.aliases)}|.i as $i|.a|to_entries[]|[(if .value.is_write_index then true else false end), .key, $i]|@tsv'
> GET _alias?human
$ cat aliases.json | jq -c 'to_entries[]|select(.value.aliases!={})|[.key, (.value.aliases|keys)]'  

> GET _all/_mapping?expand_wildcards=all
> GET _cat/fielddata

> GET _data_stream?expand_wildcards=all
> GET _data_stream/MY_DATASTREAM_NAME/_stats
```

**Problem Box**
- Empty indices (`docs.count:0`) taking up space unnecesarily
  ```bash
  $ cat cat_indices.txt | awk 'NR==1 || ($7==0)' | awk '{print($3)}' | sort
  ```
  - [elasticsearch#83345](https://github.com/elastic/elasticsearch/pull/83345): v8.4.0 stop default ILM rollover for 0 docs
- [elasticsearch#93188](https://github.com/elastic/elasticsearch/pull/93188): `docs.deleted` 33% (<8.7.0, 22% ≥8.7.0) of `docs` 
  ```bash
  $ cat cat_indices.txt | grep -v 'health' | awk 'NR!=1 && $7 && $7!=0 && $8 && $8!=0' | awk '$8/($7+$8)>0.3' | awk '{print $3}'
  ```
- copying index options
  - [Splitting](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-split-index.html) (increase `number_of_shards`, [how works](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-split-index.html#how-split-works); \* requires more space ECE/ESS as hard-links not supported)
  - [Cloning](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-clone-index.html) (same Settings/Mappings and does not apply index templates, [more info](https://stackoverflow.com/a/69268032))
  - [Reindexing](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-reindex.html) (does not copy Index Settings/Mapping)
  - Snapshot restore rename
- mapping conflicts
  ```bash
  > GET kbn:api/index_patterns/_fields_for_wildcard?pattern=MY_INDEX_PATTERN&meta_fields=_source&meta_fields=_id&meta_fields=_type&meta_fields=_index&meta_fields=_score
  $ cat output.json | jq -r '.fields[]|select(.type=="conflict")|.name,.conflictDescriptions'
  ```
- high mapping updates show via lots of `mergeAndApplyMappings` under Node Hot Threads
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-explosion.html) ([setting](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-settings-limit.html), [example](https://www.elastic.co/blog/found-crash-elasticsearch#mapping-explosion))
  ```bash
  $ cat mapping.json | jq -c 'to_entries[] | .key as $index | [.value.mappings|to_entries[]|{(.key):([.value|..|.type?|select(.!=null)]|length)}] | map(to_entries)|flatten|from_entries | ([to_entries[].value]|add) | { index: $index, field_count: select(.>1000) }' | jq '. | "\(.field_count) \(.index)"' | sed 's/\"//g'
  ```
- invalid value for existing type `mapper_parsing_exception`


## Templates

[doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-templates.html)

[CAT Templates](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-templates.html) (`cat_templates.txt`), [List Index Templates](https://www.elastic.co/guide/en/elasticsearch/reference/8.11/indices-get-template.html) (`index_templates.json`), [List Legacy Templates](https://www.elastic.co/guide/en/elasticsearch/reference/8.11/indices-get-template-v1.html) (`templates.json`), [List Components](https://www.elastic.co/guide/en/elasticsearch/reference/8.11/getting-component-templates.html) (`component_templates.json`) 

```bash
> GET _cat/templates?v&s=t,n
$ jq -nc 'input|to_entries|map_values(.value+{name:.key}) as $lt|input.index_templates as $t|($t+$lt)[]|{rank:(.order+.index_template.priority),name:.name,pattern:(.index_patterns+.index_template.index_patterns)}' templates.json index_templates.json

> GET _index_template
> GET _template # legacy
```

## Shard

[doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/size-your-shards.html) ( [segments](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-modules-merge.html) ( [blog](https://blog.mikemccandless.com/2011/02/visualizing-lucenes-segment-merges.html), [translog](https://www.elastic.co/guide/en/elasticsearch/guide/2.x/translog.html), [buffer](https://www.elastic.co/guide/en/elasticsearch/reference/current/indexing-buffer.html)) )

[CAT Shards](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-shards.html) (`cat_shards.txt`)
, [CAT Segments](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-segments.html) (`cat_segments.txt`), [Read Segments](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-segments.html) (`segments.json`)

```bash
> GET _cat/shards?v&h=s,pr,st,n,i
> GET _cluster/state/routing_table?filter_path=routing_table.indices.*.shards.*.unassigned_info
# box.unassigned
$ cat cluster_state.json | jq -rc '.routing_table.indices|to_entries[]|.key as $i|.value.shards|to_entries[]|.key as $s|.value[] as $v|select($v.state=="UNASSIGNED")|[$v.primary, $s, $i, $v.unassigned_info.allocation_status, $v.unassigned_info.reason, $v.recovery_source.type]|@tsv' | sort -r | column -t | head
$ jq -c '.routing_table.indices[].shards[][]|select(.state != "STARTED")|{reason: .unassigned_info.details, shard, primary, index}' cluster_state.json

> GET _cat/segments?v&s=i,s,seg&h=i,s,pr,seg,g,docs.*,ic,is,v
> GET INDEX_NAME/_segments
$ cat segments.json | jq '.indices|to_entries[]|.key as $i|.value.shards|to_entries[]|.key as $sh|.value[]|. as $v|.segments|to_entries[]|.key as $se|{index:$i, shard:$sh, primary:$v.routing.primary, segments:$se, status:$v.routing.state, generation:.value.generation, docs_count:.value.num_docs, docs_deleted:.value.deleted_docs, size:.value.size, committed:.value.committed, searchable:.value.search, version:.value.version, compound:.value.compound}'
```

**Problem Box**
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/size-your-shards.html#shard-size-recommendation): size 10-50GB
- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/master/size-your-shards.html): oversharding


## ILM

[doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html) ([config](https://www.elastic.co/guide/en/cloud/current/ec-configure-index-management.html), [bootstrap](https://www.elastic.co/guide/en/elasticsearch/reference/8.2/getting-started-index-lifecycle-management.html#ilm-gs-alias-bootstrap), [existing](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-with-existing-indices.html), [troubleshoot](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-error-handling.html)), [phases](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-index-lifecycle.html), [actions](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-index-lifecycle.html#ilm-phase-actions), [steps order](https://gist.github.com/stefnestor/9dc1b28fa86ccc6a9a54c3f4041a0cb6)

[Status](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-get-status.html), [Explain](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-explain-lifecycle.html), [Create/Replace Policy](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-put-lifecycle.html), [Read Policy](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-get-lifecycle.html), [Remove](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-remove-policy.html), [Move Step](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-move-to-step.html), [Start](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-start.html), [Stop](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-stop.html), [Retry](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-retry-policy.html)

```bash
> GET _ilm/status
> GET _ilm/policy
> POST _all/_ilm/retry

> GET _all/_ilm/explain
> GET _all/_ilm/explain?only_errors=true
$ cat cluster_state.json | jq '.metadata.index_lifecycle.policies' 
$ cat cluster_state.json | jq '.metadata.indices|to_entries[]|{index:.key, explain:.value.ilm}'
# common
## not waiting on timing event
$ cat ilm_explain.json | jq -c '.indices[]|select(.managed==true and .action!="complete" and .step!="check-rollover-ready")|{phase,action,step,policy,index}' | sort
## ⭐ I'm usually looking for this one
$ cat ilm_explain.json | jq -c '.indices[]|select(.managed==true)|.phase+"/"+.action+"/"+.step' | sort | uniq -c | sort -r
## ^ samesies but filtered to transient
$ cat ilm_explain.json | jq -c '.indices[]|select(.managed==true and (.action!="complete" or .phase=="new") and .step!="check-rollover-ready")|.phase+"/"+.action+"/"+.step' | sort | uniq -c | sort -r
```

**Problem Box** 
- [blog](https://www.elastic.co/blog/troubleshooting-elasticsearch-ilm-common-issues-and-fixes), [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-error-handling.html) most common errors
- `step […] for index […] with policy […] does not exist` somehow missed step, use ILM Move Step to necessary step
- policy not match cache policy
  ```bash
  $ cat ilm_explain.json | jq -r '.indices[]|select(.managed==true)|select(.policy != .phase_execution.policy)'
  ```
- missing cache
  ```bash
  $ cat ilm_explain.json | jq '.indices[]|select(.managed==true and (.phase|not))|.index'
  ```
- erring or not-sure-not-erring ( [elasticsearch#99030](https://github.com/elastic/elasticsearch/issues/99030) )
  ```bash
  $ cat ilm_explain.json | jq -rc '.indices[]|select(.managed==true)|select((.step|not) or .step=="ERROR") | .index'
  ```   

## Ingest

[doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-replication.html#basic-write-model) ( [Ingest Pipelines](https://www.elastic.co/guide/en/elasticsearch/reference/current/ingest.html) )

[Index Doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs.html), [Bulk](https://www.elastic.co/guide/en/elasticsearch/reference/current/docs-bulk.html), [Read Ingest Pipelines](https://www.elastic.co/guide/en/elasticsearch/reference/current/get-pipeline-api.html), [Simulate Ingest Pipeline](https://www.elastic.co/guide/en/elasticsearch/reference/current/simulate-pipeline-api.html)

```bash
> POST MY_INDEX_NAME/_doc 
{ "key": "value" }

> POST _bulk
{ "index" : { "_index" : "MY_INDEX_NAME", "_id" : "1" } }
{ "key" : "value" }

# add ingested timestamp
> PUT _ingest/pipeline/add_indexed_timestamp
{"description": "createdAt, not updatedAt", "processors": [{"set": {"field": "_source.indexed_at", "value": "{{_ingest.timestamp}}", "override": false } } ] }
```

**Problem Box**
- failed ingest
  ```bash
  $ cat indices_stats.json | jq -rc '.indices|to_entries[]|.value.primaries.indexing as $i | select($i.index_total>100) | select($i.index_failed > 100) | { index: .key, failed_percent:(100*$i.index_failed/$i.index_total|round), failed_count:$i.index_failed }'
  ```
- high indexing duration
  ```bash
  $ cat indices_stats.json | jq -r '[.indices | to_entries[] | select(.value.primaries.indexing.index_total>100) | select((.value.primaries.indexing.index_time_in_millis/.value.primaries.indexing.index_total) > 1) | { "key": .key, "value": (.value.primaries.indexing.index_time_in_millis/.value.primaries.indexing.index_total) }] | from_entries'
  ```
- ingest pipelines' failed_percent and total
  ```bash
  $ cat nodes_stats.json | jq -c '.nodes[]|.name as $node|.ingest.pipelines|to_entries[]|.key as $pipeline| .value.processors[] | to_entries[]|.key as $process| { pipeline:$pipeline, process:$process, node:$node, total:.value.stats.count, failed:.value.stats.failed, failed_percent:(try (100*.value.stats.failed/.value.stats.count|round) catch 0)}|select(.total>0 and .failed_percent>10)' | head
  ```

# Watcher

[ES doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/xpack-alerting.html), [KB doc](https://www.elastic.co/guide/en/kibana/current/watcher-ui.html), [api](https://www.elastic.co/guide/en/elasticsearch/reference/current/watcher-api.html) ([settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/notification-settings.html))

indices:
- `.watches`: config (service runs from nodes hosting shards)
- `.triggered_watches`: current active
- `.watcher-history*` historic/logging

[Read Watches](https://www.elastic.co/guide/en/elasticsearch/reference/current/watcher-api-get-watch.html), [Read Watcher Stats](https://www.elastic.co/guide/en/elasticsearch/reference/current/watcher-api-stats.html), [Execute](https://www.elastic.co/guide/en/elasticsearch/reference/current/watcher-api-execute-watch.html)

```bash
> GET _watcher/watch
> GET _watcher/stats

# … watch_id will come through as _inline_
> POST _watch/watch/WATCH_NAME/_execute
```

**Problem Box**

- [doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/actions-webhook.html) action webhook
  ```bash
  # debug webhooks - will emit passwords!
  > PUT _cluster/settings
  {"transient": {"logger.org.apache.http" : "DEBUG"} }
  ```
- long durations
  ```bash
  > GET .watcher-history*/_search?pretty
  {"query":{"bool":{"filter":[{"range":{"result.execution_time":{"gte":"now/d-1d"}}}]}},"size":0,"aggs":{"watch_id":{"terms":{"field":"watch_id","size":10},"aggs":{"max":{"max":{"field":"result.execution_duration"}},"avg":{"avg":{"field":"result.execution_duration"}}}}}}

  > GET .watcher-history*/_search?pretty 
  {"query":{"bool":{"filter":[{"range":{"result.execution_time":{"gte":"now/d-1d"}}}]}},"size":0,"aggs":{"watch_id":{"terms":{"field":"watch_id","size":10},"aggs":{"max_duration":{"max":{"field":"result.execution_duration"}},"avg_duration":{"avg":{"field":"result.execution_duration"}},"bucket_sort":{"bucket_sort":{"sort":[{"max_duration":{"order":"desc"}}]}},"top_hits":{"top_hits":{"size":1,"sort":[{"result.execution_time":{"order":"desc"}}],"_source":{"includes":["metadata.*"]}}}}}}} 
  ```
