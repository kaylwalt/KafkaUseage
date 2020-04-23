# KafkaUseage
If you are starting to consume from a confluent cluster, you need to have the a few things set up. This script does most of the work for you to use a console consumer.

This script will make sure that
- You have `java` installed
- You have `openssl` installed
- you have the `kafka binaries` downloaded
- you have our `Certificate Authority` in a Java Truststore
 

The only things required of you are
- Have a file `client.properties` in the same directory as this script with your credentials replacing the ones in this example.
- Replace `TOPIC` with a topic you have access to (in the firstProducer.sh file)
- Replace `BROKER_URL` with the IP or URL of the kafka cluster (in the firstProducer.sh file)
- Replace `TRUSTPASS` with a password that isn't the default  (in the firstProducer.sh file)
- Replace `bootstrap.servers` with the IP or URL of the kafka cluster (in the client.properties file)


 

 

```
#EXAMPLE client.properties
bootstrap.servers=X.X.X.X:9092
sasl.mechanism=PLAIN
security.protocol=SASL_SSL
ssl.truststore.location=truststore.jks
ssl.truststore.password=123456
#ssl.endpoint.identification.algorithm=https
sasl.jaas.config=org.apache.kafka.common.security.plain.PlainLoginModule  required username="{{REPLACEME}}"  password="{{RELPACEME}}";
retry.backoff.ms=500
```

``` bash
#!/bin/bash

TRUSTPASS=123456
BROKER_URL=X.X.X.X:9092
TOPIC=TESTTOPIC

InstallKafka () {
  wget https://downloads.apache.org/kafka/2.5.0/kafka_2.12-2.5.0.tgz -O kafka.tgz
  tar -xf kafka.tgz
  ls | grep kafka_ | xargs -L 1 -i bash -c 'mv {} kafka'
  rm kafka.tgz
}

CreateTrustStore () {
  openssl s_client -showcerts -verify 5 -connect ${BROKER_URL} < /dev/null | awk '/BEGIN/,/END/{ if(/BEGIN/){a++}; out="cert"a".pem"; print >out}'
  export TRUSTPASS=${TRUSTPASS}
  ls | grep cert | xargs -L 1 -i bash -c 'keytool -import -file {} -alias {} -storepass ${TRUSTPASS} -keystore truststore.jks -noprompt'
  ls | grep -P "^cert..*\.pem" | xargs -d"\n" rm
}

#check for prereqs

which java > /dev/null 2>&1
if [ $? != 0 ]
then
echo "We need to sudo to install java"
sudo apt install default-jre default-jdk -y || exit 1
else
echo "We found java"
fi

which openssl > /dev/null 2>&1
if [ $? != 0 ]
then
echo "We need to sudo to install openssl"
sudo apt install openssl -y || exit 1
else
echo "we found openssl"
fi

#install kafka bins
[ ! -d "kafka" ] && echo "No kafka directory, downloading and creating one" && InstallKafka

if [ -d "kafka" ]
then
echo "kafka directory exists"
else
echo "Failed to install the kafka directory"
exit 1
fi

[ ! -f "client.properties" ] && echo "There was no client.properties file in your present working directory" && exit 1

#get the truststore key and add to to a truststore
[ ! -f "truststore.jks" ] && echo "No truststore.jks found, making one" && CreateTrustStore && NEWTRUSTSTORE=1

[ ! -f "truststore.jks" ] && echo "The truststore.jks file was not created correctly" && exit 1

[ ! -z "${NEWTRUSTSTORE+x}" ] && echo "We just made the trust store, so we are letting it settle for a 5 seconds" && sleep 5

echo "running the console consumer on topic ${TOPIC}"
./kafka/bin/kafka-console-consumer.sh --from-beginning --bootstrap-server ${BROKER_URL} --topic ${TOPIC} --consumer.config client.properties

```
