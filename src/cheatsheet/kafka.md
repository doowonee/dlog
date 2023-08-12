# Kafka

[가이드 라인](https://kafka.apache.org/documentation/#config)에 따르면 Kafka와 OS는 같은 스토리지를 사용하지 않도록 권고 한다.

[Confluent community version](https://hub.docker.com/r/confluentinc/cp-kafka/) 도커 이미지가 있는데 뭘써야 할지 잘 모르겠으므로 그냥 나는 [apache version](https://hub.docker.com/r/wurstmeister/kafka)를 사용한다.

그다음 도커를 실행 해준다. 버전은 현재 시점 기준 최신 버전인 `2.13-2.7.0`을 사용한다. 참고로 앞에 `2.13`은 scala 버전이고 뒤가 kafka 버전이다.

`server.properties` 내용인 `group.initial.rebalance.delay.ms=0`을 수정하려면 `-e "KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS=3000"` 형식으로 변환해서 환경변수로 지정하면 설정값을 변경 할수 있다.

`log4j.properties` 설정도 `log4j.logger.kafka.authorizer.logger=INFO, authorizerAppender`의 내용의 경우 `-e "LOG4J_LOGGER_KAFKA_AUTHORIZER_LOGGER=INFO, stdout"` 형식으로 환경변수를 지정해서 file로 로그가 생성되지 않고 모두 stdout으로 로그가 출력 되도록 설정을 변경 할수 있다. 하지만 `rootLogger` 등 대소문자가 다른 key는 동작 하지 않으므로 그냥 외부 볼륨 바인딩해서 로그 설정을 덮어쓰기 하도록 한다. 단 `gc.log`는 여전히 컨테이너 내부에 생성 된다. 이건 JVM 생성 하는 로그도 보이며 `log4j` 설정에 포함되지 않아서 일단 놔두었다.

```bash
# 외부 바인딩 IP주소 자동으로 우분투 기준 외부 IP로 지정
# log.dirs 기본 경로인 /kafka 볼륨 바인드
docker run -d --name kafka \
  --network host \
  --log-driver json-file --log-opt max-size=2g \
  -e "KAFKA_BROKER_ID=1" \
  -e "KAFKA_ADVERTISED_HOST_NAME=$(ip addr show ens5 | grep 'inet\b' | awk '{print $2}' | cut -d/ -f1)" \
  -e "KAFKA_ZOOKEEPER_CONNECT=172.30.1.1:2181" \
  -e "KAFKA_AUTO_CREATE_TOPICS_ENABLE=false" \
  -e "KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS=3000" \
  -v /home/ubuntu/log4j.properties:/opt/kafka/config/log4j.properties \
  -v /kafka-data:/kafka \
  wurstmeister/kafka:2.13-2.7.0
```

kafka client만 사용하고 싶으면 `docker run --rm -it wurstmeister/kafka /bash`로 1회성 셀을 사용할수 있다. `rm` 옵션 없이 사용하면 재사용 가능할듯.

참고로 log directory에 저장된 `broker.id`가 서로 다르면 kafka는 기동 되지 않는다. 즉 `broker.id`는 영구적으로 사용해야 한다는 것

## CLI

```bash
#  https://kafka.apache.org/documentation/#upgrade_220_notable 이후
#  --zookeeper 옵션 안쓰고 브로커 자체 옵션인 --bootstrap-server 사용 한다
kafka-topics.sh --list --bootstrap-server localhost:9092 
kafka-topics.sh --create --topic quickstart-events --bootstrap-server localhost:9092

# consumer 사용 스크립트
kafka-console-consumer.sh --topic viewlog --from-beginning --bootstrap-server localhost:9092

# partition과 key도 출력 space로 구분 하고 key message partition 순임
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic viewlog_dev --from-beginning --property print.key=true --property key.separator=" " --property print.partition=true --property partition.separator=" "

# 특정 파티션 특정 오프셋 이후 consume 하기
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic viewlog_dev --property print.key=true --property key.separator=" " --property print.partition=true --property partition.separator=" " --partition 1 --offset 1160555

# topic 정보 보기
kafka-topics.sh --describe --topic viewlog_dev --bootstrap-server localhost:9092

# topic 삭제
kafka-topics.sh --delete --topic quickstart-events --bootstrap-server localhost:9092

# topic에 메시지가 얼만큼 있는지 확인한다. 각 파티션별로 출력된다.
kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9092 --topic viewlog_dev

# topic 생성
# partitions과 replication-factor 걍 둘다 클러스터 인스턴스 만큼 했음
kafka-topics.sh --create \
--bootstrap-server localhost:9092 \
--replication-factor 3 \
--partitions 3 \
--topic viewlog_dev

# topic retention 변경
# 근데 log.retention.check.interval.ms 이게 기본 5분이라 그 이하는 무용지물임
# 36시간 적용
kafka-configs.sh --alter \
    --entity-type topics \
    --entity-name viewlog \
    --add-config retention.ms=129600000 \
    --bootstrap-server localhost:9092

# conumer group 목록
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
# consumer group 정보
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group cg_viewlog_dev
# consumer group 정보
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --topic viewlog_dev
```

## 클러스터

```bash
# broker ID 다르게 하고 zk 서버 여러개 넣기
docker run -d --name kafka \
  --network host \
  --log-driver json-file --log-opt max-size=2g \
  -e "KAFKA_BROKER_ID=3" \
  -e "KAFKA_ADVERTISED_HOST_NAME=$(ip addr show ens5 | grep 'inet\b' | awk '{print $2}' | cut -d/ -f1)" \
  -e "KAFKA_ZOOKEEPER_CONNECT=172.1.1.1:2181,172.2.2.2:2181,172.3.3.3:2181" \
  -e "KAFKA_AUTO_CREATE_TOPICS_ENABLE=false" \
  -e "KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS=3000" \
  -v /home/ubuntu/log4j.properties:/opt/kafka/config/log4j.properties \
  -v /kafka-data:/kafka \
  wurstmeister/kafka:2.13-2.7.0
```

## log4j.properties

```properties
# prevent log to file for dockerizing
log4j.rootLogger=INFO, stdout

log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=[%d] %p %m (%c)%n

log4j.logger.org.apache.zookeeper=INFO
log4j.logger.kafka=INFO
log4j.logger.org.apache.kafka=INFO

log4j.logger.kafka.request.logger=INFO, stdout
log4j.additivity.kafka.request.logger=false

log4j.logger.kafka.network.RequestChannel$=WARN, stdout
log4j.additivity.kafka.network.RequestChannel$=false

log4j.logger.kafka.controller=INFO, stdout
log4j.additivity.kafka.controller=false

log4j.logger.kafka.log.LogCleaner=INFO, stdout
log4j.additivity.kafka.log.LogCleaner=false

log4j.logger.state.change.logger=INFO, stdout
log4j.additivity.state.change.logger=false

log4j.logger.kafka.authorizer.logger=INFO, stdout
log4j.additivity.kafka.authorizer.logger=false
```
