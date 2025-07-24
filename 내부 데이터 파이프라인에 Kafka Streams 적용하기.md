---
출처: '[내부 데이터 파이프라인에 Kafka Streams 적용하기](https://engineering.linecorp.com/ko/blog/applying-kafka-streams-for-internal-message-delivery-pipeline)'
---

- Kafka Streams가 개발 초기였던 2016년 작성된 글, 현재 상황(2025년)과 다를 수 있음

- 프로젝트의 목적

    - 사내 시스템 간의 이벤트 전달을 일원화된 방식으로 처리하는 데이터 파이프라인 개발

    - 라인 서버 시스템 구성요소 중 백그라운드 태스크 처리를 담당하는 talk-dispatcher 개선

    - 두 목적에서 동일하게 Kafka와 스트림 프로세싱 기술을 적용

- Apache Kafka

    - 링크드인에서 주로 개발하고 사용하는 대용량 분산 메시징 시스템

    - 중요 특징

        - 디스크 기반 영속적 저장 방식과 페이지 캐시를 통한 높은 처리량의 인메모리 방식

        - 여러 컨슈머가 하나의 토픽으로부터 여러번에 걸쳐 메시지를 가져올 수 있음

            - 클라이언트가 해당 큐에서 어느 부분까지 데이터를 받아갔는지 offset 제공

# Stream processing framework

## Samze

- 스트림 프로세싱에 사용되는 여러 프레임워크중 Samza를 처음 고려했음

    - Kafka와 연동되도록 설계되어 친화력이 뛰어나고, 통합을 기본적으로 지원함

- 기본적인 기능은 문제가 없으나 핵심 인프라로는 염려되는 점이 존재했음.

    - Hadoop YARN에 의존적임

        - 잘 만들어진 분산 프레임워크지만 최초에 배치 처리를 위해 설계되었고, 일부 하둡의 계승점이 스트림 프로세싱에 부적절

        - 사내 엔지니어링 설비는 호스트 개념을 따르고, 어떤 호스트가 작업을 수행하는지 결정하는 것을 YARN에 위임하기에는 부적절

            - 예를 들어 사내 서비스 모니터링용 자체 툴 IMON은 호스트별로 메트릭을 보거나 알람을 보낼 수 있으며, 키바나는 호스트별로 로그를 저장함.

        - 라인 사내 배포시 사내 배포 시스템(configuration managemenet database + 빌드, 배포 기능을 합친 서비스)과의 친화력이 전무

        - 단순한 아키텍처를 지향함

            - 중단 없이 계쏙 실행중인 어플리케이션에 리소스를 분리하거나 할당할 필요가 없었음

                - 서버는 기본적으로 서로 다른 서비스마다 별도로 할당됨

                - 대부분의 메모리를 소모하는 부분은 heap이지만 JVM에는 heap limit이 있음

                - 어플리케이션의 특성상 CPU는 문제가 되지 않음

                - 네트워크 트래픽이 문제의 소지가 있는데, YARN은 네트워크 IO를 조절할 수 없음

    - 커뮤니티의 활동이 상대적으로 저조함

- 하나의 리소스 풀로 많은 작업이나 애플리케이션을 배치하고 실행하는데는 여전히 유용.

    - 통계 작업이나 문제 조사 중 필요한 에드혹엔 여전히 사용

## Kafka Streams

- 다른 일반적인 프레임워크가 "실행 프레임워크"인 반면, Kafka Stream는 라이브러리

- Samza에서 가져온 개념도 일부 있지만 중요한 차이가 존재

    - 그냥 라이브러리이고, 실행 프레임워크가 아니기 때문에 사용자가 수동으로 구동해야 함

    - Kafka 본래의 기능을 모두 활용해 간단하게 핵심 기능을 개발할 수 있고 매우 가벼움

    - 직관적인 DSL을 사용하여 프로세싱 토폴로지를 정의할 수 있음

    - Kafka 공식 커뮤니티가 주도하고 있어 개발활동이 활발함

    - 롤링 리스타트를 지원하기 떄문에 트래픽을 처리하는 중에도 단일 인스턴스 t아에서 어플리케이션의 동작을 확인할 수 있음.

