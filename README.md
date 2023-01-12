## Why partition rebalance needed?

When we add or remove new servers, kafka topics do not automatically be assigned any data partitions, so unless partitions are moved to them they won't be doing any work until new topics are created. Therefore, you will need to migrate some existing data to these machines when you modify cluster nodes.

This document aims to cover the detail steps in establishing a new Kafka nodes.

## How to rebalance?

During a rebalance, consumers are unable to access messages, thus we can say that rebalance is a short window of unavailability to entire consumer group. 

First, we need to detect zookeeper host

```
sudo egrep -r zookeeper /opt/kafka/config/
/opt/kafka/config/server.properties:zookeeper.connect=zookeeper001:2181,zookeeper002:2181,zookeeper003:2181,zookeeper004:2181,zookeeper004:2181/env-kafkacluter/kafka
```


Export zookeeper host for later use

```
export ZK_HOSTS="zookeeper001:2181/env-kafkacluter/kafka"
```

Get all topics in cluster

```
echo '{"topics": [' > topics.json && /opt/kafka/bin/kafka-topics.sh --zookeeper $ZK_HOSTS --list | perl -ne 'chomp;print "{\"topic\": \"$_\"},\n"' >> topics.json && truncate --size=-2 topics.json && echo '],"version":1}' >> topics.json
```

Change your `kafka-topics.sh` path.

Generate partition reassignment configuration for 5 brokers.

```
/opt/kafka/bin/kafka-reassign-partitions.sh --zookeeper ${ZK_HOSTS} --topics-to-move-json-file topics.json --broker-list "1,2,3,4,5" --generate  | tee >(awk -F: '/Current partition replica assignment/ { getline; print $0 }' | jq > /tmp/topic.current.json) >(awk -F: '/Proposed partition reassignment configuration/ { getline; print $0 }' | jq > /tmp/topic.proposed.json)
```

We can check status of brokers before and after changing

```
/tmp/topic.current.json
/tmp/topic.proposed.json
```

Execute

```
/opt/kafka/bin/kafka-reassign-partitions.sh --zookeeper $ZK_HOSTS --reassignment-json-file /tmp/topic.proposed.json --execute
```

Verify if the reassignment process.

```
/opt/kafka/bin/kafka-reassign-partitions.sh --zookeeper $ZK_HOSTS --reassignment-json-file /tmp/topic.proposed.json --verify
```

Then verify number of Brokers are online.

Random check one topic

```
kafkactl describe topic topic1
```
