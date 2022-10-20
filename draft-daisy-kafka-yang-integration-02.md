## Abstract

With the introduction of YANG [RFC7950] as a data modelling language in the network for configuration and operational metrics, an automated onboarding of new models and metrics into a Data Mesh Big Data environment and therefore a closed loop operation without human intervention is at possible reach. YANG can be serialized in JSON [RFC8259], XML and CBOR [RFC7049]. With [RFC9254], YANG specific encoding rules for CBOR had been defined at IETF. 

A message broker enables real time data exchange among different Data Mesh domains. A schema registry ensure that the producer and consumer can learn from each other the schema and version for each message outband. Confluent [Apache Kafka] message broker supports custom data serialization. Since April 2020, Confluent extended their [Schema Registry] to be pluggable. Supporting besides AVRO (JSON and binary) also JSON and Protobuf natively. Described in [SR Blog Post], [SR Protobuf Provider Plugin], [Protobuf Serializer/Derializer], [Protobuf Serde for Kafka Streams] and [Protobuf Connect Converter].

With [draft-ietf-netconf-udp-notif] and [draft-ietf-netconf-https-notif] two proposed standards at the NETCONF working group which supports configured YANG push subscriptions are in progress. [draft-ietf-netconf-https-notif] supports YANG push notification messages according to [RFC8639] and application/yang-data+json and application/yang-data+xml [Media Types] for encoding. [draft-ietf-netconf-https-notif], supports YANG push notification messages according to [RFC8641] and application/yang-data+json, application/yang-data+xml and application/yang-data+cbor for encoding.

The YANG push notification message described in [RFC8641] contains a subscription id for each message as metadata. Referencing the subscribed XPath, a subsection of a YANG module. The YANG subscription id is being defined when a new XPath is subscribed to. Without this metadata, the YANG push receiver (YANG push data-collection) is unable to determine which YANG module matches the JSON/XML/CBOR encoded message today.

With [draft-ietf-netmod-yang-versioning-reqs] YANG versioning requirements have been defined and with [draft-ietf-netmod-yang-module-versioning] a YANG versioning handling has been proposed. This needs to be carried on to YANG push into the notification header [draft-tgraf-netconf-yang-notifications-versioning] and also into the YANG push subscription [RFC8641].

An [Apache Kafka] topic supports multiple subjects. Each subject maps to a dedicated schema id in the schema registry [Schema Registry Overview]. A YANG push receiver IPv4/6 address and L4 port maps 1:1 to an Apache Kafka topic.

In this document we describe how Confluent's [Apache Kafka] data serialization and [Schema Registry], the native Confluent Kafka producer and consumer, the time series data base ingestion and the [pmacct] pmtelemetryd YANG push receiver needs to be extended to enable automated data onboarding while preserving the data semantic along the chain.

![alt text](draft-daisy-kafka-yang-integration-02-1.png "Overview")


## Aim

With the establishment of a bounded context (standardized interface) between operational metrics and analytical application, we meet the following project goals

* New YANG push metrics are visible in the time series database within minutes without manual intervention
* New analytics components, regardless which network or analytics vendor provides if, can be integrated seamlessly
* Enabling a digital twin for a closed loop operated network


## Status of this memo

Project Kickoff at IETF 115 in London.


## YANG push receiver

The [pmacct] YANG push receiver needs to be extended to read

* From YANG push notification header, the YANG XPath and version as specified in [draft-tgraf-netconf-yang-notifications-versioning]
* From YANG push notification header, the subscription id as specified in Section 3.6 of [RFC8641]
* From the JSON payload, the vendor specific sensor-path metadata

To determine which YANG module and version for the JSON/CBOR payload in the YANG push notification message.

If for that YANG module and version the Confluent schema ID is not cached yet, then a <get-schema> NETCONF request as defined in section 3.1 of [RFC6022] needs to be performed to the IPv4/6 address where the YANG push message is being originated from to obtain the YANG schema. Once obtained, the YANG schema needs to be posted through a REST API to the Confluent Schema Registry [Schema Registry] and the Confluent Schema ID in return needs to be used for Kafka serialization with [libserdes].

At present it is unclear, when CBOR encoding is performed at the YANG push publisher, wherever the notification message described in RFC 8641 is being encoded in CBOR or remain JSON encoded. When CBOR encoding is performed at YANG push publisher and the notification message would be encoded in CBOR as well, the YANG push receiver would need to deserialize the message to determine the YANG XPath and version. For this purpose, the [libcbor] library could be used.


## YANG schema registry

