공부 진행용DeepResearch_Part 4_(Gemini)

# **대규모 언어 모델(LLM) 환경에서의 임베딩 기반 검색 메커니즘과 악성 페이로드 활성화 평가 연구**

## **요약 (Executive Summary)**

대규모 언어 모델(LLM) 기반의 검색 증강 생성(Retrieval-Augmented Generation, RAG) 및 자율 에이전트(Autonomous Agent) 시스템이 발전함에 따라, 시스템의 장기 메모리(Long-term Memory)와 외부 지식 베이스의 무결성 확보가 핵심적인 보안 과제로 대두되었다. 본 보고서는 학부 2학년 수준의 연구자가 메모리 포이즈닝(Memory Poisoning) 및 악성 페이로드(Malicious Payload) 생존성 연구를 수행하기 위해 반드시 이해해야 하는 '정보 검색(Information Retrieval, IR)'의 이론적 기반과 평가 방법론을 심층적으로 다룬다.

과거의 정보가 현재의 시스템 출력에 영향을 미치기 위해서는 단순한 '생존(Survival)'을 넘어 '검색(Retrieval)'이라는 필수적인 관문을 통과해야 한다. 검색 시스템은 데이터베이스에 잠들어 있는 지시어 형태의 악성 페이로드가 LLM의 컨텍스트 윈도우(Context Window)로 진입하여 공격을 활성화(Activation)하도록 만드는 트리거(Trigger) 역할을 수행한다. 본 문서는 정보 검색의 기초 개념부터 임베딩(Embedding) 기반 밀집 검색(Dense Retrieval)의 수학적 원리, RAG 파이프라인에서의 작동 방식, 그리고 Hit@k, Recall@k 등 평가 지표의 의미를 상세히 분석한다. 또한 최신 보안 연구(2024–2026)에서 밝혀진 검색 실패 모드(Failure Modes)와 LLM 소비 행동을 반영한 새로운 평가 과제를 논의함으로써, 성공적이고 타당성 있는 보안 연구 실험을 설계하기 위한 구조적 통찰을 제공한다.

## ---

**1\. 정보 검색(IR)의 개념과 중요성 (What is Information Retrieval and Why It Matters)**

### **정보 검색 시스템의 정의**

정보 검색(Information Retrieval, IR)은 대규모의 비정형(Unstructured) 또는 반정형 데이터(텍스트 문서, 웹 페이지, 대화 기록 등) 집합으로부터 사용자의 정보 요구(Information Need)와 관련된 리소스를 찾아내는 과정 및 소프트웨어 시스템을 의미한다.1 일상생활에서 가장 널리 사용되는 정보 검색 시스템의 예는 구글(Google)과 같은 웹 검색 엔진이나 추천 시스템이다.1

### **데이터베이스 검색과의 차이: 정확 일치(Exact Match) vs 관련성(Relevance)**

정보 검색 시스템을 명확히 이해하기 위해서는 이를 전통적인 관계형 데이터베이스(RDBMS)의 데이터 검색(Data Retrieval)과 비교하는 것이 필수적이다.

* **데이터 검색 (Data Retrieval):** 결정론적(Deterministic) 접근 방식을 취한다. SQL 쿼리를 통해 특정 조건(예: SELECT \* FROM accounts WHERE balance \> 1000)을 명시하면, 시스템은 조건에 '정확히 일치(Exact Match)'하는 결과만을 반환한다. 결과는 이분법적이며, 부분적인 일치나 모호성을 허용하지 않는다.3 오류가 발생할 경우 쿼리 자체의 문법적 결함이나 데이터의 부재로 명확히 귀결된다.5  
* **정보 검색 (Information Retrieval):** 확률론적(Probabilistic)이고 의미론적인 접근 방식을 취한다. 자연어로 구성된 사용자의 쿼리("최신 보안 취약점 사례")는 본질적으로 모호성을 지니며, 시스템은 쿼리와 문서 간의 '관련성(Relevance)'을 추정한다.3 문서 내에 쿼리의 정확한 단어가 존재하지 않더라도 동의어나 문맥을 통해 관련 문서로 판별할 수 있다.

### **왜 결과를 '순위(Ranking)'로 매기는가?**

정보 검색에서 완벽하게 일치하는 단 하나의 정답을 찾는 것은 불가능에 가깝다. 수백만 개의 문서가 쿼리와 다양한 수준의 관련성을 맺고 있기 때문이다. 따라서 시스템은 쿼리와 각 문서 간의 관련성 점수(Relevance Score)를 계산하고, 이 점수를 바탕으로 결과를 내림차순으로 정렬(Sorting)하는 '순위화(Ranking)' 패러다임을 따른다.6

| 비교 항목 | 데이터 검색 (Data Retrieval) | 정보 검색 (Information Retrieval) |
| :---- | :---- | :---- |
| **목표** | 조건에 맞는 특정 데이터의 완벽한 추출 | 사용자의 의도에 부합하는 관련 정보의 발견 |
| **매칭 방식** | 정확 일치 (Exact Match) | 부분 일치 및 관련성 (Partial Match, Relevance) |
| **데이터 형태** | 정형 데이터 (Structured Data) | 비정형 텍스트 (Unstructured Data) |
| **결과 형태** | 정렬되지 않은 집합 (Unordered Set) | 관련성에 따른 순위 목록 (Ranked List) |

검색 엔진(Search Engine)에 비유하자면, 사용자가 검색창에 키워드를 입력하면 검색 엔진은 인덱싱된 수십억 개의 웹페이지 중 관련 있는 페이지를 식별(Retrieval)하고, 가장 유용한 정보를 포함할 확률이 높은 페이지부터 상단에 노출(Ranking)시킨다.8 이는 LLM이 활용할 컨텍스트를 선별할 때, 가장 중요한 소수의 정보만을 제공하기 위한 필수적인 파이프라인이다.

## ---

**2\. 키워드 검색에서 임베딩 검색으로의 전환 (From Keyword Search to Embedding Search)**

정보 검색의 발전 과정은 텍스트를 매칭하는 패러다임의 변화로 요약된다. 현재 RAG 시스템 및 LLM 메모리 구조에서는 크게 두 가지 방식이 활용된다.

### **A. 어휘 검색 (Lexical Retrieval: BM25 및 키워드 매칭)**

전통적인 정보 검색은 단어의 표면적인 형태와 빈도(Frequency)에 의존하는 희소 벡터(Sparse Vector) 모델을 사용했다. 가장 대표적인 알고리즘은 \*\*BM25 (Best Matching 25)\*\*이다.10

* **작동 원리:** 문서 내에 쿼리 토큰이 얼마나 자주 등장하는지(Term Frequency, TF)를 측정하되, 'the', 'is'와 같이 모든 문서에 흔하게 등장하는 단어는 가중치를 대폭 낮춘다(Inverse Document Frequency, IDF).10 또한 극단적으로 긴 문서가 유리해지는 것을 막기 위해 문서 길이를 정규화한다.  
* **강점:** 티켓 ID, 고유 에러 코드("LDAP\_AUTH\_001"), 특정 인물의 이름 등 정확한 식별자(Identifiers)를 찾는 데 압도적인 성능을 발휘한다. 작동 원리가 투명하여 디버깅이 용이하다.11  
* **약점:** 어휘적 불일치(Vocabulary Mismatch)에 매우 취약하다. 쿼리에 "변호사"라는 단어가 쓰였을 때, 문서에 "법률가"라는 단어만 존재한다면 의미가 같음에도 불구하고 시스템은 이를 전혀 관련 없는 문서로 판단한다.12

### **B. 밀집 검색 (Dense Retrieval / Embedding-based Search)**

신경망(Neural Network)의 발전에 따라 등장한 밀집 검색은 텍스트의 표면적 단어가 아닌 '의미론적 유사성(Semantic Similarity)'을 기반으로 정보를 검색한다.12

* **작동 원리:** 언어 모델(Transformer 등)을 사용하여 텍스트를 수백\~수천 차원의 연속적인 실수로 이루어진 밀집 벡터(Dense Vector)로 변환한다. 의미가 유사한 텍스트는 이 다차원 벡터 공간 내에서 서로 가까운 위치에 군집(Cluster)을 형성한다.14  
* **강점:** 동의어, 의역(Paraphrasing), 모호하게 표현된 대화형 쿼리를 완벽하게 처리한다. "로그인이 안 됩니다"와 "비밀번호 분실"을 의미론적으로 동일한 공간에 매핑하여 관련 문서를 찾아낸다.11  
* **약점 및 한계:** 연산 비용이 높으며, 의미론적 모호성(Semantic Ambiguity)으로 인해 검색 품질이 저하될 수 있다. 질문이 너무 구체적인 기술적 용어를 포함할 경우, 임베딩 모델은 해당 용어를 노이즈로 처리하고 주변의 흔한 단어(예: "연결 오류")의 의미에만 집중하여, 관련은 있어 보이지만 실제로는 전혀 유용하지 않은(Plaucible but wrong) 문서를 최상위로 반환하는 '의미적 표류(Semantic Drift)' 현상이 발생한다.11

### **LLM 시스템에서 임베딩 검색이 주류가 된 이유**

LLM을 활용한 애플리케이션의 사용자는 정형화된 키워드가 아니라 일상적인 자연어 형태로 질문을 던진다. 임베딩 기반 검색은 이러한 자연어의 모호성과 문맥을 포착하여 인간의 의도(Intent)에 가장 잘 부합하는 지식을 외부에서 끌어올 수 있기 때문에 현대 RAG 시스템의 표준으로 자리 잡았다.11

## ---

**3\. 임베딩의 작동 원리: 직관적 및 기술적 이해 (How Embeddings Work)**

임베딩은 언어의 의미를 수학의 언어로 번역하는 과정이다. 악성 페이로드가 어떻게 검색망을 피하거나 역으로 이용하는지 이해하려면 이 변환 과정을 기술적으로 숙지해야 한다.

### **직관적 이해: 의미 공간의 좌표 체계**

