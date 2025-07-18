- 출처
	- [Amazon Bedrock과 LangGraph로 Multi Agent 시스템 구현하기](https://aws.amazon.com/ko/blogs/tech/build-multi-agent-systems-with-langgraph-and-amazon-bedrock/)
	- [Multi Agent LLM](https://x2bee.tistory.com/412)
	- [LLM Multi Agent: Customer Service를 기깔나게 자동화하는 방법](https://disquiet.io/@sunnyyy/makerlog/llm-multi-agent-customer-service%EB%A5%BC-%EA%B8%B0%EA%B9%94%EB%82%98%EA%B2%8C-%EC%9E%90%EB%8F%99%ED%99%94%ED%95%98%EB%8A%94-%EB%B0%A9%EB%B2%95)
- 기존의 모놀리식 구조의 에이전트는 모든 컨택스트를 하나의 에이전트가 핸들링함
	- 간단한 작업에서는 문제가 없지만, 컨텍스트가 길어지면 에이전트의 정확도가 떨어짐
	- 또한, 여러개의 기능을 선택하여 사용하는 경우 (예: A->B, A->B->C가 존재), 하나의 에이전트는 모두 다 구현을 해주어야 함.
	- 복잡성이 증가하고 유지보수가 어려워짐
- 하나의 작업에 대한 Divide-and-Conquer 처리 전략
	- 기능 단위로 에이전트를 나누고, 작업을 기능 단위로 나누어 에이전트별로 작은 단위를 처리하도록 구성한다.
	- 메인 에이전트가 작업을 분할하고 의사 결정을 수행하는 오케스트레이팅 역할을 하게 된다.
- 멀티 에이전트에서는 각 에이전트가 전체 목표에 기여하며 다른 에이전트와의 조화를 유지하는 조정 메커니즘 이 필요하다.
	- 에이전트간 의존성 관리, 자원 할당 및 동기화에 과제가 필요하다.
	- 이 과정에서 성능을 최적화하며, 시스템 전체의 일관성을 유지할 수 있어야 한다.
	- 단일 에이전트는 단기 대화 메모리, 장기 기록 저장, RAG로 구성된 3단 구조를 사용한다.
	- 멀티 에이전트는 에이전트간 기록 동기화를 추가로 필요로 하며 컨택스트 데이터 관리, 상호 작용 추적 및 에이전트 간 기록 동기화를 필요로 한다.
		- 실시간 상호 작용, 컨택스트 동기화, 효율적인 데이터 검색을 처리해야 한다.
	- 에이전트 프레임워크는 자율 에이전트 조정, 통신 및 자원 관리, 워크플로우 오케스트레이션을 위안 인프라를 제공하기 때문에 멀티 에이전트 시스템에 필수적이다.
- LangGraph
	- LangGraph는 멀티 에이전트 오케스트레이션을 위해 상태머신과 방향성 그래프를 구현한다.
	- 에이전트 워크플로우를 그래프로 모델링하고, 세가지 핵심 구성요소를 통해 에이전트 동작을 정의한다.
		- 상태(State) : 모든 에이전트가 공유하는 데이터 구조로, 멀티 에이전트 시스템의 공유 메모리
		- 노드(Nodes) : 각각의 전문 에이전트 또는 도구의 로직을 캡슐화하는 파이썬 함수.
		- 엣지(Edges) : 현재 상태를 기반으로 다음에 실행할 노드를 결정하는 연결선. 메인 에이전트의 라우팅 로직을 구현
	- LangGraph는 persistence layer를 구현하여 대부분의 에이전트 아키텍처에 공통적인 기능이 된다.
		- 메모리
			- 상태를 지속적으로 저장하여 대화 기록 및 작업 흐름을 유지
			- 사용자 상호작용 내부 및 상호작용 간의 대화 기억과 기타 업데이트를 지원
		- Human-in-the-loop
			- 상태가 체크포인트되어, 실행을 중단 후 재개할 수 있음.
				- 실행을 특정 지점에서 중단(체크포인트)하고, 사람의 검증이나 수정을 거치도록 함
			- 인간 입력을 통해 주요 단계에서 결정, 검증 및 수정이 가능함.
- 고려사항
	- 멀티 에이전트 아키텍처는 에이전트 조정, 상태 관리, 통신, 출력 통합, 가드레일, 처리 컨택스트 유지, 오류 처리 및 오케스트레이션을 고려해야 함.
	- 그래프 기반 아키텍처는 선형 파이프라인보다 명확한 시스템 시각화, 비선형 통신 패턴으로 복잡한 워크플로우를 수행할 수 있도록 함.
		- 동적 경로와 적응형 통신으로 동시 Agent 상호작용이 있는 대규모 배포에 이상적
		- 병렬 처리 및 리소스 할당에 뛰어남
		- 정교한 설정이 필요하며, 더 높은 컴퓨팅 리소스를 요구할 수 있음.
		- 시스템 토폴로지의 신중한 계획, 모니터링 및 실패한 상호작용에 대한 대체 매커니즘이 필요
	- 조직에서 구현시 회사의 기존 인공지능 운영 및 거버넌스와 일치시킨다.
		- 배포전에 AI 안전 프로토콜, 데이터 처리 정책 및 모델 배포 지침과의 일치 여부 확인
- 멀티 에이전트 환경에서 개별 에이전트의 프롬프트
	- 페르소나 (ROLE), 명확한 지시, 입 출력 형태, 가이드라인이 담겨야 함
	- 프롬프트 예시
	  id:: 686dd408-ad1a-45d3-b4be-072c7c190d7a
		- ```
		  <ROLE>
		  You are a smart agent with an ability to use tools. 
		  You will be given a question and you will use the tools to answer the question.
		  Pick the most relevant tool to answer the question. 
		  If you are failed to answer the question, try different tools to get context.
		  Your answer should be very polite and professional.
		  </ROLE>
		  
		  ----
		  
		  <INSTRUCTIONS>
		  Step 1: Analyze the question
		  - Analyze user's question and final goal.
		  - If the user's question is consist of multiple sub-questions, split them into smaller sub-questions.
		  
		  Step 2: Pick the most relevant tool
		  - Pick the most relevant tool to answer the question.
		  - If you are failed to answer the question, try different tools to get context.
		  
		  Step 3: Answer the question
		  - Answer the question in the same language as the question.
		  - Your answer should be very polite and professional.
		  
		  Step 4: Provide the source of the answer(if applicable)
		  - If you've used the tool, provide the source of the answer.
		  - Valid sources are either a website(URL) or a document(PDF, etc).
		  
		  Guidelines:
		  - If you've used the tool, your answer should be based on the tool's output(tool's output is more important than your own knowledge).
		  - If you've used the tool, and the source is valid URL, provide the source(URL) of the answer.
		  - Skip providing the source if the source is not URL.
		  - Answer in the same language as the question.
		  - Answer should be concise and to the point.
		  - Avoid response your output with any other information than the answer and the source.  
		  </INSTRUCTIONS>
		  
		  ----
		  
		  <OUTPUT_FORMAT>
		  (concise answer to the question)
		  
		  **Source**(if applicable)
		  - (source1: valid URL)
		  - (source2: valid URL)
		  - ...
		  </OUTPUT_FORMAT>
		  
		  ```
-