# Custom Job Monitor (from [MLO Job Monitor Scripts](https://github.com/epfml/job-monitor))
We (mainly Thijs's efforts) are working on an environment that helps you to

* keep track of experiments in a reproducible way
* compare results (metrics, graphs, and custom data like images)
* explain differences by diffing experiment configs
* with more customizability than in tensorboard (e.g. custom axes, labels, grouping etc.)

This repo (a much simplified version) will also contain utilities for using the EPFL container cluster to run these experiments.

## Timeseries data
Much of the log information that is interesting in ML training forms a timeseries. Consider training loss over time, learning rate, norm(activations) in layer 5, ... There are amazing software stacks to track timeseries like webserver throughput and hardware utilizaiton over a large cluster, and we can use the same stack to track training timeseries. We can use:

* [InfluxDB](https://docs.influxdata.com/influxdb/v1.7/): a timeseries database
    - Format: `("training_loss", {"iteration": 1, "value": 0.45}, tags={"host": "iccluster45", "user": "lin", "experiment_id": 10})`
    - Each unique set of 'tags' forms a separate timeseries that can be queried efficiently.
    - We have one node (i.e., pod/container) that runs the database and the other nodes send metrics there.
    - The tutorial can be found [here](https://docs.influxdata.com/influxdb/v1.7/introduction/getting-started/) and [here](https://docs.influxdata.com/influxdb/v1.7/query_language/spec/).
* [Telegraf](https://www.influxdata.com/time-series-platform/telegraf/): agent for collecting metrics and reporting them to InfluxDB.
    - Designed for high throughput.
    - Keeps a buffer on the worker node that is sent to the database periodically.
    - [Python client](https://github.com/paksu/pytelegraf):
    ```python
    # pip install pytelegraf
    from telegraf.client import TelegrafClient
    client = TelegrafClient(host='localhost', port=8092)
    client.metric('training_loss', 0.46) # fast & non-blocking
    ```

## Setup(s)
### InfluxDB Setup
The easiest way is to use the Docker image [influxdb](https://hub.docker.com/r/_/influxdb/) to run a server.

Running an instance locally:
```bash
docker run -p 8086:8086 -it influxdb
```

Deploying on the container cluster:
```bash
# (1) replace 'vogels' in pod-timeseries.yaml by your username
# (2) replace 'vogels' in svc-timeseries.yaml by your username
kubectl create -f databases/timeseries/
```

The command launces a pod that runs the database and a service that lets you connect to this database form outside the container cluster (in the EPFL network). To figure out the IP and port of the server:

```bash
kubectl get svc | grep ${USER}-svc-timeseries
# output: vogels-svc-timeseries LoadBalancer 10.233.10.205 <pending> 8086:10035/TCP 1m
#         ^ port = 10035, ip is only for inside the cluster
kubectl get pod ${USER}-pod-timeseries -o yaml | grep hostIP
# output: hostIP: 10.90.47.10
#         ^ that's the IP we need on the regular EPFL network
```

You are done if you can to the database with the `influx` client:
```bash
influx --host 10.90.47.10 --port 10035
# Connected to http://10.90.47.10:10035 version 1.7.0
# InfluxDB shell version: v1.7.0
# Enter an InfluxQL query
# >
```
(get the client with `brew install influxdb`, or something similar)

InfluxDB has an [SQL-like query syntax](https://docs.influxdata.com/influxdb/v1.7/introduction/getting-started/)


### Telegraf setup
So you have an InfluxDB database in which you want to store training logs. You can directly use the InfluxDB HTTP API to send your metrics to that database, but if you send data often, this can get slow. A better way is to use a local buffer like Telegraf.

Telegraf is most often used to collect system utilization metrics (like CPU/mem/disk/GPU). It has a large ecosystem of plugins that automatically report these metrics to a database like InfluxDB. We can use it for this purpose, but we can also use it as a local buffer to mediate the talking to the database. We use the packaged plugin `inputs.socket_listener` to send metrics to over `UDP`. To install the service:

```bash
# (1)
apt-get update && apt-get install -y telegraf
# (2) Copy ./databases/timeseries/telegraf.conf to /etc/telegraf/telegraf.conf and replace two occurences of http://{{ influxdb_host }}:{{ influxdb_port }}
# (3)
sudo service telegraf start
```

With the provided `telegraf.conf`, you will start sending system utilization information to the database `telegraf`, and any input from UDP into `training`.

Go ahead and try sending a test metric:

```python
# pip install pytelegraf
from telegraf.client import TelegrafClient
client = TelegrafClient(host='localhost', port=8092)

# Records a single value with no tags
client.metric('test_metric', 123)

# Records a three values with different data types
client.metric('test_metric', {'value_a': 100, 'value_b': 100, 'value_c': True})

# Records a single value with one tag
client.metric('test_metric', 123, tags={'server_name': 'my-server'})
```