임베딩을 거대한 다차원 지도(Map)로 상상해 볼 수 있다.14 가장 단순하게 '샌드위치와 얼마나 비슷한가(Sandwichness)'와 '디저트와 얼마나 비슷한가(Dessertness)'라는 두 가지 축(Dimension)만 존재하는 공간이 있다고 가정하자.15 이 공간에서 "핫도그"와 "햄버거"는 샌드위치 축의 값이 높아 서로 매우 가까운 위치(좌표)에 놓인다. 반면 "사과 파이"는 디저트 축의 값이 높아 "핫도그"와는 멀리 떨어진 공간에 위치한다.15 임베딩 모델은 단어나 문장을 읽고 그 의미를 파악하여 이 지도 상의 특정한 점(Point)으로 텍스트를 던져 넣는다(Projection). 따라서 "의미가 비슷하다(Similar meaning)"는 것은 "벡터 공간에서 두 점 사이의 거리가 가깝다(Close vectors)"는 것을 의미한다.15

### **기술적 이해: 고차원 공간과 코사인 유사도**

실제 최신 텍스트 임베딩 모델(예: OpenAI의 임베딩이나 sentence-transformers)은 2차원이 아니라 384차원, 768차원, 혹은 1536차원에 달하는 초고차원(High-dimensional space) 벡터를 생성한다.17

* **텍스트 ![][image1] 벡터 변환:** 텍스트가 입력되면, 모델의 어텐션 메커니즘(Attention Mechanism)이 각 토큰 간의 관계를 분석하여 최종적으로 해당 텍스트의 전체 의미를 담은 하나의 밀집 벡터 $v \= \[v\_1, v\_2,..., v\_n\]$를 생성한다.18  
* **코사인 유사도 (Cosine Similarity):** 고차원 공간에서 두 텍스트의 유사도를 측정할 때 가장 널리 쓰이는 수학적 도구는 코사인 유사도다.19 유클리드 거리(Euclidean Distance)가 두 점 사이의 물리적 직선거리를 측정한다면, 코사인 유사도는 두 벡터가 향하는 \*\*'방향(Angle)'\*\*의 일치성을 측정한다.19  
  ![][image2]  
  코사인 유사도 값은 \-1에서 1 사이의 범위를 가진다. 두 벡터의 각도가 0도(방향이 완벽히 일치)이면 코사인 값은 1이 되며, 각도가 90도(서로 무관)이면 0, 180도(정반대의 의미)이면 \-1이 된다.19 텍스트의 길이(벡터의 크기)가 달라도 의미하는 바의 방향성만 같다면 높은 유사도를 가지므로 검색에 최적화되어 있다.16

### **임베딩의 치명적 한계점**

보안 관점에서 임베딩의 메커니즘은 완벽하지 않으며 다음의 약점을 노출한다.

1. **의역에 대한 민감도 (Paraphrase Sensitivity):** 동일한 악성 지시어라도 "Delete all files"와 "Remove every document"는 벡터 공간에서 미세한 차이를 유발하며, 모델의 학습 편향에 따라 유사도 점수가 크게 요동칠 수 있다.  
2. **노이즈 민감도 (Noise Sensitivity):** 임베딩은 입력된 문맥 전체의 평균적인 의미를 담는다. 악성 페이로드가 무해한 내용의 긴 문서 사이에 짧게 숨겨져 있다면, 전체 문서의 벡터는 무해한 주제 쪽으로 끌려가게 되어 악성 페이로드를 활성화할 수 있는 특정 쿼리에 매칭되지 않는 희석(Dilution) 현상이 발생한다.21  
3. **길이 효과 (Length Effects):** 모델은 처리할 수 있는 입력 토큰 수에 엄격한 제한(Context Window)이 있다. 긴 문서의 경우 임베딩 과정에서 뒤쪽 정보가 잘려나가거나(Truncated) 정보의 밀도가 극도로 압축되어 정보의 손실이 발생한다.16

## ---

**4\. LLM 시스템의 검색 파이프라인 (Retrieval Pipeline in RAG)**

에이전트가 과거의 메모리를 회상하거나 외부 문서를 참조하는 RAG 환경에서, 검색은 일련의 파이프라인을 통해 실행된다.23 이 파이프라인의 흐름은 다음과 같다.

1. **데이터 저장 단계 (Ingestion):**  
   * **메모리/문서 ![][image1] 청킹(Chunking):** 방대한 양의 메모리나 문서를 의미론적으로 유의미한 작은 조각(Chunk)으로 나눈다.  
   * **임베딩 생성 (Embedding):** 각 청크를 임베딩 모델을 통해 고차원 벡터로 변환한다.  
   * **벡터 DB (Vector DB):** 변환된 벡터들은 FAISS, Milvus 등과 같은 특수 목적의 벡터 데이터베이스에 인덱싱되어 저장된다.  
2. **검색 단계 (Retrieval):**  
   * **쿼리 임베딩 (Query Embedding):** 사용자의 입력(또는 에이전트의 내부 쿼리)이 들어오면 동일한 임베딩 모델을 통해 쿼리를 벡터로 변환한다.  
   * **유사도 검색 (Similarity Search):** 벡터 DB에서 쿼리 벡터와 모든 메모리 벡터 간의 코사인 유사도 연산을 수행하여 점수를 매긴다.  
   * **Top-k 결과 추출:** 유사도 점수가 가장 높은 상위 ![][image3]개의 청크만을 추출한다.  
3. **생성 단계 (Generation):**  
   * 선별된 Top-k 청크들을 사용자의 원래 쿼리와 함께 LLM의 프롬프트(Context)에 조립하여 최종 답변을 생성한다.24

### **왜 Top-k를 사용하며, 여러 후보를 가져오는가?**

LLM은 한 번에 연산할 수 있는 컨텍스트 윈도우(Context Window)가 제한되어 있으며, 너무 많은 텍스트를 입력하면 연산 비용이 폭증하고 모델의 논리적 추론 능력이 급감한다.24 따라서 전체 데이터베이스가 아닌 소수의 핵심 정보만을 골라내는 Top-k 추출이 필수적이다. 단 하나의 후보(Top-1)만 가져오지 않고 여러 후보(Top-3, Top-5 등)를 가져오는 이유는, 임베딩의 유사도 점수가 실제 LLM이 필요로 하는 논리적 '정답'과 100% 일치하지 않기 때문이다.26 복수의 후보를 제공하여 LLM이 여러 문맥을 종합(Synthesis)하여 추론할 수 있는 재료를 확보하는 것이다.

**핵심 인사이트:** 이 파이프라인에서 가장 명백한 사실은 \*\*"정답(혹은 악성 페이로드)이 벡터 DB 내에 온전히 존재하더라도, 유사도 검색을 통해 Top-k 안에 진입하지 못하면 전체 시스템은 해당 정보를 전혀 알지 못한다"\*\*는 것이다.26

## ---

**5\. 공격 활성화에서 검색이 결정적인 이유 (Why Retrieval is Critical for Attack Activation)**

본 보고서에서 학부 연구자가 가장 깊이 이해해야 할 대목은 바로 **검색 과정이 악성 공격의 활성화(Activation)를 통제하는 결정적 메커니즘**이라는 사실이다.

### **생존(Survival)은 활성화(Activation)의 충분조건이 아니다**

첫 번째와 두 번째 보고서에서 살펴본 것처럼, 공격자가 에이전트와의 상호작용 과정에서 교묘한 프롬프트 인젝션을 시도하고 이 내용이 에이전트의 요약 알고리즘을 견뎌내어 장기 메모리(Vector DB)에 저장되었다고 가정해 보자. 페이로드가 메모리에 텍스트 형태로 온전히 보존(Survive)되어 있다면 공격이 성공한 것일까? 그렇지 않다. 메모리 안에 존재하는 정보는 수동적(Passive)이다. LLM은 현재의 컨텍스트 윈도우에 주어지지 않은 정보에 대해서는 연산을 수행하지 않는다.27 즉, **페이로드가 생존해 있더라도 향후의 작업에서 검색(Retrieve)되지 않는다면 해당 공격은 아무런 효과(No effect)도 미치지 못하며 영원히 동면 상태에 머무르게 된다**.30

### **검색: 메모리 포이즈닝의 활성화 메커니즘 (Activation Mechanism)**

최신 에이전트 보안 연구(2025–2026)에서는 정보 검색 알고리즘이 공격의 '방아쇠(Trigger)' 역할을 한다는 점을 명확히 입증하고 있다.

* **eTAMP (Environment-Injected Memory Poisoning):** 2026년 연구에 따르면, 웹 에이전트가 악성 지시어가 포함된 웹페이지를 스크랩하여 자신의 메모리(Trajectory)에 저장한다. 이후 사용자가 다른 도메인에서 전혀 무관해 보이는 작업을 요청할 때, 에이전트의 의미론적 검색(Semantic Retrieval) 시스템이 과거의 오염된 메모리를 '현재 작업과 관련 있는 문맥'으로 착각하여 Top-k로 불러온다. **이 검색되는 순간, 잠들어 있던 악성 지시어가 LLM의 컨텍스트에 주입되면서 공격이 활성화(Activate)된다**.32  
* **MemoryGraft (경험 이식 공격):** 공격자는 에이전트의 메모리에 '성공적으로 수행한 과거의 작업 경험'으로 위장한 악성 템플릿을 심어둔다. 에이전트가 나중에 이와 유사한 작업을 수행하려 할 때, 유사도 검색이 오염된 메모리를 최상위로 끌어올린다. 에이전트는 검색된 이 악성 패턴을 과거의 모범 사례로 착각하여 그대로 모방(Semantic imitation)하게 된다.30

결론적으로, 메모리 포이즈닝(Memory Poisoning) 공격의 성패는 전적으로 "유사도 검색 알고리즘이 페이로드가 담긴 청크를 현재 쿼리에 대한 높은 관련성(Relevance)을 가진 것으로 판별하여 Top-k에 포함시키는가"에 달려 있다. 검색은 과거의 오염된 기억이 현재의 시스템 행동에 영향을 미칠 수 있는지 여부를 결정하는 절대적인 \*\*수문장(Gatekeeper)\*\*이다.30

## ---

**6\. Hit@k, Recall@k, Precision@k의 이해 (Understanding Evaluation Metrics)**

