---
published: false
layout: post
---

## A New Post\n\nEnter text in [Markdown](http://daringfireball.net/projects/markdown/). Use the toolbar above, or click the **?** button for formatting help.
# Setup #

Before we start querying druid, we're going to finish setting up a complete cluster on localhost. In our previous posts, we setup a Realtime node. In this tutorial we will also setup the other Druid node types: Compute, Master and Broker.

## Booting a Broker Node ##

1. Setup a config file at config/broker/runtime.properties that looks like this: [https://gist.github.com/rjurney/5818837](https://gist.github.com/rjurney/5818837)
2. Run the broker node:
```bash
java -Xmx256m -Duser.timezone=UTC -Dfile.encoding=UTF-8 \
-Ddruid.realtime.specFile=realtime.spec \
-classpath services/target/druid-services-0.5.6-SNAPSHOT-selfcontained.jar:config/broker \
com.metamx.druid.http.BrokerMain
```

## Booting a Master Node ##

1. Setup a config file at config/master/runtime.properties that looks like this: [https://gist.github.com/rjurney/5818870](https://gist.github.com/rjurney/5818870)
2. Run the master node:
```bash
java -Xmx256m -Duser.timezone=UTC -Dfile.encoding=UTF-8 \
-classpath services/target/druid-services-0.5.6-SNAPSHOT-selfcontained.jar:config/master \
com.metamx.druid.http.MasterMain
```

## Booting a Realtime Node ##

1. Setup a config file at config/realtime/runtime.properties that looks like this: [https://gist.github.com/rjurney/5818774](https://gist.github.com/rjurney/5818774)

2. Setup a realtime.spec file like this: [https://gist.github.com/rjurney/5818779](https://gist.github.com/rjurney/5818779)
3. Run the realtime node:
```bash
java -Xmx256m -Duser.timezone=UTC -Dfile.encoding=UTF-8 \
-Ddruid.realtime.specFile=realtime.spec \
-classpath services/target/druid-services-0.5.6-SNAPSHOT-selfcontained.jar:config/realtime \
com.metamx.druid.realtime.RealtimeMain
```

## Booting a Compute Node ##

1. Setup a config file at config/compute/runtime.properties that looks like this: [https://gist.github.com/rjurney/5818885](https://gist.github.com/rjurney/5818885)
2. Run the compute node:
```bash
java -Xmx256m -Duser.timezone=UTC -Dfile.encoding=UTF-8 \
-classpath services/target/druid-services-0.5.6-SNAPSHOT-selfcontained.jar:config/compute \
com.metamx.druid.http.ComputeMain
```

# Querying Your Data #

Now that we have a complete cluster setup on localhost, we need to load data. To do so, refer to [Loading Your Data](previous post). Having done that, its time to query our data!

## Querying Different Nodes ##

As a shared-nothing system, there are three ways to query druid, against the Realtime, Compute or Broker node. Querying a Realtime node returns only realtime data, querying a compute node returns only historical segments. Querying the broker will query both realtime and compute segments and compose an overall result for the query. This is the normal mode of operation for queries in druid.

### Construct a Query ###

For constructing this query, see below at: Querying Against the realtime.spec
```json
{
    "queryType": "groupBy",
    "dataSource": "druidtest",
    "granularity": "all",
    "dimensions": [],
    "aggregations": [
        {"type": "count", "name": "rows"},
        {"type": "longSum", "name": "imps", "fieldName": "impressions"},
        {"type": "doubleSum", "name": "wp", "fieldName": "wp"}
    ],
    "intervals": ["2010-01-01T00:00/2020-01-01T00"]
}
```

### Querying the Realtime Node ###

Run our query against port 8080:
```bash
curl -X POST "http://localhost:8080/druid/v2/?pretty" \
-H 'content-type: application/json' -d @query.body
```
See our result:
```json
[ {
  "version" : "v1",
  "timestamp" : "2010-01-01T00:00:00.000Z",
  "event" : {
    "imps" : 5,
    "wp" : 15000.0,
    "rows" : 5
  }
} ]
```

### Querying the Compute Node ###
Run the query against port 8082:
```bash
curl -X POST "http://localhost:8082/druid/v2/?pretty" \
-H 'content-type: application/json' -d @query.body
```
And get (similar to):
```json
[ {
  "version" : "v1",
  "timestamp" : "2010-01-01T00:00:00.000Z",
  "event" : {
    "imps" : 27,
    "wp" : 77000.0,
    "rows" : 9
  }
} ]
```
### Querying both Nodes via the Broker ###
Run the query against port 8083:
```bash
curl -X POST "http://localhost:8083/druid/v2/?pretty" \
-H 'content-type: application/json' -d @query.body
```
And get:
```json
[ {
  "version" : "v1",
  "timestamp" : "2010-01-01T00:00:00.000Z",
  "event" : {
    "imps" : 5,
    "wp" : 15000.0,
    "rows" : 5
  }
} ]
```

Now that we know what nodes can be queried (although you should usually use the broker node), lets learn how to know what queries are available.

## Querying Against the realtime.spec ##

How are we to know what queries we can run? Although [Querying](https://github.com/metamx/druid/wiki/Querying) is a helpful index, to get a handle on querying our data we need to look at our Realtime node's realtime.spec file:

```json
[{
  "schema" : { "dataSource":"druidtest",
               "aggregators":[ {"type":"count", "name":"impressions"},
                                  {"type":"doubleSum","name":"wp","fieldName":"wp"}],
               "indexGranularity":"minute",
           "shardSpec" : { "type": "none" } },
  "config" : { "maxRowsInMemory" : 500000,
               "intermediatePersistPeriod" : "PT10m" },
  "firehose" : { "type" : "kafka-0.7.2",
                 "consumerProps" : { "zk.connect" : "localhost:2181",
                                     "zk.connectiontimeout.ms" : "15000",
                                     "zk.sessiontimeout.ms" : "15000",
                                     "zk.synctime.ms" : "5000",
                                     "groupid" : "topic-pixel-local",
                                     "fetch.size" : "1048586",
                                     "autooffset.reset" : "largest",
                                     "autocommit.enable" : "false" },
                 "feed" : "druidtest",
                 "parser" : { "timestampSpec" : { "column" : "utcdt", "format" : "iso" },
                              "data" : { "format" : "json" },
                              "dimensionExclusions" : ["wp"] } },
  "plumber" : { "type" : "realtime",
                "windowPeriod" : "PT10m",
                "segmentGranularity":"hour",
                "basePersistDirectory" : "/tmp/realtime/basePersist",
                "rejectionPolicy": {"type": "messageTime"} }

}]
```

### dataSource ###

```json
"dataSource":"druidtest"
```
Our dataSource tells us the name of the relation/table, or 'source of data', to query in both our realtime.spec and query.body!

### aggregations ###

Note the [Aggregations](https://github.com/metamx/druid/wiki/Aggregations) in our query:

```json
    "aggregations": [
        {"type": "count", "name": "rows"},
        {"type": "longSum", "name": "imps", "fieldName": "impressions"},
        {"type": "doubleSum", "name": "wp", "fieldName": "wp"}
    ],
```

this matches up to the aggregators in the schema of our realtime.spec!

```json
"aggregators":[ {"type":"count", "name":"impressions"},
                                  {"type":"doubleSum","name":"wp","fieldName":"wp"}],
```

### dimensions ###

Lets look back at our actual records (from [Loading Your Data](add link):

```json
{"utcdt": "2010-01-01T01:01:01", "wp": 1000, "gender": "male", "age": 100}
{"utcdt": "2010-01-01T01:01:02", "wp": 2000, "gender": "female", "age": 50}
{"utcdt": "2010-01-01T01:01:03", "wp": 3000, "gender": "male", "age": 20}
{"utcdt": "2010-01-01T01:01:04", "wp": 4000, "gender": "female", "age": 30}
{"utcdt": "2010-01-01T01:01:05", "wp": 5000, "gender": "male", "age": 40}
```

Note that we have two dimensions to our data, other than our primary metric, wp. They are 'gender' and 'age'. We can specify these in our query! Note that we have added a dimension: age, below.

```json
{
    "queryType": "groupBy",
    "dataSource": "druidtest",
    "granularity": "all",
    "dimensions": ["age"],
    "aggregations": [
        {"type": "count", "name": "rows"},
        {"type": "longSum", "name": "imps", "fieldName": "impressions"},
        {"type": "doubleSum", "name": "wp", "fieldName": "wp"}
    ],
    "intervals": ["2010-01-01T00:00/2020-01-01T00"]
}
```

Which gets us grouped data in return!

```json
[ {
  "version" : "v1",
  "timestamp" : "2010-01-01T00:00:00.000Z",
  "event" : {
    "imps" : 1,
    "age" : "100",
    "wp" : 1000.0,
    "rows" : 1
  }
}, {
  "version" : "v1",
  "timestamp" : "2010-01-01T00:00:00.000Z",
  "event" : {
    "imps" : 1,
    "age" : "20",
    "wp" : 3000.0,
    "rows" : 1
  }
}, {
  "version" : "v1",
  "timestamp" : "2010-01-01T00:00:00.000Z",
  "event" : {
    "imps" : 1,
    "age" : "30",
    "wp" : 4000.0,
    "rows" : 1
  }
}, {
  "version" : "v1",
  "timestamp" : "2010-01-01T00:00:00.000Z",
  "event" : {
    "imps" : 1,
    "age" : "40",
    "wp" : 5000.0,
    "rows" : 1
  }
}, {
  "version" : "v1",
  "timestamp" : "2010-01-01T00:00:00.000Z",
  "event" : {
    "imps" : 1,
    "age" : "50",
    "wp" : 2000.0,
    "rows" : 1
  }
} ]
```

### filtering ###

Now that we've observed our dimensions, we can also filter:

```json
{
    "queryType": "groupBy",
    "dataSource": "druidtest",
    "granularity": "all",
    "filter": {
        "type": "selector",
        "dimension": "gender",
        "value": "male"
    },
    "aggregations": [
        {"type": "count", "name": "rows"},
        {"type": "longSum", "name": "imps", "fieldName": "impressions"},
        {"type": "doubleSum", "name": "wp", "fieldName": "wp"}
    ],
    "intervals": ["2010-01-01T00:00/2020-01-01T00"]
}
```

Which gets us just people aged 40:

```json
[ {
  "version" : "v1",
  "timestamp" : "2010-01-01T00:00:00.000Z",
  "event" : {
    "imps" : 3,
    "wp" : 9000.0,
    "rows" : 3
  }
} ]
```

Check out [Filters](https://github.com/metamx/druid/wiki/Filters) for more.

## Learn More ##

Finally, you can learn more about querying at [Querying](https://github.com/metamx/druid/wiki/Querying)!
