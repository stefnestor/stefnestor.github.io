+++
title = 'Three Node Elasticsearch Docker'
date = 2024-05-14T12:00:00-07:00
draft = false
type = 'post'
showTableOfContents = false
tags = [ 'elastic', 'docker', 'bash', 'elasticsearch', 'docker-compose' ]
+++

**GOAL**: Automate spinning up v7.x or v8.x Elasticearch clusters with 3 nodes for testing. 

# Guide 

Elastic's published a quick-start guide: [Start a multi-node cluster with Docker Compose](https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html#docker-compose-file) . Following this, we can quickly get up and running via commands

```bash
$ where
~/downloads/docker3es
$ curl -s -o ".env" https://raw.githubusercontent.com/elastic/elasticsearch/8.13/docs/reference/setup/install/docker/.env
$ curl -s -o "docker-compose.yml" https://raw.githubusercontent.com/elastic/elasticsearch/8.13/docs/reference/setup/install/docker/docker-compose.yml

# fill in .env details
$ cat .env | grep -v "#"
ELASTIC_PASSWORD=changeme
KIBANA_PASSWORD=changeme
STACK_VERSION=8.13.4
CLUSTER_NAME=my-docker-cluster
LICENSE=trial
ES_PORT=127.0.0.1:9200
KIBANA_PORT=5601
MEM_LIMIT=1073741824

$ docker-compose up -d
$ open -a Firefox http://localhost:5601
```

# Automate

From this, we can reformat this default guide to allow multiple version to simultaneously spin-up against this `docker-compose.yml` within a single host by dynamically handling their ports based on version `major.minor`. 

So in a directory (with Elastic's unmodified `docker-compose.yml`) ... 
```bash
$ ls
docker-compose.yml
elk.sh
```

... we can create a wrapping Bash function `elk.sh` which reads in `-v` version (required) and `-p` password (optional). 

```bash
#!/bin/bash

while getopts 'v:p:h' opt; do
  case "$opt" in
    v)
      v="$OPTARG" ;;

    p)
      p="$OPTARG" ;;

    ?|h)
      echo "Usage: $(basename $0) [-v arg] [-p arg]"
      exit 1
      ;;
  esac
done
shift "$(($OPTIND -1))"
if [[ -z $p ]]; then p="changeme"; fi
if [[ -z $v ]]; then echo "👻 provide version, e.g. 8.13.0" && exit 1; fi


function short_version {
  version_re="([0-9]+)\.([0-9]+)\.([0-9]+)"
  version=$1
  
  [[ $version =~ $version_re ]]
  v_max="${BASH_REMATCH[1]}"
  v_min="${BASH_REMATCH[2]}"

  if [[ ${#v_min}<2 ]]; then v_min="0${v_min}"; fi
  cur_version="${v_max}${v_min}"
  echo $cur_version
}

vs=$(short_version $v)
ves=$vs"0"

env="""
CLUSTER_NAME=es-$vs
COMPOSE_PROJECT_NAME=es-$vs
STACK_VERSION=$v

ELASTIC_PASSWORD=$p
ES_PORT=127.0.0.1:$ves
KIBANA_PASSWORD=$p
KIBANA_PORT=$vs
LICENSE=trial
MEM_LIMIT=1073741824
"""

( export $env; docker compose -f $(eval dirname "$(realpath $0)")/docker-compose.yml up -d ) 
echo ""
echo "🦖"
echo "KB: http://localhost:$vs"
echo "ES: https://localhost:$ves"
```

> *Note*: This will not let you run simultaneous `patch` versions of a `major.minor.patch` version, but it will let you run various majors/minors. So for example `8.13.4` cannot be ran simultaneous to `8.13.3` but it can be ran along side `8.11.1`. 

I prefer to set the reference to `elk.sh` in my Dotfiles / `.bash_profile` as `eck`
```bash
$ alias elk
alias elk='bash ~/Documents/docker3ek/elk.sh'
```

So running two versions at once to cross-compare would then appear

```bash
$ bash start.sh -v 8.13.4
[+] Building 0.0s (0/0)
[+] Running 6/6
 ✔ Network es-813_default     Created                           0.1s
 ✔ Container es-813-setup-1   Healthy                           1.7s
 ✔ Container es-813-es01-1    Healthy                          22.1s
 ✔ Container es-813-es02-1    Healthy                          22.1s
 ✔ Container es-813-es03-1    Healthy                          12.6s
 ✔ Container es-813-kibana-1  Started                          22.2s

🦖
KB: http://localhost:813
ES: https://localhost:8130


docker3es$ bash start.sh -v 8.11.1
[+] Building 0.0s (0/0)
[+] Running 11/11
 ✔ Network es-811_default      Created                          0.0s
 ✔ Volume "es-811_esdata02"    Created                          0.0s
 ✔ Volume "es-811_esdata03"    Created                          0.0s
 ✔ Volume "es-811_certs"       Created                          0.0s
 ✔ Volume "es-811_kibanadata"  Created                          0.0s
 ✔ Volume "es-811_esdata01"    Created                          0.0s
 ✔ Container es-811-setup-1    Healthy                          2.8s
 ✔ Container es-811-es01-1     Healthy                         23.2s
 ✔ Container es-811-es02-1     Healthy                         23.1s
 ✔ Container es-811-es03-1     Healthy                         23.6s
 ✔ Container es-811-kibana-1   Started                         23.9s

🦖
KB: http://localhost:811
ES: https://localhost:8110
```

> *Disclaimer*: This was tested on Mac running Terminal with default Bash Shell. 

