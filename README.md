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
stream create --name to_hdfs --definition ":windsock_queue > hdfs --fsUri=hdfs://localhost:9000 --directory=/user/windsock/ --fileName=data --flushTimeout=5 --idleTimeout=5000" --deploy
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

# Results

## Check files in HDFS 
bin/hdfs dfs -ls /user/windsock/
16/05/07 00:51:43 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Found 6 items
-rw-r--r--   3 wmarkito supergroup        966 2016-05-07 00:35 /user/windsock/data-0.txt
-rw-r--r--   3 wmarkito supergroup        477 2016-05-07 00:36 /user/windsock/data-1.txt
-rw-r--r--   3 wmarkito supergroup        159 2016-05-07 00:37 /user/windsock/data-2.txt
-rw-r--r--   3 wmarkito supergroup        158 2016-05-07 00:42 /user/windsock/data-3.txt
-rw-r--r--   3 wmarkito supergroup        159 2016-05-07 00:42 /user/windsock/data-4.txt
-rw-r--r--   3 wmarkito supergroup        954 2016-05-07 00:46 /user/windsock/data-5.t

## Query data in GemFire

gfsh> query --query="select * from /windsock"

Result     : true
startCount : 0
endCount   : 20
Rows       : 5

Result
--------------------------------------------------------------------------------------------------------------------------------------------------------------------
{"wind_speed": 7.325088960678481, "timestamp": 123457, "wind_direction": 6, "longitude": 112.56126350408175, "device_id": "26A", "latitude": 76.8716224029603}
{"wind_speed": 7.325088960678481, "timestamp": 12345, "wind_direction": 6, "longitude": 112.56126350408175, "device_id": "26A", "latitude": 76.8716224029603}
{"wind_speed": 7.325088960678481, "timestamp": 146251559919, "wind_direction": 6, "longitude": 112.56126350408175, "device_id": "26A", "latitude": 76.8716224029603}
{"wind_speed": 7.325088960678481, "timestamp": 149918, "wind_direction": 6, "longitude": 112.56126350408175, "device_id": "26A", "latitude": 76.8716224029603}
{"wind_speed": 7.325088960678481, "timestamp": 123456, "wind_direction": 6, "longitude": 112.56126350408175, "device_id": "26A", "latitude": 76.8716224029603}

# Troubleshooting

## Log sink in SCDF

``` BASH
stream create --name to_log --definition ":windsock_queue > log --keyExpression=#jsonPath(payload,'$.timestamp')" --deploy
```
## jsonPath issues in SCDF

Use a properties file pointing to `1.0.0.BUILD-SNAPSHOT` and re-import all modules.

`module import --uri file:///tmp/apps-starter.properties --force`
