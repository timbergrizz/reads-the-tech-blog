---
출처: '[Scaling Kubernetes to 2,500 nodes | OpenAI](https://openai.com/index/scaling-kubernetes-to-2500-nodes/)'
---

- 딥러닝 연구로 쿠버네티스를 2년간 사용했다.

    - 가장 큰 워크로드는 베어 클라우드VM을 사용한다.

    - 빠른 반복사이클, 합리적인 스케일 기능, 보일러 플레이트의 부재로 대부분의 실험과 적합하다.

- 클러스터 여러 개를 동작시키고 있으며 (일부는 베어메탈, 일부는 클라우드), 가장 큰 클러스터는 2500개의 노드가 있다.

    - Azure에서 D15v2랑 NC24 VM으로 작동시킨다

- 이정도 스케일이 된까 많은 시스템 컴포넌트가 장애를 만든다.

    - etcd, 마스터 노드, 도커의 이미지 풀링, 네트워크, KubeDNS, ARP 캐시까지

- 이슈들을 공유해보려 한다.

# etcd

- 클러스터에 노드가 500개 넘어갔을때, kubectl에서 타임 아웃 난다는 리포트가 나오기 시작했다.

    - 마스터 노드를 추가하면 일시적으로 해결되는 것 처럼 보였다.

    - 근데 레플리카 10개를 넘어가니 원인을 해결하는게 아니라 증상을 해결하고 있었다는 것을 깨달았다.

- 마스터노드의 state를 저장하는 중앙 저장소인 etcd 클러스터를 의심하기 시작했다.

    - 데이터독을 살펴보니 DS15v2 VM의 디스크에서 쓰기 레이턴시에 스파이크가 발생했다.

        - 다른 머신은 P30 SSD로 5000IOPS를 처리할 수 있었다.

