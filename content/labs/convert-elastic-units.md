+++
title = 'Convert Elastic Units'
date = 2024-01-01T12:00:00-07:00
draft = false
type = 'post'
showTableOfContents = true
tags = [ 'elastic', 'python', 'bash', 'analysis', 'elasticsearch' ]
+++

> Python functions to convert (Elastic) bytes/timing units to human 
> readable format.

# Human API flag

[Elasticsearch's API](https://www.elastic.co/guide/en/elasticsearch/reference/current/rest-apis.html) 
offers multiple performance statistics for [bytes](https://www.elastic.co/guide/en/elasticsearch/reference/current/api-conventions.html#byte-units)/[timing](https://www.elastic.co/guide/en/elasticsearch/reference/current/api-conventions.html#time-units)
that can have human readable, 
sister-columns emitted via [`?human`](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#_human_readable_output) API flag. 

This human view can be very helpful on interpretting e.g. 
[Node Stats](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-nodes-stats.html), 
[Index Stats](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-stats.html), 
[Repository Metering](https://www.elastic.co/guide/en/elasticsearch/reference/current/get-repositories-metering-api.html), 
[Index Recovery](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-recovery.html), 
[Node Tasks](https://www.elastic.co/guide/en/elasticsearch/reference/current/tasks.html), and 
[ILM Explain](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-explain-lifecycle.html). 

However, sometimes we need a middle-step calculation to make these 
emitted stats more usable/interpretable. E.g. since Node Stats increments 
[since Node uptime](https://medium.com/@stefnestor/es-node-uptime-755a76acb5e9), 
we could back-up from [total rejections/breakers](https://medium.com/@stefnestor/elasticsearch-ingest-rejections-ce97e2e9da00) 
back to [hourly averages (see "examples")](https://medium.com/@stefnestor/use-method-for-elasticsearch-d976802d8ba6). 
But once calculated, seeing it in human-readable format again would be 
a nice little cherry-on-top.

## E.g. disk usage

For example, let's check Elasticsearch disk for cache storage issues. As 
a human, we'd check [CAT Allocation](https://www.elastic.co/guide/en/elasticsearch/reference/current/cat-allocation.html) 
to see if we show `disk.indices << disk.used` anywhere.

```diff
# GET _cat/allocation?v
shards disk.indices disk.used disk.avail disk.total disk.percent host       ip         node
-    51       76.8mb    23.6gb     34.6gb     58.3gb           40 172.18.0.5 172.18.0.5 es0
```

👻 This _very_ clearly shows a huge  issue where `disk.indices` is `76.8mb` 
but `disk.used` is `23.6gb`. However, it required a human thinking to 
run this API. Since Elastic advises against [programming against its CAT API's](https://medium.com/@stefnestor/elasticsearch-cat-alternatives-315f72ea6d5e) 
we'll instead look at [Node Stats](https://www.elastic.co/guide/en/elasticsearch/reference/current/cluster-nodes-stats.html):

```diff
# GET _nodes/stats?human&filter_path=nodes.*.indices.store
{ "nodes": { "_zFc4lREQ4iXaHmZuID1Hg": {
    "indices": { "store": {
      "reserved": "0b",
      "reserved_in_bytes": 0,
-      "size": "76.8mb",
+      "size_in_bytes": 80584938,
      "total_data_set_size": "76.8mb",
      "total_data_set_size_in_bytes": 80584938
} } } } }
```

This shows the same `disk.indices` at `76.8mb` that we saw before, but 
also includes `size_in_bytes` which we can use to build an automation on 
top. We'd compare this data to `disk.used` which Elastic calculates as 
`disk.total - disk.available`. (Note: _not_ related only to Elastic's 
`path.data` folder but disk as reported by the underlying OS. My example 
is a test environment running in a Docker container which is why the 
`disk.indices` is drastically far off `disk.used`.) To find these values, 
we'd poll a variant `&filter_path=`

```diff
# GET _nodes/stats?human&filter_path=nodes.*.fs.total
{ "nodes": { "_zFc4lREQ4iXaHmZuID1Hg": {
    "fs": { "total": {
-      "available": "34.7gb",
+      "available_in_bytes": 37283065856,
      "free": "37.7gb",
      "free_in_bytes": 40499781632,
-      "total": "58.3gb",
+      "total_in_bytes": 62671097856
} } } } }
```

So our `disk.used` calculates as `disk.total - disk.available` or 
`58.3gb - 34.7gb` so `23.6gb` which is what we saw above. 

This example is egregious so is easily human inspected. However, say the 
difference was closer or a human didn't coincidentally think to poll these 
API's. Instead, we'd have a cron job polling this endpoint and then 
calculating the `disk.cache` via the sister-columns ending in `*_in_bytes`. 

```
disk.cache_in_bytes = (disk.total_in_bytes - disk.available_in_bytes) - disk.indices
disk.cache_in_bytes = (62671097856 - 37283065856) - 80584938
disk.cache_in_bytes = 25307447062
```

(Noting per API doc's disclaimer, this is really an "other" category and 
not guaranteed to be only an Elasticsearch caching issue; however, for 
our purposes, I'm assuming we've already established base ignorable disk 
rate. I'm fingers-crossed on getting better data from Elasticsearch's 
API on this ballpark in the future.)

Now that we've gotten to `disk.cache_in_bytes` as `25307447062`, we could 
alert based on that being higher than a threshold predetermined by our 
team. This circles us back to "and now a human needs to take a look" 
reporting `disk.cache` along side `disk.cache_in_bytes` might be helpful. 

## E.g. index refreshes 

Pivoting for a second example, a human may ask "how would I know if any indices'
`refresh_interval` setting is set too low compared to how long its 
refreshes are actually taking?". There isn't an easy Elasticsearch CAT 
API to investigate this so we'll jump straight into programatically 
inspecting. We'll poll both [Index Setting](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-get-settings.html) 
and [Index Stats](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-stats.html):

```diff
# GET _settings?human&expand_wildcards=all&filter_path=*.settings.index.refresh_interval
{ ".kibana_task_manager_8.8.0_001": {"settings": {"index": {
-	"refresh_interval": "1s"
} } } }

# GET _stats?human&expand_wildcards=all&filter_path=indices.*.total.refresh
{ "indices": { ".kibana_task_manager_8.8.0_001": {
    "total": { "refresh": {
+      "external_total": 642,
-      "external_total_time": "9.3s",
+      "external_total_time_in_millis": 9303,
      "listeners": 0,
      "total": 670,
      "total_time": "8.4s",
      "total_time_in_millis": 8415
} } } } }
```

(Tribal-knowledge, humans only care about `external_*` instead of 
`total_*`, but I'm not sure where that's publicly documented.) 

So here we run into a new data interpretation category: Elastic uses human 
"[time units](https://www.elastic.co/guide/en/elasticsearch/reference/current/api-conventions.html#time-units)". 
So a `refresh_interval` of `1s` has no sister-column like 
`refresh_interval_in_millis` of `1000` which we would have wanted 
for-programming reviews. We'll need to programatically account for it. 

The second highlight while we humans are here is just outlining what 
we'd want our Python program to alert on:

```python
if external_total not in [0,None]:
	if refresh_interval_in_millis < (external_total_time_in_millis / external_total):
		print("👻 alert")
```

which for us this human, manual calculation would show like

```
1000ms < (9303ms/642)
1000ms < 14.49ms
# nope, so we're good 👌
```

The right-hand of this equation/conditional highlights another helpful 
analytical ballpark for Elasticsearch diagnostics. I may not necessarily 
want to alert vs report for a human to read later the average time 
duration per event. (Some of Elastic's API's return below milliseconds, 
but the majority are that level so we'll do the same.) Elasticsearch's 
API only currently emits total counts and total durations, so anywhere 
we either want an alert/report on average durations per event, we'll 
need to manually calculate and/or manually format as desired for human 
usage later. 

# Functions 

I'm liking Elasticsearch's API's human-readable flag. It makes human 
intuiting a lot quicker during investigations. In the same spirit that 
mathematics claims "don't round along the way, only at the end", since 
I would claim there's further analytics we'd want to do with their data, 
we need to take a step back to the non-human-readable data, make our new 
calculations, and then format back for humans to see. (This outline is 
purposely going to leave the sister-columns for human-readable along 
side for-programming rather than prescribing which to use when. Snippets 
are Python.)

## To human bytes

I haven't found a need yet for "human to bytes" so skipping that. On 
converting `*_in_bytes`, we can just repurpose this [StackOverflow](https://stackoverflow.com/a/1094933/7938558) 
answer noting [Elastic only reports](https://www.elastic.co/guide/en/elasticsearch/reference/current/api-conventions.html#byte-units) 
up to petabytes (and you could go higher but I'm less familiar with 
higher and it's non-intuitive for me so living with their cap):

```python
def b2human(num):
    if num == None: return num
    for unit in ("b", "kb", "mb", "gb", "tb", "pb"):
        if abs(num) < 1024.0:
            return f"{num:3.1f}{unit}b"
        num /= 1024.0
    return f"{num:.1f}Yib"

```

## From human time

OK, the next ballpark we run in to is where Elasticsearch's API _only_ 
reports the human-readable [time unit](https://www.elastic.co/guide/en/elasticsearch/reference/current/api-conventions.html#time-units) 
and doesn't include it's sister-column for-programming. E.g. [Index Settings](https://www.elastic.co/guide/en/elasticsearch/reference/current/indices-get-settings.html) 
`refresh_interval` (exemplified above). E.g. [ILM Explain](https://www.elastic.co/guide/en/elasticsearch/reference/current/ilm-explain-lifecycle.html) 
`age`. So here we need to determine the time unit and then convert from 
that unit down to some base unit/metric. AFAICT Elastic goes down to 
nanoseconds though the vast majority of its statistics are milliseconds; 
however, avoiding the edge case, we end up with:

```python
def humanTime2nanos(x):
    if x == None: return x
    num = None
    unit = None
    TIME_UNITS = [["nanos", 1000], ["micros", 1000], ["ms", 1000], ["s", 60], ["m", 60], ["h", 24], ["d", 30], ["M", 12], ["y", 1]]
    for u in TIME_UNITS:
        if x.endswith(u[0]) and unit==None:
            num = float(x[:-len(u[0])])
            unit = u[0]
    if num == None: return num
    for u in TIME_UNITS:
        if unit==u[0]:
            return num
        num *= u[1]
```

## To human time

So we have this conversion one-way, let's now outline backwards. (Note: 
I only care about estimates and not exact conversions so we're 
simplifying some real-life nuance, e.g. days-to-months is assumed 30. This 
code could potentially simplify, but I hit my "better done than perfect".)

```python
def ms2humanTime(num):
    if num == None: return num    
    unit = "ms"
    if abs(num)/1000>=1:
        num = num/1000 
        unit = "s"
        if abs(num)/60>=1:
            num = num/60
            unit = "m"
            if abs(num)/60>=1:
                num = num/60
                unit = "h"
                if abs(num)/24>=1:
                    num = num/24
                    unit = "d"
                    if abs(num)/30>=1:
                        num = num/30
                        unit = "M"
                        if abs(num)/12>=1:
                            num = num/12
                            unit = "y"
    return f"{num:.2f}{unit}"

def nanos2humanTime(num):
    if num == None: return num
    unit = "nanos"
    if abs(num)/1000>=1:
        num = num/1000
        unit = "micros"
    if abs(num)/1000>=1:
        num = num/1000
        unit = "ms"
        return ms2humanTime(num)
    return f"{num:.2f}nanos"
```

## Better human time

You'll note Elastic's [time units](https://www.elastic.co/guide/en/elasticsearch/reference/current/api-conventions.html#time-units) 
don't include `M` (month) or `y` (year); however, its [date math](https://www.elastic.co/guide/en/elasticsearch/reference/current/common-options.html#date-math) 
does _and_ especially on ILM's statistics I get annoyed trying to 
interpret an index's `age` of `700d` being somewhere human-guestimated 
of around `2y`.

Vs now if you append a convert-to-and-back-immediately function to 
check the above code ...

```python
def humanTime2humanTime(x):
    return nanos2humanTime( humanTime2nanos(x) )
```

... then we end up easily showing `700d` is `1.94y` in a Bash console via 
[Ipython](https://ipython.org) ...

```bash
$ ipython -i -c "import sys,os; sys.path.append(os.environ[\"HOME_LIB\"]) ; from elastic import *""
In [1]: humanTime2humanTime("700d")
Out[1]: '1.94y'
```

... Much better.

