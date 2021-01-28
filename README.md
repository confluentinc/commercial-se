# Original demo designed for Confluent Platform, by Senior Solutions Engineer: Alex Woolford 
find Alex's demo here: https://github.com/alexwoolford/rtd-kafka 

# Demo summary
realtime bus data from denver transit authority -> springboot -> Confluent Cloud -> S3

•Springboot app scrapes realtime bus location data from denver transit authority

•app produces to topic 

•fully managed S3 connector writes out to S3 bucket

# Prerequisites to run demo 
•IntelliJ

•S3 connector

•S3 bucket

•Confluent Cloud account

•Confluent Cloud connection details (api keys for schema registry, your cluster, see the "application.properties" within rtd-feed/src/main/resources/application.properties for what you need to configure from your Confluent Cloud account) 

# RTD Bus Feed
The __rtd-feed__ module publishes the latest Denver RTD bus positions, every 30 seconds, to the `rtd-bus-position` Kafka topic. This is telemetry data from buses. The regional transport district (RTD) publishes data as a Protobuf file, which we scrape from here: https://www.rtd-denver.com/business-center/open-data/gtfs-developer-guide#gtfs-realtime-feeds. 


The messages are serialized as Avro and look like this when deserialized:

    {
      "id": "985CC5EC1D3FC176E053DD4D1FAC4E39",
      "timestamp": 1574904869,
      "location": {
        "lon": -104.98970794677734,
        "lat": 39.697723388671875
      }
    }

The message key is the vehicle ID, which means that telemetry for a vehicle is always sent to the same Kafka partition.

Finally, used fully managed Confluent Cloud S3 sink connector to easily write out data from the rtd-bus-position topic and persist in S3. 
