PromQL is a built in query-language made for Prometheus. Here at Timber [we've found Prometheus to be awesome](https://timber.io/blog/prometheus-the-good-the-bad-and-the-ugly/), but PromQL difficult to wrap our heads around. This is our attempt to change that.

# Basics

## Instant Vectors

_Only Instant Vectors can be graphed._

`http_requests_total`

![http requests total](//images.ctfassets.net/h6vh38q7qvzk/47m73p4320Qm4Cs6AGOC8A/0afdd1172d332697d4a0563d7ef6dce5/http_requests_total.png)

This gives us all the http requests, but we've got 2 issues.
1. There are too many data points to decipher what's going on.
2. You'll notice that `http_requests_total` only goes up, because it's a [counter](https://prometheus.io/docs/concepts/metric_types/#counter). These are common in Prometheus, but not useful to graph.

I'll show you how to approach both.

### It's easy to filter by label.

`http_requests_total{job="prometheus", code="200"}`

![filter-by-label](//images.ctfassets.net/h6vh38q7qvzk/1YYMQf9uS4Oi0Q2KeiSWc6/aa1493fe8cef2e23d4430b84c9f9feb4/filter-by-label.png)

### You can check a substring using regex matching.

`http_requests_total{status_code=~"2.*"}`

![substring](//images.ctfassets.net/h6vh38q7qvzk/49euNSlsFGU4kA0qa44oEG/0538688f02c42cafd31054922ffd8b10/substring.png)

If you're interested in learning more, [here](https://docs.python.org/3/library/re.html) are the docs on Regex.

## Range Vectors

_Contain data going back in time._

Recall: Only Instant Vectors can be graphed. You'll soon be able to see how to visualize Range Vectors using functions.

`http_requests_total[5m]`

You can also use (s, m, h, d, w, y) to represent (seconds, minutes, hours, ...) respectively.

# Important functions

## For Range Vectors

_You'll notice that we're able to graph all these functions. Since only Instant Vectors can be graphed, they take a Range Vector as a parameter and return a Instant Vector._

### Increase of `http_requests_total` averaged over the last 5 minutes.

`rate(http_requests_total[5m])`

![rate](//images.ctfassets.net/h6vh38q7qvzk/1oQ9xOzINe4mIqIwkIwcu8/a421fa29abb27648e967a38e8425ef65/rate.png)

### Irate

Looks at the 2 most recent samples (up to 5 minutes in the past), rather than averaging like `rate`

`irate(http_requests_total[5m])`

![irate](//images.ctfassets.net/h6vh38q7qvzk/3MCqVRoOPCOuKG6MOsqYcu/0aac574e965b4d960bf83ad8477ce5f9/irate.png)

It's best to use `rate` when alerting, because it creates a smooth graph since the data is averaged over a period of time. _Spikey graphs can cause alert overload, fatigue, and bad times for all due to repeatedly triggering thresholds._

### # of http requests in the last hour. This is equal to the `rate` * # of seconds

`increase(http_requests_total[1h])`

![increase](//images.ctfassets.net/h6vh38q7qvzk/4UqNZH5G80WaowamI884MY/9e991a116fd1606b42c9978b72edb26f/increase.png)

These are a small fraction of the functions, just what we found most popular. You can find the rest [here](https://prometheus.io/docs/prometheus/latest/querying/functions/).

## For Instant Vectors

### Broken by Status Code

`sum(rate(http_requests_total[5m]))`

You'll notice that `rate(http_requests_total[5m])` above provides a large amount of data. You can filter that data using your labels, but you can also look at your system as a whole using `sum` (or do both).

You can also use `min`, `max`, `avg`, `count`, and `quantile` similarly.

![sum-rate](//images.ctfassets.net/h6vh38q7qvzk/6t6cQFBJ7i6yqOU2CCsQwW/4216d8b6efef36ec0b49ae601a65e545/sum-rate.png)

This query tells you how many total HTTP requests there are, but isn't directly useful in deciphering issues in your system. I'll show you some functions that allow you to gain insight into your system.

### Sum by Status Code

`sum by (status_code) (rate(http_requests_total[5m]))`

You can also use `without` rather than `by` to sum on everything not passed as a parameter to without.

![sum-by-rate](//images.ctfassets.net/h6vh38q7qvzk/MyLjzpWl4kcm0gwGSSaOA/4473db77e505c9322f68b533c2a5cc86/sum-by-rate.png)

Now, you can see the difference between each status code.

## Offset

You can use `offset` to change the time for Instant and Range Vectors. This can be helpful for comparing current usage to past usage when determining the conditions of an alert.

`sum(rate(http_requests_total[5m] offset 5m))`

Remember to put `offset` directly after the selector.

# Operators

Operators can be used between scalars, vectors, or a mix of the two. Operations between vectors expect to find matching elements for each side (also known as one-to-one matching), unless otherwise specified.

There are Arithmetic (+, -, \*, /, %, ^), Comparison (==, !=, >, <, >=, <=) and Logical (and, or, unless) operators.

## Vector Matching

__One-to-One__

Vectors are equal i.f.f. the labels are equal.

### API 5xxs are 10% of HTTP Requests

`rate(http_requests_total{status_code=~"5.*"}[5m]) > .1 * rate(http_requests_total[5m])`

![api5xx](//images.ctfassets.net/h6vh38q7qvzk/22x7QobKuMEIU88AYkea0Y/ded5098f90c33fb9e9cf7b978580de3f/api5xx.png)

We're looking to graph whenever more than 10% of an instance's HTTP requests are errors. Before comparing rates, PromQL first checks to make sure that the vector's labels are equal.

You can use `on` to compare using certain labels or `ignoring` to compare on all labels except.

## Many-to-One

It's possible to use comparison and arithmetic operations where an element on one side can be matched with many elements on the other side. _You must explicitly tell Prometheus what to do with the extra dimensions._

You can use `group_left` if the left side has a higher cardinality, else use `group_right`.

# Examples

_Disclaimer_: We've hidden some of the information in the pictures using the `Legend Format` for privacy reasons.

### CPU Usage by Instance

`100 * (1 - avg by(instance)(irate(node_cpu{mode='idle'}[5m])))`

Average CPU Usage per instance for a 5 minute window.

![cpu](//images.ctfassets.net/h6vh38q7qvzk/3rbFceQUaQ0w2O2sogi4w6/bbf45cfd8a92ac6a7fcfe24eca268b6a/cpu.png)

### Memory Usage

`node_memory_Active / on (instance) node_memory_MemTotal`

Percentage of memory being used by instance.

![memory](//images.ctfassets.net/h6vh38q7qvzk/5wKZrzMiHu6y80au0AOQ2G/3a7419d79587b9e8690787b1099a66d0/memory.png)

### Disk Space

`node_filesystem_avail{fstype!~"tmpfs|fuse.lxcfs|squashfs"} / node_filesystem_size{fstype!~"tmpfs|fuse.lxcfs|squashfs"}`

Percentage of disk space being used by instance. We're looking for the available space, ignoring instances that have `tmpfs`, `fuse.lxcfs`, or `squashfs` in their `fstype` and dividing that by their total size.

![disk](//images.ctfassets.net/h6vh38q7qvzk/66tln8JKuWuyQ2CwA4gaCY/40de7b35900834ac2de4c47757db8d63/disk.png)

### HTTP Error Rates as a % of Traffic

`rate(http_requests_total{status_code=~"5.*"}[5m]) / rate(http_requests_total[5m])`

![http error rates](//images.ctfassets.net/h6vh38q7qvzk/11WaHWtONckUeYcqY4YYyc/ba85a9ba85c37d2b74d81ff60d6f4dff/http_error_rates.png)

### Alerts Firing in the last 24 hours

`sum(sort_desc(sum_over_time(ALERTS{alertstate="firing"}[24h]))) by (alertname)`

![alerts firing](//images.ctfassets.net/h6vh38q7qvzk/bk7qkd88FykSwoI8gIeoa/1b865a815087b373da64d13483fc3d23/alerts_firing.png)

You can find more useful examples [here](https://github.com/infinityworks/prometheus-example-queries).