검색 시스템이 이 수문장 역할을 얼마나 잘 수행하고 있는지(또는 방어에 실패하여 페이로드를 얼마나 잘 통과시키는지)를 정량적으로 평가하기 위해 정보 검색의 평가 지표들을 활용한다. RAG 시스템 평가에서 가장 핵심적인 세 가지 지표를 분석한다.26

지표 계산을 위해 검색 결과는 이진 분류(Binary Classification)에 따라 다음과 같이 구분된다.26

* **True Positive (TP):** 상위 ![][image3]개 내에 검색되었으며, 실제로 정답(관련 문서)인 경우.  
* **False Positive (FP):** 상위 ![][image3]개 내에 검색되었으나, 오답(무관한 문서)인 경우 (노이즈).  
* **False Negative (FN):** 상위 ![][image3]개 내에 검색되지 못했으나, 실제로는 정답인 경우 (치명적 누락).  
* **True Negative (TN):** 검색되지 않았고, 실제로도 무관한 문서인 경우.

### **A. Hit@k (Hit Rate at K)**

* **정의:** 검색된 상위 ![][image3]개의 결과 안에 타겟 문서(정답 또는 악성 페이로드)가 **적어도 하나 이상 존재하는가**를 나타내는 이진(Binary) 지표다.26  
* **설명:** 단일 쿼리에 대해 페이로드가 포함되었으면 1, 포함되지 않았으면 0이다. 전체 테스트 셋에 대해 이를 평균 내어 비율(%)로 표시한다.  
* **직관적 예시:** 과녁에 5발의 다트(k=5)를 던질 때, 단 한 발이라도 과녁(정답)에 맞으면 1점(Hit), 모두 빗나가면 0점으로 처리하는 방식이다.37

### **B. Recall@k (재현율)**

* **정의:** 전체 데이터베이스에 존재하는 모든 정답 문서 중에서, 상위 ![][image3]개 안에 **성공적으로 찾아낸(Retrieve) 문서의 비율**을 의미한다.26  
* **공식:** ![][image4] 35  
* **직관적 예시:** 데이터베이스 안에 특정 질병에 대한 중요 논문이 총 10편(전체 정답) 존재한다고 가정하자. 검색 시스템이 상위 5개의 문서(k=5)를 가져왔고 그중 3편이 해당 논문이었다면, Recall은 $3/10 \= 0.3 (30%)$이 된다. 시스템이 유용한 정보를 '얼마나 빠짐없이' 포획했는지 나타내는 \*\*포괄성(Completeness)\*\*의 지표다.26

### **C. Precision@k (정밀도)**

* **정의:** 시스템이 검색한 상위 ![][image3]개의 문서 중에서, **실제로 정답인 문서의 비율**을 의미한다.26  
* **공식:** ![][image5] 35  
* **직관적 예시:** 앞선 예시와 같이 시스템이 상위 5개의 문서(k=5)를 가져왔고 그중 3편이 정답이었다면, Precision은 $3/5 \= 0.6 (60%)$이 된다. 검색된 결과가 얼마나 노이즈 없이 깨끗한지 나타내는 \*\*정확성(Quality)\*\*의 지표다.26

### **보안 및 LLM 파이프라인에서 Hit@k가 가장 중요한 이유**

정보 검색 시스템의 종류에 따라 중요시되는 지표가 다르지만, LLM 에이전트의 보안(메모리 포이즈닝) 연구에서는 **Hit@k가 압도적으로 중요한 지표**로 취급된다.26 일반적으로 데이터베이스 내에 존재하는 특정 악성 페이로드는 단 1\~2개의 문서(또는 청크) 형태로 존재한다. LLM은 고도의 추론 및 지시 추종 능력을 갖추고 있기 때문에, 프롬프트 내에 해당 페이로드가 '단 하나라도' 노출되기만 하면(Hit=True) 악성 행동을 100% 실행할 수 있다.31 즉, 공격자의 목표는 시스템에 존재하는 모든 악성 코드를 전부 다 찾아내는 것(Recall)이 아니라, 시스템의 컨텍스트 윈도우 안에 단 하나라도 침투시키는 것(Hit)이다. 따라서 학생의 연구는 **Hit@1** (검색 결과 최상단에 페이로드가 노출되는가) 및 **Hit@3** (상위 3개 이내에 노출되는가)라는 단순화된 지표를 사용하는 것이 현상의 본질을 가장 정확하게 측정하는 방법이 된다.26

## ---

**7\. 왜 Top-k 매개변수가 중요한가 (Why Top-k Matters)**

검색 파이프라인에서 ![][image3] 값의 크기를 결정하는 것은 시스템의 성능과 방어 체계에 막대한 영향을 미치는 매우 중요한 설계 요소다.

### **K 크기에 따른 정밀도와 재현율의 상충 관계 (Trade-off)**

정보 검색에서 정밀도(Precision)와 재현율(Recall)은 본질적으로 시소와 같은 상충 관계를 가진다.26

* **작은 k (예: k=1, k=2):** 시스템은 가장 확신하는 극소수의 정보만 가져온다. 가져온 정보가 정답일 확률(Precision)은 매우 높지만, 데이터베이스 어딘가에 있는 다른 중요한 정보를 놓칠 확률(False Negative)이 급증하여 재현율(Recall)은 극도로 낮아진다.40  
* **큰 k (예: k=20, k=50):** 시스템이 그물을 넓게 던져 데이터베이스 내의 정답을 대부분 건져 올리게 된다(높은 Recall). 하지만 그 과정에서 정답과 무관한 노이즈(False Positive) 문서들이 대거 섞여 들어오므로 정밀도(Precision)는 바닥으로 떨어진다.40

### **검색기(Retriever) 단계가 재현율(Recall)을 우선하는 이유**

LLM 기반의 RAG 파이프라인 설계에서 검색기 단계는 전통적으로 **재현율(Recall)을 극대화하는 방향**으로 튜닝된다.26 그 이유는 명확하다. 검색기가 초기에 관련 문서를 놓쳐버리면(Low Recall), 뒤에 이어지는 그 어떠한 강력한 LLM이나 재순위화(Reranking) 알고리즘도 존재하지 않는 정보를 마법처럼 창조해 낼 수 없기 때문이다.26 따라서 실무 시스템은 약간의 노이즈를 감수하더라도 k 값을 크게 설정하여 잠재적 정답(혹은 악성 페이로드)을 일단 컨텍스트 윈도우 안에 포함시키는 전략을 취한다. 이는 보안 관점에서 볼 때 공격자에게 매우 유리한 환경(k가 클수록 Hit@k 달성 확률 급증)을 제공한다.41

### **소규모 실험에서 k=1과 k=3이 합리적인 이유**

반면, 통제된 학술 연구나 소규모 실험 환경에서는 k를 무한정 늘리는 것이 타당하지 않다. 최근의 연구들은 컨텍스트 윈도우가 너무 길어지거나 여러 문서가 주입될 경우, LLM이 문맥의 중간에 위치한 정보를 무시하거나 추론 능력을 상실하는 **'가운데서 길을 잃다(Lost-in-the-middle)'** 현상이 발생함을 증명했다.42 만약 실험에서 k=20을 설정하여 페이로드가 15위에 랭크되었다면, Hit@20은 성공하겠지만 LLM이 이를 인지하지 못해 공격은 최종적으로 실패할 수 있다. 이는 '검색 시스템의 성능'과 'LLM의 독해 능력'이라는 두 가지 변수가 섞여 실험 결과를 오염시킨다. 따라서 **검색 시스템(임베딩)의 순수한 페이로드 탐지 능력**을 격리하여 평가하기 위해서는, 노이즈 간섭이 배제되는 **k=1** 및 **k=3** 환경에서 엄격하게 Hit 여부를 평가하는 것이 가장 과학적이고 합리적인 접근이다.26

## ---

**8\. 검색 실패 모드 (Retrieval Failure Modes)**

정보가 데이터베이스에 명확히 저장되어 있고 요약 과정을 잘 견뎌냈음에도 불구하고, 유사도 검색의 태생적 한계로 인해 페이로드가 검색되지 않는(False Negative) 치명적인 실패 모드들이 존재한다.45

1. **의미적 표류 (Semantic Drift) 및 과도한 일반화:** 임베딩은 단어의 표면적 형태를 버리고 '의미'를 취하기 때문에, 질문이 고도로 구체적일 경우 그 의미를 포괄적인 상위 개념으로 뭉뚱그려 해석하는 경향이 있다.46 공격자가 sys.exit(1)과 같은 구체적인 코드 페이로드를 심어두었을 때, 시스템은 이를 '시스템 종료'나 '오류 처리'라는 광범위한 의미 공간으로 표류(Drift)시켜, 정확한 페이로드가 아닌 일반적인 운영체제 매뉴얼을 Top-k로 반환해 버릴 수 있다.11  
2. **핵심 토큰 누락 (Missing Key Tokens):** 임베딩은 문맥의 평균적 의미를 포착하므로 고유한 식별자(Identifiers)에 대한 가중치가 낮다. 사용자가 특정 환자 번호나 에러 코드("LDAP\_AUTH\_001")를 쿼리할 때, 모델이 이 코드를 노이즈로 취급해버리면 텍스트 상에 정확히 같은 코드가 존재함에도 검색 점수는 하위권으로 밀려난다.11  
3. **노이즈에 의한 희석 (Noisy Embeddings):** 악성 페이로드가 단독으로 존재하지 않고 1,000단어 분량의 일상적인 문서 한가운데 한 줄로 숨겨져 있다고 가정하자. 임베딩 모델이 이 문서를 하나의 벡터로 압축(Pooling)할 때, 압도적으로 많은 비율을 차지하는 일상적인 내용이 벡터의 방향을 지배하게 된다. 결국 악성 페이로드의 의미적 특성은 철저히 희석(Diluted)되어, 공격 관련 쿼리에 반응하지 않게 된다.16  
4. **의역 불일치 (Paraphrase Mismatch):**  
   유사도 모델이 모든 형태의 동의어를 완벽히 같은 공간에 매핑하는 것은 아니다. 특정 훈련 데이터의 편향으로 인해, 질문과 페이로드의 문장 구조가 미묘하게 다를 경우 거리가 멀어져 매칭에 실패할 수 있다.  
