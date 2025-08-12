---
출처: '[How we run GPT OSS 120B at 500+ tokens per second on NVIDIA GPUs](https://www.baseten.co/blog/sota-performance-for-gpt-oss-120b-on-nvidia-gpus/)'
---

- 데이 제로모델 최적화는 실험, 버그 픽스, 직관과 경험 기반의 벤치마킹의 조합으로 구성된다.

    - Baseten Inference Stack에 대해 알아보자.

- 새로운 모델의 최적화는 큰 엔지니어링 챌린지를 낳는다.

    - 유연한 인퍼런스 스택과 모델 퍼포먼스 엔지니어링 팀 집단의 전문성으로 한시간만에 크게 모델의 성능을 향상시킬 수 있었다.

        - 이 글을 작성하는 시간동안 다운타임 없이 처리량이 100t/s 증가했다.

- 인퍼런스 프레임워크간 테스트와 벤치마크 : TensorRT-LLM, vLLM, SGLang

- Hopper와 Blackwell GPU간 호환성 보장하기

- NVIDIA Dynamo를 포함해 인퍼런스 스택의 중요 조각들을 통합하기

- KV-cache 인식 라우팅과 Eagle을 통한 추론적 디코딩 등의 레이어링 (가장 좋아하는 성능 최적화)

# Step by step

## Step 1: Running first Instance

- 처음은 그냥 인퍼런스하여 가능한 베이스라인을 만든다.

    - 인퍼런스를 돌리려면 인퍼런스 프레임워크, 하드웨어 아키텍처, 모델 서버 레벨의 지원이 필요하다.

- GPU에서 영감을 얻어, 이러한 노력을 엔지니어간 병렬화했다.

    - 엔지니어 한명은 vLLM, 한명은 SGLang, 한명은 TensorRT-LLM 이런식으로 일한다.

    - 빠르게 TensorRT-LLM이 작동했고, 다행히 일반적으로 가장 성능이 좋은 인퍼런스 프레임워크였다.

- 모델을 Hopper와 Blackwell 아키텍처 모두에서 작동하는것이 중요했다.

    - H100이 흔하게 있었고 B200의 속도도 퍼블릭 모델 API에서 사용할 수 있어야 했다.

- Baseten 인퍼런스 런타임의 가장 중요한 요소는 유연성이다.

    - 새로운 모델을 훌륭한 아키텍처와 같이 작동시킬때 유용하다.

    - 네비게이팅과 업데이트(필요시)에서 전반적인 스택에서 빠르게 툴을 교체할 수 있다.

## Step 2: Fixing compatibility bugs

- 새로운 모델 아키텍처가 출시되면, 기존 프레임워크로 이걸 배포했을 때 당연히 버그와 이슈가 있다.

    - GPT-OSS 출시에는 'Harmony'라는 새로운 응답 형식을 포함한 신기술이 추가되었다.

- 엔지니어링의 일 대부분은 버그 픽스하고 속도와 정확도 개선을 위해 모델을 테스트하는것.

    - 가능하면 오픈소스로 다시 기여하려고 한다.

## Step 3: Optimizing model configuration

- OpenAI는 gpt-oss를 단일 H100에서 돌릴 수 있다고 한다.

    - 배포 최적화해서 병렬처리하면 4 / 8 GPU에서 동시에 돌려서 성능과 처리량을 개선할 수 있다.

- 두가지 병렬 처리 접근 방법이 있다. Tensor / Expert Parallelism

    - Tensor 병렬화가 더 나은 지연시간을 보였고, Expert 병렬화는 시스템 처리량이 더 나았다.

    - 레이턴시를 우선하기 때문에 Tensor를 채택

- 추가로 TensorRT-LLM MoE 백엔드를 채택했다.

    - Blackwell에서만 지원하고 Hopper에서는 지원 안된다.

    - 개선된 쿠다 커널을 제공하여 기존 트리톤 백엔드보다 빠른 성능을 보인다.

- 설정들을 패키징해 120B, 20B 모델을 Hopper GPU들에 맞추어 모델 라이브러리에 추가했고, 모델API에는 블랙웰 사용했다.

## Next steps in performance optimization

- 이러한 첫 퍼포먼스들은 SOTA 지연시간과 처리량을 개선한다.

    - 120B모델은 성능을 개선할 여지가 남아있다.

    - 성능 개선 지점중 하나는 speculative 디코딩을 추가하는것

        - 미래의 토큰에 대해 더 작은 draft 모델을 사용하고, 타겟 모델로 검증하는 것.

        - Eagle3 좋지만, 인퍼런스 스택은 10개 이상의 모델을 지원하여 모델과 워크로드에 적절한 알고리즘을 선택하도록 할 수 있다.