The pluggable Confluent [Schema Registry] needs to be extended to support YANG [RFC7950]. This enables schema evolution between Kafka producer and consumer and automated data ingestion into a timeseries database at a consumer.

In order to support YANG [RFC7950] the following items need to be defined and developed

* Which [Confluent Compatibility Checks] defined in section 11 [RFC7950] should be performed for different [Confluent Compatibility Types]
* How YANG yang-version statement, namespace statement, prefix statement and the revision-statements are preserved and revision-statements possibily being used in the [Confluent Compatibility Checks]


## YANG Kafka producer/consumer

The JSON message, including the notification message header specified in section 2.6 of [RFC8639], and the YANG push header specified in section 3.7 and figure 9 under section 4.1 of [RFC8641] needs to be serialized with the Confluent Schema ID. It needs to be determined if the same [AVRO Wire Format] should be used. When being consumed from a Kafka topic, the Confluent schema ID needs to be read and a REST HTTP GET call to the Confluent Schema Registry needs to be performed.

The changes should be developed first to the native JAVA Kafka producer/consumer and then to [libserdes] second.


## YANG push subscription

The YANG push subscription process for a network node needs to be extended with the YANG version. During the subscription process the subscription id is obtained for each XPath and version and all stored in a mapping file for use at the YANG push receiver when YANG XPath and version is not present in the YANG push notification header yet.


## YANG Times Series Data Base Ingestion

The schema from the schema registry is being used to define the times series database schema and the ingestion rule for the operational metrics. The time when the message was being produced by the YANG publisher can be optionally obtained by the <eventTime> as defined in section 4 of [RFC5277]. The subscription id and in the future also the observation domain id, the XPath and the version can be obtained from the YANG push schema specified in section 3.7 and figure 9 under section 4.1 of [RFC8641].


## Project Team

* Christoph Schubert, Senior Solutions Architect at Confluent, https://www.linkedin.com/in/christophschubert/
* Senad Jukic, Senior Solutions Engineer at Confluent, https://www.linkedin.com/in/senad-jukic/
* Perry Krol, Head of Solutions Engineering EMEA at Confluent, https://www.linkedin.com/in/perrykrol/
* Paolo Lucente, Network Analytics Architect at NTT, https://www.linkedin.com/in/plucente/
* Diego R. Lopez, Senior Technology Expert at Telefonica I+D, https://www.linkedin.com/in/dr2lopez/
* Ignacio Dominguez Martinez-Casanueva, Technology Exploration and Standards at Telefonica, * https://www.linkedin.com/in/ignacio-dominguez-martinez-casanueva-3b8b6ab2/
* Daniel Voyer, Technical Fellow at Bell Canada, https://www.linkedin.com/in/daniel-voyer-52a11414/
* Joe Clarke, Chair at OPSAWG IETF and Distinguished Services Engineer at Cisco Systems, * https://www.linkedin.com/in/joeclarke2/
* Benoit Claise, Network Automation CTO at Huawei, https://www.linkedin.com/in/benoit-claise/
* Michael Mackey, Leader Of The SDN Automation Research Team at Huawei Technologies, https://www.linkedin.com/in/mikemackey/
* Olga Havel, Technical Expert & Research Team Leader at Huawei Ireland Research Center, https://www.linkedin.com/in/olgahavel/
* Jean Quilbeuf, Senior Software R&D Engineer at Huawei Ireland labs, https://www.linkedin.com/in/jean-quilbeuf/
* Eric Tschetter, Field CTO at Imply, https://www.linkedin.com/in/cheddar/
* Dean Sheehan, Field CTO at InfluxData, https://www.linkedin.com/in/dsheehan/
* Pierre Francois, Head of Research at INSA Lyon, https://www.linkedin.com/in/pierre-fran%C3%A7ois-2280901/
* Alex Huang Feng, R&D Researcher at INSA Lyon, https://www.linkedin.com/in/ahuangfeng/
* Zhuoyao Lin from Institut Polytechnique de Paris, Intern at Swisscom and Huawei Research Center Dublin, https://www.linkedin.com/in/zhuoyao-lin-6683b1221/
* Urs Baumann, Network Automation and Research Engineer at OST Rapperswil, https://www.linkedin.com/in/ubaumannch/
* Marco Tollini, Data and DevOps Engineer at Swisscom, https://www.linkedin.com/in/marco-tollini/
* Ahmed Elhassany, Data and Software Engineer at Swisscom, https://www.linkedin.com/in/ahassany/
* Uwe Storbeck, Senior Software Engineer at Swisscom
* Thomas Graf, Distinguished Network Engineer and Network Analytics Architect at Swisscom, https://www.linkedin.com/in/thomas-graf-25212516a/