- [fio](https://github.com/axboe/fio) 벤치마크 퍼포먼스를 확인했을 때, etcd가 IOPS의 10%밖에 사용 못하는 것을 발견했다.

    - etcd가 sequential IO를 사용해서 레이턴시가 커졌다.

    - etcd의 디렉토리를 각 인스턴스와 직결된 노드의 로컬 temp 디스크로 옮기니 레이턴시가 감소하고 etcd 문제가 해결되었다.

- 노드 1000개까지는 문제가 없었고, 그 지점에서 다시 레이턴시가 높아졌다.

    - kube-apiservers가 etcd부터 500MB/s로 데이터를 읽었다.

    - 프로메테우스를 세팅하고, `--audit-log-path`와 `--audit-log-maxbackup` 플래그로 apiserver에 더 많은 로깅을 하도록 했다.

        - 느린 쿼리와 쓸데없이 많은 콜을 찾아낼 수 있었다.

- 근본 원인은 Fluentd와 Datadog의 모니터링 프로세스가 클러스터의 모든 노드의 apiservers에 쿼리를 날리는 것이었다.

    - 각 노드에서 apiserver를 풀링하는 대신 중앙 위치를 풀링하도록 덜 공격적으로 바꾸어 문제가 해결되었다.

- 쿠버네티스 이벤트를 분리된 etcd에 저장하면 이벤트 생성에서 스파이크가 발생해도 메인 인스턴스에 영향을 끼치지 않는다.

    - `--etcd-servers-overrides` 옵션 사용하면 된다.

- 또 다른 오류: 노드 1000개 되니까 etcd 스토리지의 하드리밋 걸려서 쓰기 멈추고, 연쇄적인 오류로 이어졌다.

    - 모든 쿠버네티스 노드의 헬스체크가 죽고, 오토스케일러가 모든 워커를 죽였다.

- etcd 사이즈를 늘리고 `--quota-backend-bytes` 플래그를 줬으며, 오토스케일러에 sanity check를 추가하여 50% 이상의 클러스터를 죽여야 하면 아무것도 하지 않도록 했다.

# Kube masters

- kube-apiserver, kube-controller-manager, kube-scheduler 프로세스를 같은 머신에 위치시킨다.

    - 고가용성을 위해, 최소 2개의 마스터를 갖도록 하고, `--apiserver-count` 플래그로 apiservers 개수를 설정한다.

- 쿠버네티스를 주로 배치 스케줄링 시스템으로 사용하고, 오토스케일러에 동적 스케일링을 의존한다.

    - 아이들 노드를 없애 비용 절약에 도움이 되고, 낮은 레이턴시와 빠른 반복을 제공한다.

- 기본 쿠베 스케줄러 정책은 모든 노드에 최대한 고르게 로드를 분배하는 것

    - 오픈AI는 반대로 작동하여 안쓰는 노드가 종료될 수 있으며, 큰 파드가 빠르게 스케줄링 될 수 있도록 해야 했다.

- 다음과 같이 정책을 구성했다.

    ```json
    {
      "kind" : "Policy",
      "apiVersion" : "v1",
      "predicates" : [
        {"name" : "GeneralPredicates"},
        {"name" : "MatchInterPodAffinity"},
        {"name" : "NoDiskConflict"},
        {"name" : "NoVolumeZoneConflict"},
        {"name" : "PodToleratesNodeTaints"}
      ],
      "priorities" : [
        {"name" : "MostRequestedPriority", "weight" : 1},
        {"name" : "InterPodAffinityPriority", "weight" : 2}
      ]
    }
    ```

- KubeDNS를 서비스 디스커버리를 위해 많이 사용하는데, 새로운 정책 적용 후 신뢰도 문제가 발생하기 시작했다.

    - 특정 파드의 KubeDNS에서만 실패가 발생하는 것이 확인되었다.

    - 새로운 스케줄링 정책으로 인해 일부 머신은 10개 이상의 KubeDNS를 실행하고, 핫스팟이 생겼으며, Azure VM에서 외부 도메인 조회에 허용되는 200QPS를 초과했다.

- Anti-affinity 정책을 추가하여 해결했다.

    ```yaml
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
        labelSelector:
          matchExpressions:
            - key: k8s-app
              operator: In
              values:
              - kube-dns
        topologyKey: kubernetes.io/hostname
    ```

# Docker image pulls

- Dota 프로젝트는 쿠버네티스에서 시작되었다.

    - 스케일이 커지니 스타트업에 펜딩 상태로 엄청 오래 걸리는 현상이 발생했다.

        - 게임 이미지가 17GB정도였고, 새로운 노드에 시작하는데 30분 이상을 소모했다.

- 파고 들어가니 kubelet의 `--serialize-image-pull`가 true로 설정되어 도타 이미지를 가져올 때 다른 이미지를 모두 블락하고 있었다.

- 이걸 false로 바꾸기 위해서 Docker의 파일시스템을 AUFS가 아닌 overlay2로 변경해야 한다.

    - 스피트업을 더 늘리기 위해, 도커 루트를 etcd에 했던 것 처럼 인스턴스에 부착된 ssd로 이동시켰다.

- 풀 속도를 최적화한 이후에도 pod가 시작을 실패하는 경우가 발생했다.

    - `rpc error: code = 2 desc = net/http: request canceled` 에러가 발생했다.

    - kubelet과 도커의 로그에서 진척이 없어 이미지 풀이 취소된 경우도 발생했다.

- 큰 이미지의 풀링에 너무 오래걸리는지, 혹은 이미지 풀링에 몇번이나 긴 백로그가 생기는지 추적했다.

    - 이를 위해 kubelet의 `--image-pull-progress-deadline`을 30분으로 설정했고, 도커 데몬의 `max-concurrent-downloads`를 10으로 설정했다. 

- 도커 풀 이슈는 구글 GCR때문 이었다.

    - 기본적으로 kubelet은 어떤 컨테이너를 시작할때도 특수한 이미지를 gcr.id에서 가져온다.

        - `--pod-infra-container-image` 플래그를 사용한다.

    - pull이 어떤 이유로든 실패하면 노드는 어떤 컨테이너도 실행할 수 없다.

    - 노드에서 gcr.io로 갈때 NAT로 요청이 가는데, 이 과정에서 IP에 대한 quota limit이 걸렸다.

- 머신 이미지에 사용하는 도커 이미지를 쿠버네티스 워커에 프리로드하여 문제를 해결했다.

    - `docker image save -o /opt/preloaded_docker_images.tar` and `docker image load -i /opt/preloaded_docker_images.tar`

    - 성능을 개선하기 위해 도타같은 오픈AI 내부 이미지에 대해 동일한 작업을 수행했다.

# Networking

- 실험이 커질수록, 인프라는 네트워크에 크게 의존하는 복잡한 분산 시스템이 되었다.

    - 처음 분산 실험을 돌렸을 때 네트워크가 잘 설정되어 있지 않다는 사실을 바로 알 수 있었다.

    - 머신간 직결은 10-15Gbit/s의 Throughput을 가졌지만 Flannel을 이용한 pod끼리의 통신은 2Gbit/s 가 최대였다.

    - [CNI 퍼블릭 벤치마크](https://machinezone.github.io/research/networking-solutions-for-kubernetes/) 를 확인했을 때 비슷한 수치를 보였다.

        - 설정 문제가 아니라, 환경 자체에 내제적인 문제였던 것이다.

    - 유저는 `hostNetwork: true`와 `dnsPolicy: ClusterFirstWithHostNet` 2개의 설정을 통해 파드에 Flannel을 비활성화 할 수 있었다.

# ARP cache

- DNS 튜닝에도 불구하고 DNS resolution 과정에서 간헐적인 이슈가 발생했다.

- 레디스 서버에 `nc -v`를 날렸을 때 커넥션까지 30초가 걸린 케이스가 제보되었다.

    - 커널의 ARP 스택을 추적했다.

    - 레디스 파드를 최초 조사했을 땐 네트워크에 심각한 문제가 있는 것으로 보였다.

        - 어떤 포트로도 통신이 hang이 걸렸고, 로컬 `dnsmsq` 데몬을 사용했을 때 어떤 DNS name으로도 resolve되지 않았다.

- `dmesg` 로그는 neighbor table overflow 에러를 보였고, ARP cache 공간이 없는 것을 알았다.

    - ARP는 네트워크 주소를 물리 주소로 매핑할 때 사용한다.

    - `/etc/sysctl.conf`에 설정 추가하면 바로 해결할 수 있다.

        ```text
        net.ipv4.neigh.default.gc_thresh1 = 80000
        net.ipv4.neigh.default.gc_thresh2 = 90000
        net.ipv4.neigh.default.gc_thresh3 = 100000
        ```

- HPC 클러스터에서 이런 세팅을 튜닝하는것은 일반적이다.

    - 쿠버네티스가 모든 파드가 IP를 가지면 그만큼 ARP 캐시를 사용하기 때문에 특히 관련이 있다.



- 쿠버네티스 3개월째 장애 없이 운영중이고, 더 스케일을 키울 예정이다.

