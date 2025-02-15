# Grafana Dashboards for JupyterHub

Grafana Dashboards for use with [Zero to JupyterHub on Kubernetes](http://z2jh.jupyter.org/)

![Grafana Dasboard Screencast](demo.gif)

## What?

Grafana dashboards displaying prometheus metrics are *extremely* useful in diagnosing
issues on Kubernetes clusters running JupyterHub. However, everyone has to build their
own dashboards - there isn't an easy way to standardize them across many clusters run
by many entities.

This project provides some standard [Grafana Dashboards as Code](https://grafana.com/blog/2020/02/26/how-to-configure-grafana-as-code/)
to help with this. It uses [jsonnet](https://jsonnet.org/) and
[grafonnet](https://github.com/grafana/grafonnet-lib) to generate dashboards completely
via code. This can then be deployed on any Grafana instance!

## Pre-requisites

1. Locally, you need to have
   [jsonnet](https://github.com/google/jsonnet#packages) installed.  The
   [grafonnet](https://grafana.github.io/grafonnet-lib/) library is already
   vendored in, using
   [jsonnet-builder](https://github.com/jsonnet-bundler/jsonnet-bundler).

2. A recent version of prometheus installed on your cluster. Currently, it is assumed that your prometheus instance
   is installed using the prometheus helm chart, with [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics),
   [node-exporter](https://github.com/prometheus/node_exporter) and [cadvisor](https://github.com/google/cadvisor)
   enabled. In addition, you should scrape metrics from the hub instance as well.

3. A recent version of Grafana, with a prometheus data source already added.

4. An API key with 'admin' permissions. This is per-organization, and you can make a new one
   by going to the configuration pane for your Grafana (the gear icon on the left bar), and
   selecting 'API Keys'. The admin permission is needed to query list of data sources so we
   can auto-populate template variable options (such as list of hubs).

## Deployment

There's a helper `deploy.py` script that can deploy the dashboard to any grafana installation.

```bash
export GRAFANA_TOKEN="<API-TOKEN-FOR-YOUR-GRAFANA>
./deploy.py <your-grafana-url>
```

This creates a folder called 'JupyterHub Default Dashboards' in your grafana, and adds
a couple of dashboards to it.

**NOTE: ANY CHANGES YOU MAKE VIA THE GRAFANA UI WILL BE OVERWRITTEN NEXT TIME YOU RUN deploy.bash.
TO MAKE CHANGES, EDIT THE JSONNET FILE AND DEPLOY AGAIN**

## Upgrading grafonnet version

The grafonnet jsonnet library is bundled here with [jsonnet-bundler](https://github.com/jsonnet-bundler/jsonnet-bundler).
Just running `jb update` in the git repo root dir after installing jsonnet-bunder should bring
you up to speed.

## Metrics guidelines

Interpreting prometheus metrics and writing PromQL queries that serve a particular
purpose can be difficult. Here are some guidelines to help.

### Container memory usage metric

"When will the OOM killer start killing processes in this container?" is the most useful
thing for us to know when measuring container memory usage. Of the many container memory
metrics, `container_memory_working_set_bytes` tracks this (see [this blog post](https://faun.pub/how-much-is-too-much-the-linux-oomkiller-and-used-memory-d32186f29c9d)
and [this issue](https://github.com/jupyterhub/grafana-dashboards/issues/13)).
So prefer using that metric as the default for 'memory usage' unless specific reasons
exist for using a different metric.

### Available metrics

The most common prometheus on kubernetes setup in the JupyterHub community seems
to be the [prometheus helm chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus).


1. [kube-state-metrics](https://github.com/kubernetes/kube-state-metrics)
   ([metrics documentation](https://github.com/kubernetes/kube-state-metrics/tree/master/docs))
   collects information about various kubernetes objects (pods, services, etc)
   by scraping the kubernetes API. Anything you can get via `kubectl` commands,
   you can probably get via a metric here. Very helpful as a way to query other
   metrics based on the kubernetes object they represent (like pod, node, etc).

2. [node-exporter](https://github.com/prometheus/node_exporter)
   ([metrics documentation](https://github.com/prometheus/node_exporter#enabled-by-default))
   collects information about each node - CPU usage, memory, disk space, etc. Since hostnames
   are usually random, you usually join these metrics with `kube-state-metrics` node
   metrics to get useful information out. If you are running a manual NFS server,
   it is recommended to run a node-exporter instance there as well to collect server
   metrics.

3. [cadvisor](https://github.com/google/cadvisor)
   ([metrics documentation](https://github.com/google/cadvisor/blob/master/docs/storage/prometheus.md))
   collects information about each *container*. Join these with pod metrics from
   `kube-state-metrics` for useful queries.

4. [jupyterhub](https://jupyterhub.readthedocs.io/en/latest/)
   ([metrics documentation](https://jupyterhub.readthedocs.io/en/latest/reference/metrics.html))
   collects information directly from the JupyterHubs.

5. Other components you have installed on your cluster - like prometheus,
   nginx-ingress, etc - will also emit their own metrics.

### Avoid double-counting container metrics

It seems that one container's resource metrics can be reported multiple times,
with an empty `name` label and a `name=k8s_...` label.
Because of this, if we do `sum(container_resource_metric) by (pod)`,
we will often get twice the actual resource consumption of a given pod.
Since `name=""` is always redundant, make sure to exclude this in any query
that includes a sum across container metrics.
For example:

```promql
sum(
    irate(container_cpu_usage_seconds_total{name!=""}[5m])
) by (namespace, pod)
```
