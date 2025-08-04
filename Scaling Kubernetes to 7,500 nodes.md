---
type: Study
title: Scaling Kubernetes to 7,500 nodes
tags: [Kubernetes, LLM]
출처: '[Scaling Kubernetes to 7,500 nodes](https://openai.com/index/scaling-kubernetes-to-7500-nodes/)'
---

- 단일 쿠버네티스 클러스터를 이런 사이즈로 작동시키는것은 매우 흔하지 않고, 특별한 관리를 필요로 한다

    - 그럼에도 이러한 간단한 인프라가 ML 리서치 팀을 빠르게 움직이고, 코드 변경 없이 스케일업 할 수 있도록 한다.

- [Scaling Kubernetes to 2,500 nodes](https://app.capacities.io/d1cf29da-a5be-492d-839a-e228b21e0129/26f2238e-ede2-415d-aea5-30b0fb363d94) 이후로 연구자들의 필요에 따라 인프라가 커졌고, 이러한 과정에서 추가적으로 많은 것들을 배울 수 있었다.

# Our workload

- 쿠버네티스에서 사용하는 어플리케이션과 하드웨어가 일반적인 회사에서 보는 것과는 다르다.

- 거대한 ML 작업은 많은 노드에 퍼져 각 노드의 모든 하드웨어 자원에 접근하여 작동할 때 가장 효율적으로 작동한다.

    - GPU가 [NVLink](https://www.nvidia.com/en-us/data-center/nvlink/)를 통해 상호 통신을 하거나, NIC와 GPU가 [GPUDirect](https://developer.nvidia.com/gpudirect)로 직접 통신하도록 한다.

        - 이러한 직접 통신으로 호스트 CPU를 거치지 않고 최고의 성능을 내도록 할 수 있다.

- 대부분의 워크로드에서 단일 파드가 전체 노드를 차지한다.

    - 어떤 NUMA, CPU, PCIe 자원 논쟁도 스케줄에 고려되지 않는다.

    - Bin-packing이나 fragmentation은 일반적으로 문제가 되지 않는다.

    - 현재 클러스터는 full bisection 대역폭을 가져, 어떤 랙이나 네트워크 topology를 만들지도 않는다.

    - 많은 노드를 가지고 있지만 스케줄러에 걸리는 부하는 적다.

        - kube-scheduler로 걸리는 요청은 스파이크 성이다.

            - 새로운 잡으로 몇백개의 파드가 한번에 생성되고, 다시 요청이 없어진다.

- 가장 큰 잡은 MPI를 실행시키고, 잡에 속한 파드는 단일 MPI 커뮤니케이터로 참여한다.

    - 참여 파드중에 하나라도 죽으면, 전체 잡이 버려지고 체크포인트로부터 재시작되어야 한다.

    - 따라서, 죽은 파드가 대체되어 작업이 다시 이어질 수 있도록 파드를 semi-stateful하도록 파드를 설계하여야 한다.

        - 이렇게 작동하는 것은 지장을 주며, 최소한으로 유지되어야 한다.

- 쿠버네티스의 로드밸런싱에 의존하지 않는다. https 트래픽이 매우 적으며, A/B 테스팅이나 블루그린, 카나리 배포를 필요로 하지 않는다.

    - 파드는 서비스 엔드포인트가 아닌 SSH를 통한 MPI로 다른 파드와 직접 통신한다.

    - 시작 시점에 어떤 파드가 MPI에 참여하는지 확인만 하기 때문에, 서비스 "디스커버리"는 제한적이다.

- 대부분의 잡은 몇개의 형태의 blob 저장소와 상호작용 한다.

    - 데이터셋의 일부나 체크포인트를 blob 스토리지로 스트림하거나, 빠른 로컬 임시 디스크로 캐시한다.

    - POSIX 시멘틱이 유용한 경우를 위한 PV를 갖고 있으나, blob 스토리지가 확장도 가능하고 느린 detach/attach 동작을 하지 않아 사용한다.

- 연구가 기본 워크로드고, 워크로드가 항상 달라진다.

    - 슈퍼컴퓨팅 팀이 프로덕션 레벨 컴퓨팅 인프라를 만들려고 노력하지만, 클러스터에 동작하는 어플리케이션의 수명은 짧고 개발자들은 빠르게 반복 수행한다.

    - 트렌드와 적절한 트레이드오프에 대한 예상에 도전하는 새로운 사용 패턴은 언제라도 나타날 수 있다.

# Networking

- 노드와 파드의 개수가 증가하면서, Flannel이 스케일업 과정에서 성능이 문제가 있음을 발견했다.

    - Azure VMSS의 IP 구성에 대해 네이티브 파드 네트워킹 기술과 관련 CNI 플러그인을 사용하도록 전환했다.

    - 이를 통해 호스트 레벨 수준의 네트워크 처리량을 달성할 수 있었다.

- 바꾼 또다른 이유는 우리의 최대 클러스터에서 alias-based IP addressing로 전환했기 때문이다.

    - 이론상 동시에 20만개의 IP 주소를 사용할 수 있었고, 기존의 라우트 기반 파드 네트워킹에는 네트워크 주소의 효율적인 사용에 제한이 있었다.

    - 캡슐화를 피하는 것은 라우팅 엔진이나 내포된 SDN의 요청을 늘렸지만, 네트워크를 심플하게 유지할 수 있엇다.

    - VPN이나 터널링을 구현하는 것은 어댑터 없이 잘 작동했다.

    - 네트워크의 일부가 더 낮은 MTU를 가져, 패킷 파편화를 피할 수 있었다..

    - 네트워크 정책과 트래픽 모니터링은 패킷의 소스와 도착지에 대한 모호함이 없어져 단순해졌다.

- 호스트에 태깅된 iptable로 네임스페이스와 파드당 네트워크 자원 사용량을 추적했다.

    - 리서쳐들이 네트워크 사용 패턴을 시각화할 수 있었다.

    - 많은 실험이 분리된 인터넷과 파드간 커뮤니케이션을 기반으로 하여, 병목을 조사하기 용이했다.

- iptable의 mangle 규칙은 특정 규칙을 만족하는 패킷을 임의로 마킹하기 위해 사용되었다.

    - Forward 규칙은 파드로부터의 트래픽을, Input, Output은 호스트로부터의 트래픽이다.

        ```text
        iptables -t mangle -A INPUT ! -s 10.0.0.0/8 -m comment --comment "iptables-exporter openai traffic=internet-in"
        iptables -t mangle -A FORWARD ! -s 10.0.0.0/8 -m comment --comment "iptables-exporter openai traffic=internet-in"
        iptables -t mangle -A OUTPUT ! -d 10.0.0.0/8 -m comment --comment "iptables-exporter openai traffic=internet-out"
        iptables -t mangle -A FORWARD ! -d 10.0.0.0/8 -m comment --comment "iptables-exporter openai traffic=internet-out"
        ```

    - 마킹되면 iptable은 카운터로 규칙을 만족하는 바이트와 패킷 수를 센다.

        ```text
        % iptables -t mangle -L -v
        Chain FORWARD (policy ACCEPT 50M packets, 334G bytes)
         pkts bytes target     prot opt in     out     source               destination
        ....
        1253K  555M            all  --  any    any     anywhere            !10.0.0.0/8           /* iptables-exporter openai traffic=internet-out */
        1161K 7937M            all  --  any    any    !10.0.0.0/8           anywhere             /* iptables-exporter openai traffic=internet-in */
        ```

    - [오픈소스 프로메테우스 익스포터로](https://github.com/madron/iptables-exporter/) 데이터를 모니터링 시스템으로 가져와 추적했다.

- 네트워크 모델의 특수한 점으로 노드, 파드, 서비스 네트워크 CIDR을 연구자에게 노출한다는 점이다.

    - 허브와 스포크 네트워크 모델이 있고, 트래픽을 라우팅하기 위해 네이티브 노드와 파드의 CIDR 레인지를 사용한다.

    - 연구자는 허브에 연결해 개별 클러스터에 접근한다.

    - 클러스터 그 자체는 아무말도 하지 않고, 이는 클러스터를 독립시켜 failure 격리를 무너뜨리는 클러스터간의 의존성이 발생하지 않도록 한다.

# API Servers

- 쿠버네티스 API서버와 etcd는 건강한 클러스트의 중요한 요소이고, 이러한 시스템에 스트레스를 줄때 많은 주의를 기울인다.

- [kube-prometheus](https://github.com/coreos/kube-prometheus)와 그라파나, 인하우스 대시보드를 사용한다.

    - HTTP 429와 5XX 알림이 높은 수준의 문제 신호라는것을 찾았다.

- 항상 쿠버네티스 외부의 지정된 노드에서 etcd와 API 서버를 작동시킨다.

    - 가장 큰 클러스터는 5개의 API 서버와 5개의 etcd 노드로 로드를 분산하고 장애 여파를 최소화한다.

    - 분리된 etcd 클러스터로 쿠버네티스 이벤트를 분리한 이후로 etcd 관련된 큰 문제는 없었다.

    - API 서버는 stateless하고 일반적으로 자가 복구 가능한 인스턴스 그룹이나 스케일셋에서 작동시킬 수 있다.

    - etcd 클러스터에 대해서는 사고가 거의 발생하지 않아 자가 복구 자동화를 수행하지 않았따.

- API 서버는 꽤 많은 메모리를 먹고, 노드 개수가 증가하면 선형적으로 증가한다.

    - 7500개의 노드에서 API 서버마다 70GB의 힙이 사용되었다. 

- API서버의 큰 제한은 엔드포인트의 WATCH이다.

    - kubelet이나 node-exporter같은 모든 노드가 멤버로 있는 일부 서비스이다.

    - 노드가 클러스터에 추가되거나 제거되면 WATCH에 부하가 온다.

    - 일반적으로 각 노드들이 kubelet 서비스를 kube-proxy를 통해 보고 있어, 응답을 위해 필요한 #와 대역폭은 n^2 이상의 큰 값이며, 일반적으로 1GB/s이다.

        - [EndpointSlice](https://kubernetes.io/docs/concepts/services-networking/endpoint-slices/)로 1/1000 규모로 감소시킬 수 있었다.

- 클러스터의 사이즈를 조정하는 API 서버 요청에 대해 집중한다.

    - API서버의 부하를 줄이기 위해 DaemonSet이 API 서버와 직접 상호작용하는 것을 최소화한다.

    - 각 노드가 변경을 확인하려는 경우 API 서버에 직접 요청하는 대신 [Datadog Cluster Agent](https://docs.datadoghq.com/agent/cluster_agent/)같은 중간 캐싱 서비스를 사용하여 클러스터 전반의 병목을 피했다.

- 클러스터가 커지면서 실질적으로 클러스터의 오토스케일링이 줄어들었다.

    - 오토스케일링을 한번에 많이 하여 주기적으로 문제가 발생했다.

    - 노드를 클러스터에 추가하는 요청이 많이 발생하고, 몇백개의 노드가 한번에 추가되는 것이 API 서버의 추가적이 부하를 만들었다.

    - 몇초라도 이러한 부하를 부드럽게 하는것이 장애를 막을 수 있었다.

# Time-series metrics with Prometheus and Grafana

- 시계열 메트릭 수집을 위해 프로메테우스를, 그래프와 대시보드, 얼럿을 위해 그라파나를 사용한다.

    - [kube-prometheus](https://github.com/coreos/kube-prometheus) deployment로 시작하여 다양한 종류의 메트릭을 수집하고 좋은 시각화 대시보드를 얻었으며, 시간이 지나 많은 대시보드, 메트릭, 얼럿이 추가되었다.

- 노드를 추가할수록, 프로메테우스가 수집하는 엄청나게 많은 양의 메트릭으로 문제가 생겼다.

    - kube-prometheus는 많은 유용한 데이터를 제공하지만, 사용하지 않는 데이터도 있었고 효율적으로 수집, 저장, 쿼리하기 어려운 데이터도 이었다.

    - [Prometheus Rules](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#relabel_config)를 사용하여 수집되는 메트릭 중 일부를 버리도록 하였다.

- 프로메테우스가 더 많은 메모리를 사용하면서 메모리를 추가해도 OOM이 발생하게 되었다.

    - 크러쉬가 나면 WAL 로그를 리플레이하기 때문에 스타트업도 엄청나게 오래 걸렸다.

    - OOM의 원인은 그라파나와 프로메테우스의 상호 작용이었다.

        - 그라파나가 `/api/v1/series` API를 `{le!=""} (모든 히스토그램 메트릭을 가져옴)`로 사용하기 때문이다.

        - `/api/v1/series` API의 구현은 시간과 공간에 묶여있지 않고, 많은 결과를 포함한 쿼리를 할때 더 많은 메모리와 시간을 소모하게 된다.

            - 요청이 취소되고 커넥션이 끊겨도 메모리를 계속 사용하였다.

        - 사용 용도상 충분한 메모리가 없었고, 프로메테우스의 사망으로 이어졌다.

        - 컨텍스트에 타임아웃을 추가하였고 이 문제를 해결할 수 있었다.

    - 프로메테우스의 크러쉬가 적게 나면서, WAL 리플레이가 오래 걸리는건 여전히 문제로 남아있었다.

        - `GOMAXPROCS=24` 옵션을 적용하여 프로메테우스의 코어수를 늘리는 것이 문제를 해결했다.

# Healthchecks

- 큰 규모의 클러스터에서 당연히 이상한 노드를 감지하고 제거하는 것을 자동화했다.

## Passive Healthchecks

- 헬스체크 몇개는 정적이고, 항상 모든 노드에서 작동중이다.

    - 네트워크 접근 가능 여부, 디스크 오류 혹은 디스크 가득 참, GPU 에러

- GPU는 문제를 드러내는 방법의 종류가 많은데, 가장 일반적인건 "Uncorrectable ECC error"이다.

    - DCGM 툴은 이 에러를 쉽게 쿼리할 수 있게 하고 Xid 에러도 파악 가능하다.

    - [dcgm-exporter](https://github.com/NVIDIA/gpu-monitoring-tools#dcgm-exporter)를 통해 메트릭을 프로메테우스로 가져와 조사한다.

        - `DCGM_FI_DEV_XID_ERRORS` 플래그로 나타나 가장 최근에 발생한 에러 코드로 설정된다.

    - 추가로 [NVML Device Query API](https://docs.nvidia.com/deploy/nvml-api/group__nvmlDeviceQueries.html#group__nvmlDeviceQueries)로 GPU에 관련된 더 디테일한 정보를 얻을 수 있다.

    - 에러를 발견하면, 보통은 GPU나 시스템 리셋으로 해결이 된다.

        - 일부 케이스는 GPU를 물리적으로 교체해야하는 경우도 있다.

- 또다른 형태의 헬스체크로 클라우드의 메인터넌스 이벤트를 따라가는 것이 있다.

    - 클라우드 업체들은 현재 사용중인 VM의 점검 예정을 알려준다.

    - 하이퍼바이저 패치나 피지컬노드 교체가 이루어질 수 있어 VM이 리부팅되어야 할 수도 있다. 

- 이러한 정적 헬스체크는 모든 노드의 백그라운드에서 지속적으로 동작한다.

    - 헬스체크가 실패하면 노드를 cordon 처리하여 새로운 파드가 스케줄되지 않도록 한다.

    - 더 중대한 헬스체크 실패가 나는 경우 현재 구동중인 모든 파드를 즉각 종료시키는 것을 시도한다.

    - 결과적으로 모든 파드가 종료되거나 7일이 지나면 VM을 강제 종료한다. 

## Active GPU tests

- 모든 GPU 문제가 DCGM의 에러 코드로 나타나지 않는다.

    - 추가적인 문제를 파악하고 하드웨어와 드라이버가 예상대로 작동하는지 보장하기 위한 GPU 테스트 라이브러리를 만들었다.

    - 백그라운드에서 동작할 수 없고, 몇초에서 몇분동안 GPU를 독접적으로 동작시켜야 한다.

- 처음에는 부팅하자마자 이 테스트를 작동시켰고, 이를 preflight라고 했다.

    - 모든 노드는 preflight taint와 라벨이 적용되어 클러스터에 추가된다.

        - DaemonSet은 테스트 파드를 preflight 라벨이 붙은 모든 노드에서 작동시키도록 구성되었다.

        - 테스트 성공하면 라벨과 taint를 제거하고 노드를 사용할 수 있도록 한다.

- 테스트를 노드의 라이프타임에 크론잡으로 만들었다.

    - 모든 가용 노드에 이를 작동하도록 하며, 어떤 노드가 테스트될지는 무작위이다.

    - 부팅 시점의 결정적 테스트와 운영 중의 무작위 주기적 테스트를 조합하여, 최소한의 코디네이션으로 충분한 테스트 커버리지를 제공하고, 상관관계가 있는 장애를 방지할 수 있었다.

# Quotas & Resource Usage

- 클러스터를 스케일업했을때, 연구자들은 자신들에게 할당된 총 할당량을 확인하지 못했다.

    - 전통적인 잡 스케줄링 시스템은 경쟁하는 팀에 공평하게 워크로드를 분배하는 기능을 제공했고, 쿠버네티스는 이를 제공하지 않았다.

    - 이러한 스케줄링 시스템에서 영감을 얻어 쿠버네티스의 방식으로 몇가지 기능을 구현했다.

## Team taints

- 클러스터에 team-resource-manager 서비스를 만들었다.

    - ConfigMap을 데이터 소스로 주어진 클러스터에서 가용한 모든 리서치 팀의 노드 셀렉터, 적용할 팀 레이블, 할당량등의 데이터를 특정한다.

    - 클러스터의 현재 노드와 조화하여 태그로 적절한 개수의 노드만큼을 tainting한다.

- team-resource-manager는 또한 admission 웹훅을 갖고 있어, 각 잡이 제출되면 대응되는 제출자의 팀 멤버십에 따라 toleration을 적용한다.

- taint를 사용하여 쿠버네티스 파드 스케줄러의 유연성을 제한한다.

    - 낮은 우선순위의 아이템을 any toleration으로 배정하여 복잡한 코디네이션 없이 다른팀의 캐퍼를 빌릴 수 있도록 한다.

## CPU & GPU balloons

- 클러스터 오케스케일러를 통한 VM 기반 클러스터 동적 확장에 추가로 unhealthy 멤버를 제거하고 재추가하여 노드를 치료하는 프로세스가 있다.

    - 클러스터의 최소 크기를 0으로, 최대 크기를 최대 가용량으로 설정하여 수행한다

- 클러스터 오토스케일러는 아이들 노드를 보면 필요한 만큼만으로 스케일 다운하려 한다.

    - 여러가지 이유로 (VM 스핀업 지연 시간, pre-allocated 비용, API 서버 영향) 이상적이지 않다.

- GPU만 사용하는 호스트와 GPU 호스트에 balloon Deployment를 도입했다.

    - 이 Deployment는 낮은 우선순위를 가진 파드의 최대 사이즈를 설정한 레플리카셋이 존재한다.

    - 이 파드들은 노드의 자원을 점유하여 오토스케일러에서 아이들하지 않다고 판단하도록 한다.

        - 우선순위가 낮기 때문에 스케줄러는 실제 작업이 발생하면 파드를 비워 작동시키도록 할 수 있다.

        - DaemonSet 대신 Deployment를 사용해 idle 워크로드로 인식되지 않도록 했다.

- pod anti-affinity를 사용해 파드가 노드 간에 고르게 배포될 수 있도록 하였다.

    - 초기 쿠버네티스 버전에선 Anti-affinity가 O(N^2) 로 성능 이슈가 있었고, 1.18에서 해결되었다.

## Gang Scheduling

- 우리의 실험은 보통 하나 이상의 StatefuleSet을 포함한다.

    - 각각은 서로 다른 트레이닝 노력을 필요로 한다.

    - Optimizer에 대해 연구자는 StatefulSet의 모든 멤버가 훈련 완료 이전에 스케줄 되어야 한다.

- 쿠버네티스틑 기본적으로 하나의 StatefulSet에서 발생한 요청을 충족하는것을 우선하지 않는다.

    - 예를 들어 두개의 실험이 각각 클러스터의 100%의 자원을 요청했으면, 하나의 실험에 100%를 할당하는 것이 아닌 각각에 50%를 할당하여 실험간의 데드락으로 이어진다.

    - 커스텀 스케줄러는 엣지 케이스로 작동하여 문제가 발생했으나 1.18버전에서 도입된 코어 쿠버네티스 스케줄러에 대한 플러그인 아키텍처로 [Coschduling](https://github.com/kubernetes/enhancements/pull/1463)이란 것을 구현하여 문제를 해결했다.

# Unsolved problems

## Metrics

- 스케일이 크기 때문에 프로메테우스 빌트인 TSDB 스토리지 엔진이 느려지고, 프로메테우스 재시작에서 WAL 리플레이가 긴 시간을 소요했다.

    - 쿼리도 query processing would load too many sample 에러로 이어졌다.

- 다른 프로메테우스 호환 스토리지와 쿼리 엔진으로 마이그레이션하는 프로세스를 진행중이다.

## Pod network traffic shaping

- 클러스터를 스케일업하면서 각 파드는 전체 인터넷 대역폭 중 특정 부분을 할당받도록 계산된다.

- 사람마다 필요한 통합 인터넷 대역폭이 커졌고, 연구자들이 의도치 않게 인터넷 상의 다른 위치의 자원에 큰 제한을 넣는 경우가 발생한다.

    - 데이터셋이나 소프트웨어 패키지 다운로드 할때 그렇다.

# Conclusion

- 쿠버네티스는 우리 연구의 요구사항에 맞출 수 있는 유연한 플랫폼이다.

    - 우리가 필요로 하는 가장 큰 워크로드를 소화할만큼 스케일링될 수 있다.

- 발전이 필요한 부분도 있으며 오픈AI 슈퍼컴퓨팅 팀은 쿠버네티스를 얼마나 스케일할 수 있을지를 계속 탐색할 것이다.

