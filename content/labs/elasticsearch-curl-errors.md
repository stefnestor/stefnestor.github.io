+++
title = 'Common Elasticsearch CURL Errors'
date = 2024-05-14T12:00:00-07:00
draft = false
type = 'post'
showTableOfContents = true
tags = [ 'elastic', 'api', 'curl', 'bash', 'elasticsearch', 'network' ]
+++

**GOAL**: The following are common `curl` errors which surface in context to Elasticsearch but do not necessarily indicate an issue with its service vs how the client-side request was made. Errors are organized by networking > certificates > Elasticsearch layers. 


# routing
Examples: (`Could not retrieve the Elasticsearch version due to a system or network error - unable to continue.`, `Unable to connect to Elasticsearch at localhost:9200`, `No living connections`). 

## connection refused

```bash
$ curl https://myElasticsearchURL.com:9200
url: (7) Failed connect to myElasticsearchURL.com:9200; Connection refused
```

Alternatives: (`ERR_CONNECTION_REFUSED`, `ECONNREFUSED`, `This site can’t be reached`, `This page is unavailable`, `Unable to connect`, `Hmmm…can’t reach this page`, `connection failed to node at`, `failed to connect to`). This generically claims network routing issues to the `host:port` pair or responsiveness issues from it. It is quite a generic and also fairly common non-Elasticsearch-related networking error. 