### Masterless

- Kafka Stream은 일반적인 분산 시스템의 장애 감지, 처리 노드간 조율, 파티션 할당을 수행하기 위한 마스터라는 개념이 없음

    - Kafka의 자체 조율 알고리즘에 의존. 작업 노드 간의 통신이 필요 없음

- 주어진 applicationId로 새 카프카스트림 인스턴스를 실행하면 해당 applicationId의 consumer중 하나로 Kafka broker를 subscribe

- 파티션 할당이 조정되거나 failover 발생시 카프카 브로커가 감지하고 처리하여 작업 노드들의 통신이 필요 없어짐

### High-level-DSL API와 Low-level API

- 스트림 프로세싱 프로그래밍을 위해 high-level-DSL과 low-level API 두가지를 지원

- High-level-DSL

    - 일반적인 스트림 프로세싱은 스트림에 transform, filter, join, aggregate 연산 처리를 적용하고 결과를 저장

        - 이런 기본적인 연산 처리에는 high-level-DSL 인터페이스가 적합

    - Scala Collections API와 유사한 형태로 collection, transform 연산을 프로그래밍 할 수 있음.

        ```java
        KStreamBuilder builder = new KStreamBuilder();
        // 1. topic과 key-value serializer를 지정하여 KStream 생성
        KStream<Long, OperationLog> stream =
             builder.stream(sourceTopic.keySerde(), sourceTopic.valSerde(), sourceTopic.name());
        Map<String, Set<OpType>> categories = loadCategories();
        
        for (Map.Entry<String, Set<OpType>> entry : categories.entrySet()) {
            String topic = entry.getKey();
            Set<OpType> opTypes = entry.getValue();
            TalkOperationLogV1 destTopic =
                    InternalMessageTopics.getTopic(topic, TalkOperationLogV1.class);
        // 2. 각 요소에 필터 적용 
            stream.filter((key, value) -> {
                      TalkOperation op = value.getOperation();
                      return op != null && opTypes.contains(op.getOpType());
                  })
        // 3. 결과를 해당 카테고리에 대응하는 토픽에 저장
                  .to(destTopic.keySerde(), destTopic.valSerde(), destTopic.name());
        }
        Properties props = loadStreamsProps();
        
        KafkaStreams streams = new KafkaStreams(builder, props);
        streams.start();
        ```

        - 특정 기준으로 원본 토픽의 메시지를 필터링하여 새 토픽에 저장하는 예제 로직

- Low-level API

    -  통상적이지 않은 방법을 필요로 하는 몇몇 희귀한 경우에는 로우레벨 API사용

        - 메시지를 내용에 기반하여 특정 다운스트림으로 보내는 등

    - 프로세서 API는 대체로 직관적이고, 이해하기 어려운 부분은 없음.

## Fault-tolerance local state DB

- 스트림 프로세싱 구현을 하다보면 다양한 목적의 state를 보관해두어야 함

    - 로컬 state는 aggregation, join, windowing을 구현할 떄 사용하지만, 다른 경우에도 사용

- Kafka Streams에서는 각 프로세서가 고유의 state store를 가질 수 있음.

- 카프카 스트림의 changelog 메커니즘은 장애가 발생하여 프로세스가 다른 호스트로 failover하면 state DB도 새 프로세스로 이전됨

    - Kafka Streams가 state DB를 위한 물리 store를 업데이트하는 동안, changelog 토픽을 위한 특수한 메시지를 생성

        - 이 changelog topic은 로컬 state의 WAL로 간주됨

        - 물리 store는 pluggable하여, 인메모리DB나 RocksDB로 바꿀 수 있다.

    - 토픽은 반복해서 읽을 수 있고, 프로세서에 failover가 발생할 때 마다 새로운 프로세서가 changelog 토픽에서 읽은 mutation log를 리플레이하여 로컬 state DB 복구가 가능해진다.

