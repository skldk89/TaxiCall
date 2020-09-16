```
curl -o kafka2.5.tgz -l http://mirror.navercorp.com/apache/kafka/2.5.0/kafka_2.13-2.5.0.tgz
tar -xvf kafka2.5.tgz
zookeeper 실행
cd  kafka_2.13-2.5.0/bin
./zookeeper-server-start.sh ../config/zookeeper.properties 
kafka 실행
cd  kafka_2.13-2.5.0/bin
./kafka-server-start.sh ../config/server.properties





./kafka-topics.sh --zookeeper localhost:2181 --topic TaxiCall --create --partitions 1 --replication-factor 1

./kafka-topics.sh --zookeeper localhost:2181 --list

./kafka-console-consumer.sh --bootstrap-server http://localhost:9092 --topic RestaurantReservaiton --from-beginning

```