## Project Milestones

* IETF 115 London - Official Project Kickoff
* IETF 115 London - IETF 117 San Francisco, Confluent Schema Registry, Kafka native producer/consumer YANG JSON serializer integration code development at INSA/Swisscom.
* IETF 116 Yokohama - IETF 118 Prague, YANG push data-collection code development at Huawei/Swisscom.
* IETF 117 San Francisco - 118 Prague, Imply Druid YANG JSON Data Ingestion Specification Transformation code development at Imply.


### Normative References

* [RFC7950] The YANG 1.1 Data Modeling Language
  https://datatracker.ietf.org/doc/html/rfc7950

* [RFC9254] CBOR Encoding of Data Modeled with YANG
  https://datatracker.ietf.org/doc/html/rfc9254

* [RFC8259] The JavaScript Object Notation (JSON) Data Interchange Format
  https://datatracker.ietf.org/doc/html/rfc8259

* [RFC7049] Concise Binary Object Representation (CBOR)
  https://datatracker.ietf.org/doc/html/rfc7049

* [RFC8641] Subscription to YANG Notifications for Datastore Updates
  https://datatracker.ietf.org/doc/html/rfc8641

* [draft-tgraf-netconf-yang-notifications-versioning] Support of Versioning in YANG Notifications Subscription
  https://www.ietf.org/archive/id/draft-tgraf-netconf-yang-notifications-versioning

* [Apache Kafka] Confluent Apache Kafka
  https://github.com/apache/kafka

* [Schema Registry] Confluent Schema Registry
  https://github.com/confluentinc/schema-registry

* [pmacct] Network Telemtry data-collection
  https://github.com/pmacct/pmacct

* [libcbor] libcbor
  https://github.com/PJK/libcbor

* [draft-ietf-netmod-yang-versioning-reqs]
  https://datatracker.ietf.org/doc/html/draft-ietf-netmod-yang-versioning-reqs

* [draft-ietf-netmod-yang-module-versioning] 
  https://datatracker.ietf.org/doc/html/draft-ietf-netmod-yang-module-versioning

* [draft-tgraf-netconf-yang-notifications-versioning]
  https://github.com/graf3net/draft-tgraf-netconf-yang-notifications-versioning

* [RFC6022]  YANG Module for NETCONF Monitoring
  https://datatracker.ietf.org/doc/html/rfc6022.html

* [libserdes] libserdes
  https://github.com/confluentinc/libserdes

* [Confluent Compatibility Checks] Confluent Schema Compatibility Checks
  https://docs.confluent.io/platform/current/schema-registry/serdes-develop/index.html#sr-serdes-schemas-compatibility-checks


### Informative References

* [RFC5277] NETCONF Event Notifications
  https://datatracker.ietf.org/doc/html/rfc5277

* [RFC8639] Subscription to YANG Notifications
  https://datatracker.ietf.org/doc/html/rfc8639

* [Media Types] IANA - Media Types
  https://www.iana.org/assignments/media-types/media-types.xhtml

* [Schema Registry Overview] Confluent Schema Registry, Schemas, Subjects, and Topics
  https://docs.confluent.io/platform/current/schema-registry/index.html

* [SR Blog Post]
  https://yokota.blog/2020/02/26/playing-chess-with-confluent-schema-registry/

* [SR Protobuf Provider Plugin]
  https://github.com/confluentinc/schema-registry/tree/master/protobuf-provider 

* [Protobuf Serializer/Derializer]
  https://github.com/confluentinc/schema-registry/tree/master/protobuf-serializer

* [Protobuf Serde for Kafka Streams]
  https://github.com/confluentinc/schema-registry/tree/master/protobuf-serde

* [Protobuf Connect Converter]
  https://github.com/confluentinc/schema-registry/tree/master/protobuf-converter

* [draft-ietf-netconf-udp-notif]] UDP-based Transport for Configured Subscriptions
  https://datatracker.ietf.org/doc/html/draft-ietf-netconf-udp-notif

* [draft-ietf-netconf-https-notif]  An HTTPS-based Transport for YANG Notifications
  https://datatracker.ietf.org/doc/html/draft-ietf-netconf-https-notif

* [AVRO Wire Format] AVRO Wire Format
  https://docs.confluent.io/platform/current/schema-registry/serdes-develop/index.html#wire-format

* [Confluent Compatibility Types] Confluent Schema Compatibility Types
  https://docs.confluent.io/platform/current/schema-registry/avro.html#compatibility-types