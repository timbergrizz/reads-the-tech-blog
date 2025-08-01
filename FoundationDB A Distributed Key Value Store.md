---
type: Study
title: 'FoundationDB: A Distributed Key Value Store'
tags: [Data, Database]
출처: '[FoundationDB: A Distributed Key Value Store](https://dl.acm.org/doi/abs/10.1145/3542700.3542707?download=true)'
---

# Abstract

- FoundationDB는 10년전에 만들어진 오픈소스 트랜젝셔널 키밸류 스토어

    - ACID 트랜젝션과 NoSQL의 유연성, 확장 가능성을 갖춘 최초의 시스템 중 하나

- 인메모리 트랜젝션 관리 시스템과 분산 스토리지 시스템, 내장 분산 config 시스템을 분리한 unbundled 아키텍처를 채택함

    - 각 서브시스템은 독립적으로 배포되고 컨피그되어 확장 가능성과 고가용성, 오류 저항성을 갖춤

- 최소한의 핵심 기능만 제공하며, 다양한 시스템이 레이어로 위에 구성될 수 있도록 함.

# Introduction

- 많은 클라우드 서비스는 어플리케이션 state 저장을 위해 확장 가능한 분산 스토리지 백엔드에 의존함

    - 이러한 시스템은 오류에 저항성이 있어야 하고, 고가용성을 갖춰야 함

    - 동시에 충분히 강한 시멘틱과 유연한 데이터 모델을 제공해야 함

- NoSQL 시스템이 어플리케이션 개발을 쉽게 하고, 오류 저항성과 다양한 데이터모델을 제공하면서 스케일링과 스토리지 운영을 단순화함

    - 스케일링을 위해 트랜젝션을 희생했고, 결과적 동일성을 제공하지 않고 개발자에게 동시성 명령에 대해 고려하도록 함.

- FoundationDB는 하이 레벨의 분산 시스템을 만들기 위한 블록으로 만들어짐

    - 순차적, 트랜젝션이 지원되는, 전체 키 공간에 대해 다중 키를 지원하는 순차성이 엄격히 보장되는 트랜젝션을 지원하는 키밸류 스토리지

- 스토리지 엔진, 데이터 모델, 쿼리 언어가 번들된 다른 DB와 다르게 모듈러 접근을 사용함

    - 최소한의 필요한 기능을 갖춘 확장 가능한 트랜잭셔널 스토리지 엔진을 제공함

- NoSQL 모델은 개발자에게 확장성을 남겨둠.

    - 어플리케이션은 데이터를 간단한 키밸류 쌍으로 다룰 수도 있고, 추가적인 기능을 구현할 수도 있음

- FDB는 순차적 트랜젝션을 엄격히 지키도록 구성되었으나, 엄격함이 필요 없을경우 시멘틱을 완화함

- FDB는 DB에 있어 하위 계층(트랜젝션 처리 및 저장) 집중하고, 상위 계층(데이터 모델, 쿼리)은 레이어로 위임함

    - 레이어는 데이터 모델이나 다른 기능을 지원하기 위해 DB 위에 올려진 stateless application

    - 이를 통해 다른 형태의 스토리지를 원하던 어플리케이션도 FDB에서 사용할 수 있음.

        - 관계형으로 FDB Record Layer, 그래프DB로 JanusGraph등이 존재

- 분산시스템의 테스트와 디버깅은 설계만큼 어려움

    - 여러 오류가 있을 수 있어 재구현과 디버깅이 어렵고, 작은 버그가 전체 시스템에 크게 작용할 수 있으며, DB에선 작은 오류가 데이터 오염으로 이어질 수도 있음

    - FDB는 DB 그자체를 만들기 전에, 결정론적 DB 시뮬레이션 프레임워크를 만들어 프로세스의 상호작용과 디스크, 프로세스, 네트워크, 요청 레벨 실패와 회복 과정을 단일 프로세스 시뮬레이션으로 구현

- FDB는 컨트롤 플레인과 데이터 플레인으로 분리된 unbundled 아키텍처를 채택

    - 컨트롤 플레인은 클러스터의 메타데이터를 관리하고 고가용성을 위해 Active Disk Paxos를 사용한다

    - 데이터 플레인은 업데이트를 처리하는 트랜젝션 관리 시스템, 읽기를 제공하는 분산 스토리지 레이어를 제공한다.

        - 둘 다 분리되어 확장 가능하다.

