# Elasticsearch, Logstash, Kibana (ELK) Docker image

Maintained by Hatch Baby. Forked from https://github.com/spujadas/elk-docker.

Changes made to this image from the origin:
- activate the TCP input logstash plugin, so that it works with [The Logstash Logback Appender](https://github.com/logstash/logstash-logback-encoder).
- DISABLE ECS compatibility. This was done to preserve compatibility with our previous schema. To re-enable, edit 01-lumberjack-input.conf and 30-output.conf
- change the output format back to the non-Filebeats pattern
- add a GeoIp database and activate the transformations in the input.
- Customize to allow passing in the ES\_MAX\_MEM parameter thru Docker environment vars
- Add the repository-s3 plugin for backups
- enable CORS for localhost to allow elasticsearch-HQ plugin to work (elasticsearch.yml)

In order to test the image, you'll want to create at least one record in the ES index to enable Kibana to load.  You can use this:
    
    sysctl -w vm.max_map_count=262144
    docker run -d -p 5601:5601 -p 5000:5000 --name elk --ulimit nofile=65536:65536 kenwdelong/elk-docker:latest
    nc -w 3 localhost 5000 < ./test/test.json
    
This will create the logstash index and allow Kibana to work.  Point your browser at port 5601.

To specify data without using a file, try (fix the timestamp):

    echo '{"@timestamp": "2016-07-18T14:05:00.000+00:00", "@version": 1, "message": "this is a cmd line test" }' |  nc -w 3 localhost 5000

In order to output logstash activity to /var/log/logstash/logstash.stdout, add this line to /etc/logstash/conf.d/30-output.conf 

    stdout { codec => rubydebug }

---

## Cluster / Multi-Node Configuration

### Service-disable flags

Set any of these to `0` to skip starting that service (useful for dedicated nodes):

| Variable | Default | Effect when set to `0` |
|---|---|---|
| `ELASTICSEARCH_START` | `1` | Skip Elasticsearch |
| `LOGSTASH_START` | `1` | Skip Logstash |
| `KIBANA_START` | `1` | Skip Kibana |

### Elasticsearch cluster env vars

When `ES_CLUSTER_NAME` or `ES_SEED_HOSTS` is set, `start.sh` surgically rewrites only the cluster-topology keys in `elasticsearch.yml`; all other settings from the image are preserved.

| Variable | elasticsearch.yml key | Default |
|---|---|---|
| `ES_CLUSTER_NAME` | `cluster.name` | `elasticsearch` |
| `ES_NODE_NAME` | `node.name` | `elk` |
| `ES_PUBLISH_HOST` | `network.publish_host` | *(not set)* |
| `ES_SEED_HOSTS` | `discovery.seed_hosts` | *(not set)* — comma-separated list |
| `ES_INITIAL_MASTER_NODES` | `cluster.initial_master_nodes` | *(not set)* — bootstrap only, remove after cluster is green |
| `ES_DATA_PATH` | `path.data` | `/data/elasticsearch` |
| `ES_HEAP_SIZE` | JVM heap (`-Xmx`/`-Xms`) | *(image default)* |

### Logstash output env vars

`30-output.conf` uses Logstash's native `${VAR:default}` substitution. Set all three to point Logstash at an external cluster; when unset they default to `localhost:9200` (single-node / staging behaviour unchanged).

| Variable | Default |
|---|---|
| `LS_ES_HOST_1` | `localhost:9200` |
| `LS_ES_HOST_2` | `localhost:9200` |
| `LS_ES_HOST_3` | `localhost:9200` |

### Kibana env vars

| Variable | Effect |
|---|---|
| `KIBANA_ES_HOSTS` | Comma-separated `http://host:port` list written into `kibana.yml` as `elasticsearch.hosts`. When unset Kibana defaults to `localhost:9200`. |

### Resource recommendations

| Role | Instance | `--memory` |
|---|---|---|
| Data node (ES only) | r8i.2xlarge (64 GB) | `60g` |
| Misc host (Logstash + Kibana) | m7i.2xlarge (32 GB) | `28g` |

### Example: data node (Elasticsearch only)

```bash
docker run -d \
  --name elk-data \
  --memory=60g \
  --ulimit nofile=65536:65536 \
  -p 9200:9200 -p 9300:9300 \
  -v /data/elasticsearch:/data/elasticsearch \
  -e LOGSTASH_START=0 \
  -e KIBANA_START=0 \
  -e ES_HEAP_SIZE=31g \
  -e ES_CLUSTER_NAME=hatch-elk \
  -e ES_NODE_NAME=elk-data-1 \
  -e ES_PUBLISH_HOST=elk-data-1.hatch.corp \
  -e ES_SEED_HOSTS=elk-data-1.hatch.corp,elk-data-2.hatch.corp,elk-data-3.hatch.corp \
  -e ES_DATA_PATH=/data/elasticsearch \
  hatch-baby/elk-docker:ELK-9.1.3
```

### Example: misc host (Logstash + Kibana only)

```bash
docker run -d \
  --name elk-misc \
  --memory=28g \
  --ulimit nofile=65536:65536 \
  -p 5000:5000 -p 5000:5000/udp -p 5044:5044 -p 5601:5601 \
  -v /opt/kibana/config/kibana.yml:/opt/kibana/config/kibana.yml \
  -e ELASTICSEARCH_START=0 \
  -e LS_HEAP_SIZE=6g \
  -e LS_ES_HOST_1=elk-data-1.hatch.corp:9200 \
  -e LS_ES_HOST_2=elk-data-2.hatch.corp:9200 \
  -e LS_ES_HOST_3=elk-data-3.hatch.corp:9200 \
  -e ELASTICSEARCH_URL=http://elk-data-1.hatch.corp:9200 \
  hatch-baby/elk-docker:ELK-9.1.3
```
