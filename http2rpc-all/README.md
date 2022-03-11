# http2rpc

Code is available here https://github.com/stm-labs/http2rpc

In firewall you must open ports below from Inside to iDMZ:

 - to redis nodes: 22000 (redis port)
 - to redis nodes: 22001 (redis sentinel port)
 - to kafka nodes: 31190,31191,31192

## Kafka and Redis deploy

### Deploy Redis

(Sentinel)

```bash
RELEASE_PLATFORM_NAME=st-$NAMESPACE
RELEASE_COMPONENT_NAME=comp-${NAMESPACE}
RELEASE_REDIS_NAME=redisha-${NAMESPACE}
REDIS_SENTINEL_PORT=22001
```

```bash
helm upgrade -i --namespace ${NAMESPACE} \
 --set "redis.redisHostPort=22000" \
 --set "redis.sentinelHostPort=$REDIS_SENTINEL_PORT" \
 --set-string nodeSelector."SOME_DMZ_POOL_LABEL"=true \
 --atomic \
 --debug \
 --wait \
 $RELEASE_REDIS_NAME \
 ./redis-ha
```

### Deploy kafka

You can use any kafka - below - are confluence charts for cp kafka 5.2.2

(3 brokers)

you should change 31190 to your ports pool

```bash
helm upgrade --install --namespace ${NAMESPACE} \
    --atomic \
    --set 'cp-schema-registry.enabled=false' \
    --set 'cp-kafka-connect.enabled=false' \
    --set 'cp-kafka-rest.enabled=false' \
    --set 'cp-ksql-server.enabled=false' \
    --set 'cp-control-center.enabled=false' \
    --set cp-kafka.configurationOverrides."auto\.create\.topics\.enable"=false \
    --set cp-kafka.configurationOverrides."offsets\.topic\.replication\.factor"=3 \
    --set cp-kafka.configurationOverrides."auto\.create\.topics\.enable"=false \
    --set-string cp-kafka.configurationOverrides."message\.max\.bytes"="10485760" \
    --set-string cp-kafka.configurationOverrides."replica\.fetch\.max\.bytes"="10485760" \
    --set 'cp-kafka.persistence.enabled=true' \
    --set 'cp-kafka.customEnv.KAFKA_LOG4J_LOGGERS=kafka.controller=WARN,state.change.logger=WARN' \
    --set 'cp-kafka.customEnv.KAFKA_LOG4J_ROOT_LOGLEVEL=WARN' \
    --set 'cp-kafka.customEnv.KAFKA_TOOLS_LOG4J_LOGLEVEL=ERROR' \
    --set 'cp-kafka.brokers=3' \
    --set 'cp-kafka.nodeport.enabled=true' \
    --set 'cp-kafka.nodeport.firstListenerPort=31190' \
    --set 'cp-zookeeper.servers=3' \
    --wait \
    $RELEASE_COMPONENT_NAME \
    ./cp-helm-charts
```

## http2rpc

You should deploy pair of idmz/inside component for each connection.
In http2rpc/inside you should set "stm.proxyUrl"
In http2rpc/inside and http2rpc/idmz you should set "stm.systemCode" to globally uniq name of idmz/inside (that would be kafka topic, so you don't want to share it between different pair)

By default redis sentinel (HA) is used. But you can downgrate to signle Redis instance - just set 
- "--set stm.rpcDmzRedisSentinelEnable=false" and 
- "--set stm.rpcDmzRedis=some-redis-host"
- also you can set port via stm.rpcDmzRedisPort (6379 by default)

### Deploy Inside

```bash
RELEASE_PLATFORM_NAME_INSIDE=http2rpc-inside-auth
SYSTEM_CODE=someSystemComponentAuth

RPC_BUILD=25
RELEASE_COMPONENT_NAME=comp-stage
REDIS_SENTINEL_PORT=22001
NAMESPACE='stage-component-inside'
```

```bash
helm upgrade -i --namespace ${NAMESPACE} \
--atomic \
--set "image.build=$RPC_BUILD" \
--set "stm.systemCode=$SYSTEM_CODE" \
--set "stm.proxyUrl=http://changeme-to-address.$NAMESPACE" \
--set "stm.rpcDmzRedisSentinelEnable=true" \
--set "stm.rpcDmzRedisNodes=redis1.local:$REDIS_SENTINEL_PORT\,redis1.local.local:$REDIS_SENTINEL_PORT\,redis1.local.local:$REDIS_SENTINEL_PORT" \
--set "stm.rpcDmzKafka=kafkaNode1:31190,kafkaNode2:31191,kafkaNode3:31191" \
--wait \
$RELEASE_PLATFORM_NAME_INSIDE \
./http2rpc/inside
```

## Deploy idmz

```bash
RELEASE_PLATFORM_NAME_DMZ=http2rpc-dmz-auth
SYSTEM_CODE=someSystemComponentAuth
NAMESPACE_IDMZ='stage-component-idmz'
```

```bash
helm upgrade -i --namespace ${NAMESPACE_IDMZ} \
--atomic \
--set "image.build=$RPC_BUILD" \
--set "stm.systemCode=$SYSTEM_CODE" \
--set "stm.rpcDmzRedisSentinelEnable=true" \
--set "stm.rpcDmzRedisNodes=redis1.local:$REDIS_SENTINEL_PORT\,redis1.local.local:$REDIS_SENTINEL_PORT\,redis1.local.local:$REDIS_SENTINEL_PORT" \
--set "stm.rpcDmzKafka=$RELEASE_COMPONENT_NAME-cp-kafka.stage:9092" \
--wait \
$RELEASE_PLATFORM_NAME_DMZ \
./http2rpc/dmz
```