As escalating investigation steps:
-  **IF** Elasticsearch is not responsive on `localhost`, it's probably exited (and if you recently started it up you might grep its logs for `StartupException`).  
- **IF** Elasticsearch is responsive on `localhost` on local box, then this error indicates something network related. 
- **IF** Elasticsearch is responsive on `localhost` but not desired IP/FQDN on local box, you'll need to review [Elasticsearch network settings](https://www.elastic.co/guide/en/elasticsearch/reference/master/modules-network.html#common-network-settings) (usually `network.host`).
- Otherwise, it's likely a non-Elasticsearch-related network error. At this point, it usually indicates inability to route the `curl` request to the target `host:port`. This could be for various reasons, but listing some common:
	- incorrect `port` 
	- firewall blocking/timeout
	- proxy rejection/timeout 
	- DNS cache
	- enabled/disabled VPN
	- [More ideas](https://support.site24x7.com/portal/en/kb/articles/troubleshoot-connection-refused-error111-for-elasticsearch-plugins).

> *NOTE*: Kibana reporting `connection error` to Elasticsearch is not the same ballpark of error; instead check Kibana logs for root error. 

## empty reply from server

```bash
$ curl localhost:9200
curl: (52) Empty reply from server
```

This occurs when Elasticsearch is not running on the `host:port` pair. This commonly comes up when either
- the Elasticsearch service is stopped
- [Elasticearch is SSL-enabled](https://www.elastic.co/guide/en/elasticsearch/reference/current/secure-cluster.html) . Then it will reject HTTP traffic and only allow HTTPS traffic. So instead, the command will work by adding an explicit `https://` in front of the `localhost` host. 

## connect to host 

```bash
$ curl 127.0.0.1:9200
curl: (7) couldn't connect to host
```

This occurs when Elasticsearch is not running on the `host:port` pair (usually when not `localhost`). Usually if `127.0.0.1` specifically induces this error, you'll check `localhost` instead for further troubleshooting. 


# certificates

Example:  (`Unable to retrieve version information from Elasticsearch nodes. unable to get issuer certificate`). 

Certificate issues usually source in setup file reference issues or outside-Elasticsearch certificate setup issues. The following therefore offer first troubleshooting steps but will not normally be able to tell you how to resolve your certificate issue vs recommend recreating. 

> *NOTE*: Various of the below issues can by bypassed (not necessarily recommended) by disabling certificate verification or running the request insecure (e.g. `-k` or `--insecure` for `curl` requests). This is a common pre-production troubleshooting method. Example: 
> ```bash
> $ curl -k https://localhost:9200
> ```
> See also [Elastic's blog](https://www.elastic.co/blog/configuring-ssl-tls-and-https-to-secure-elasticsearch-kibana-beats-and-logstash) & [Opsters Certificate guide](https://opster.com/guides/elasticsearch/security/elasticsearch-cluster-security/) ([alt](https://opster.com/guides/elasticsearch/how-tos/update-elasticsearch-security-certificates/)). 

## local issuer

```bash
$ curl https://localhost:9200
curl: (60) SSL certificate problem: unable to get local issuer certificate
```

When Elasticsearch is HTTP SSL-enabled with certificates (so `elasticsearch.yml` contains [`xpack.security.http.ssl`](https://www.elastic.co/guide/en/elasticsearch/reference/master/security-settings.html#http-tls-ssl-settings) settings related to certificates), then it will error when client-side attempts traffic without certificates. Depending on your setting's (`verification_mode`, `verificationMode`, `verify_certs`)  you will be required to supply the certificates.

## invalid chain 

```bash
$ curl -u "elastic:changeme" https://localhost:9200 
curl: (35) OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to localhost:9200 
```

Where backend client-side/Elasticsearch logs report (`Certificate chain is not valid` , ` Certificate chain was invalid [Invalid Entry: expected X.509 Certificate`, `Caused by: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target`). 

This has occurred historically because ... requiring recreating. 
- the certificates in the chain were in the incorrect order
- the root CA is not available / is invalid

## failed verify 

```bash
$ curl -u "elastic:changeme" https://localhost:9200 
CERTIFICATE_VERIFY_FAILED
```

Where backend client-side/Elasticsearch logs report (`ConnectionError([SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed`). Usually Go/Python/Java will error this and you'll need to `curl` to further investigate the issue. Alternatively, this is a generic certificate error and further down `Caused By` messages in Elasticsearch logs would tell you what to investigate. The most common causes are the next couple of errors below. 

## self signed 

```bash
$ curl -u "elastic:changeme" https://localhost:9200
curl (60) ssl certificate problem self signed certificate in certificate chain
```

Highlighting this is not an Elasticsearch-induced error but a certificate setup error. See [Elasticsearch's setup guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-basic-setup-https.html), but for example can resolve by 
- not using self-signed certificate 
- declaring certificate reference in `curl` requests, for example via `--cacert` 
- setting (`elasticsearch.ssl.certificateAuthorities`, `ssl.certificate_authorities`) 
- (not recommended) disable this check client-side `elasticsearch.ssl.verificationMode:none` (e.g. [for Kibana](https://www.elastic.co/guide/en/kibana/current/settings.html))

## trust error

```bash
$ curl -u "elastic:changeme" https://localhost:9200 --cacert ./mycert.pem 
curl: (60) schannel: CertGetCertificateChain trust error CERT_TRUST_REVOCATION_STATUS_UNKNOWN
curl: (60) The certificate chain was issued by an authority that is not trusted. SEC_E_UNTRUSTED_ROOT 
```

Both of these errors can occur in Windows invalidly as an certificate path look-up erring from being invalid or user not having permissions. Outside of that it is a certificate level error and which is usually recreated to resolve. 

## unknown authority 

```bash
$ curl -u "elastic:changeme" https://localhost:9200
Error dialing x509: certificate signed by unknown authority
```

This sometimes is a parent-level error to other errors on this page. Otherwise, you'll need to investigate writing/fixing settings like `certificate_authorities` (e.g. [Filebeat](https://www.elastic.co/guide/en/beats/filebeat/current/configuration-ssl.html#server-verification-mode), [Elasticsearch](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-basic-setup-https.html)). 

## alternative subject name

```bash
curl: (60) no alternative certificate subject name matches target host name 'XXXXX'
```

Alternatively (`ERR_TLS_CERT_ALTNAME_INVALID`). This is a certificate setup issue. If using [elasticsearch-certutil](https://www.elastic.co/guide/en/elasticsearch/reference/current/certutil.html) to create your certificates, see settings (`--dns`, `--ip`). 

## expired or inactive

```bash
$ curl -u "elastic:changeme" https://localhost:9200
Error dialing x509: certificate has expired or is not yet valid
```

This is a certificate setup issue which is resolved by recreating with valid time ranges. I don't know how to handle this ballpark so just linking [Sematext's blog](https://sematext.com/blog/ssl-certificate-error/) about this ballpark. 

## bad

```bash
$ curl -u "elastic:changeme" https://localhost:9200 -v  | grep 'SSL_ERROR_BAD_CERT_ALERT'

# can error for various reasons but for example
curl: (58) NSS: client certificate not found (nickname not specified)
Error pinging server: ConnectionError: Invalid or malformed certificate
```

There's a multitude of rabbit-hole reasons a certificate can be bad. Here's the most common reasons I've found as seen client-side or within Elasticsearch logs:

```bash
Caused by: javax.net.ssl.SSLHandshakeException: Received fatal alert: bad_certificate 

# sometimes sub-errors 
Caused by: javax.net.ssl.SSLHandshakeException: null cert chain 
Caused by java.security.cert.CertificateException: failed to parse any certificates from [MY_DIRECTORY] 
Caused by: io.netty.handler.ssl.NotSslRecordException: not an SSL/TLS record 
Caused by: javax.net.ssl.SSLHandshakeException: Received fatal alert: unknown_ca 
Caused by: javax.net.ssl.SSLHandshakeException: unable to get issuer certificate 
Caused by: sun.security.validator.ValidatorException: PKIX path validation failed: java.security.cert.CertPathValidatorException: Path does not chain with any of the trust anchors 
Caused by javax.net.ssl.SSLHandshakeException: no cipher suites in common 
Caused by: javax.net.ssl.SSLHandshakeException: Client requested protocol SSLv3 not enabled or not supported 
Caused by: java.security.cert.CertificateException: No name matching XXXXX found 
Caused by: javax.net.ssl.SSLException: Received fatal alert: certificate_unknown

# TCP-only
Caused by: java.io.StreamCorruptedException: invalid internal transport message format
```

Historically this occurred for typo reasons (invalid/empty certificate file, missing permissions, inconsistent settings across nodes). Usually you'd [go through the steps again](https://www.elastic.co/guide/en/elasticsearch/reference/current/security-basic-setup.html) instead of trying to fix the certificates. 

## not match subject provided

```bash
"Host name 'XXXXX' does not match the certificate subject provided by the peer (YYYYY)"
```

This is a certificate vs usage line-up issue. For example, certificate does not cover the attempted IP/FQDN as listed under `elasticsearch.hosts`. 

## version number

```bash
$ curl -u "elastic:changeme" https://localhost:9200
curl: (35) error:1408F10B:SSL routines:ssl3_get_record:wrong version number
```

I've only seen this a handful of times, but AFAICT it occurs when you hodge-podge have certificate settings without having SSL/TLS settings enabled. You'd confirm by running with `http` instead of `https` and seeing Elasticsearch responsive while the `curl` error goes away. 

## verify location

HTTPS traffic might see ...

```bash
$ curl -u "elastic:changeme" https://localhost:9200 --cacert ./myCert.crt
curl: (77) error setting certificate verify locations:
  CAfile: /myCert.crt
  CApath: /etc/ssl/certs
```

... while Elasticsearch logs might see TLS traffic like ...

```bash
ElasticsearchException[failed to initialize SSL TrustManager - not permitted to read truststore file [XXXXX]]
```

I've only seen these once each but it indicates the requesting user does not have permissions to the certificate files, so try the `curl` / Elasticsearch start-up request with `sudo`. 

Same ballpark, but an alternative issue can occur if your certificates require password and the password is incorrect will error `File does not contain valid private key`. This could be either directory/file look-up issues or password issues. 


# Elasticsearch

Now we'll dive into the most common Elasticsearch errors which return from the server after the CURL request was successfully received by the server. These errors spread across `curl` / client-side issues still as well as now also adding in Elasticsearch-side issues. Elasticsearch errors show via both the response status code and body. 

## HTTP 401 

`UNAUTHENTICATED` : The additional information in the response body will usually indicate either that your `username:password` pair (or API Key, etc.) is invalid, or that your `.security` index is unavailable and you need to setup a temporary `role:superuser` [File-realm user to bypass](https://medium.com/@stefnestor/elasticsearch-file-based-realm-2508ff0e5de2).  

### missing credentials 

Example: when SSL-enabled, running `curl` without authenticating credentials will error like ... 

```bash
$ curl -k https://localhost:9200
{
  "error": {
    "header": {
      "WWW-Authenticate": ["Basic realm=\"security\" charset=\"UTF-8\"","Bearer realm=\"security\"","ApiKey"]
    },
    "reason": "missing authentication credentials for REST request [/]",
    "root_cause": [{"header": {"WWW-Authenticate": ["Basic realm=\"security\" charset=\"UTF-8\"","Bearer realm=\"security\"","ApiKey"] }, "reason": "missing authentication credentials for REST request [/]", "type": "security_exception" } ],
    "type": "security_exception"
  },
  "status": 401
}
```

... highlighting `reason: missing authentication credentials for REST request`. 

### invalid credentials

Example: when attempting a valid `username` but invalid `password` in our `username:password` pair, it will error like ... 

```bash
$ curl -k -u "elastic:invalidPassword" https://localhost:8130
{
  "error": {
    "header": {
      "WWW-Authenticate": ["Basic realm=\"security\" charset=\"UTF-8\"","Bearer realm=\"security\"","ApiKey"]
    },
    "reason": "unable to authenticate user [elastic] for REST request [/]",
    "root_cause": [{"header": {"WWW-Authenticate": ["Basic realm=\"security\" charset=\"UTF-8\"","Bearer realm=\"security\"","ApiKey"] }, "reason": "unable to authenticate user [elastic] for REST request [/]", "type": "security_exception" } ],
    "type": "security_exception"
  },
  "status": 401
}
```

... highlighting `reason: unable to authenticate user [elastic] for REST request`. 

This error can also occur when 
- the `username`, `api_key`, etc. fails to look-up and not just the `password`
- the credential backs up to the `.security-7` index (see [which objects it hosts](https://medium.com/@stefnestor/diving-into-elasticsearch-security-f0a10ebd1d43)) which is `status:red` 
- (infrequent) [issues induced client-side from special characters](https://github.com/elastic/logstash/issues/11193) 

## HTTP 403 

`UNAUTHORIZED` : Your `username` is recognized and `password` accepted (or alternative `api_key`, etc.) but has insufficient permissions to run the request. The response body will explicitly state which permissions were lacking. Example: user `test` created with no permissions at all ...

```bash
$ curl -k -u "test:changeme" https://localhost:9200
{
  "error": {
    "reason": "action [cluster:monitor/main] is unauthorized for user [test] with effective roles [], this action is granted by the cluster privileges [monitor,manage,all]",
    "root_cause": [{"reason": "action [cluster:monitor/main] is unauthorized for user [test] with effective roles [], this action is granted by the cluster privileges [monitor,manage,all]", "type": "security_exception" } ],
    "type": "security_exception"
  },
  "status": 403
}
```

... highlighting `reason: action [XXXXX] is unauthorized for user [test] with effective roles [YYYYY], this action is granted by the cluster privileges [ZZZZZ]`. 

## HTTP 429 

`TOO_MANY_REQUESTS` : Your username authenticated and authorized, but the cluster is under sufficiently high strain that it's not responding to the attempted API call. 

> *NOTE*: These responses are usually intermittent and directly relate to the cluster's resource strain, particularly when [hot spotting](https://www.elastic.co/guide/en/elasticsearch/reference/master/hotspotting.html). I'll outline example response bodies here but for their resolutions, see [ingest rejections](https://medium.com/@stefnestor/elasticsearch-ingest-rejections-ce97e2e9da00) (which also serves high-level for all other conversations). 

### circuit breakers

This occurs when Elasticsearch heap is hitting its `parent` or child-breaker limits to protect the service from JVM OOM'ing. See also Elasticsearch's [troubleshooting doc](https://www.elastic.co/guide/en/elasticsearch/reference/current/circuit-breaker-errors.html). Example:

```bash
$ curl -k -u "elastic:changeme" https://localhost:9200
{
  "bytes_limit": 6844055552,
  "bytes_wanted": 6844240272,
  "reason": "[request] Data too large, data for [<parent>] would be larger than limit of [6844055552/6.3gb]",
  "type": "circuit_breaking_exception"
}
```

... highlighting `reason: Data too large` which in example points to `parent` as the circuit breaker in question. Alternative: (`CircuitBreakingException`). 

### rejected  - coordinating

```bash
$ curl -k -u "elastic:changeme" https://localhost:9200/_bulk ... 
{
  "error": {
    "root_cause": [
      {
        "reason": "rejected execution of coordinating operation [coordinating_and_primary_bytes=0, replica_bytes=0, all_bytes=0, coordinating_operation_bytes=68295401, max_coordinating_and_primary_bytes=53687091]",
        "type": "es_rejected_execution_exception"
      }
    ]
  }
}
```

... highlighting `reason: rejected execution of coordinating operation`. This is the same ballpark as alternatives (`rejected execution of primary operation` , `rejected execution of replica operation`). 

### rejected  - queues

```bash
$ curl -k -u "elastic:changeme" https://localhost:9200/_bulk ... 
{
   "shard" : 0,
   "node" : "my-node",
   "reason" : {
      "reason" : "rejected execution of org.elasticsearch.common.util.concurrent.TimedRunnable@26c03d4a on QueueResizingEsThreadPoolExecutor[name = esdX/search, queue capacity = 1000, min queue capacity = 1000, max queue capacity = 1000, frame size = 2000, targeted response rate = 1s, task execution EWMA = 968.1ms, adjustment amount = 50, org.elasticsearch.common.util.concurrent.QueueResizingEsThreadPoolExecutor@70be0765[Running, pool size = 13, active threads = 13, queued tasks = 1000, completed tasks = 616499351]]",
      "type" : "es_rejected_execution_exception"
   },
   "index" : "my-index"
}
```

... highlighting `reason: rejected execution of ... on QueueResizingEsThreadPoolExecutor`. 

## HTTP 503 

`SERVICE_UNAVAILABLE` : This indicates a cluster-level issue with Elasticsearch. Most frequently it'll be [cluster formation/fault](https://medium.com/@stefnestor/es-cluster-fault-detection-6dfc10cd0d8d) issues surfacing in various stages. You'll usually port over to Elasticsearch logs to investigate as this situation is a prerequisite to the API's general responsiveness. 

### license 

This is a infrequent error but when it surfaces it's usually via Kibana as it's failing one of it's first connecting to Elasticsearch APIs. This can happen either during license setup or most commonly happens as master node overwhelms (right before next error example). Example: 

```bash
$ curl -k -u "elastic:changeme" https://localhost:9200
{
  "error": "Service Unavailable",
  "message": "License is not available.",
  "statusCode": 503
}
```

... highlighting `License is not available`. 

### master not discovered

When nodes are forming/defaulting from cluster, while a master node is not elected, Elasticsearch will error ...

```bash
$ curl -k -u "elastic:changeme" https://localhost:9200
{
  "error": {
    "reason": null,
    "root_cause": [{"reason": null,"type": "master_not_discovered_exception"}],
    "type": "master_not_discovered_exception"
  },
  "status": 503
}
```

... highlighting `master_not_discovered_exception`. Alternative: (`MasterNotDiscoveredException`).  

### cluster state recovery

Cluster fault detection issues can surface as delayed recovery ... 

```bash
$ curl -k -u "elastic:changeme" https://localhost:9200
{
  "error": {
    "caused_by": {
      "reason": "Cluster state has not been recovered yet, cannot write to the [null] index",
      "type": "status_exception"
    },
    "reason": "failed to promote the auto-configured elastic password hash",
    "root_cause": [{"reason": "Cluster state has not been recovered yet, cannot write to the [null] index", "type": "status_exception" } ],
    "type": "authentication_processing_error"
  },
  "status": 503
}

$ curl -k -u "elastic:changeme" https://localhost:9200
{
  "error": "ClusterBlockException[blocked by: [SERVICE_UNAVAILABLE/1/state not recovered / initialized];]",
  "status": 503
}
```

... highlighting ( `reason: Cluster state has not been recovered yet`, `state not recovered / initialized` ).

### unavailable shards

```bash
$ curl -k -u "elastic:changeme" https://localhost:9200/my-index/_search ... 
{
  "error": "Service Unavailable",
  "message": "[all shards failed: search_phase_execution_exception\n\tRoot causes:\n\t\tno_shard_available_action_exception: null]: all shards failed",
  "statusCode": 503
}

$ curl -X POST -k -u "elastic:changeme" https://localhost:9200/my-index/_doc/1 ... 
{
  "index": {
    "_id": "1",
    "_index": "my-index",
    "_type": "_doc",
    "error": {
      "reason": "[my-index][0] primary shard is not active Timeout: [1m], request: [BulkShardRequest [[my-index][0]] containing [1] requests]",
      "type": "unavailable_shards_exception"
    },
    "status": 503
  }
}
```

... highlighting `unavailable_shards_exception`. [Cluster Health](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-health.html) is `status:red` on target indices & you need to [resolve that first](https://medium.com/@stefnestor/elasticsearch-data-health-5e760d303b49). 

### timeout exception

```bash
$ curl -X PUT -u "elastic:changeme" https://localhost:9200/_snapshot/my_repo ... 
{
  "error": {
    "reason": "failed to process cluster event (put_repository [my_repo]) within 30s",
    "root_cause": [{"reason": "failed to process cluster event (put_repository [my_repo]) within 30s", "type": "process_cluster_event_timeout_exception" } ],
    "type": "process_cluster_event_timeout_exception"
  },
  "status": 503
}
```

... highlighting `process_cluster_event_timeout_exception`. 

### queue full

Noting Elasticsearch used to HTTP status code error 503 for but in the future should error 429 for ... 

```bash
$ curl -k -u "elastic:changeme" https://localhost:9200/_bulk ... 
{
  "accepted": 0,
  "errors": [{"message": "queue is full"}]
}
```

... see earlier `QueueResizingEsThreadPoolExecutor`. 

## HTTP 504 

`BAD_GATEWAY` : Your network is experiencing issues reaching the cluster. Frequently, Elasticsearch itself will not report the `504` but an intermediate proxy/firewall will. You'll usually want to check Elasticsearch `localhost` / logs for more troubleshooting data.

```bash
$ curl -k -u "elastic:changeme" https://localhost:9200
{
  "error": "Gateway Time-out",
  "message": "Client request timeout",
  "statusCode": 504
}
```