- 프로세서 state를 보관하기 위한 별도 외부 스토리지가 필요 없이, 카프카 스트림으로 해결 가능

# Kafak Streams를 이용하여 구현한 것들

## Loopback replicator

- 토픽 레플리케이터를 구현했다.

    - 단순한 클러스터간 토픽 복제가 아닌, 맵 / 필터 등의 연산을 메시지에 적용하여 토픽을 복제하기 위함

    - 가장 주된 사용 목적은 원본 토픽에서 메시지를 분류, 용량이 적은 파생 토픽을 제공하여 컨슈머가 더 적은 메시지를 읽도록 하여 네트워크 트래픽과 리소스 사용량을 감축

        - 피크인 시간대에는 토픽으로 수신되는 메시지 수가 100만개에 육박하고, 모든 컨슈머가 필요하지 않음.

- 어떤 컨슈머가 라인의 TalkOperation의 일부 메시지만 받고싶다면, 전체 스트림을 읽을 필요가 없다.

    - 이때 loopback replicator를 사용해서 원하는 관련된 메시지만 읽어들일 수 있음.

## Decaton

- talk-dispatcher를 대체하기 위한 백그라운드 태스크 처리 시스템

- 기존 talk-dispatcher의 문제

    - 처리방식의 확장성 문제

        - 특정 talk-server에서 생성된 모든 태스크는 같은 호스트 내에서 구동중인 로컬 레디스 큐에 인입

        - 컨슈머인 talk-dispatcher도 같은 서버에서 구동되며, 로컬 레디스 인스턴스에 있는 태스크만 꺼내 처리

        - 서버에 버스트 발생하면, 태스크를 소비하는 인스턴스가 하나밖에 없어 queue의 크기가 엄청나게 커짐

    - In-memory queue

        - 레디스를 큐 서버로 사용하기 때문에, 서버가 종료되면 큐의 모든 내용이 유실됨

        - 물리적 메모리 한계로 등록할 수 있는 태스크의 수가 제한적임

        - 이미 대량의 태스크를 유실한 경험이 몇 차례 있음.

    - Out-of-order 처리

        - talk-server의 요청은 유저아이디 기반의 라우팅이 아니기 때문에, 어떤 talk-server도 처리할 수 있음.

        - 요청에 의해 생성된 태스크는 로컬 큐로 가겠지만,다른 서버로 간 요청으로 발생한 태스크는 다른 호스트의 큐로 등록됨

            - 다른 호스트로 보내지면서 각기 다른 talk-dispatcher 인스턴스에 의해 수행되고, 처리 순서와 등록 순서가 뒤섞일 수 있게 됨

- 카프카를 이용함으로서 얻는 것

    - 확장성 있는 분할된, 충분히 빠른 속도와 디스크 기반의 영속적인 큐

    - 메시지 키로 셔플링하여 순서를 지키며 태스크 처리 (userId를 키로 사용)

- 작동 원리 다이어그램

    [image](https://app.capacities.io/d1cf29da-a5be-492d-839a-e228b21e0129/51a33a63-4e68-49d2-81a1-2296e36c36fc)

    - talk-dispatcher의 큰 문제점을 해결하고, 서로 다른 프로세싱을 격리함

        - 카프카 토픽의 비휘발성 특징을 이용해 서로 다른 프로세서들이 같은 태스크에 대해 다른 처리를 격리되어 독립적으로 수행할 수 있음

        - 동일 컨슈머 컨택스트 내에서 연관되지 않은 태스크 처리 중 발생한 스토리지 요청이 오래 걸리거나 실패처럼 보이는 상황에도 다른 프로세스들의 태스크 처리가 멈추지 않도록 함.

            - TaskProcessorA와 B는 HBase 서버 장애로 처리가 무기한 멈춰도 StorageMutationProcessor 로부터 영향을 받지 않음.

                - 태스크가 queue에 쌓지만 토픽이 영속적이고 실질적인 용량제한이 없어 문제가 발생하지 않음

