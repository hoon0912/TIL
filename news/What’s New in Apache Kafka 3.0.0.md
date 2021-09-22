
> source : https://www.confluent.io/blog/apache-kafka-3-0-major-improvements-and-new-features/

## 개요

Apache Kafka 3.0의 릴리즈 소식을 전해서 매우 기쁘다.  
3.0은 여러 면에서 중요한 릴리즈이다. Apache Zookeeper를 대체할 KRaft-Kafka 기능, 주요 API의 변경 사항 및 새로운 기능이 도입되었다.

KRaft가 아직 실무에 적용되기에는 추천되지 않는 상황이기 때문에, 우리는 KRaft metadata와 API에 많은 개선사항을 추가했다.  
Exactly-once와 partition reassignment는 강조할 가치가 있으며 KRaft의 새로운 기능을 확인하고 개발 환경에서 시도해보기 바란다.  

3.0부터는 producer는 기본 값으로 강력한 delivery guarantee를 가지게 된다 (acks=all, enable.idempotence=true)

또한, Kafka Connect의 task restart 개선 사항, timestamp 기반 동기화에서의 KStreams의 개선 사항, MirrorMaker2의 더 유연한 속성 옵션도 같이 살펴보아라  

feaure 및 개선사항에 대한 전체 리스트를 확인하려면 [release note](https://downloads.apache.org/kafka/3.0.0/RELEASE_NOTES.html) 를 참조하여라  
또한 3.0에서 무엇이 새로운지 요약한 관련 비디오도 확인할 수 있다

## Universal changes

[KIP-750 (Part I): Deprecate support for Java 8 in Kafka](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=181308223)
* Java 8에 대한 지원은 3.0에서 모든 구성 요소에서 deprecate 된다. 이를 통해 Java 8 지원이 제거될 예정인 다음 주요 릴리즈(4.0) 이전에 적응할 수 있다

[KIP-751 (Part I): Deprecate support for Scala 2.12 in Kafka](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=181308218)
* Scala 2.12에 대한 지원또한 3.0에서 모두 deprecate 된다. 마찬가지로 Scala 2.12에 대한 지원이 다음 릴리즈(4.0)에서 제거 될 예정이므로 이에 대한 적응 시간을 준다.

## Kafka broker, producer, consumer and adminclient

[KIP-630: Kafka Raft Snapshot](https://cwiki.apache.org/confluence/display/KAFKA/KIP-630%3A+Kafka+Raft+Snapshot)
* 3.0버전의 큰 특징 중 하나는 KRaft controller 및 KRaft borker에 대한 사항이다  
KRaft broker는 __cluster_metadata라는 토픽 파티션의 metadata의 snapshot을 생성하고 복제하고 저장하는 기능을 제공한다  
__cluster_metadata 토픽은 broker config, topic partition assignment, leadershipt 등의 클러스터 metadata 정보를 저장하고 복제하는데 사용된다 
Kafka Raft Shapshot은 이러한 정보를 저장하고 불러오고 복제하는 일련의 작업을 효율적으로 제공해준다

[KIP-746: Revise KRaft metadata records](https://cwiki.apache.org/confluence/display/KAFKA/KIP-746%3A+Revise+KRaft+Metadata+Records)
* Kafka가 Zookeeper 없이 실행되도록 구성될 때 사용되는 몇 가지 metadata record 타입을 수정해야 할 필요성이 있다  

[KIP-730: Producer ID generation in KRaft mode](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=177049344)
* 3.0 및 KIP-730에서 Kafka Controller는 이제 kafka producer의 ID 생성 책임을 완전히 담당한다. Kafka Controller는 이제 ZK와 KRaft 모드를 지원한다. 이를 통해 기존에 ZK를 사용하는 구조에서 KRaft를 사용하는 새 배포로 전환할 수 있는 다리 역할을 할 것이다

[KIP-679: Producer will enable the strongest delivery guarantee by default](https://cwiki.apache.org/confluence/display/KAFKA/KIP-679%3A+Producer+will+enable+the+strongest+delivery+guarantee+by+default)
* 3.0부터 kafka producer가 기본값으로 모든 replica에 대한 ack 확인 등의 옵션이 적용될 것이다

[KIP-735: Increase default consumer session timeout](https://cwiki.apache.org/confluence/display/KAFKA/KIP-735%3A+Increase+default+consumer+session+timeout)
* Kafka consumer의 config 옵션 중에 `session.timeout.ms` 옵션이 10초에서 45초로 증가되었다. 이를 통해 일시적인 네트워크 오류로 인해 consumer group rebalance가 무의미하게 발생하는 경우를 비할 수 있다

[KIP-709: Extend OffsetFetch requests to accept multiple group ids](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=173084258)
* Kafka Consumer Group의 현재 offset을 요청하는 것은 가능했지만, 여러 Consumer Group에 대한 offset 요청은 각 그룹 별로 개별 요청이 필요하였다
하지만 3.0부터는 AdminClient API가 단일 request/response 뿐만 아니라 동시에 여러 Consumer Group offset 읽기를 지원하도록 확장되었다
  
[KIP-699: Update FindCoordinator to resolve multiple Coordinators at a time](https://cwiki.apache.org/confluence/display/KAFKA/KIP-699%3A+Update+FindCoordinator+to+resolve+multiple+Coordinators+at+a+time)
* 지금까지 여러 Consumer Group에 대한 operation은 Client가 얼마나 효율적으로 Consumer Group Coordinator를 발견하는지에 따라 달라졌다
이 업데이트로 인해서 하나의 request로 여러 Consumer Group에 대한 coordinator를 찾는 작업이 가능해졌다. Kafka Client는 이 요청을 지원하는 Kafka Broker와 통신할 때
  이 최적화 방식을 사용하도록 업데이트 되었다
  
[KIP-724: Drop support for message formats v0 and v1](https://cwiki.apache.org/confluence/display/KAFKA/KIP-724%3A+Drop+support+for+message+formats+v0+and+v1)
* 2017년 kafka 0.11.0 버전이 도입된 후 4년 동안 기본 메세지 형식은 v2였다
따라서 3.0에서는 v0과 v1 메세지 포맷을 deprecate 처리한다. 이 메세지 포맷은 요즈음에 거의 사용되지 않는다
  3.0에서는 메세지 포맷을 v0 또는 v1로 Kafka broker를 구성하면 경고가 표시된다. 해당 포맷은 4.0 버전에서 제거될 것이다
  
[KIP-707: The future of KafkaFuture](https://cwiki.apache.org/confluence/display/KAFKA/KIP-707%3A+The+future+of+KafkaFuture)
* Kafka AdminClient의 구현을 용이하기 위해서 Kafka Future 타입을 추가하였을 당시만해도 Java 8이 널리 사용되고 있었고 Kafka에서 공식적으로 Java 7이 지원되고 있었다
시간이 빠르게 지나가면서 이제 카프카는 CompletionState와 CompletableFuture 클래스 타입을 지원하는 Java 버전에서 작동된다. KafkaFuture는 CompletionStage 객체를 반환하는 메서드를 추가하여
  이전 버전과 호환되는 방식으로 KafkaFuture의 사용성을 향상시킨다
  
[KIP-466: Add support for List<T> serialization and deserialization](https://cwiki.apache.org/confluence/display/KAFKA/KIP-466%3A+Add+support+for+List%3CT%3E+serialization+and+deserialization)
* serialization / deserialization 를 위한 새로운 클래스 및 메서드를 추가한다.
이 기능은 Kafka client와 Kafka Streams에 유용한 기능이 될 것이다
  
[KIP-734: Improve AdminClient.listOffsets to return timestamp and offset for the record with the largest timestamp](https://cwiki.apache.org/confluence/display/KAFKA/KIP-734%3A+Improve+AdminClient.listOffsets+to+return+timestamp+and+offset+for+the+record+with+the+largest+timestamp)
* kafka topic/partition에 대한 offset 리스트를 얻는 사용자의 기능이 확장되었다
사용자는 이제 AdminClient에 topic/partition 중에서 가장 높은 timestamp를 가지고 있는 레코드의 offset과 timestamp를 반환하도록 request 할 수 있다
  (기존에 AdminClient가 최신 offset을 반환하는 것과 혼동하지 마라. 해당 기능은 topic/partition에 기록 될 next record offset 을 의미한다)
  이 업데이트로 인한 ListOffsets API에 대한 확장 기능은 가장 최근에 기록된 record offsset / timestamp를 확인하여서 partition의 활성 상태를 조사할 수 있게 할 것이다
  
## Kafka Connect

[KIP-745: Connect API to restart connector and tasks](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=181308623)
* 런타임 기간 동안 Kafka Connect에서 connector는 Connector 클래스 인스턴스와 하나 이상의 Task 클래스 인스턴스의 그룹으로 표시된다
Connect REST API를 통해 connector에 대한 대부분의 작업은 그룹 전체에 적용할 수 있었다.
  런타임 중에 주목할만한 exception으로는 Connector와 Task 인스턴스에 대한 restart 엔드포인트였다. 왜냐하면 connector를 전체 재시작하기 위해서는 Conecctor와 Task 인스턴스에게 개별적으로 요청을 보내야하기 때문이다
  3.0에서는 사용자가 모든 혹은 실패한 Conenctor와 Task 인스턴스를 재시작할 수 있는 단일 호출 방식을 제공한다
  이 기능은 추가 기능이며 이전의 restart REST API는 여전히 변경되지 않은 상태로 유지될 것이다
  
[KIP-738: Removal of Connect’s internal converter properties](https://cwiki.apache.org/confluence/display/KAFKA/KIP-738%3A+Removal+of+Connect%27s+internal+converter+properties)
* 2.0 릴리즈에서 deprecated 된 `internal.key.converter` , `internal.value.converter` 옵션이 제거되고 Connect worker의 옵션에 값이 고정되었다.

[KIP-722: Enable connector client overrides by default](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=177047383)
* 2.3.0 버전부터 connector worker는 kafka client properties를 override하기 위해서 connector config값을 설정할 수 있었다. 이 기능은 널리 사용되었고 이제는 default로 활성화되게 되었다
  (`connector.client.config.override.policy=All`)
  
[KIP-721: Enable connector log contexts in Connect Log4j configuration](https://cwiki.apache.org/confluence/pages/viewpage.action?pageId=177047379)
* 2.3.0에 다시 도입되었지만 지금까지 default로 활성화되지 않은 또 다른 기능으로는 connector log context 이다
3.0 부터는 connect work의 log4j 로그 패턴이 default로 connector context에 추가되었다
  
## Kafka Streams & Mirror Maker

생략 (Source 참조)
