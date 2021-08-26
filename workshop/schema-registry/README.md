# UNDER CONSTRUCTION

# 0 - Setup

docker-compose --env-file docker-compose-ccloud.env -f docker-compose-schema-registry.yml up -d

export PATH="$PATH:<path to CP bin folder>"

# 1 - Create Topics

	ccloud kafka topic create avro_topic1 --partitions 1;
	ccloud kafka topic create avro_topic2 --partitions 1;
	ccloud kafka topic create avro_topic3 --partitions 1;

	ccloud kafka topic create proto_topic1 --partitions 1;
	ccloud kafka topic create proto_topic2 --partitions 1;
	ccloud kafka topic create proto_topic3 --partitions 1;

	ccloud kafka topic create json_topic1 --partitions 1;
	ccloud kafka topic create json_topic2 --partitions 1;
	ccloud kafka topic create json_topic3 --partitions 1;

	ccloud kafka topic create multi_event --partitions 1;

# 2 - List Schema Subjects

	curl -s -X GET localhost:8081/subjects | jq

# 3 - Add Schemas

Note: if no schemaType is supplied, schemaType is assumed to be AVRO.


## AVRO


	curl -X POST -H "Content-Type: application/json" -d \
	   "{\"schemaType\":\"AVRO\", \
	     \"schema\": $(jq '.|tostring' ./avro/avro_topic1.v1.avsc)}" \
	   "http://localhost:8081/subjects/avro_topic1-value/versions" | jq;

View schemas:

	curl -s -X GET localhost:8081/subjects | jq

Consume:

	kafka-avro-console-consumer \
	   --topic avro_topic1 \
	   --from-beginning \
	   --bootstrap-server pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --consumer.config producer.config \
	   --property value.schema.id=1 \
	   #--property value.schema.file=./avro/avro_topic1.v1.avsc \
	   --property schema.registry.url=http://localhost:8081

Produce:

	kafka-avro-console-producer \
	   --topic avro_topic1 \
	   --broker-list pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --producer.config producer.config \
	   --property value.schema.id=1 \
	   #--property value.schema.file=./avro/avro_topic1.v1.avsc \
	   --property schema.registry.url=http://localhost:8081

Test:

	GOOD: {"customer_id":4,"first_name":"John","last_name":"Doe","favourite_colour":"black"}
	BAD:  {"f1":2}


## JSON SCHEMA


	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"JSON\", \"schema\": $(jq '.|tostring' ./json/json_topic1.v1.json)}" "http://localhost:8081/subjects/json_topic1-value/versions" | jq;

#-- VIEW SCHEMAS

	curl -s -X GET localhost:8081/subjects/json_topic1-value/versions/latest | jq
	curl -X GET http://localhost:8081/schemas/ids/2 | jq

#-- CONSUME

	kafka-json-schema-console-consumer \
	   --topic json_topic1 \
	   --from-beginning \
	   --bootstrap-server pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --consumer.config producer.config \
	   #--property value.schema=$(jq '.|tostring' ./json/json_topic1.v2.json) \
	   --property value.schema.id=2 \
	   --property schema.registry.url=http://localhost:8081

#-- PRODUCE

	kafka-json-schema-console-producer \
	   --broker-list pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --property schema.registry.url=http://localhost:8081 \
	   --topic json_topic1 \
	   #--property value.schema=$(jq '.|tostring' ./json/json_topic1.v2.json) \
	   --property value.schema.id=2 \
	   --producer.config producer.config

Test:

	GOOD: {"customer_id":4,"first_name":"John","last_name":"Doe"}
	BAD:  {"f1":2}

## PROTOBUF

	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"PROTOBUF\", \"schema\": \"$(cat ./protobuf/proto_topic1.v1.proto | tr -d '\n' | tr -d '\t')\"}" "http://localhost:8081/subjects/proto_topic1-value/versions" | jq;

View schemas:

	curl -s -X GET localhost:8081/subjects | jq
	curl -s -X GET localhost:8081/subjects/proto_topic1-value/versions/latest | jq
	curl -X GET http://localhost:8081/schemas/ids/3| jq

Consume:

	kafka-protobuf-console-consumer \
	   --topic proto_topic3 \
	   --from-beginning \
	   --bootstrap-server pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --consumer.config producer.config \
	   --property value.schema.id=3 \
	   --property schema.registry.url=http://localhost:8081

