# efk-k8s
Kubernetes Cluster log analyse and monitoring solution with ElasticSearch + Filebeat + Kibana + MetricBeat 

Assuming that you know essential concept of EFK package and its target , lets start stept by step Running EFK on kubernetes cluster. Before starting , consider that this tutorial is ElasticSearch configuration without x-pack security configuration. 


First of all we need create specific namespace .

``` 
kubectl create namespace logging
```
# ElasticSearch
In this senario we decide run ElasticSearch cluster inside of Kubernetes cluster. First node of the cluster we're going to setup is the master which is responsible for controlling the cluster .
elasticsearch-master file consist of configmap , service and deployment.

```
kubectl create -f 1-elasticsearch-master.yml
```
************* NOTE **************
If you faced with ```max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]``` 
You should use this command on scheduled worker node for Elasticsearch on kubernetes :
``` sysctl -w vm.max_map_count=262144 ``` and confirm your changes by ``` sudo sysctl -a | grep vm.max_map_count ```

The second node of the cluster we're going to setup is the data which is responsible for hosting the data and executing the queries (CRUD, search, aggregation) which is similar to master node deploy.

```
kubectl create -f 2-elasticsearch-data.yml
```
The last but not least node of the cluster is the client which is responsible of exposing an HTTP interface and pass queries to the data node.

```
kubectl create -f 3-elasticsearch-client.yml
```

# Kibana
In terms of setup in k8s, this is very similar to ElasticSearch, we first use ConfigMap to provide a config file to our deployment with all the required properties. This particularly includes the access to ElasticSearch (host, username and password) which are configured as environment variables. Let's create kibana resources.

```
kubectl create -f 4-kibana.yml
```
And after a couple of minutes, check the logs for Status changed from yellow to green
```
kubectl logs -f -n logging $(kubectl get pods -n logging | grep kibana | sed -n 1p | awk '{print $1}')      | grep "Status changed from yellow to green" 
```
Once, the logs say "green", you can access Kibana from your browser.
You can check kibana panel on assigned worker's IP port.

# Metricbeat

Metricbeat is a lightweight shipper installed on a server to periodically collect metrics from the host and services running. This represents the first pillar of observability to monitor our stack.
Metricbeat captures by default system metrics but also includes a large list of modules to capture specific metrics about services such as proxy (NGINX), message bus (RabbitMQ, Kafka), Databases (MongoDB, MySQL, Redis) and many others. First we need to install kube-state-metrics which is a service listening the Kubernetes API to exposes a set of useful metrics about the state of each Object.

```
kubectl create -f 5-1-kube-system-metric.yml
```

Let's deploy metricbeat with :
```
kubectl create -f 5-2-metricbeat.yml
```

# Filebeat

Similarly to Metricbeat, Filebeat requires a settings file to configure the connections to ElasticSearch (endpoint, username, password), the connection to Kibana (to import pre-existing dashboards) and the way to collect and parse logs from each container of the k8s environment.
Let`s deploy filebeat with:
```
kubectl create -f 6-filebeat.yml

```

Finally , on Kibana panel and in the Infrastructure view, the logs are now integrated and can be accessed easily for each pod by clicking on "View logs" on a pod or container.