- 순차성을 Optimistic Concurrency Control(OCC)과 Multi-Version Concurrency Control을 조합하여 제공

- FDB와 다른 DB랑 다른 점은 오류를 대응하는 방식

    - 오류 판정을 위해 quorum을 사용하지 않고, 오류를 적극적으로 탐지하고 시스템을 재설정하여 오류에서 회복하려 함

    - f+1개의 레플리카만 있으면 f개의 오류에서 회복할 수 있음.

# Design

- 메인 디자인 원칙은 다음과 같다

    - Divide-and-Conquer

        - 트랜젝션 관리 시스템(쓰기)을 분산 스토리지(읽기)에서 분리 후 각각 확장 가능하도록 구성

        - 트랜젝션 시스템 내부에서 독립 프로세스들이 트랜젝션의 특정 역할을 수행하도록 시스템을 구성

        - 클러스터 단위의 오케스트레이팅도 추가적인 룰 기반으로 분리되고 서비스된다.

    - Make failure a common case

        - 분산 시스템에서는 failure가 익셉션보다 일반적이다.

            - 이에 맞추어 모든 실패를  하나의 recovery path로 처리하도록 트랜젝션 시스템을 구성했다.

            - 뭐가 다른가요? failure는 일반적인 시스템 장애로 회복 가능한 경로로 수렴시킴, exception은 코드 수준의 예외 처리

        - 모든 failure 처리가 하나의 회복 명령으로 이어진다.

        - 가용성을 최대화하기 위해 Mean-Time-To-Recovery를 최소화하도록 구현한다.

    - Simulation testing

        - 랜덤화된 결정론적 시뮬레이션 프레임워크를 통해 분산 DB의 적절성을 테스트한다.

        - 깊은 버그를 찾아내며, 개발자의 생산성과 코드 퀄리티를 향상시킨다.

## Architecture

