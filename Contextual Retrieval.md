- AI모델에 백그라운드 지식을 필요로 하는 경우가 있다.
	- 이를 위해서 RAG를 구성했는데, 문제는 기존의 RAG는 인코딩 과정에서 컨텍스트 정보가 포함되지 않아 성능이 떨어진다는 것
	- 이러한 문제를 해결하여 백그라운드 성능을 향상시키기 위해 Context Retrieval이 도입됨
- 토큰이 20만개보다 작으면, 그냥 전체 지식 베이스를 모델에 때려박아도 괜찮다.
	- 지식 베이스는 커지고, 그렇기에 스케일 가능한 솔루션이 필요한거다.
- 컨택스트 윈도우에 들어가지 않는 지식 베이스에서, RAG가 일반적인 솔루션이고, 다음과 같이 동작한다.
	- 지식 베이스를 몇백 토큰 정도의 작은 청크로 쪼갠다
	- 임베딩 모델을 사용하여 벡터 임베딩으로 변환한다.
	- 벡터 데이터베이스에 저장하여 이를 탐색할 수 있도록 한다.
- 임베딩 모델은 시멘틱 관계는 잘 잡는데, 실제로 매치되는 글자는 놓칠 수 있다.
	- 임베딩 모델과 BM25를 같이 사용하는 이유
- 이러한 일련의 과정에서, 컨택스트가 무너진 RAG시스템이 구성된다.
	- 작은 청크로 쪼개서 탐색하기 때문에 탐색에서 충분한 컨택스트를 담기 어려워진다.
	- Context Retrieval은 청크에 근처에 특정된 추가적인 컨텍스트를 부여하여 임베딩을 만들고 BM25인덱스를 생성하여 이를 해결한다.
	- 원래도 이런 작업 자체가 비용이 얼마 안들지만, 프롬프트 캐싱등의 방법을 이용하여 더 효율적으로 처리할 수 있다.
- 도입시 다음과 같은 점을 고려해야 한다
	- 청크를 어떤 단위로 쪼갤지 바운더리에 따라 retrieval 성능이 달라진다.
	- 임베딩 모델에 따라서도 성능 향상 폭이 달라지는데, gemini와 voyage가 효과적이었다.
	- 프롬프트를 커스텀하여 컨택스트를 더 잘 담도록 하면 좋다.
	- 많은 정보는 모델을 혼란스럽게 하므로 컨택스트 윈도우에 몇개의 관련된 청크를 담을건지도 고려
- Reranking같은 방법을 도입하여 추가적인 성능 향상도 이룰 수 있다.
	- 가장 많이 관련된 청크들만 모델로 전달되도록 하는 기법이다.
	- 리랭킹 모델로 Top-K개의 청크만 추출하는 방법이다.
- 다음과 같이 정리할 수 있다.
	- 임베딩과 BM25를 같이 쓰는것이 임베딩만 쓰는 것보다 좋다.
	- Gemini와 Voyage가 임베딩 성능이 가장 좋았다.
	- 청크는 top-20이 top-10, top-5보다 좋았다.
	- 청크에 컨텍스트를 추가하는것이 retrieval 성능 향상에 큰 도움이 되었다.
	- 리랭킹은 들어가는게 좋다.
	- 이 모든 성능 향상을 동시에 사용할 수 있다.