### Introduction

There are some situations in which it is necessary to stream messages into a single topic with different schemas.  Not many but some! For example, transactions requiring time sequenced events may require a single topic with messages conforming to different schemas.  

In this walk through we will create two different schemas in the schema registry, each will have a different subject name and we’ll see how we can use the subject name strategy to validate both of them and push them into a single topic.

First off we need an environment to work with.  What could be better than the Confluent Platform all in one example?  Go and get it here:

[https://github.com/confluentinc/cp-all-in-one](https://github.com/confluentinc/cp-all-in-one)

This will provide pretty much all that is  needed to get up and running with an environment to work in.

Take the following steps:

```
git clone [https://github.com/confluentinc/cp-all-in-one.git](https://github.com/confluentinc/cp-all-in-one.git)

cd cp-all-in-one

sudo docker-compose up -d
```

### Create a topic and start to produce some messages
Connect to the Confluent Control Center (C3) and take a look at your cluster.  Use the IP address of the machine hosting C3 on port 9021.  It should look a bit like this:

![](https://lh3.googleusercontent.com/xiA575qHODNILzTc-YI2GakhkUvTzyQibNZEk3f7wyfeHwLiMO9Oqp7FL8_5jFoMprQUxX_QvHDscLnVNppnSuitW4QMhZuZXFhM2ZaXKfebnSNMXgw_CaSLi-ztWiEOjYvhb1Mf9miYYVGmx0bRB50VxVx91FmXjmyt_PPEacprnnXAfiIxuYEROQ)

  

Click on the cluster and select Topics from the left menu

![](https://lh6.googleusercontent.com/4CtcyIHG5CGxK-bBvrgpGExLUcJmF5T0GiZsGub5haUw7vrOitIaGOUfUCtwuDlWfqOfKVeRQT21PFAmk69aC-BApRR5UgOUM4926n4O64KrBfaCotnfRcNp9lRpeRHsHlcpJ6Hviqob-LW1VHkPyB1YPHyd-O7dC-BtQBnteRLKEgjZHrZ9Myr9YA)

From this view select Add Topic on the left and create a new topic called “transactions”.  Accept the default settings by clicking on “Create with defaults”.

Select the Configuration tab and note the settings.

![](https://lh5.googleusercontent.com/7l0jO_oz2mWxf2QJZNGO2cJhv6JatjtoZ81_9hyLbPhv8-eGGLtSfIKCmU8pUsrQeV47w1NUps40glz1zTyEm2Sjb9Vc_JdDcMc3S_kKNGbUvh9ofSLRn9MOss4H1moR6vhlLW0av0hXkGFxxLAP_v4Wc4ZpUh28ak-0VKN2dXengkMGWzKcSChVCg)

You can see that the confluent.value.schema.validation is set to false and both the confluent.key.subject.name.strategy and the confluent.value.subject.name.strategy are both set to TopicNameStrategy  

This sets schema validation on the broker level and will not be used for this article.  We will be enforcing schema validation on the producer and consumer side.

### Publish 2 different schemas to the schema registry
Publish a schema to the schema registry with the record name of our choosing.  The record name is the friendly identifier for the schema.  You can publish the schema to the registry in multiple ways. There is a Maven plugin but you can also use the Schema Registry API. In this example we will use the latter.  The API details regarding posting a new schema to the registry can be found here:

[https://docs.confluent.io/platform/current/schema-registry/develop/api.html#post--subjects-(string-%20subject)-versions](https://docs.confluent.io/platform/current/schema-registry/develop/api.html#post--subjects-(string-%20subject)-versions)

We will use an Avro schema file (asvc) for our example.  It is shown below.  You should create this as a text file call transactiontype1.asvc

If you have the following asvc file:

```
{
    "type": "record",
    "namespace": "io.confluent.csta.examples",
    "name": "transactiontype1",
    "fields": [
        {"name": "id", "type": "int"},
        {"name": "amount", "type": "double"},
        {"name": "accountname", "type": "string"}
    ]
}
```

Use this command to register it with the schema registry:

```
jq '. | {schema: tojson}' transactiontype1.asvc  | \
curl -X POST http://localhost:8081/subjects/transactiontype1/versions \
         -H "Content-Type:application/json" \
         -d @-
```

Create the second schema now.  Create the following in a file called transactiontype2.asvc


```
{
    "type": "record",
    "namespace": "io.confluent.csta.examples",
    "name": "transactiontype2",
    "fields": [
        {"name": "id", "type": "string"},
        {"name": "amount", "type": "double"},
        {"name": "accounttype", "type": "string"}
    ]
}
```

Use this command to register it with the schema registry:

```
jq '. | {schema: tojson}' transactiontype2.asvc  | \
    curl -X POST http://localhost:8081/subjects/transactiontype2/versions \
         -H "Content-Type:application/json" \
         -d @-
```

In each of the commands to register the schema the key element is highlighted in bold below

``curl -X POST http://localhost:8081/subjects/transactiontype2/versions``

This is what sets the subject name in the registry.

Query the registry and see the schemas are registered

``curl -GET http://localhost:8081/subjects/
  
This will return the following:

``["transactiontype1","transactiontype2"]

Later it will be necessary to produce messages to the topic using these two schemas. To do this we will need the schema ids for each schema. These can be retrieved with the following commands:

``curl -GET http://localhost:8081/subjects/transactiontype1/versions/1 | jq

This will return some JSON that looks like this:

```
{
  "subject": "transactiontype1",
  "version": 1,
  "id": 1,
  "schema": "{\"type\":\"record\",\"name\":\"transactiontype1\",\"namespace\":\"io.confluent.csta.examples\",\"fields\":[{\"name\":\"id\",\"type\":\"int\"},{\"name\":\"amount\",\"type\":\"double\"},{\"name\":\"accountname\",\"type\":\"string\"}]}"
}
```

Make a note of the id (in this case it is 1) and then make the same call again on the second schema type:

``curl -GET http://localhost:8081/subjects/transactiontype2/versions/1 | jq

The output will be something like this:

```
{
  "subject": "transactiontype2",
  "version": 1,
  "id": 2,
  "schema": "{\"type\":\"record\",\"name\":\"transactiontype2\",\"namespace\":\"io.confluent.csta.examples\",\"fields\":[{\"name\":\"id\",\"type\":\"string\"},{\"name\":\"amount\",\"type\":\"double\"},{\"name\":\"accounttype\",\"type\":\"string\"}]}"
}
```
  
Take note of this schema id as well (in this case 2).

### Produce and Consume against the Schema

The setup is now in place to produce the two different types of message to the topic.  First off we will fire up the consumer to pick messages off the topic as they arrive.  Use the following command to consume from the topic whilst validating against the schema registry:

'''sudo docker exec -i schema-registry kafka-avro-console-consumer \
    --topic transactions  \
    --from-beginning \
    --bootstrap-server broker:29092   \
    --property schema.registry.url=http://schema-registry:8081 \
    --property value.subject.name.strategy=io.confluent.kafka.serializers.subject.TopicRecordNameStrategy
'''
  

(The above assumes a containerized environment)

Leave this running in your terminal window and open new terminal window.

Now produce a message to the topic, validated against the first schema type that was created with the following command:  

'''sudo docker exec -i schema-registry kafka-avro-console-producer \
    --topic transactions \
    --broker-list broker:29092 \
    --property value.schema.id=1 \
    --property schema.registry.url=http://schema-registry:8081 \
    --property value.subject.name.strategy=io.confluent.kafka.serializers.subject.TopicRecordNameStrategy
'''
  

This is interactive (the -i option) so you just paste your message to the command line and it will be consumed correctly and validated against the first schemas in the schema registry.  Paste this into the command terminal window:

{"id":1,"amount":23.45,"accountname":"primary"}

If you move over to the terminal window where your consumer is running you should see this message output on the screen as it is consumed.

Now send a message using the format of the second schema.  Issue the following producer command:

```
sudo docker exec -i schema-registry kafka-avro-console-producer \
    --topic transactions \
    --broker-list broker:29092 \
    --property value.schema.id=2 \
    --property schema.registry.url=http://schema-registry:8081 \
    --property value.subject.name.strategy=io.confluent.kafka.serializers.subject.TopicRecordNameStrategy
```

Note that this is now using the schema with id 2.  The id should be a string and the final key value pair will have the name accounttype.  Paste the following in to the terminal as a value message

``{"id":"001","amount":23.4,"accounttype":"premier"}

The consumer will output this in its terminal window indicating that it is valid.

We have now produced to the topic successfully using two different schemas.

Paste the following message into the producer and you should see an exception.  In this example we are using a message that doesn’t conform to schema 2.  The type of the id is not a string and the last key name is invalid (it should be accounttype not accountid).

``{"id":1,"amount":12.0,"accountid":"account1"} 

Note that it is still possible to produce to the topic without enforcing schema validation.  We have only seen in this example enforcement occurring at the producer and consumer level and not at the broker level.  With this configuration there is nothing to stop a rogue consumer or producer interacting with the topic.  Run an ultra simple consumer in a new terminal window with the following command:

```
sudo docker exec broker kafka-console-consumer \
   --topic transactions \
   --bootstrap-server broker:29092 
   --from-beginning 
```

Now produce to the topic with the following command

```
sudo docker exec -i broker kafka-console-producer \
   --topic transactions \
   --broker-list broker:29092
```

And paste in the interactive terminal the following message:

``{"account":"invalidaccount","amount":12.5}

It will still be accepted at the broker level

To enforce the TopicRecordName strategy at the broker level update the settings in C3.

Update the following settings 

  
```
confluent.value.schema.validation = true

confluent.key.subject.name.strategy = io.confluent.kafka.serializers.subject.TopicRecordNameStrategy

confluent.value.subject.name.strategy = io.confluent.kafka.serializers.subject.TopicRecordNameStrategy
```

Edit the settings in “expert mode” to do this:

![](https://lh3.googleusercontent.com/RnFP3H0HLAaz1EOF5AfXpaamYrKz_DtyvnJ2gGV3uIIZXC_3ovB-bqHgWWWb9Anfiml8Ts50-sYW2yISwxnvT_l5IXHOM1whBGaRjf8FmXo0_xd-lE6_6Fo9tUJ56i66RWZ0XM-Jx26jTnSACch8uD4kO7DV1I1qTsERthtLu5JZIqIZCSeIUXDdng)

  

Now repeat the previous command produce to the topic:

```
sudo docker exec -i broker kafka-console-producer \

   --topic transactions \

   --broker-list broker:29092
```

  
Paste in the interactive terminal the following message:

``{"account":"invalidaccount","amount":12.5}

An exception is thrown.

However, repeating previous commands that validate against the schema registry will still work.  Run the consumer again (dropping the from beginning property as this could well cause issues now).

```
sudo docker exec -i schema-registry kafka-avro-console-consumer \

    --topic transactions  \

    --bootstrap-server broker:29092   \

    --property schema.registry.url=http://schema-registry:8081 \

    --property value.subject.name.strategy=io.confluent.kafka.serializers.subject.TopicRecordNameStrategy
```


And then produce again to the topic with the following command:

```
sudo docker exec -i schema-registry kafka-avro-console-producer \

    --topic transactions \

    --broker-list broker:29092 \

    --property value.schema.id=1 \

    --property schema.registry.url=http://schema-registry:8081 \

    --property value.subject.name.strategy=io.confluent.kafka.serializers.subject.TopicRecordNameStrategy
```

And send this message:

``{"id":1,"amount":23.45,"accountname":"primary"}

This will be processed by the cluster and the consumer.
