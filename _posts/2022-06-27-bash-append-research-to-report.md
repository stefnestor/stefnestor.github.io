---
layout: post 
title: "[LAB] Append Console Research to Report via Bash"
---

## lab

### code

```bash
### use
# execute command, then run: append "!!"
function append {
  if [[ -z $NOTE ]]; then
    echo "👻 not saved; set NOTE & repeat"
  else
    input="$*"
    FILEa='/Users/stef/downloads'/$NOTE'.md'

    echo >> $FILEa
    echo "\`\`\`" >> $FILEa
    echo "$ #" $(date +%Y%m%d-%H%M%S) >> $FILEa
    echo "$ "$input >> $FILEa
    echo "$(eval $input)" >> $FILEa
    echo "\`\`\`" >> $FILEa
  fi
}
```

### run

```
Last login: Mon Jun 27 11:59:23 on ttys000
$ NOTE="TMP"
$ echo "stef"
stef
$ append "!!"
append "echo "stef""
```

### result
`TMP.md` (auto-created and) added
`````````
```
$ # 20220627-120212
$ echo stef
stef
```
`````````

## use case

[Obsidian](https://obsidian.md/) lets you wiki embed sub-pages. 
[Elastic Cloud](https://cloud.elastic.co) lets you maintain your deployment
via [their API](https://www.elastic.co/guide/en/cloud/current/ec-restful-api.html). 
You can also capture an Elasticsearch [diagnostic](https://github.com/elastic/support-diagnostics#usage-examples).

### ess api

You can setup your Bash dotfiles to quick cURL your ESS/ECE deployment's 
[ES API Console](https://www.elastic.co/guide/en/cloud/current/ec-api-console.html).

```bash
function ess {
  AUTH="Authorization: APIKey ${ELASTIC_ECE_KEY}"
  MGMNT="X-Management-Request: true"
  DOMAIN="https://adminconsole.found.no/api/v1/deployments/$1/elasticsearch/"
  optA="main-elasticsearch/"
  optB="elasticsearch/"
  URI_END="proxy/"

  if [[ -z "$2" ]]; then URI="/"; else URI=$2; fi
  if [[ URI != "/" ]]; then URI_END+=$2; fi
  response=$(curl -s -XGET -H "$AUTH" -H "$MGMNT" $DOMAIN$optA$URI_END)

  if [[ $response == *"resource_not_found"* ]]; then
    response=$(curl -s -XGET -H "$AUTH" -H "$MGMNT" $DOMAIN$optB$URI_END)
  fi
  echo $response
}
```

Your simplified cURL can now be ran in terminal, e.g.

```
$ ess e406cfd66dc944119bd8da2bb7290d2b _cluster/health?filter_path=status
{"status":"yellow"}
```

### investigate

Let's say I've captured a diagnostic showing my cluster's `status:yellow`. 
JSON filtering done via [JQ](https://stedolan.github.io/jq/manual).

```
$ where
/Users/stef/downloads/diag
$ append "!!"
$ cat cluster_health.json | jq '.status'
"yellow"
```

My [Allocation Cheatsheet](https://github.com/stefnestor/elastic/blob/main/Elasticsearch/Index/Shard/Allocation/allocation%20cheatsheet.pdf) 
suggests we need to check if we're currently assigning/recovering shards.

```
$ cat cluster_health.json | jq '{status,number_of_nodes,active_shards,unassigned_shards,active_shards_percent_as_number,relocating_shards,initializing_shards,delayed_unassigned_shards}'
{
  "status": "yellow",
  "number_of_nodes": 2,
  "active_shards": 92,
  "unassigned_shards": 3,
  "active_shards_percent_as_number": 96.84210526315789,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "delayed_unassigned_shards": 0
}
$ append "!!"
```

We are not, so we'll kick off a [Cluster Reroute](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-reroute.html). 
After 3min, I poll [Cluster Health](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-health.html).

```
$ ess e406cfd66dc944119bd8da2bb7290d2b _cluster/health | jq '{status,number_of_nodes,active_shards,unassigned_shards,active_shards_percent_as_number,relocating_shards,initializing_shards,delayed_unassigned_shards}'
{
  "status": "yellow",
  "number_of_nodes": 2,
  "active_shards": 92,
  "unassigned_shards": 3,
  "active_shards_percent_as_number": 96.84210526315789,
  "relocating_shards": 0,
  "initializing_shards": 0,
  "delayed_unassigned_shards": 0
}
$ append "!!"
```

We're unchanged. This suggests a non-transient issue's at play. We 
continue to use the cheatsheet to investigate.

### (opt) write up

I can leave the file as is for internal tracking. 

![raw report](/images/2022-06-27-bash-append-research-to-report-A.png)


If I'm sending it to a customer, I'll add in quotes (previously written 
by me) to flush out Doc contexts.

![report with annotations](/images/2022-06-27-bash-append-research-to-report-B.png)

### result

This exports as a ready-to-go PDF.

![frontend report](/images/2022-06-27-bash-append-research-to-report-C.png)

