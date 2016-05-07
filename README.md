# All The Things - Pivotal Data

## GemFire

Open `gfsh` and execute:

``` BASH
start locator --name=locator1
start server --name=server1 --J=-Dgemfire.start-dev-rest-api=true --J=-Dgemfire.http-service-port=8181 --server-port=0
start server --name=server2 --J=-Dgemfire.start-dev-rest-api=true --J=-Dgemfire.http-service-port=8282 --server-port=0
create region --name=windsock --type=PARTITION
```

If using the PCF Tile, skip this step but you will need the locator IP and PORT information.

## SCDF streams

### Setup

1. Kafka & Redis
1. Start local SCDF server
(`java -jar spring-cloud-dataflow-server-local-1.0.0.BUILD-SNAPSHOT.jar`)
1. Start SCDF shell and create streams (`java -jar spring-cloud-dataflow-shell-1.0.0.BUILD-SNAPSHOT.jar`)

Details here http://cloud.spring.io/spring-cloud-dataflow/

### Creating streams

#### Ingestion endpoint

```
stream create --name http_server --definition "http --server.port=8888 --spring.cloud.stream.bindings.output.contentType='application/json' > :windsock_queue" --deploy
```

#### GemFire

```
stream create --name to_gemfire --definition ":windsock_queue > gemfire --useLocator=true --hostAddresses='localhost:10334'  --keyExpression=#jsonPath(payload,'$.timestamp') --regionName='windsock'" --deploy
```

#### HDFS

```
stream create --name to_hdfs-2 --definition ":windsock_queue > hdfs --fsUri=hdfs://localhost:9000 --directory=/user/windsock/ --fileName=data --flushTimeout=5 --idleTimeout=5000" --deploy
```

* Change namenode to your host:port information.
* For large volumes we need to consider partitioning in HDFS.  

# Sample data

``` JSON
{"wind_speed": 7.325088960678481, "timestamp": 1462555989, "wind_direction": 6, "longitude": 112.56126350408175, "device_id": "26A", "latitude": 76.8716224029603}'
```

# Sample request

``` BASH
curl -H "Content-Type: application/json" -X POST -d '{"wind speed": 7.325088960678481, "timestamp": 1462555989, "wind direction": 6, "longitude": 112.56126350408175, "device id": "26A", "latitude": 76.8716224029603}' http://localhost:8888
```

# Troubleshooting

## Log sink in SCDF

``` BASH
stream create --name to_log --definition ":windsock_queue > log --keyExpression=#jsonPath(payload,'$.timestamp')" --deploy
```
## jsonPath issues in SCDF

Use a properties file pointing to `1.0.0.BUILD-SNAPSHOT` and re-import all modules.

`module import --uri file:///tmp/apps-starter.properties --force`