- FDB 클러스터는 중요한 메타데이터를 다루고, 클러스터를 오케스트레이션하기 위한 컨트롤 플레인이 존재한다.

    [image](https://app.capacities.io/d1cf29da-a5be-492d-839a-e228b21e0129/43e2b82c-1e34-45c0-9337-a97131cf166e)

### Control Plane

- 시스템의 중요 메타데이터에 영속성을 부여하는 역할

- 코디네이터는 Paxos 그룹을 형성하고, ClusterController를 선출한다.

- ClusterController는 클러스터의 모든 서버를 모니터링하고 3개의 프로세스를 선출하여 역할을 부여한다.

    - Sequncer

    - Distributer : 실패를 모니터링하고 StorageServer 간의 데이터를 밸런싱한다.

    - Ratekeeper : 클러스터에서 오버로드에 대한 보호를 제공한다.

    - 오류 나면 다시 선출한다.

## Data Plane

- FDB는 읽기가 주로 발생하고, 트랜젝션에 작은 셋의 키를 읽고 쓰며, 스케일링이 필요한 OLTP 워크로드를 타겟한다.

- Sequencer, Proxies, Resolver라는 stateless 프로세스들로 분산 트랜젝션 관리 시스템을 구성한다.

    - Sequencer

        - 읽기를 할당하고 각 트랜지션에 버전을 커밋하며, Proxies, Resolvers, LogServers를 선출한다.

    - Proxies

        - 클라이언트에 MVCC 읽기 버전을 제공하고, 트랜젝션 커밋을 오케스트레이션한다.

    - Resolvers

        - 트랜젝션 사이의 컨플릭트를 확인한다.

- Log system

    - 트랜젝션 시스템의 WAL 로그를 저장한다

- Storage System

    - 데이터를 저장하고 읽기를 제공한다.

    - StorageServer로 구성된다.

    - StorageServers

        - 데이터 샤드를 저장하며, 클라이언트 읽기를 제공한다.

        - 시스템의 대부분의 프로세스이며, 분산 B-tree를 구성한다.

### Read-Write Separation and Scaling

- 프로세스들엔 다른 역할이 부여되고, FDB는 각 롤의 프로세스를 추가하여 확장을 수행한다.

    - 클라이언트는 샤딩된 StorageServer에서 데이터를 읽어오므로, 이 프로세스를 추가하면 읽기가 선형저긍로 확장된다.

    - 쓰기는 Proxies, Resolvers, LogServers를 추가하면 확장된다.

- 컨트롤 플레인의 싱글톤 프로세스와 코디네이터는 한정된 메타데이터 작동만 하기 때문에 병목이 아니다.

### Bootstrapping

- FDB는 외부 코디네이션 서비스에 의존성이 없다.

    - 모든 유저 데이터와 대부분의 시스템 메타데이터는 StorageServers에 저장된다.

    - StorageServers의 메타데이터는 LogServers에, LogServers의 메타데이터는 Coordinator에 저장된다.

- 과정

    - Coordinator는 디스크 Paxos 그룹이고, 서버는 ClusterController가 없으면 되려고 시도한다.

    - 새로 선출된 컨트롤러는 코디네이터에서 기존 LS 설정을 확인해 새로운 Sequencer를 선출하고 TS와 LS를 새롭개 생성한다.

    - 프록시는 기존 LS로부터 SS에 대한 정보를 포함한 시스템 메타데이터를 복구한다.

    - 시퀀서는 TS가 복구될때까지 기다리고, 새로운 LS의 config를 모든 코디네이터에 작성한다.

    - 이제 새로운 트랜젝션 시스템이 클라이언트의 요청을 받을 수 있다.

### Reconfiguration

- Sequencer는 Proxies, Resolvers, LogServers의 상태를 모니터링한다.

- TS나 LS에 failure가 발생하거나  config가 변경되면 Sequencer가 죽는다.

    - ClusterController는 이걸 시퀀서 failure로 여기고, 새로운 Sequencer를 선출한다.

        - 과정은 새로운 TS와 LS를 생성하는 것과 동일하다

- 트랜젝션 프로세싱은 epoch로 분할되며, 각 epoch는 트랜젝션 관리 시스템의 생성으로 대표된다.

## Transaction Manangement

### End-to-end Transactional Processing

- 트랜젝션은 프록시로 트랜젝션 버전을 읽으면서 시작된다.

    - 프록시는 시퀀서에 읽기 버전을 질의하하고, 클라이언트로 읽기 버전을 반환한다.

    - 클라이언트는 StorageServer로 읽기 요청을 보내고 읽기 버전에 대한 결과값을 얻는다.

- 클라이언트의 쓰기는 클라이언트에 요청하지 않고 로컬에 버퍼된다.

    - read-your-write는 lookup을 아직 커밋되지 않은 트랜젝션 데이터와 결합하여 지켜진다.

    - 커밋 시점에 클라이언트는 읽기 쓰기 셋이 포함된 트랜젝션 데이터를 프록시로 보내고, 커밋할지 버릴지 여부를 결정한다.

        - 트랜젝션이 커밋이 안된다면, 클라이언트가 재시작 될 수도 있다.

- 프록시는 클라이언트의 트랜젝션을 3단계로 커밋한다.

    - 시퀀서로부터 커밋 버전을 받아, 지금 존재하는 모든 읽기 / 커밋 버전보다 큰지 확인한다.

        - 시퀀서는 커밋 버전을 100만초에 1번씩 업데이트하여 선택한다.

    - 프록시는 트랜젝션 정보를 범위로 파티션된 Resolver로 보낸다.

        - 모든 Resolver에서 충돌이 없다면, 트랜젝션은 최종 커밋으로 이어진다.

        - 문제가 있다면 프록시는 이 트랜젝션을 실패한 것으로 간주한다.

    - 트랜젝션은 LogServer로 전송되어 영속성을 가지며, 커밋된다.

        - 모든 LogServer가 프록시에 응답하면 커밋되었다고 여기고 시퀀서로 커밋된 버전을 리포트하고, 클라이언트에 응답을 한다.

- StorageServer는 LogServer로부터 변경 로그를 풀링하여 디스크로 커밋한다.

- Read-write 트렌젝션뿐만이 아니라 read-only와 snapshot read도 지웒나다.

    - 읽기 전용 트렌젝션은 연속적이며 성능이 고려되었으며, 클라이언트가 DB에 컨택하지 않고도 수행할 수 있다.

    - Snapshot read는 컨플릭트를 줄여 트랜젝션의 독립된 속성을 줄인다.

### Strict Serializability

- OCC와 MVCC를 조합하여 Serializable Snapshot Isolation를 구현했다.

    - 트랜젝션은 읽기 버전과 커밋 버전을 시퀀서로부터 모두 부여받는다.

        - 읽기 버전은 트랜젝션이 시작할 때 어떤 커밋 버전보다 작지 않음이 보장된다.

        - 커밋 버전은 존재하는 모든 읽기 버전과 커밋 버전보다 크다.

            - 트랜젝션의 연속적인 히스토리를 정의하고, Log Sequence Number로 작동한다.

- 트랜젝션이 이전의 모든 트랜젝션의 결과를 관측하기 때문에 엄격한 Serializability를 구현할 수 있다.

    - LSN사이에 간격이 없도록 보장하기 위해, 시퀀서는 각 커밋 버전에 대해 이전 커밋 버전을 같이 반환하도록 한다.

    - 프록시는 LSN과 이전의 LSN을 리졸버와 로그서버로 보내 LSN 순서로 트렌젝션을 처리할 수 있도록 한다.

    - 스토리지 서버는 로그서버로부터 LSN이 증가하는 순서대로 로그를 가져온다.

- Resolver는 write-snapshot isolation과 유사한 lock-free 충돌 감지 알고리즘을 사용한다.

    - FDB는 커밋 버전을 충돌 감지 전에 선택한다는 점이 다르다.

    - 트랜젝션은 모든 Resolver가 트랜젝션을 승인해야 커밋되고, 아니면 abort된다.

    - 문제는 일부 리졸버가 이미 abort된 트랜젝션을 승인 상태로 간주하고 히스토리를 업데이트 할 수 있으며, 이로 인해 불필요한 충돌이 발생할 수 있다..

        - 이렇게 되면 다른 트랜젝션과 충돌이 발생할 수 있긴 한데, 프로덕션 워크로드에서는 일반적으로 키가 하나의 리졸버로 가기 때문에 문제가 없었다.

            - 추가로, modified key는 MVCC 윈도우 이후 만료되기 때문에, 짧은 시간동안만 존재한다.

    - OCC는 락을 얻고 푸는 복잡한 로직을 피했다.

        - 이를 통해 TS와 SS 사이에 상호 작용을 단순화한다.

        - 비용은 abort된 트랜젝션이 한 작업

    - 멀티 테넌트 프로덕션 워크로드에서 트랜젝션 충돌이 1%정도로 낮고 OCC가 잘 작동했다.

        - 컨플릭트 나면 클라이언트가 단순하게 트랜젝션 다시 시작하면 된다.

### Logging Protocol

- 프록시가 트랜젝션을 커밋하기로 결정하면, 모든 로그 서버로 메시지를 보낸다.

    - 수정된 키 범위를 책임지는 모든 로그서버에 변경이 전송되고, 나머지는 빈 메시지가 전송된다.

- 로그 메시지 헤더는 시퀀서로부터 받은 현재와 과거의 LSN과 프록시에 가장 많이 알려진 커밋 버전을 같이 보낸다.

    - 로그 서버는 로그 데이터가 저장되면 프록시에 응답한다.

    - 이후 프록시는 모든 로그서버 레플리카로부터 응답을 받고, 현재 KVC보다 LSN이 크면 KVC를 LSN으로 업데이트한다.

- 로그 서버에서 스토리지 서버로 redo 로그를 보내는 것은 커밋 경로의 일부가 아니며, 비동기적으로 백그라운드에서 일어난다.

    - 스토리지 서버는 강건하지 않은 redo 로그를 로그서버에서 인메모리 인덱스로 적용한다.

        - 일반적으로 읽기 버전을 클라이언트에 반환하기 전에 이를 수행한다.

        - 클라이언트의 읽기 요청이 스토리지 서버에 도달하면 요청한 버전이 항상 이미 준비되어 있다.

- 클라이언트가 새로운 데이터가 반영되지 않은 스토리지 서버로 요청을 보낸 경우, 클라이언트는 기다리거나 다른 레플리카로 요청을 보낼 수 있다.

    - 모든 읽기 요청이 타임아웃되면 트랜젝션을 다시 시작한다.

    - 로그 데이터가 이미 로그 서버에서 durable하다.

        - 스토리지 서버는 업데이트를 메모리에 업데이트하고 주기적으로 배치성으로 디스크에 저장하여 IO성능을 개선하는데 집중한다.

### Transaction System Recovery

- 전통 DB 시스템은 ARIES 복구 프로토콜을 채택한다.

    - 복구동안 시스템 프로세스는 마지막 체크포인트로부터 redo 로그를 처리하도록 한다.

    - 이를 통해 DB가 일관된 상태를 유지하도록 한다.

        - 오류상태에서 발생한 트랜젝션은 undo를 수행하여 롤백된다.

- FDB에서 복구는 최대한 비용이 작도록 설계되었다.

    - 디자인 초이스를 단순화하여 undo를 적용할 필요가 없도록 만들었다.

        - redo 로그를 처리하는 것이 그냥 로그를 처리하는 것과 동일한 경로로 작동한다.

    - 스토리지서버는 로그서버로부터 로그를 풀링하고 백그라운드에서 적용한다.

- 복구 프로세스는 failure를 발견하고 새로운 트랜젝션 시스템을 채용하며 시작된다.

    - 새로운 TS는 기존 로그 서버의 데이터를 모두 처리하기 전에 트랜젝션을 수용할 수 있다.

    - 복원은 redo log를 찾을때까지만 수행하면 되고, 이 과정에서 스토리지 서버는 비동기적으로 로그를 다시 수행한다.

- 각 epoch마다 시퀀서는 여러 단계로 복구를 수행한다.

    1. 기존 TS 설정을 코디네이터로부터 읽고, 다음 시퀀서가 동시에 복구하지 않도록 락을 건다.

    2. 오래된 LogServer의 정보를 포함한 이전의 TS 시스템의 상태를 복원하고, 트랜젝션을 받지 못하도록 한다.

    3. 새로운 프록시, Resolver, 로그 서버를 채용한다.

    4. 이전 로그서버가 중단되고 새로운 TS가 채용되면 시퀀서는 TS의 정보를 코디네이터에게 준다.

- 프록시와 Resolver는 stateless 해서 복구에서 따로 할 일이 없다.

- 로그서버는 커밋된 모든 트랜젝션의 로그를 저장하므로, 모든 트랜젝션이 durable하며 스토리지 서버가 얻을 수 있도록 해야 한다.

- 기존 로그서버의 복구는 redo log의 종료 여부를 결정하는 것이 핵심이다.

    - undo 로그로 롤백하는 것은 로그서버와 스토리지서버의 복원 버전 이후의 모든 데이터를 버리는 것과 동일하다.

    - 복원 버전 (RV)를 결정하는 알고리즘은 다음과 같이 결정된다.

        [image](https://app.capacities.io/d1cf29da-a5be-492d-839a-e228b21e0129/3633bbe3-c3df-459a-87e4-c05adbec416f)

    - 각 로그 서버는 프록시로부터 받은 KCV와 로그서버가 갖고있는 LSN의 최댓값인 Durable Version을 갖고있다.

        - 복원동안 시퀀서는 모든 기존 LogServer를 중지하려 한다.

        - replication degree가 k일 때, 시퀀서가 m-k개보다 많은 reply를 받으면 시퀀서는 이전 에포크가 모든 KCV의 최대값까지 트랜젝션을 커밋했다는 것을 알 수 있다.

            - 이는 Previous Epoch's end Version이라 한다. 

        - 현재 Epoch에 대해 시작 버전은 PEV+1이고, 시퀀서는 모든 DV의 최소가 RV가 되도록 한다.

            - 그 사이의 로그는 이전 에포크의 LS로부터 현재로 복제되어 상태 복구에 사용된다.

        - 수 초간에 발생한 로그 데이터만을 포함하기 떄문에 복제에 발생하는 오버헤드는 매우 작다.

- 새로운 시퀀서가 처음으로 수행하는 트랜젝션은 모든 스토리지 서버에 RV를 알려주는 작업이다.

    - 이를 통해 RV보다 큰 데이터를 롤백할 수 있다.

- 스토리지 엔진은 버전이 없는 SQLite B트리와 인메모리 버전 redo 로그 데이터로 구성된다.

    - MVCC윈도우를 벗어난 mutation만 SQLite로 작성된다.

    - 롤백은 스토리지 서버의 인메모리의 버전 데이터를 버리는 방식으로 작동한다.

        - 이후 PEV보다 큰 데이터를 로그 서버에서 가져오는 방법으로 작동한다.;

## Replication

- FDB는 다양한 복제 전략을 사용한다.

- Metadata replication

    - 컨트롤 플레인의 시스템 메타데이터는 코디네이터에 동적 디스크 Paxos를 통해 저장된다.

    - 코디네이터의 quorum이 살아있으면 메타데이터를 회복할 수 있다.

- Log Replication

    - 프록시가 로그서버에 쓰기를 수행하면, 샤드된 로그 레코드는 동기적으로 k=f+1개의 로그 서버로 복제된다.

        - 모든 k가 성공적으로 영속성을 했을 때 프록시가 클라이언트로 커밋 응답을 보낸다.

    - 로그서버가 Failure나면 트랜젝션 시스템 복구로 들어간다.

- Storage Replication

    - 모든 샤드는 비동기적으로 k = f + 1 스토리지 서버로 복제된다.

    - 스토리지 서버는 샤드의 개수만큼 호스트되어 데이터가 고르게 분포될 수 있도록 한다.

    - 스토리지 서버에 failure나면 DataDistributer를 트리거하여 실패한 팀에서 건강한 팀으로 옮기도록 한다.

    - 스토리지 팀 추상화는 Copyset보다 훨씬 복잡하다.

- 동시적인 실패로 인한 데이터 손실 가능성을 낮추기 위해 FDB는 레플리카의 최소한 하나의 프로세스가 Fault domain에 있도록 한다.

    - 각 팀의 최소 하나의 프로세스는 fault domain에서 데이터 손실이 없도록 보장하도록 한다

# Simulation Testing

- 분산 시스템에서의 테스트와 디버깅을 일종의 도전 과제고, 충분하지 않은 프로세스

    - 따라서 처음부터 E2E 테스트가 고려되었다.

        - 실제 DB가 돌아가고, 결정론적인 이벤트 시뮬레이터 안에서 임의로 생성된 워크로드와 오류들이 삽입된다.

## Deterministic Simulator

[image](https://app.capacities.io/d1cf29da-a5be-492d-839a-e228b21e0129/ab729ad3-afc6-4d08-8bce-43ba16e87935)

- FDB는 이러한 테스트 접근을 기반으로 만들어졌다.

- 모든 DB 코드는 결정론적이고, 멀티쓰레드 동시성을 피했다.

- 시뮬레이터 프로세스는 네트워크, 디스크, 시간, 랜덤 생성같은 모든 커뮤니케이션과 비결정론적인 프로세스를 추상화했다.

- C++의 async-await 동시성 모델인 Flow로 작성되어 Actor 모델을 사용하였다.

    - FDB 서버의 다양한 액션을 Flow 런타임 라이브러리로 스케줄된 actor로 추상화했다.

- 시뮬레이터는 하나의 프로세스 내에 여러개의 서버를 생성하고,  이들 간의 통신을 시뮬레이션 네트워크를 통해 구현한다..

    - Flow로 작성된 워크로드를 시뮬레이션의 서버과 시뮬레이션 네트워크로 구동한다.

    - 워크로드에는 fault injection 지시나, 목 어플리케이션,  DB 설정 변경, DB 내부 기능 실행들이 포함된다.

## Test oracles

- 시뮬레이션에서 실패를 감지하기 위해 다양한 테스트 오라클을 사용한다

- 대부분의 만들어진 워크로드는 데이터베이스의 속성과 contract를 확인하기 위한 Assertion이 있다.

    - Assertion은 코드베이스 전반에서 로컬로 확인할 수 있는 속성을 확인하는데 사용된다.

    - 회복 가능성같은 속성은 모델링된 하드웨어 환경을 복원이 가능해야 하는 상황으로 보내고, 클러스터가 복원되는지를 확인한다.

## Fault Injections

- 시뮬레이션은 하드웨어의 failure와 리부트, 네트워크 오류, 파티션과 레이턴시 문제, 디스크의 행동, 랜덤화된 이벤트 시간을 주입한다.

    - 버그를 찾기 위해 인위적인 장애를 주입하는 매커니즘이다.

- 다양한 fault의 주입은 특정 오류에 대한 회복력을 테스트하고 시뮬레이션의 상태를 다양화한다.

- fault injection 분포는 신중하게 조정되어 과도한 장애주입으로 상태 공간이 비정상적으로 좁이지지 않게 한다..

- FDB는 시뮬레이션과 협동하여 특수한 상태와 이벤트를 더 흔하게 만든다. buggification

    - 코드베이스의 많은 자리에서 시뮬레이션은 일반적이지 않은 동작을 주입할 수 있게 된다.

        - 보통 성공하는 동작에 에러를 반환하고, 튜닝 파라미터로 일반적이지 않은 값을 삽입하는 등

    - 네트워크와 하드웨어 레벨에서 fault injection을 채운다.

- Swarm testing은 시뮬레이션 동작의 다양성을 최대화하기 위하여 사용된다.

    - 각 런은 랜덤 사이즈의 클러스터와 설정, 워크로드, fault injection 파라미터, 튜닝 파라미터를 사용하고 buggification 부분집합을 활성화/비활성화 한다.

## Latency to bug discovery

- 버그를 빠르게 찾는 것은 중요하다.

    - 프로덕션 직전에 테스트를 마주했을 때를 위해, 그리고 엔지니어링 생산성을 위해

-  별개의 이벤트 시뮬레이션은 시뮬레이션의 CPU 사용량이 낮을 경우 리얼타임보다 빠르게 작동한다.

    - 시뮬레이터가 다음 이벤트로 빠르게 시계를 돌릴 수 있기 때문

- 많은 분산 시스템의 버그는 찾는데 오래걸리고, 낮은 사용률의 시뮬레이션을 길게 작동시키는 것은 실제 환경에서의 E2E 테스트보다 더 오래걸린다.

- 추가로, 랜덤화된 테스트는 병렬적으로 동작하고, FDB 개발자들이 엄청냔 양의 테스트를 메이저 배포 전에 수행한다.

    - 찾을 공간이 효율적으로 무한하기 떄문에, 테스트를 더 돌릴수록 커버리지가 올라가고 버그가 될 수 있는 지점을 찾을 수 있다.

## Limitations

- 시뮬레이션은 성능 병목이나 실제 하드웨어 상의 성능 저하 이슈를 포착하는데 한계가 있다

- 서드 파티 라이브러리나 의존성, Flow로 구현된 퍼스트 파티 코드를 확인할 수 없다.

    - 결과적으로, 외부 시스템에 의존성을 갖는 것을 피하게 되었다.

- 파일 시스템이나 OS같은 중요한 연관 시스템의 버그, 그들의 보장에 대한 오해도 FDB의 버그로 이어진다.

# Evaluation

- 실질적인 사용 속도에 관한 데이터로, 요약에서 생략

# Lessons Learned

## Architecture Design

- 분할 정복 디자인 원칙이 유연한 클라우드 배포를 가능하도록 함이 증명되었다.

    - 데이터베이스를 확장장 가능하며 좋은 성능으로 디자인했다.

- 트랜젝션 시스템과 스토리지 레이어를 분리함으로서 연산과 스토리지 자원을 스케일링하는 것에 큰 유연성을 주었다.

    - LogServer는 증인 복제본과 유사하고, 우리의 멀티리전 클라우드 프로덕션 워크로드에서 동일한 정도의 고가용성 속성을 얻기 위해 필요한 Storage Server 개수를 줄일 수 있었다.

    - 오퍼레이터는 다른 서버 인스턴스 타입에 따라 FDB의 다양한 역할을 자유롭게 위치시킬 수 있고, 성능과 비용으로 최적화할 수 있다.

- 디커플링 디자인으로 DB의 기능을 확장할 수 있었다.

- 최근 성능 향상의 많은 부분은 적절한 역할에 특수화된 기능을 부여하여 가능했다.

    - 시퀀서로부터 DataDistributor와 Ratekeeper를 분리하고, 스토리지 캐시를 추가하고, 프록시를 get-read-version과 commit로 분리하였다.

## Simulation Testing

- 시뮬레이션 테스팅은 버그가 들어가고 이를 찾는 과정을 빠르게 하여 작은 팀으로 빠르게 개발할 수 있도록 하였다.

    - 또한 결정론적인 이슈의 재현이 가능하도록 한 점도 도움이 되었다.

    - 추가적인 로깅을 하는것은 일반적으로 이벤트의 결정론적 순서에 영향을 끼치지 않고, 따라서 재현이 보장된다.

- 이러한 디버깅 접근법은 일반적인 프로덕션 디버깅보다 강하다

    - 자연 상태에서 버그를 찾으면 디버깅 프로세스에서는 처음으로 하는 것은 이를 재현할 수 있는지를 시뮬레이션으로 발전시키고, 디버깅 프로세스로 이어진다.

    - 시뮬레이션을 통한 교정 프로세스를 통해 FDB의 신뢰도를 높일 수 있었다.

- 시뮬레이션의 성공은 의존성을 없애고 플로우로 재구현을 하여 시뮬레이션 테스팅의 바운더리를 넓히도록 발전했다.

    - 초기 버전엔 주키퍼 썼는데 리얼월드 fault injection이 2개의 개별 버그를 발견한 후 제거되었다.

    - Flow로 작성된 Paxos 구현으로 대체되었다.

## Fast Recovery

- 빠른 복구는 가용성을 늘릴 뿐 아니라 소프트웨어 업그레이드와 설정 변경을 단순화하고 빠르게 했다.

    - 모든 트랜젝션 시스템은 Stateless하며 재시작만으로 북구될 수 있다.

- 전통적인 분산 시스템 업그레이드는 롤링 업그레이드 해서 롤백 가능하도록 하는 것이 일반적이었다..

    - FoundationDB는 프로세스를 모두 동시에 재시작하여 업데이트를 수행할 수 있고, 몇 초 안에 가능하다.

    - 이러한 업그레이드 경로는 충분히 테스트되었고, 애플의 프로덕션 환경 표준 업그레이드 방식이다.

## 5s MVCC Window

- FDB는 TS와 스토리지 서버의 메모리 사용량을 제한하기 위해 5초 MVCC윈도우를 선택했다.

    - 멀티 버전 데이터는 리졸버와 스토리지 서버의 메모리에 저장되고, 트랜젝션 사이즈를 제한한다.

- 경험상 5초면 대부분의 OLTP 유즈케이스를 처리할 수 있다.

    - 시간 제한을 넘면 클라이언트가 비효율적인 작업을 하는 것.

        - 병렬 읽기 대신 하나씩 읽는다던가

    - 결과적으로 시간 제한 초과는 어플리케이션의 비효율성을 드러낸다.

- 트랜젝션이 5초보다 길어지는 경우, 대부분은 더 작은 트랜젝션으로 분할될 수 있다.

    - FDB의 연속적인 백업 프로세스는 키스페이스를 스캔해서 키 스냅샷을 만드는데, 스캐닝 프로세스는 작은 범위로 나누어져 5초 내에 수행할 수 있도록 구성된다.

    - 일반적인 패턴이다. 하나의 트랜젝션이 분할된 잡을 생성하고, 각 잡을 트랜젝션으로 수행하는 것.

# Conclusions

- 파운데이션DB는 OLTP 클라우드 서비스를 위해 설계되었다.

- 메인 아이디어는 트랜젝션 프로세스와 로깅, 스토리지를 분리하는 것

    - 이러한 언번들 아키텍처가 읽기 / 쓰기 핸들링에 대한 분리와 수직 스케일링을 가능하게 했다.

- 트랜젝션 시스템은 OCC와 MVCC를 조합하여 강력한 serializability를 보장했다.

- 로깅의 분리와 트랜젝션 순서의 결정성은 복원을 단순화하고, 빠른 복구와 가용성 향상으로 이어졌다.

- 결정론적 랜덤화 테스트는 데이터베이스의 correctness를 강화했다.