5. **길이 편향 및 절사 (Length Bias and Truncation):** 대부분의 신경망 모델은 512토큰 등 입력 길이 제한이 있다.16 문서를 적절히 자르지(Chunking) 않고 밀어 넣으면, 후반부에 존재하는 정보(페이로드)는 임베딩 연산 과정에서 완전히 무시(Truncated)되어 검색의 대상조차 되지 못한다.16

즉, **"데이터베이스에 정보가 존재한다"는 사실이 "해당 정보가 검색될 수 있다"는 결과를 결코 보장하지 않는다**는 것이 이 섹션의 핵심 교훈이다.47

## ---

**9\. 임베딩과 BM25: 두 방식이 모두 중요한 이유 (Embedding vs BM25)**

LLM 시대를 맞아 RAG 시스템에서 임베딩(Dense Retrieval)이 검색의 표준이 되었지만, 정보 검색 연구에서는 여전히 BM25 기반의 어휘 검색(Sparse Retrieval)을 필수적인 비교군(Baseline)으로 활용한다. 그 이유는 두 시스템이 텍스트를 인지하는 방식이 극명하게 대비되기 때문이다.11

* **임베딩 (Embedding):** 텍스트의 구조와 단어를 파괴하고 '의미론적 매칭(Semantic Match)'을 수행한다. 의도(Intent)를 파악하는 데 강하지만, 지엽적인 식별자를 놓치기 쉽다.11  
* **BM25:** 텍스트의 의미는 전혀 모르지만, 단어의 표면적 일치(Exact Keyword Match)와 희귀 단어의 가중치(IDF)를 철저히 계산한다. 데이터베이스 내에 해당 단어가 물리적으로 존재하는지를 100% 확신할 수 있게 해준다.10

### **해석적 가치: BM25를 베이스라인으로 사용하는 이유**

학생의 연구에서 악성 페이로드의 생존과 검색을 평가할 때, 이 두 시스템을 병행하여 테스트하면 훌륭한 연구적 통찰을 얻을 수 있다. 만약 특정 조건에서 \*\*"BM25로는 악성 페이로드가 완벽하게 Top-1으로 검색되지만, 임베딩 기반 검색으로는 완전히 누락되는 현상"\*\*이 관찰된다면 어떻게 해석해야 할까? 11

이는 페이로드의 텍스트가 요약 및 메모리 통합 과정을 견디고 문자 그대로 생존(Structural Survival)했음을 입증한다. 그러나 주변의 정상적인 텍스트 노이즈와 결합되면서 해당 청크의 전체적인 '의미적 벡터 좌표(Semantic Space)'가 변질되어, 임베딩 모델이 페이로드의 존재를 의미론적으로 인식하지 못하는 \*\*해석의 오류(Interpretation Issue)\*\*에 빠졌음을 과학적으로 증명하는 것이다.11 따라서 BM25는 임베딩 검색의 '의미적 실패'를 증명하는 완벽한 대조군이 된다.

## ---

**10\. LLM 검색 환경에서의 평가 과제 (Evaluation Challenges in LLM Retrieval)**

정보 검색(IR) 시스템을 평가하는 고전적인 지표들(nDCG, MAP, MRR 등)은 20여 년간 검색 엔진을 평가하는 데 사용되었으나, LLM의 등장과 함께 그 근본적인 가정이 흔들리고 있다.49

### **인간의 읽기 행동 vs LLM의 소비 행동**

* **인간의 한계와 순위 가중치:** 전통적 IR 지표들은 인간 사용자가 검색 결과를 1위부터 순차적으로(Sequentially) 읽어 내려가며, 아래로 갈수록 집중력을 잃고 읽기를 포기할 확률이 높다는 것을 수학적으로 모델링했다(Position Discounting).  
* **LLM의 동시 소비 특성:** 그러나 RAG 시스템에서 문서를 소비하는 주체는 인간이 아닌 대규모 언어 모델(LLM)이다. LLM은 검색된 10개의 문서(Top-k)를 순차적으로 읽는 것이 아니라, 프롬프트에 담아 \*\*한꺼번에 병렬적으로 처리(All at once)\*\*한다.49 따라서 1위 문서의 가치와 5위 문서의 가치가 인간의 경우처럼 급격히 차이 나지 않는다.

### **무관한 정보(Distractor)가 미치는 악영향**

전통적 평가에서는 무관한 문서가 중간에 섞여 있어도 인간은 이를 시각적으로 '무시'하므로 평가 점수에 0점을 줄 뿐 감점을 하지는 않는다. 반면 LLM의 경우, 관련 없는 문서(Distractor)가 프롬프트에 섞여 들어가면 모델의 어텐션 메커니즘을 교란시켜 답변을 조작하거나 환각(Hallucination)을 유발하는 등 시스템 출력 품질을 능동적으로 훼손(Actively degrade)한다.49 이러한 패러다임의 차이 때문에 최근 2025-2026년 연구들은 LLM의 특성을 반영하여 관련 없는 문서의 악영향을 계산에 포함시키는 UDCG(Utility and Distraction-aware Cumulative Gain)와 같은 새로운 평가지표를 제안하고 있다.49

### **평가 자동화의 난제: Ground Truth 부재 및 LLM-as-a-Judge**

기존의 평가는 사람이 직접 질문과 정답 문서 쌍(Ground Truth)을 라벨링하는 수동 평가(Manual Evaluation)에 의존했다.52 그러나 실시간으로 데이터가 변하고 무한한 질문이 생성되는 LLM 환경에서는 사전에 모든 정답지를 마련할 수 없다.53 이에 따라 최근에는 강력한 LLM(예: GPT-4)에게 검색된 문서와 질문을 주고 "이 문서가 질문에 대한 적절한 문맥을 포함하고 있는가?"를 판별하게 하는 **자동화된 평가(LLM-as-a-Judge)** 방식이 대안으로 활용되고 있다.53 다만 이는 평가를 수행하는 LLM 자체의 편향성이나 환각이라는 새로운 리스크를 수반한다.54

## ---

