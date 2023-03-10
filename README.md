# Setting up ES Rally


# Setup local Elasticsearch cluster

## For local use: set up minikube

```
minikube start -p es-minikube --cpus=6 --memory=12g
minikube profile es-minikube
```

Test the installation:
```
minikube kubectl -- get pods -A
```

## Deploy Elastic with Helm

### Deploy Elasticsearch

```
helm repo add elastic https://helm.elastic.co
helm install elasticsearch elastic/elasticsearch --values elasticsearch-values.yaml
```
Test installation:
```
kubectl port-forward elasticsearch-master-0 9200 &
curl http://localhost:9200
```

Should give:
```json
{
  "name" : "elasticsearch-master-0",
  "cluster_name" : "elasticsearch",
  "cluster_uuid" : "O5RGPFe4SE-0fwtnAJL6PQ",
  "version" : {
    "number" : "7.17.5",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "8d61b4f7ddf931f219e3745f295ed2bbc50c8e84",
    "build_date" : "2022-06-23T21:57:28.736740635Z",
    "build_snapshot" : false,
    "lucene_version" : "8.11.1",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

### Deploy Kibana

```
helm install kibana elastic/kibana --values kibana-values.yaml
```
Test:
```
kubectl port-forward $(kubectl get pods --namespace=default -l release=kibana -ojsonpath='{.items[0].metadata.name}') 5601
```
and open http://localhost:5601


## Configure Stack monitoring

```
helm install stack-monitoring-metricbeat elastic/metricbeat --values metricbeat-values.yaml
```

# Ready to play with ES Rally

## Launch geonames race

```
kubectl apply -f esrally-geonames-job.yaml
```

Follow activity:
```
kubectl logs -f $(kubectl get pods --namespace=default -l job-name=esrally-geonames -ojsonpath='{.items[0].metadata.name}')
```

```

    ____        ____
   / __ \____ _/ / /_  __
  / /_/ / __ `/ / / / / /
 / _, _/ /_/ / / / /_/ /
/_/ |_|\__,_/_/_/\__, /
                /____/

[INFO] Downloading track data (20.5 kB total size)                                [100.0%]
[INFO] Decompressing track data from [/rally/.rally/benchmarks/data/geonames/documents-2-1k.json.bz2] to [/rally/.rally/benchmarks/data/geonames/documents-2-1k.json] ... [OK]
[INFO] Preparing file offset table for [/rally/.rally/benchmarks/data/geonames/documents-2-1k.json] ... [OK]
[WARNING] merges_total_time is 5534 ms indicating that the cluster is not in a defined clean state. Recorded index time metrics may be misleading.
[WARNING] indexing_total_time is 8297 ms indicating that the cluster is not in a defined clean state. Recorded index time metrics may be misleading.
[WARNING] refresh_total_time is 22797 ms indicating that the cluster is not in a defined clean state. Recorded index time metrics may be misleading.
Running delete-index                                                           [100% done]
.
.
.
|                                                     error rate |             asc_sort_geonameid |     0          |       % |
|                                                 Min Throughput |  asc_sort_with_after_geonameid |    69.63       |   ops/s |
|                                                Mean Throughput |  asc_sort_with_after_geonameid |    69.63       |   ops/s |
|                                              Median Throughput |  asc_sort_with_after_geonameid |    69.63       |   ops/s |
|                                                 Max Throughput |  asc_sort_with_after_geonameid |    69.63       |   ops/s |
|                                       100th percentile latency |  asc_sort_with_after_geonameid |    19.6687     |      ms |
|                                  100th percentile service time |  asc_sort_with_after_geonameid |     4.75491    |      ms |
|                                                     error rate |  asc_sort_with_after_geonameid |     0          |       % |


[INFO] Race id is [c74f0153-625d-43de-bc86-80de00e63505]

--------------------------------
[INFO] SUCCESS (took 63 seconds)
--------------------------------
```

## Persist results in Elasticsearch

Results are persisted in /root/.rally directory by default. We can tell ES Rally to store results in Elasticsearch.

### Deploy dedicated metricstore

```
helm install metricstore-elasticsearch elastic/elasticsearch --values metricstore-elasticsearch-values.yaml
helm install metricstore-kibana elastic/kibana --values metricstore-kibana-values.yaml
```

```
kubectl port-forward $(kubectl get pods --namespace=default -l release=metricstore-kibana -ojsonpath='{.items[0].metadata.name}') 15601:5601
```

Open http://localhost:15601

### Update benchmark.ini

```
kubectl apply -f esrally-geonames-metricstore-job.yaml
kubectl logs -f $(kubectl get pods --namespace=default -l job-name=esrally-geonames-metricstore -ojsonpath='{.items[0].metadata.name}')
```