Produce:

	kafka-protobuf-console-producer \
	   --topic proto_topic3 \
	   --broker-list pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --producer.config producer.config \
	   --property value.schema.id=3 \
	   --property schema.registry.url=http://localhost:8081

Test:

	GOOD: {"customer_id":4,"first_name":"John","last_name":"Doe","height":179.0}
	BAD:  {"f1":2}

# 4 - Change Compatibility

	curl -s -X GET localhost:8081/config

	curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" --data '{"compatibility": "FORWARD"}' http://localhost:8081/config/json_topic1-value;

	curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" --data '{"compatibility": "FULL"}' http://localhost:8081/config/proto_topic1-value;

	curl -X GET http://localhost:8081/config/json_topic1-value
	curl -X GET http://localhost:8081/config/proto_topic1-value

# 5 - Evolve Schemas

## AVRO - Add Optional Field

Check compatibility:

	curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" --data "{\"schema\": $(jq '.|tostring' ./avro/avro_topic1.v2.avsc)}" http://localhost:8081/compatibility/subjects/avro_topic1-value/versions/latest | jq;

Evolve schema:

	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"AVRO\", \"schema\": $(jq '.|tostring' ./avro/avro_topic1.v2.avsc)}" "http://localhost:8081/subjects/avro_topic1-value/versions" | jq;

View schema:

	curl -s -X GET localhost:8081/subjects/avro_topic1-value/versions/latest | jq
	curl -s -X GET localhost:8081/subjects/avro_topic1-value/versions/1 | jq

Consume(consumer v2):

	kafka-avro-console-consumer \
	   --topic avro_topic1 \
	   --from-beginning \
	   --bootstrap-server pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --consumer.config producer.config \
	   --property value.schema.id=4 \
	   --property schema.registry.url=http://localhost:8081

Produce(producer v1):

	kafka-avro-console-producer \
	   --topic avro_topic1 \
	   --broker-list pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --producer.config producer.config \
	   --property value.schema.id=1 \
	   --property schema.registry.url=http://localhost:8081

Test:

	python3 python-avro-producer.py ./avro/avro_topic1.v1.avsc avro_topic1
	{"customer_id":4,"first_name":"Jane","last_name":"Doe","favourite_colour":"green"}

## JSON - Remove Optional Field

Check compatibility:

	curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" -d "{\"schemaType\":\"JSON\", \"schema\": $(jq '.|tostring' ./json/json_topic1.v2.json)}" http://localhost:8081/compatibility/subjects/json_topic1-value/versions/latest | jq

Evolve schema:

	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"JSON\", \"schema\": $(jq '.|tostring' ./json/json_topic1.v2.json)}" "http://localhost:8081/subjects/json_topic1-value/versions" | jq;

View schema:

	curl -s -X GET localhost:8081/subjects/json_topic1-value/versions/latest | jq
	curl -s -X GET localhost:8081/subjects/json_topic1-value/versions/1 | jq

Consume(consumer v1):

	kafka-json-schema-console-consumer \
	   --topic json_topic1 \
	   --from-beginning \
	   --bootstrap-server pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --consumer.config producer.config \
	   --property value.schema.id=2 \
	   --property schema.registry.url=http://localhost:8081

Produce(producer v2):

	kafka-json-schema-console-producer \
	   --broker-list pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --property schema.registry.url=http://localhost:8081 \
	   --topic json_topic1 \
	   --property value.schema.id=5 \
	   --producer.config producer.config

Test:

	{"customer_id":4,"first_name":"Benny","last_name":"Boy","date_of_birth":"11/03/81"}
	{"customer_id":4,"firstname":"Benny","last_name":"Boy"}

## PROTOBUF - Add and Remove Optional Fields

Check compatibility:

	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"PROTOBUF\", \"schema\": \"$(cat ./protobuf/proto_topic1.v2.proto | tr -d '\n' | tr -d '\t')\"}"  "http://localhost:8081/compatibility/subjects/proto_topic1-value/versions/latest" | jq;

Evolve schema:

	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"PROTOBUF\", \"schema\": \"$(cat ./protobuf/proto_topic1.v2.proto | tr -d '\n' | tr -d '\t')\"}"  "http://localhost:8081/subjects/proto_topic1-value/versions" | jq;

View schema:

	curl -s -X GET localhost:8081/subjects/proto_topic1-value/versions/latest | jq
	curl -s -X GET localhost:8081/subjects/proto_topic1-value/versions/1 | jq

# 6 - Test References

## AVRO

	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"AVRO\", \"schema\": $(jq '.|tostring' ./avro/avro_topic2.avsc)}" "http://localhost:8081/subjects/avro_topic2-value/versions" | jq

	curl -X POST -H "Content-Type: application/json" -d \
	   "{\"schemaType\":\"AVRO\", \
	     \"schema\": $(jq '.|tostring' ./avro/avro_topic3.avsc), \
	     \"references\": \
	      [{\"name\": \"com.example.Customer\",\"subject\":\"avro_topic2-value\",\"version\": 1}]}" "http://localhost:8081/subjects/avro_topic3-value/versions" | jq

	curl -X GET http://localhost:8081/subjects/avro_topic2-value/versions/1/referencedby

	kafka-avro-console-consumer \
	   --topic avro_topic3 \
	   --from-beginning \
	   --bootstrap-server pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --consumer.config producer.config \
	   --property value.schema.id=8 \
	   --property schema.registry.url=http://localhost:8081

	kafka-avro-console-producer \
	   --topic avro_topic3 \
	   --broker-list pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --producer.config producer.config \
	   --property value.schema.id=8 \
	   --property schema.registry.url=http://localhost:8081

	{"customer":{"first_name":"Bill","last_name":"Bob"}}

## PROTOBUF

	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"PROTOBUF\", \"schema\": \"$(cat ./protobuf/proto_topic2.proto | tr -d '\n' | tr -d '\t')\"}" "http://localhost:8081/subjects/proto_topic2-value/versions" | jq

	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"PROTOBUF\", \"schema\": \"$(cat ./protobuf/proto_topic3.proto | tr -d '\n' | tr -d '\t')\",\"references\":[{\"name\": \"proto_topic2.proto\",\"subject\":\"proto_topic2-value\",\"version\": 1}]}" "http://localhost:8081/subjects/proto_topic3-value/versions" | jq

	curl -X GET http://localhost:8081/subjects/proto_topic2-value/versions/1/referencedby

	kafka-protobuf-console-consumer \
	   --topic proto_topic3 \
	   --from-beginning \
	   --bootstrap-server pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --consumer.config producer.config \
	   --property value.schema.id=11 \
	   --property schema.registry.url=http://localhost:8081

	kafka-protobuf-console-producer \
	   --topic proto_topic3 \
	   --broker-list pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --producer.config producer.config \
	   --property value.schema.id=11 \
	   --property schema.registry.url=http://localhost:8081

	{"order_id":"1","customer":{"customer_name":"Ben"}}
	{"orde_id":"1","customer":{"customer_name":"Ben"}}
	{"order_id":"1","customer":{"custome_name":"Ben"}}

## JSON SCHEMA

	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"JSON\", \"schema\": $(jq '.|tostring' ./json/json_topic2.json)}" "http://localhost:8081/subjects/json_topic2-value/versions" | jq

	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"JSON\", \"schema\": $(jq '.|tostring' ./json/json_topic3.json),\"references\":[{\"name\": \"json_topic2.json\",\"subject\":\"json_topic2-value\",\"version\": 1}]}" "http://localhost:8081/subjects/json_topic3-value/versions" | jq

	curl -X GET http://localhost:8081/subjects/json_topic2-value/versions/1/referencedby

	kafka-json-schema-console-consumer \
	   --topic json_topic3 \
	   --from-beginning \
	   --bootstrap-server pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --consumer.config producer.config \
	   --property value.schema.id=13 \
	   --property schema.registry.url=http://localhost:8081

	kafka-json-schema-console-producer \
	   --broker-list pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --property schema.registry.url=http://localhost:8081 \
	   --topic json_topic3 \
	   --property value.schema.id=13 \
	   --producer.config producer.config

	{"f1":1,"customer":{"customerid":2}}
	{"f2":1,"customer":{"customerid":2}}
	{"f1":1,"customer":{"custid":2}}

# 7 - Multiple Topic Schemas

## AVRO

	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"AVRO\", \"schema\": $(jq '.|tostring' ./avro/customer.avsc)}" "http://localhost:8081/subjects/customer/versions" | jq;
	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"AVRO\", \"schema\": $(jq '.|tostring' ./avro/product.avsc)}" "http://localhost:8081/subjects/product/versions" | jq;
	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"AVRO\", \"schema\": $(jq '.|tostring' ./avro/order.avsc)}" "http://localhost:8081/subjects/order/versions" | jq;


	curl -X POST -H "Content-Type: application/json" -d \
	   "{\"schemaType\":\"AVRO\", \
	   \"schema\": $(jq '.|tostring' ./avro/multi_event.avsc), \
	   \"references\":[ \
	      {\"name\": \"com.example.Customer\", \"subject\": \"customer\", \"version\": 1}, \
	      {\"name\": \"com.example.Product\",\"subject\": \"product\",\"version\": 1}, \
	      {\"name\": \"com.example.Order\",\"subject\": \"order\",\"version\": 1}]}" \
	"http://localhost:8081/subjects/multi_event-value/versions" | jq;

	kafka-avro-console-consumer \
	   --topic multi_event \
	   --from-beginning \
	   --bootstrap-server pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --consumer.config producer.config \
	   --property value.schema.id=17 \
	   #--property value.schema.file=multi_event.avsc \
	   --property schema.registry.url=http://localhost:8081

	kafka-avro-console-producer \
	   --topic multi_event \
	   --broker-list pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --producer.config producer.config \
	   --property value.schema.id=17 \
	   #--property value.schema.file=multi_event.avsc \
	   --property schema.registry.url=http://localhost:8081

	{"oneof_type":{"com.example.Customer":{"first_name":"Benny","last_name":"Boy"}}}
	{"oneof_type":{"com.example.Order":{"order_id":2,"order_amount":4}}}
	{"oneof_type":{"com.example.Product":{"product_name":"Efexor","price":4.56}}}

	{"oneof_type":{"product":{"product_name":2,"price":4.56}}
	{"oneof_type":{"com.example.Order":{"order_id":2,"orderamount":4}}

	curl -s -X DELETE "http://localhost:8081/subjects/multi_event-value";
	curl -s -X DELETE "http://localhost:8081/subjects/multi_event-value?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/order";
	curl -s -X DELETE "http://localhost:8081/subjects/order?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/product";
	curl -s -X DELETE "http://localhost:8081/subjects/product?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/customer";
	curl -s -X DELETE "http://localhost:8081/subjects/customer?permanent=true";

## JSON

	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"JSON\", \"schema\": $(jq '.|tostring' ./json/customer.json)}" "http://localhost:8081/subjects/customer/versions" | jq;
	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"JSON\", \"schema\": $(jq '.|tostring' ./json/product.json)}" "http://localhost:8081/subjects/product/versions" | jq;
	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"JSON\", \"schema\": $(jq '.|tostring' ./json/order.json)}" "http://localhost:8081/subjects/order/versions" | jq;

	curl -X POST -H "Content-Type: application/json" -d \
	   "{\"schemaType\":\"JSON\", \
	   \"schema\": $(jq '.|tostring' ./json/multi_event.json), \
	   \"references\":[ \
	      {\"name\": \"customer.json\", \"subject\": \"customer\", \"version\": 1}, \
	      {\"name\": \"product.json\",\"subject\": \"product\",\"version\": 1}, \
	      {\"name\": \"order.json\",\"subject\": \"order\",\"version\": 1}]}" \
	"http://localhost:8081/subjects/multi_event-value/versions" | jq;

	kafka-json-schema-console-consumer \
	   --topic multi_event \
	   --from-beginning \
	   --bootstrap-server pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --consumer.config producer.config \
	   --property value.schema.id=4 \
	   --property schema.registry.url=http://localhost:8081

	kafka-json-schema-console-producer \
	   --broker-list pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --property schema.registry.url=http://localhost:8081 \
	   --topic multi_event \
	   --property value.schema.id=4 \
	   --producer.config producer.config

	{"first_name":"Benny","last_name":"Boy"}
	{"order_id":2,"order_amount":4}
	{"product_name":"Efexor","price":4.56}

	{"firstname":"Benny","lastname":"Boy"}

	curl -s -X DELETE "http://localhost:8081/subjects/multi_event-value";
	curl -s -X DELETE "http://localhost:8081/subjects/multi_event-value?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/order";
	curl -s -X DELETE "http://localhost:8081/subjects/order?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/product";
	curl -s -X DELETE "http://localhost:8081/subjects/product?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/customer";
	curl -s -X DELETE "http://localhost:8081/subjects/customer?permanent=true";

## PROTO

	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"PROTOBUF\", \"schema\": \"$(cat ./protobuf/customer.proto | tr -d '\n' | tr -d '\t')\"}" "http://localhost:8081/subjects/customer/versions" | jq;
	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"PROTOBUF\", \"schema\": \"$(cat ./protobuf/product.proto | tr -d '\n' | tr -d '\t')\"}" "http://localhost:8081/subjects/product/versions" | jq;
	curl -X POST -H "Content-Type: application/json" -d "{\"schemaType\":\"PROTOBUF\", \"schema\": \"$(cat ./protobuf/order.proto | tr -d '\n' | tr -d '\t')\"}" "http://localhost:8081/subjects/order/versions" | jq;

	curl -X POST -H "Content-Type: application/json" -d \
	   "{\"schemaType\":\"PROTOBUF\", \
	   \"schema\": \"$(cat ./protobuf/multi_event.proto | tr -d '\n' | tr -d '\t')\", \
	   \"references\":[ \
	      {\"name\": \"customer.proto\", \"subject\": \"customer\", \"version\": 1}, \
	      {\"name\": \"product.proto\",\"subject\": \"product\",\"version\": 1}, \
	      {\"name\": \"order.proto\",\"subject\": \"order\",\"version\": 1}]}" \
	"http://localhost:8081/subjects/multi_event-value/versions" | jq;

	kafka-protobuf-console-consumer \
	   --topic multi_event \
	   --from-beginning \
	   --bootstrap-server pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --consumer.config producer.config \
	   --property value.schema.id=82 \
	   --property schema.registry.url=http://localhost:8081

	kafka-protobuf-console-producer \
	   --topic multi_event \
	   --broker-list pkc-e8mp5.eu-west-1.aws.confluent.cloud:9092 \
	   --producer.config producer.config \
	   --property value.schema.id=82 \
	   --property schema.registry.url=http://localhost:8081


	{"customer":{"first_name":"Benny","last_name":"Boy"}}
	{"order":{"order_id":2,"order_amount":4}}
	{"product":{"product_name":2,"price":4.56}}

	{"prodct":{"product_name":2,"price":4.56}}
	{"order":{"order_id":2,"orderamount":4}}

# 8 - Blow it all away

	curl -s -X DELETE "http://localhost:8081/subjects/avro_topic1-value";
	curl -s -X DELETE "http://localhost:8081/subjects/avro_topic1-value?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/avro_topic3-value";
	curl -s -X DELETE "http://localhost:8081/subjects/avro_topic3-value?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/avro_topic2-value";
	curl -s -X DELETE "http://localhost:8081/subjects/avro_topic2-value?permanent=true";

	curl -s -X DELETE "http://localhost:8081/subjects/proto_topic1-value";
	curl -s -X DELETE "http://localhost:8081/subjects/proto_topic1-value?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/proto_topic3-value";
	curl -s -X DELETE "http://localhost:8081/subjects/proto_topic3-value?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/proto_topic2-value";
	curl -s -X DELETE "http://localhost:8081/subjects/proto_topic2-value?permanent=true";

	curl -s -X DELETE "http://localhost:8081/subjects/json_topic1-value";
	curl -s -X DELETE "http://localhost:8081/subjects/json_topic1-value?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/json_topic3-value";
	curl -s -X DELETE "http://localhost:8081/subjects/json_topic3-value?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/json_topic2-value";
	curl -s -X DELETE "http://localhost:8081/subjects/json_topic2-value?permanent=true";

	curl -s -X DELETE "http://localhost:8081/subjects/multi_event-value";
	curl -s -X DELETE "http://localhost:8081/subjects/multi_event-value?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/order";
	curl -s -X DELETE "http://localhost:8081/subjects/order?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/product";
	curl -s -X DELETE "http://localhost:8081/subjects/product?permanent=true";
	curl -s -X DELETE "http://localhost:8081/subjects/customer";
	curl -s -X DELETE "http://localhost:8081/subjects/customer?permanent=true";

	ccloud kafka topic delete multi_event;

	ccloud kafka topic delete avro_topic1;
	ccloud kafka topic delete avro_topic2;
	ccloud kafka topic delete avro_topic3;

	ccloud kafka topic delete proto_topic1;
	ccloud kafka topic delete proto_topic2;
	ccloud kafka topic delete proto_topic3;

	ccloud kafka topic delete json_topic1;
	ccloud kafka topic delete json_topic2;
	ccloud kafka topic delete json_topic3;

	docker-compose --env-file docker-compose-ccloud.env -f docker-compose-schema-registry.yml down -v;

	ccloud kafka topic delete _schemas;