**11\. 학생의 연구와의 연결성 (Connection to the Student's Research)**

현재 준비 중인 "지시어 형태의 악성 페이로드가 메모리 통합을 거쳐 생존하고, 이후 검색되어 공격을 수행할 수 있는가?"라는 보안 프로젝트는 지금까지 논의한 개념들과 다음과 같이 구체적으로 연결되어야 한다.

* **RQ1 (Survival \- 메모리 생존):** 페이로드가 LLM의 요약 알고리즘이나 압축 메커니즘을 견디고 벡터 DB의 청크 내에 텍스트 형태로 물리적으로 남아 있는지를 확인하는 단계다. 이는 정보의 보존을 확인하는 것일 뿐, LLM이 이를 인지할 수 있음을 의미하지 않으므로 **검색 가능성(Retrievability)과는 완전히 분리하여 사고**해야 한다.47  
* **RQ2 (Retrieval \- 활성화 검색):** 이 연구의 핵심 평가(Core Evaluation) 단계다. 살아남은 페이로드가 특정 사용자의 쿼리에 의해 코사인 유사도 측면에서 높은 점수를 받아 **Top-k (예: Top-3) 내에 성공적으로 진입(Hit)하는지**를 측정한다.26 만약 여기서 Hit@k가 0이라면, 페이로드는 영원히 비활성화된 상태로 남게 된다.  
* **RQ3 (Benign Retention \- 정상 정보의 유지):**  
  악성 페이로드만 잘 검색된다고 해서 의미가 있는 것이 아니다. 에이전트 시스템이 정상적인 기억(Benign Info)들 역시 높은 Recall@k로 원활하게 검색해 낼 수 있어야만, 시스템이 본연의 기능을 유지하면서도 공격에 취약하다는 타당한 결론을 도출할 수 있다.

이 세 가지 연구 질문(RQ)의 결합은 "공격의 생존을 증명하는 것을 넘어, **검색 메커니즘이 어떻게 공격을 촉발하는가(Measuring retrieval is essential)**"를 입증하는 견고한 논리적 틀을 제공한다.

## ---

**12\. 실험 설계를 위한 시사점 (Design Implications for Experiment)**

학생이 실제 코드를 작성하고 실험을 설계할 때 반드시 반영해야 할 실무적 시사점은 다음과 같다.

1. **임베딩 모델의 선택이 결과를 지배한다:** 임베딩 모델의 아키텍처와 학습 데이터에 따라 유사도 거리가 완전히 달라지므로, 동일한 페이로드라도 모델에 따라 공격 성공 여부가 갈린다. 학부 수준의 소규모, 통제된 실험 환경에서는 허깅페이스(Hugging Face) 생태계의 sentence-transformers 라이브러리를 사용하는 것이 가장 합리적이다. all-MiniLM-L6-v2와 같이 가볍고 표준화된 모델은 연산 비용을 최소화하면서도 명확한 의미론적 특성을 보여준다.57  
2. **쿼리 디자인의 고도화 (Direct vs Paraphrase):** 검색 시스템의 취약점을 테스트할 때는 쿼리를 이원화해야 한다. 페이로드의 문장 구조와 정확히 동일한 단어를 포함하는 '직접 쿼리(Direct Query)'와, 의미만 같고 단어가 완전히 다른 '의역 쿼리(Paraphrased Query)'를 각각 벡터 DB에 투척해 보아야 한다.11 이를 통해 임베딩 모델이 의미를 추적하는지, 얕은 표면적 유사성에 의존하는지 명확히 분별할 수 있다.  
3. **단순하고 직관적인 평가의 수용 (Simple Evaluation):** 복잡한 nDCG와 같은 순위 가중치 지표를 억지로 도입할 필요가 없다. 보안 공격 실험에서는 페이로드가 LLM의 시야에 '들어왔느냐 아니냐'의 이분법적 결과가 가장 중요하므로, **Hit@1** 및 **Hit@3**와 같이 단순 명료한 지표를 채택하는 것이 훨씬 과학적이고 타당성 있는 실험 설계로 인정받는다.26

## ---

**13\. 검색 시스템에 대한 흔한 오해 (Common Misconceptions)**

AI 보안 및 에이전트 연구를 시작하는 단계에서 가장 흔히 빠지는 기술적 오해들을 교정한다.

| 흔한 오해 (Myth) | 실제 현실 (Reality) |
| :---- | :---- |
| **"메모리 DB에 정보가 담겨있으면 무조건 사용될 것이다"** | **(False)** LLM은 자신이 직접 DB를 뒤지는 것이 아니다. 유사도 검색(Top-k) 알고리즘이 해당 텍스트 청크를 잘라서 프롬프트에 넣어주지 않으면(Retrieval Failure), LLM 입장에서는 해당 정보가 세상에 존재하지 않는 것과 같다.27 |
| **"임베딩은 언제나 문맥과 의미를 완벽히 매칭한다"** | **(False)** 임베딩은 '의미'를 숫자로 압축하는 과정에서 고유 명사, 에러 코드, 짧은 지시어 등 치명적인 세부 정보를 뭉개버리는(Dilution) 맹점을 뚜렷하게 가지고 있다.11 |
| **"정확도를 위해 Top-1 검색만 평가하면 충분하다"** | **(False)** 실무 RAG 및 에이전트 시스템은 풍부한 컨텍스트를 위해 Top-3에서 Top-10 이상의 청크를 한 번에 주입한다. 페이로드가 1위가 아니더라도 3위로 컨텍스트 윈도우에 잠입하면 공격은 똑같이 성공한다. |
| **"학습/저장 데이터가 많을수록 검색 성능이 무조건 향상된다"** | **(False)** 벡터 DB 내에 데이터가 많아질수록 텍스트 간의 의미적 거리가 조밀해져 관련 없는 정보들이 정답의 위치와 겹치는 '충돌(Collision)'이 발생한다. 이는 오히려 검색의 정밀도(Precision)를 하락시킨다.59 |
| **"의미론적 유사성(Semantic Similarity)은 곧 실제 관련성(Relevance)이다"** | **(False)** 두 벡터가 수학적으로 가깝다는 것이 논리적인 정답을 의미하지 않는다. "방화벽 허용 정책"과 "방화벽 차단 정책"은 구조적으로 비슷해 벡터 유사도는 매우 높지만, 현실에서는 정반대의 결과를 낳는 무관한(Irrelevant) 정보일 수 있다.11 |

## ---

**14\. 결론 및 요약 (Conclusion)**

### **검색은 근본적으로 무엇을 하는가?**

정보 검색은 수십만 개의 데이터 조각이 혼재하는 혼돈의 벡터 공간에서, 현재 사용자의 상황(쿼리)과 수학적 방향성(코사인 유사도)이 가장 일치하는 한 줌의 지식을 걸러내어 LLM의 시야에 공급하는 지능형 필터링 과정이다.3

### **검색이 메모리 사용의 수문장(Gatekeeper)인 이유**

공격자의 치밀한 악성 페이로드가 에이전트의 메모리 통합 과정을 견뎌내고 DB 속에 웅크리고 있다 하더라도, 검색 알고리즘이라는 문턱을 넘지 못하면 아무 일도 발생하지 않는다. 검색은 잠들어 있는 텍스트를 LLM의 실행 가능한 컨텍스트로 격상시키는 유일한 도구이며, 따라서 메모리 포이즈닝 공격의 성패를 최종적으로 결정짓는 수문장(Gatekeeper)이다.30

### **Hit@k가 본 연구에서 지니는 의미**

메모리 포이즈닝은 단 한 번의 노출만으로도 시스템 전체를 장악할 수 있는 치명성을 띠고 있다.37 페이로드가 검색 목록에서 1위를 차지했든 3위를 차지했든 간에(순위), 모델의 눈(Context Window)에 들어왔다는 사실(Hit) 자체가 공격의 완성을 의미한다.26 따라서 Hit@k는 이 연구의 가설을 입증하는 가장 본질적이고 강력한 평가 지표다.

### **실험 구현 전 연구자의 마음가짐**

연구자는 유사도 검색이 결코 마법이 아니라는 사실을 명심해야 한다. 임베딩 모델은 고차원 공간의 방향성을 측정할 뿐이며, 이 과정에서 노이즈 병합, 의미적 표류, 길이 편향 등 수많은 손실이 발생한다.11 이러한 검색 시스템의 기계적인 취약점을 완벽히 이해할 때, 비로소 악성 페이로드가 어떻게 시스템을 속이고 활성화되는지 논리적으로 증명하는 견고한 실험을 설계할 수 있을 것이다.

## ---

**핵심 요약 (Key Takeaways)**

1. **분석의 분리:** 정보의 '물리적 보존(Survival)'과 시스템에 의한 '탐지 및 주입(Retrieval)'은 완전히 별개의 메커니즘이다. 공격은 오직 검색을 통해 컨텍스트 윈도우에 진입해야만 활성화된다.  
2. **평가 지표의 최적화:** LLM 보안 환경에서는 순위 중심의 복잡한 평가(nDCG)보다, 페이로드의 최소 한 번의 노출 여부를 추적하는 **Hit@1, Hit@3** 기반의 직관적인 측정이 현상의 본질을 가장 잘 드러낸다.  
3. **교차 검증의 필수성:** 임베딩(밀집 검색)이 가지는 의미론적 왜곡 현상(Semantic Drift)을 명확히 규명하기 위해, 전통적인 키워드 일치 알고리즘인 **BM25**를 대조군으로 활용하는 것이 매우 효과적이다.  
4. **검색의 내재적 취약점:** 벡터 공간에서 거리가 가깝다는 것(유사성)이 논리적인 정답(관련성)을 보장하지 않으며, 이 간극이 바로 RAG 시스템을 겨냥한 메모리 포이즈닝의 핵심 침투 경로다.

## ---

**📚 추천 연구 문헌 및 도구 (Recommended Papers / Tools)**

* **핵심 도구 (Tools):**  
  * sentence-transformers: 허깅페이스의 파이썬 기반 라이브러리로, 로컬 환경에서 통제된 소규모 임베딩 및 검색 실험을 구성하는 데 필수적인 업계 표준.  
  * BM25: 렉시컬 검색 구현 알고리즘으로, 임베딩의 의미론적 실패를 디버깅하기 위한 베이스라인 측정 도구.  
* **주요 문헌 (Key Literature 2024-2026):**  
  * *Redefining Retrieval Evaluation in the Era of LLMs (arXiv 2510.21440):* LLM의 텍스트 동시 소비 특성을 반영한 새로운 평가 지표(UDCG)를 제안하며 전통적 IR 지표의 한계를 논증한 핵심 논문.  
  * *Poison Once, Exploit Forever (arXiv 2604.02623):* 웹 에이전트의 작업 기록(Trajectory)에 삽입된 악성 지시어가 어떻게 미래의 유사도 검색을 통해 활성화(eTAMP)되는지 증명한 최신 연구.  
  * *Persistent Compromise of LLM Agents via Poisoned Experience Retrieval (MemoryGraft):* 의미론적 모방(Semantic imitation)을 활용하여 에이전트의 RAG 스토어를 오염시키고 행동을 조작하는 메커니즘 분석.  
  * *BEIR (Benchmarking Information Retrieval):* 이기종 데이터 환경에서 다양한 임베딩 검색 모델의 성능을 일관되게 평가하기 위한 제로샷(Zero-shot) 평가 프레임워크.

#### **Works cited**

1. Information retrieval \- Wikipedia, accessed April 28, 2026, [https://en.wikipedia.org/wiki/Information\_retrieval](https://en.wikipedia.org/wiki/Information_retrieval)  
2. Search and ranking for information retrieval (IR) | by Nitin Agarwal | Data Science \+ AI at Microsoft | Medium, accessed April 28, 2026, [https://medium.com/data-science-at-microsoft/search-and-ranking-for-information-retrieval-ir-5f9ca52dd056](https://medium.com/data-science-at-microsoft/search-and-ranking-for-information-retrieval-ir-5f9ca52dd056)  
3. How does IR differ from data retrieval? \- Milvus, accessed April 28, 2026, [https://milvus.io/ai-quick-reference/how-does-ir-differ-from-data-retrieval](https://milvus.io/ai-quick-reference/how-does-ir-differ-from-data-retrieval)  
4. Difference Between Information Retrieval and Data Retrieval \- Action Sync, accessed April 28, 2026, [https://actionsync.ai/blog/what-is-difference-between-information-retrieval-and-vs-data-retrieval-comparison](https://actionsync.ai/blog/what-is-difference-between-information-retrieval-and-vs-data-retrieval-comparison)  
5. what is the difference between data retrieval and information retrieval \- School of Computing Science, accessed April 28, 2026, [https://www.dcs.gla.ac.uk/Keith/Chapter.1/Ch.1.html](https://www.dcs.gla.ac.uk/Keith/Chapter.1/Ch.1.html)  
6. Database Search vs. Information Retrieval: A Novel Method for Studying Natural Language Querying of Semi-Structured Data \- ACL Anthology, accessed April 28, 2026, [https://aclanthology.org/2020.lrec-1.219.pdf](https://aclanthology.org/2020.lrec-1.219.pdf)  
7. Ranking (information retrieval) \- Wikipedia, accessed April 28, 2026, [https://en.wikipedia.org/wiki/Ranking\_(information\_retrieval)](https://en.wikipedia.org/wiki/Ranking_\(information_retrieval\))  
8. Ranking vs. Relevance \- Daniel Tunkelang \- Medium, accessed April 28, 2026, [https://dtunkelang.medium.com/ranking-vs-relevance-ba672c79625e](https://dtunkelang.medium.com/ranking-vs-relevance-ba672c79625e)  
9. What is the difference between ranking and retrieval? \- Milvus, accessed April 28, 2026, [https://milvus.io/ai-quick-reference/what-is-the-difference-between-ranking-and-retrieval](https://milvus.io/ai-quick-reference/what-is-the-difference-between-ranking-and-retrieval)  
10. Sparse Retrieval and BM25: When Lexical Search Wins | Ailog RAG, accessed April 28, 2026, [https://app.ailog.fr/en/blog/guides/sparse-retrieval-bm25](https://app.ailog.fr/en/blog/guides/sparse-retrieval-bm25)  
11. BM25 vs Dense Retrieval for RAG: What Actually Breaks in Production | Ranjan Kumar, accessed April 28, 2026, [https://ranjankumar.in/bm25-vs-dense-retrieval-for-rag-engineers](https://ranjankumar.in/bm25-vs-dense-retrieval-for-rag-engineers)  
12. RAG Techniques & BM25 vs Dense Retrievers — A Complete Practical Guide \- Medium, accessed April 28, 2026, [https://thisissiddharthhudda.medium.com/rag-techniques-bm25-vs-dense-retrievers-a-complete-practical-guide-b1302ee35b7b](https://thisissiddharthhudda.medium.com/rag-techniques-bm25-vs-dense-retrievers-a-complete-practical-guide-b1302ee35b7b)  
13. Sparse vs Dense Vectors: How Lexical and Semantic Search Actually Work, accessed April 28, 2026, [https://bigdataboutique.com/blog/sparse-vs-dense-vectors-how-lexical-and-semantic-search-actually-work](https://bigdataboutique.com/blog/sparse-vs-dense-vectors-how-lexical-and-semantic-search-actually-work)  
14. What Are Vector Embeddings? An Intuitive Explanation \- DataCamp, accessed April 28, 2026, [https://www.datacamp.com/blog/vector-embedding](https://www.datacamp.com/blog/vector-embedding)  
15. Embedding space and static embeddings | Machine Learning \- Google for Developers, accessed April 28, 2026, [https://developers.google.com/machine-learning/crash-course/embeddings/embedding-space](https://developers.google.com/machine-learning/crash-course/embeddings/embedding-space)  
16. How Text Embeddings Actually Work Under the Hood | Let's Data Science, accessed April 28, 2026, [https://letsdatascience.com/blog/text-embeddings-explained-from-intuition-to-production-ready-search](https://letsdatascience.com/blog/text-embeddings-explained-from-intuition-to-production-ready-search)  
17. An intuitive introduction to text embeddings \- The Stack Overflow Blog, accessed April 28, 2026, [https://stackoverflow.blog/2023/11/09/an-intuitive-introduction-to-text-embeddings/](https://stackoverflow.blog/2023/11/09/an-intuitive-introduction-to-text-embeddings/)  
18. A visual introduction to vector embeddings | Microsoft Community Hub, accessed April 28, 2026, [https://techcommunity.microsoft.com/blog/educatordeveloperblog/a-visual-introduction-to-vector-embeddings/4418793](https://techcommunity.microsoft.com/blog/educatordeveloperblog/a-visual-introduction-to-vector-embeddings/4418793)  
19. What Is Cosine Similarity? | IBM, accessed April 28, 2026, [https://www.ibm.com/think/topics/cosine-similarity](https://www.ibm.com/think/topics/cosine-similarity)  
20. Understanding Cosine Similarity and Word Embeddings | by Spencer Porter \- Medium, accessed April 28, 2026, [https://spencerporter2.medium.com/understanding-cosine-similarity-and-word-embeddings-dbf19362a3c](https://spencerporter2.medium.com/understanding-cosine-similarity-and-word-embeddings-dbf19362a3c)  
21. The curious case of poisoned context | by Elina Maliarsky | Mar, 2026 \- Medium, accessed April 28, 2026, [https://medium.com/@elinamaliarsky/the-curious-case-of-poisoned-context-b6e523e1d507](https://medium.com/@elinamaliarsky/the-curious-case-of-poisoned-context-b6e523e1d507)  
22. Retrieval-Augmented Generation: A Comprehensive Survey of Architectures, Enhancements, and Robustness Frontiers \- arXiv, accessed April 28, 2026, [https://arxiv.org/html/2506.00054v1](https://arxiv.org/html/2506.00054v1)  
23. What is RAG? Retrieval-Augmented Generation Explained, accessed April 28, 2026, [https://www.youtube.com/watch?v=KNvkUH50xXM](https://www.youtube.com/watch?v=KNvkUH50xXM)  
24. Advanced RAG Techniques for High-Performance LLM Applications \- Neo4j, accessed April 28, 2026, [https://neo4j.com/blog/genai/advanced-rag-techniques/](https://neo4j.com/blog/genai/advanced-rag-techniques/)  
25. How should memory/RAG benchmarks separate retrieval quality from LLM's reasoning ability? \- Reddit, accessed April 28, 2026, [https://www.reddit.com/r/Rag/comments/1shfhx1/how\_should\_memoryrag\_benchmarks\_separate/](https://www.reddit.com/r/Rag/comments/1shfhx1/how_should_memoryrag_benchmarks_separate/)  
26. How to Evaluate Retrieval Quality in RAG Pipelines: Precision@k, Recall@k, and F1@k, accessed April 28, 2026, [https://towardsdatascience.com/how-to-evaluate-retrieval-quality-in-rag-pipelines-precisionk-recallk-and-f1k/](https://towardsdatascience.com/how-to-evaluate-retrieval-quality-in-rag-pipelines-precisionk-recallk-and-f1k/)  
27. RAG evaluation guide: metrics, frameworks & infrastructure \- Redis, accessed April 28, 2026, [https://redis.io/blog/rag-system-evaluation/](https://redis.io/blog/rag-system-evaluation/)  
28. RAG metrics: how to measure & optimize your retrieval pipeline \- Redis, accessed April 28, 2026, [https://redis.io/blog/rag-metrics/](https://redis.io/blog/rag-metrics/)  
29. Shifting from Ranking to Set Selection for Retrieval Augmented Generation \- arXiv, accessed April 28, 2026, [https://arxiv.org/html/2507.06838v1](https://arxiv.org/html/2507.06838v1)  
30. Agent Memory Poisoning The Attack Waits | Medium, accessed April 28, 2026, [https://medium.com/@michael.hannecke/agent-memory-poisoning-the-attack-that-waits-9400f806fbd7](https://medium.com/@michael.hannecke/agent-memory-poisoning-the-attack-that-waits-9400f806fbd7)  
31. AI Memory Security: Best Practices and Implementation \- Mem0, accessed April 28, 2026, [https://mem0.ai/blog/ai-memory-security-best-practices](https://mem0.ai/blog/ai-memory-security-best-practices)  
32. Observation Poisons Agent Memory | LLM Security Database \- Promptfoo, accessed April 28, 2026, [https://www.promptfoo.dev/lm-security-db/vuln/observation-poisons-agent-memory-5a088378](https://www.promptfoo.dev/lm-security-db/vuln/observation-poisons-agent-memory-5a088378)  
33. Poison Once, Exploit Forever: Environment-Injected Memory Poisoning Attacks on Web Agents \- arXiv, accessed April 28, 2026, [https://arxiv.org/html/2604.02623v1](https://arxiv.org/html/2604.02623v1)  
34. MemoryGraft: Persistent Compromise of LLM Agents via Poisoned Experience Retrieval, accessed April 28, 2026, [https://arxiv.org/html/2512.16962v1](https://arxiv.org/html/2512.16962v1)  
35. RAG evaluation \- Anyscale Docs, accessed April 28, 2026, [https://docs.anyscale.com/rag/evaluation](https://docs.anyscale.com/rag/evaluation)  
36. RAG'S Evaluation Metrics and Standard Industry Pipeline to do Evaluation | by Ayushi Gupta, accessed April 28, 2026, [https://medium.com/@ayushigupta9723/rags-evaluation-metrics-and-standard-industrial-pipeline-to-do-evaluation-f37c3791a2f8](https://medium.com/@ayushigupta9723/rags-evaluation-metrics-and-standard-industrial-pipeline-to-do-evaluation-f37c3791a2f8)  
37. How Hit Rate and MRR Measure LLM Retrievers \- AI Simplified Series., accessed April 28, 2026, [https://tamilselvan-subramanian.medium.com/how-hit-rate-and-mrr-measure-llm-retrievers-ai-simplified-series-7203ba2d4032](https://tamilselvan-subramanian.medium.com/how-hit-rate-and-mrr-measure-llm-retrievers-ai-simplified-series-7203ba2d4032)  
38. Precision and recall at K in ranking and recommendations \- Evidently AI, accessed April 28, 2026, [https://www.evidentlyai.com/ranking-metrics/precision-recall-at-k](https://www.evidentlyai.com/ranking-metrics/precision-recall-at-k)  
39. Why Are Hit Rate and MMR Particularly Important for RAG Systems? | GigaSpaces AI, accessed April 28, 2026, [https://www.gigaspaces.com/question/why-are-hit-rate-and-mmr-particularly-important-for-rag-systems](https://www.gigaspaces.com/question/why-are-hit-rate-and-mmr-particularly-important-for-rag-systems)  
40. Optimizing RAG Evaluation: Key Techniques and Metrics \- Galileo AI, accessed April 28, 2026, [https://galileo.ai/blog/rag-evaluation-techniques-metrics-optimization](https://galileo.ai/blog/rag-evaluation-techniques-metrics-optimization)  
41. Memory Poisoning Attack and Defense on Memory Based LLM-Agents \- arXiv, accessed April 28, 2026, [https://arxiv.org/html/2601.05504v1](https://arxiv.org/html/2601.05504v1)  
42. Lost in the Middle: How Language Models Use Long Contexts, accessed April 28, 2026, [https://arxiv.org/abs/2307.03172](https://arxiv.org/abs/2307.03172)  
43. Lost in the Middle: How Language Models Use Long Contexts, accessed April 28, 2026, [https://teapot123.github.io/files/CSE\_5610\_Fall25/Lecture\_12\_Long\_Context.pdf](https://teapot123.github.io/files/CSE_5610_Fall25/Lecture_12_Long_Context.pdf)  
44. Lost in the Middle: How Language Models Use Long Contexts \- Stanford Computer Science, accessed April 28, 2026, [https://cs.stanford.edu/\~nfliu/papers/lost-in-the-middle.arxiv2023.pdf](https://cs.stanford.edu/~nfliu/papers/lost-in-the-middle.arxiv2023.pdf)  
45. Ten Failure Modes of RAG Nobody Talks About (And How to Detect Them Systematically), accessed April 28, 2026, [https://dev.to/kuldeep\_paul/ten-failure-modes-of-rag-nobody-talks-about-and-how-to-detect-them-systematically-7i4](https://dev.to/kuldeep_paul/ten-failure-modes-of-rag-nobody-talks-about-and-how-to-detect-them-systematically-7i4)  
46. Retrieval-augmented generation (RAG) failure modes and how to fix them, accessed April 28, 2026, [https://snorkel.ai/blog/retrieval-augmented-generation-rag-failure-modes-and-how-to-fix-them/](https://snorkel.ai/blog/retrieval-augmented-generation-rag-failure-modes-and-how-to-fix-them/)  
47. The $4.2 Million Embedding Error: Why Your RAG Pipeline Is Failing \- Medium, accessed April 28, 2026, [https://medium.com/startup-insider-edge/the-4-2-million-embedding-error-why-your-rag-pipeline-is-failing-c4f7ae69dd0e](https://medium.com/startup-insider-edge/the-4-2-million-embedding-error-why-your-rag-pipeline-is-failing-c4f7ae69dd0e)  
48. From BM25 to Corrective RAG: Benchmarking Retrieval Strategies for Text-and-Table Documents \- arXiv, accessed April 28, 2026, [https://arxiv.org/html/2604.01733v1](https://arxiv.org/html/2604.01733v1)  
49. Redefining Retrieval Evaluation in the Era of LLMs \- ACL Anthology, accessed April 28, 2026, [https://aclanthology.org/2026.eacl-long.391.pdf](https://aclanthology.org/2026.eacl-long.391.pdf)  
50. Redefining Retrieval Evaluation in the Era of LLMs \- Semantic Scholar, accessed April 28, 2026, [https://www.semanticscholar.org/paper/Redefining-Retrieval-Evaluation-in-the-Era-of-LLMs-Trappolini-Cuconasu/d0d9507e7ced09030457397d756e77f2418e77bc](https://www.semanticscholar.org/paper/Redefining-Retrieval-Evaluation-in-the-Era-of-LLMs-Trappolini-Cuconasu/d0d9507e7ced09030457397d756e77f2418e77bc)  
51. Redefining Retrieval Evaluation in the Era of LLMs \- arXiv, accessed April 28, 2026, [https://arxiv.org/html/2510.21440v1](https://arxiv.org/html/2510.21440v1)  
52. Don't Use LLMs to Make Relevance Judgments \- PMC, accessed April 28, 2026, [https://pmc.ncbi.nlm.nih.gov/articles/PMC11984504/](https://pmc.ncbi.nlm.nih.gov/articles/PMC11984504/)  
53. LLM-Evaluation Tropes: Perspectives on the Validity of LLM-Evaluations \- arXiv, accessed April 28, 2026, [https://arxiv.org/html/2504.19076v1](https://arxiv.org/html/2504.19076v1)  
54. LLM-based Relevance Assessment Still Can't Replace Human Relevance Assessment, accessed April 28, 2026, [https://research.nii.ac.jp/ntcir/workshop/OnlineProceedings18/pdf/evia/01-EVIA2025-EVIA-ClarkeC.pdf](https://research.nii.ac.jp/ntcir/workshop/OnlineProceedings18/pdf/evia/01-EVIA2025-EVIA-ClarkeC.pdf)  
55. How do you evaluate retrievers in RAG systems: IR metrics or LLM-based metrics? \- Reddit, accessed April 28, 2026, [https://www.reddit.com/r/Rag/comments/1rtjged/how\_do\_you\_evaluate\_retrievers\_in\_rag\_systems\_ir/](https://www.reddit.com/r/Rag/comments/1rtjged/how_do_you_evaluate_retrievers_in_rag_systems_ir/)  
56. LLM Evaluation Metrics, Best Practices and Frameworks \- Aisera, accessed April 28, 2026, [https://aisera.com/blog/llm-evaluation/](https://aisera.com/blog/llm-evaluation/)  
57. SentenceTransformers Documentation — Sentence Transformers documentation, accessed April 28, 2026, [https://sbert.net/](https://sbert.net/)  
58. Sentence Transformers \- Hugging Face, accessed April 28, 2026, [https://huggingface.co/sentence-transformers](https://huggingface.co/sentence-transformers)  
59. Semantic Cache Poisoning: Corrupting the “Fast Path” | by InstaTunnel | Medium, accessed April 28, 2026, [https://medium.com/@instatunnel/semantic-cache-poisoning-corrupting-the-fast-path-e14b7a6cbc1f](https://medium.com/@instatunnel/semantic-cache-poisoning-corrupting-the-fast-path-e14b7a6cbc1f)

[image1]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAABMAAAAYCAYAAAAYl8YPAAAAXElEQVR4XmNgGAWjYIQABXQBSkArugAlgB+Ig9AFKQEXgVgeXZBcwA3Ei4FYBl1iGhDPIgMvAOJfQNzHgASoahg5AGTYdnRBcsEVBipFgAsQC6ILkguommhHMgAAbC0a/+RrKe8AAAAASUVORK5CYII=>

[image2]: <data:image/png;base64,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>

[image3]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAAsAAAAZCAYAAADnstS2AAAA4UlEQVR4XuWRrQpCQRCFR1QQDGI0GEQUTL6GFhGr0SJGQfQFTLeZBfEZrAajYNVoNojR5O+ZnRXmjvsG98DH3Zk5d5idJUqQcqAGuqBtan9ago9na2pBnUjMHVsI6Q5uoGELVimSrgtbCKlIYu6rXB7USRrFxKYDyU9psAJlkgZT5XPibbCBjROQJVkhm8fK57px1xGYgcjnm2AHSj526pF0ePrvkQJz/jQnMWVAC1zBMOZQuoCHivmy/Iq8jQ2ZR3qDs4r59mt/3oOKqrkRBiougBfJOHyOqUqyMpvjB0mOvlI+KKSC1UnrAAAAAElFTkSuQmCC>

[image4]: <data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAAAXUAAAAeCAYAAADTuY7lAAASQUlEQVR4Xu2dC8hlVRXH/1FpL3tNZFExM77LsYdpg1opYlZYERlaVGpFLzNKxcpMmbCISsuyNEuZSiQpwWRMS6KOBpUVmZEaZvhNpFJiYVj0sMf5zT7Ls+66e5/HzDfjzDfnB4vv3n32PWefffb+77XX3vd+0sTEluFhtd1a21Eh/ZW1PTWkGc+o7dW17Vfbw136V2u7xL3fVLj+/bUdW9vza/v97OFOdqzt7NrOUbpHyoYNpS//Y5q/j1cq54G1Hd4e1t+U6rDEbkp1aKxXub4NrvG/2v6jVBdmf6rtHS7fEKhPyjiUvWq7p7ZHxwMZHhcTNpHFPt/ExJLnD7WtCWk5UX+Ikkgub94/tLZTa1vZvC8J4T5KwlOpFUPPDrV9ydn3avtzbfsqfeYD6hb1FUpCd3xIBytPLBv38v7a3qg0MB1T273ueMzfB9fnXEYUdV53iegQUYdcvsOU6qcEgvy5kDZW1P9d29/VL+o/qO3JzetPa3bw8fbDJh8OBQMV9i9nz9IsX9SsAzExMdHBX2t7TkjLifqTarsppO1S2xua111CiIhWyos6vFhpsMDzxbOulPLyt0/UAZEgX6Qk6nvWdqd7DyfU9rTmdczfBeK0rrZHurQtKeqc++khzfOy2l4V0oaKOs/taKUB8LG1LdT2YZ/Bwf2/zr2n/nZuXldKgwI8orY/qm1z+9d2mdJzB66DsL+meQ/Ur38/MTGRAc8Hz/iJSh73s90xhALRQ0hPUeqIdHDvqfOezr53875PCCuVRd2wa6xs3lcaJuqVxon6C2v7i3sPCN8zm9cxfwnu/77aTlYquwnTWFG/Q0nkEOyuOmI2Yc/F2wt8Jscqtfe5k9rZ0OW1/dcyFcBbxsNHZCNn1PYP9557X+vecw+fce8rzd4/9cYMA3i2sa7XazY/zsNt7v3ExEQGOtaNzWs6JVNs0iDnqQPxTUTis0resRfGPiGslDo73hvnXqZ0XQ8DBF6cUdX2S6XwDTOKEpXyon67kojx15eNclyjNl7LwPZ9tVP8vnsByk7o6oDarq3tU+5Yn6jjWR9e27dq+52GDXhwkMaFIS5UCg3Zfb5bSUxfqtZzznF6bR+LiQEGeiPO4pi9EH4xKs3e/zvVLep3KcXwDcI+hOUmJiYKsEiHkEVWNH8RoSEiYxA2IZ4aO6enUjrn1bW9ZfbQBhDJW9TOBKDS5vHUjRVKdcFfP8CU8ht4sMwoPE+pbXXzmrLGOLctqnoxNGJ4xHOi5j3zaBaXRgx3Tx/bMHAwSHNfDGBXNOnQFX6h/ilnzjhG23htbR9REn8EnTSuXaJS+XqUxQZfrFKaHcbBi+c7pk1OTGw3EPuMggN0GDouncl7mUBoho6MfVKp8+FBIyh40XhR16lbCPFIn6vZXSKeIzUfRqg0XtQpP9fA0+wT9RJd+fFCEW8/CESoi9xMh89anBkI15yp5FF3LUJS74i1DQh99UEdsLjIYjYgxixKmih2iTqDE6EZnqsJLYMCafbe7Gyl+6S9rOfDBSqVr0dZfqy0I4hz+frx8HyfEBO3VT6kdiTO2VqlhZ8HCzqrlYXGaVhabiEOvqG0IMJC2yG1fUIptnio8h2GDsCCip0XD2Fi07FQhHnuByrtPukiF57pEkKg0z+vtrvVxuGNK9XOEjyVhok6goiQI5omZJATddqiiZL3dLnnC5S2UF7q8vdBWyXWbTtpjlAbY49Q11VM1LDwC+VaoxRKoT5unjk6C2XIXd/i412iniOGjyKbKuqV+u9/SXnq5kEheDS+XZv3GB2QNOJmuYe4JVim5GFRDtsJAZSPxZKfaX6ERbjP1fwiDB1yobazQrrBFPYqpWs9WPe71GDRMHY4vNz4zDwbI+qVUlvGY2QQMZZrdueIp1Lag90n6iVyog44QqwLcB9cP7almL+LlUo7ZzzE2kmPmKj70Al5SesTLPoGeQmrUB8/nz08isUWddrQnTHRUan8+aGiTpvsms1skyBkuRVrKpxjcS/qloSOF8XbFk+80AOLc0c3r/FsfqW0aPMetZ2bjsZ0PAeNg0W9icUB8UEsDJ7Bd5SP/Rq0uThNLgnhPmr3I+MNE1PnNdfkWBfrlLa8bayoW0imVLYSY/KzDTLG8/tEPVKpX9RgveYH041hbH32eeKl+2JBlLrgeWOEdFhgN36t5JByjF1EJdCVn8bEbR08CesIkZOUjg1thIsNZcPzwXhtmDfAXw9hF8ScaSQd/FQl7+Pbml3h/oXaLzJ4uFcGi4nFgWfGghqxXeK2LKj1hV/YuRC9Jtpf346JsbxNaSbIVkNi+GNhjzbg8IwpG/dC2HMohBGtHX+wtvfNHn4AE78Ym75e84NkDp4Lzo+tb2D0f57ZijZbL2Prk+f9o5gYWKN2S+dic4DSIvqSgjggYpZraMQoOUbjz7FK6eH7WKMHgSXGSQgl552RVjoGJfFG5CmXF3oah3nzzDriIIWo8Bk8A7wfzuHB++e4/1r0fkpfXpmYmHhwYRuj7++LBc7e8pi4rUMIAxFEFD14u4gc05gIXx7AE+bYJc3fWDH27S08gAXNfyEDb4zjhFbYssQCDCvefkRGYClDjMHiTZPuwaswOEa5PEwtTdRz3gEDB6EXYnhwsNJ0jqnh5mhMExMTw2GdrBQ23VhwOpfct0kRS0QVsfOGyJ/m8hk2nSaO6UWc8IaPu3OceJZhu1hMnBkdvShbOWLsPifeQBr5PRYiYhrK8RiL9KLOVJatYYaFeS5TetAMCAxq9yh9JoYDPAxefnFqiE1MTExsFgi9sC3Qh14IpSDIeNERto3hcccVeUIkL2lem6h+vT28QYD9+Tju41jE/BD+OGpy3riybWsAiLDH4pqEcjge45xe1In7+cURW3g9pbb3qhVxtsP5byFuaSxkNNlkk00WLRtBWKt0EM/Vg8iTHmPZ5tWbx8mXRXzYhtAJ3u6C2h8wiuTCPRZm8av0JfEuxdmJ769sXrMWEAcDBh3OZ+GXy90xysSx+5u/R6m8TrAtwWB5Xm0vjwcK8BXv2zRsYQ2Y1VB3zLy6ZjMG+YEy5XZb5eAzlIlBf0tAu9tNqR0Mqbedmr/UHX1iSD0Ae89po4+KBwbCAiZlHAKzThy1C+OBDGztzcHaV2l7aA6+DHaD8t8jyUE9XKQU2s2K1cQwTKTjQiXCTLr9CJFBfLmrIZnHzedLq9U0LBYx/UBCuCOel5AMaXHbIu8pd4yz0+BsBkHsjUZsX3rhPhAGzsdsgC8j+fAMAwfHdlQr/ue749saiCZbOXkGDH5xAOyCehm7ta1S/7Y5Bkl+qIkZ3Nhtb5SJtlcC4T9R5QX9IdAHfD+gfLHeYj8BQo//bF5bCHAoXfmpT6tT+xmAA9UOjObclKC+WR+yQaZS9/UM6yf0UR8yJM1vIhgCDlSswy720jAHgXtbqsKPrm3SvfGgch4TaRyLwmmx8QgPAZEuhVGAxkhhaVixcbFASfzaPH3w4s2IT+Ow2LsJPWLsKwAhplEY7Fxhd4519j2a9/GLSYSgEEGDezQRoSF3VTLH7AtbQ22xWK3ZLWiEkL7QHEMIEYWcqB+htAj8G7U/tGVEUedr/NQH+4D5e59SJ2f7m1GpFaCVSj8mhacGPHc7zrl5nRN1OiplIp0y+WfUJ+qwt9otrUPAG/eDAMLsB/qcqIM9Q7sO91U1r2O7pm1QX3jlDAjHaHZLYszfBX3Kr1PlRJ3zlQbXSsOuR9/K1TUbHnL1YTDLjaI/RtSPVNIBtm12ifrJan/Qi23Jcb3KG/nYz84zoO1idzZ/ceyiDmwKdv5o6BpOJW2bEDTP0ELVYPlIP1ypXX3NHR8FDY6Txa1/YBeyBoKQ0mnwskiPXKxUEESZPL5zAAVdozQN5JhvXLbLhmvs0vwFPHo8eMp5XZPP1gD21/xvLAN5rlb+B6U8VJ6fWnN9BNG/r5Susc6lb20gTFfV9uXa3qok7HhzUBJ1nsW9SvlfpLS2sNwdj6IOiAcdtESlWTG5Q7P5K3WLOmViTzVlwjmgTCy2G32izufXKq35bAyUjR1c/rsLUdRNaBh8yIcXTD11ifqemv0HF7Cg2d9WHwrl821xc4k6zy1X13F2HcmFRIeKOn0cwWMwZ01rQWXBRR9WNq+pfwYSBkyrDwuHoR3ewaNP+1k/zmduZ9+mQNjaO7S0S0QdvQLKSzlueSBH0pivaNZxXKX0X8AGYyeO5gXyrCbt4435Tn+w0gOg0ePdUDG+QMAUjpGHPOSNMWqOk45AsziJh83s4Ga116KcnONuzf7bqXcpeQ1UVgnickyL8S4pAw2Sc12v1Gg8xPLi/Vt5+Nx+Ln1rxMQ7khN1mwnFMAKd2IR8qKgTEuD85K2a1wbnwytl8GSgseMlUadMPC8PHdCXKSc0QNujfCfU9nqlZ9bl6UWIFbP4z3kQLb6YxgDT5ZmS97dKHa9L1BF+zEN7trBmzJ+DayF2C0peKs/QwmpjRB1P8ZtKdbpzOObhvAz60etlVkdd5eCLWCZmaAf1h12jch0aOGxx1xsg6mcoDfQGg+Fh7r0/t9WH3T/iThkM+rhvw7wmrVRfBvU15DkB7dSugaDTDqkX0oFzoW84LUc2aWDHPRalWDRoSHsoVcrp4RhQOMS9VCEIOHl2Vz6PeTsmLlxvmeanzrsqlSOyr4b9f0Ea7yFKZSktAMEKzQ5MvPbl25rxok5jQdSYYjLljKLuBchjU0QoiTodlPbAIP7Z2l6h9lcUK82LuuXH6DwM4pSp0ryokxY7Dp3Dlykn6jyn45QGEHt+iAGhtEuVbzsRHIzzlM7j20D01D2rlab10CXqHKN+rK0yg/Qhopg/x3FKz5RfOLxWbcioT9TJR/9jM8NPlIQ6Jx4RhOigmNiB3b/9WzgGRhwn2hDPpVSHOyhpi91PCd8HEXQ/W4gC70Wd+7AwDURRt80RfX18Y0Ud7/x4JW2hrwDnonzMkG9V68Dmngvt3Tz8ie0ML+oeS/eiDoQoTlU7czpQyQM0SqIePXVPpXlR9/mtU3DuSvOiTpnwjH2ZeG/kRJ3ZGLO80o4MzuW9oQgdhpmbQQdjwLLzlUTdvHjyIxwMIlVzLNf5GShWqP1tdU8uv8G58WDjrIMZDYNKrEMwZ8qHkQzCQDhJJRY07517454RQeyu9JEN8Jp6Rmxv0mzYpCv8skqprDmjbm2dyDuW1BezlBxR1COU2xyNi5QGcxzTPsaKOvVkdRXF2kQdmKHYembMB5Xy6RPbAWNFHfDaaNSVUljKz5AWQ9SZOo4RdWCNgzLR4SgTC1lGFHXWEt7s3o8F8btYyfv1ICa2BS8n6gj0jUrijAeO593lqffRlf/zav8xRg5EFGGI4Hny/CirsbPS9y266mylkojyWfNec23HwwzkaLXXIjxzvlrh7RL1tUqiRj3ajA7Rvdy9Nzu7+UxXfQ0R9TPVDhy+fiK22wijPVjYyqx0DdqptXsGoijK/rMW9luh+XxQaf67NhPbCUyV8QyIedKgbKrfJepd5EQdITMvjWsR47XQCFSabehMxf1xo0vUu4iivpico/wX7QhlIYbGaZqN/VLPCFJJ1L+rJEjk4V5tBwY/dnWBUmiEENFQTlYajBmAMWLzpOXIPfNKefGIUOY1zWvOsUt7aIY3KR8eIL/NdrpEPUdf/k0V9SH3HxnrqceYuicOCLQhynW7SzMqbVx5J5YwNAi8kVwH7yIn6jnvz1Op3Jk8iyXqeKnec8oZ03timseq+5+eeFH2rFbrcfL3UM3fP/VSEnXKiDfKAjwdfZnmPz9ULIC4+gnu/XFNWg7q1zYJmA0VtberPS/n8QPbWPpEOtKXH881CqWxtYl6jijqgLdO2SLMdk+KiRPbN9aAx4o6DSk2vD6GNnoWp8g7VtQpUxXSSrHVsZREfSh8nrABDK0HY0z+Bc2KEgLfJerxmVcaJmqI0mLNijhXdBC66MvPPcUZICLPjMdmkoTufL2woM0AZ8f94uoQhoq6vwav4+BzQ3MMI7RmMKtZ594brE/Y1teJiQ0QowY6QWkKnYOO1bcjIHJhTChAh2LaTpnG/OY2ZWKBcHPAvXJuW5gzY8fIkIGHzntu8zq3Na+LIWJh8KUlwkTMADB2+JR+Wx3xi7FpwjUf9ZkK8IwIscX6ILx3hcs3BM41ZlteX37CFX6v+ZZgqKgvNnzfZ7Ecl4mJiYmtlis1+43MpQg7leL27omJiYklyXKNnzFsS+ym9K34iYmJiYntkf8DtlmHi/S42i8AAAAASUVORK5CYII=>

[image5]: <data:image/png;base64,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>