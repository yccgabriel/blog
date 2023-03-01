---
title: Kafka CDC
excerpt: This project is a rather complicated project with a lot of moving parts.  This is what makes it interesting right.  At a very top level I’m trying to connect two systems with Kafka, and pushing live updates from one side to the other.  On the very left we have Odoo the open-source ERP platform.  At the other end I’m writing a website that show the products that Odoo is managing.  Product details changes in Odoo are reflected in near-real-time on my tailormade website.
coverImage: '/assets/blog/kafka-cdc/cover.png'
date: '2023-03-01T00:00:00.000Z'
author:
  name: G
  picture: '/assets/blog/authors/g.png'
ogImage:
  url: /assets/blog/kafka-cdc/cover.png
---

This project is a rather complicated project with a lot of moving parts.  This is what makes it interesting right.  At a very top level I’m trying to connect two systems with Kafka, and pushing live updates from one side to the other.  On the very left we have Odoo the open-source ERP platform.  At the other end I’m writing a website that show the products that Odoo is managing.  Product details changes in Odoo are reflected in near-real-time on my tailormade website.

## Why Odoo
There are no short of SaaS products in the market to solve different real-life problems.  I consider it a waste of time to reinvent the wheel, and I probably couldn’t craft a competing product in ten years time.  On the other hand, we make prototype websites or mobile apps for new business opportunities, which implies there is almost certainly no one-size-fit-all solutions in the market.  Creating this kind of tailor-made apps are effort-consuming enough, usually at least 3 months to reach prototype or proof-of-concept stage.  There is almost no resources left for crafting the backend - usually a content management system.

The beauty of this prototype is that Kafka is connecting the best tools from both ends, and Kafka itself enables us to add more listeners later on with great degree of flexibility.

Here comes the abstract level architecture diagram, and some highlights:
- Odoo application is using the docker image released by Odoo Inc. https://hub.docker.com/_/odoo
- Postgresql database needs a minor tweak to make Capture-Data-Change(CDC) work.  For example the WAL(Write Ahead Log) level needs to be changed to “logical”.  For this demo I’m using the Postgres image offered by Debezium, which had done these configurations.  Just remember to refer to official Debezium documentations when integrating with SaaS database systems, e.g. odoo.sh which (by default) has the database managed in odoo cloud, thus need to manually do the changes.
- Debezium is a opensource project which solve the problem of reading change logs from relational databases.  For MySQL that’s binlog, Postgres’ WAL, and even MongoDB uses the same mechanism OpLog.  As a side note, Debezium works together with Kafka at its infancy, but then evolve to be event producer for other message brokers like AWS Kinesis and RabbitMQ.  For our demo, Debezium work as a Kafka Connect Producer to listen to the WAL from Postgres.  As a result, we now have table content changes streaming into the Kafka message broker.
- Kafka broker is relatively straight forward.  This is a pretty generic development setup which consist of a zookeeper and Kafka instance each.
- For the Kafka sink, we’re using the AWS Lambda Sink Connector offered by Confluent.  This sink collects the table content change event from the Kafka topics, and invoke the lambda function on AWS for each of the incoming messages.
- Next we cross the dotted line and enter the realm of AWS serverless computing.  The Sink connector mentioned in the last step will invoke this lambda function to upsert the entry into DynamoDB via AppSync.
- It is worth mentioning that lambda is invoking AppSync because we are taking advantage of the subscription feature offered by AppSync.  Subscription enables web and mobile apps listen to database changes and perform live updates.  However to make this possible is required mutations also go through AppSync to let it notify the subscribed clients.  AppSync is not capable of listening to DynamoDB changes and then notify subscribers.  Therefore lambda should not write directly to DynamoDB.
