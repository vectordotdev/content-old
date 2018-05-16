---
title: Prometheus: The Good, the Bad, and the Ugly
author: David Antaramian
---
Operational insight is critical to the success of any software deployment, and
Timber is no exception. We need to know that service is being delivered on a
consistent basis to our users and that any anomaly is reported to our team
quickly. There are a multitude of solutions that exist for gaining this type of
insight, and each has their own benefits and trade-offs as well as cost
considerations.

At Timber, we use our own product to analyze our logs which offers us rich and
powerful insight into the granular health of our applications. At the same time,
we recognize the need to have a higher-level view of how our system is
functioning as a whole. This type of view is better served by quantitative data
that can be tested against SLA and internal expectation thresholds. I took
responsibility for the strategy and implementation of this view in late 2017 at
Timber.

While developing the strategy we would use to approach our own visibility, I
briefly considered a hosted solution for this, but I decided against that based
on an evaluation of our infrastructure and the potential costs. This left me
considering self-hosted solutions including InfluxDB, statsd, riemann, OpenTSDB,
Graphite, and Prometheus.

I was also looking for a solution that would grow with our system but didn't
require a high-degree of specialized knowledge. OpenTSDB's documentation
starts with "First you need to setup HBase"; maintaining an HBase cluster for
our operational metrics alone is not worth the psychological and temporal
toll.

Another key concern for me was how we got data _into_ the service. Our core
services are written in Elixir which does not (yet) have many integrations with
popular metrics protocols (or otherwise the integrations are poorly maintained).

InfluxDB and Prometheus ended up being the major competitors. They both offered
stand-alone binaries, did not require the management of external data systems,
and allowed for rich metadata on metrics. The two key differences between the
offerings were:

  - Prometheus will _pull_ data from services, while InfluxDB needs data to be
    pushed to the InfluxDB instance
  - InfluxDB collects _every_ data point while Prometheus only collects
    _summaries_ of data points

Both of these points have their benefits and trade-offs. By collecting every
data point, InfluxDB can support complex, high-resolution queries at the cost of
higher network traffic and larger on-disk storage. And pushing data to InfluxDB
means that the origin system can be located anywhere; whereas Prometheus ingests
data by scraping metric summaries (in plain text format) from HTTP endpoints on
every server.

Reasoning through the differences, I found myself drawn towards Prometheus as
the better solution for our team:

  - Prometheus has [rich set of community-supported libraries for Elixir](https://hex.pm/packages/prometheus_ex).
  - Prometheus makes testing metrics easy, since I can view the HTTP endpoint it
    scrapes myself during development. There's no need to run Prometheus locally
    or mock its destination.
    
Here's an example of the summary format Prometheus scrapes from each host:

```
# TYPE erlang_vm_process_count gauge
# HELP erlang_vm_process_count The number of processes currently existing at the local node.
erlang_vm_process_count 848
# TYPE erlang_vm_process_limit gauge
# HELP erlang_vm_process_limit The maximum number of simultaneously existing processes at the local node.
erlang_vm_process_limit 262144
```

Overall, the fact that Prometheus summarizes data hasn't hurt us and has even
benefited us because it can recover easily from temporary network failures and
other issues. It also requires less configuration on our servers, since our
servers don't need to know where the Prometheus instance is located. Instead,
Prometheus finds all of the appropriate servers by querying Amazon's EC2 API.

I've set up all of our instances running on EC2 to expose metrics on a
dedicated port. This allows us to maintain some fine-grained security rules for
access between instances and limits the likelihood that the metrics would be
exposed to the outside world, even on public facing systems. I also run
Prometheus' node exporter on every EC2 machine which collects system level
metrics.

The only major detractor I've found with Prometheus' pull system is with a
service we have running on Heroku with multiple dynos. Heroku doesn't expose
dynos to the outside world except via a load-balancer, so there's no way for
Prometheus to reliably scrape _all_ the dynos; the load-balancer will randomly
assign a dyno to respond to each scrape request. This means that to Prometheus,
the service appears as a single host with erratic differences in metrics on each
scrape. In the future, I plan to solve this by moving that system into our AWS
account ecosystem, but an alternative would be to use Prometheus' "push gateway"
which allows services to push data from the dynos to a centralized host which
Prometheus can then reliably scrape from.

Prometheus overall has been a huge boon to us. It allows us to understand the
aggregate view of our system’s operational health while also letting us drill
down to the health status of individual components. It also has a powerful
built-in alerting system that we integrate with PagerDuty to allow us quickly
react to abnormal situations. It still has a number of warts, however, that make
it cumbersome to use and teach to other team members.

Chief among Prometheus’ obstacles is its query language: PromQL. It is difficult
to understand and requires comprehension of Prometheus’ data structures (which
seem to have some basis in linear algebra). Understanding how to combine and
reduce data into the form you need is tricky, and the documentation is difficult
to approach. For example, we use the following query to break out the number of
HTTP responses by status code:

```
sum by (status_code)(irate(http_requests_total{job="collector", method="POST", request_path="/frames"}[10s]))
```

Beyond the phrase “sum by,” it my assumption that the query above is not readily
comprehensible to most people. This makes it difficult to formulate and maintain
the queries as well as introduce new team members to the system.

Another difficulty in working with Prometheus comes with the histogram data
type. Jack Neely has [a very good
write-up](http://linuxczar.net/blog/2017/06/15/prometheus-histogram-2/) on his
blog, LinuxCzar, about the various issues surrounding Prometheus' histogram
implementations, and Reliable Insights, written in part by Prometheus
contributor Brian Brazil, has [a
counter-argument](https://www.robustperception.io/why-are-prometheus-histograms-cumulative/)
about why the histograms are cumulative. Regardless of the operational costs
that Brazil points to, though, I've found it to be immensely difficult to
forecast (as I write code) the correct buckets that a histogram should have. In
fact, I have re-bucketed histograms on occasion because I was so woefully off in
my initial assumption of buckets.

Even with these difficulties, though, Prometheus has given us an invaluable view
of our product that allows us to monitor our system as a whole. Prometheus
allows us to understand the "what" when things go wrong, so that when we go to
our logs, we know where to log to understand the "why".
