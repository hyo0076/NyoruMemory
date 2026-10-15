# 두 임베딩으로 검색

0.28.0 통합 배포에 포함됩니다. 최초 구현: 2026-10-09 / `codex/jev-memory-shadow`.

## 사용 방법

1. [NyoruMemory.js](../NyoruMemory.js)로 플러그인을 교체한다. 로컬 프로젝트의 최신 설치 파일은 `release/NyoruMemory.js`이다.
2. **모델 및 설정 → 표현이 달라도 찾기 · 임베딩 연결**에서 **임베딩 서비스**를 고른다. 주소·모델이 즉시 입력되며, 해당 서비스의 API 키를 입력하고 저장한다. 이미 연결한 주소·모델은 설정을 여는 것만으로 바뀌지 않는다.
3. **임베딩 두 개로 검색**을 켜면 바로 아래의 두 번째 연결도 같은 방식으로 설정할 수 있다.
4. **기억 보관함 → 주입 미리보기**에서 두 모델의 실제 검색 여부, API 호출 수, 기억/검색 문장 캐시 재사용, 오류를 볼 수 있다. 미리보기도 유료 API를 사용할 수 있다. 연결 확인 버튼은 누른 연결만 짧은 예시로 한 번 호출한다.

서비스를 직접 선택할 때만 아래 기본 모델을 채운다. 주소·모델은 자유롭게 수정할 수 있으며, 저장한 모델로 기억 본문과 검색 문장을 임베딩한다. 다른 모델이나 Lite 모델로 자동 대체하지 않는다.

## 연결 가능한 API 형식

| 서비스 | 자동 입력 모델 | 자동 입력 주소 |
|---|---|---|
| Voyage 일반 | `voyage-4-large` | `https://api.voyageai.com/v1/embeddings` |
| Voyage 문맥 | `voyage-context-4` | `https://api.voyageai.com/v1/contextualizedembeddings` |
| OpenAI 호환 | `text-embedding-3-large` | `https://api.openai.com/v1/embeddings` |
| Google Gemini | `gemini-embedding-2` | `https://generativelanguage.googleapis.com/v1beta/models/gemini-embedding-2:batchEmbedContents` |
| Cohere v2 | `embed-v5.0-pro` | `https://api.cohere.com/v2/embed` |

예를 들어 `voyage-context-4`와 `voyage-4-large` 또는 Voyage와 Gemini/Cohere를 함께 연결할 수 있다. 이는 API 호환 구조의 설명이며, 그 조합이 RP에서 가장 좋다는 성능 검증 결과가 아니다. Gemini Embedding 2에는 검색용 지시문 접두어를 붙이고 지원하지 않는 `taskType`을 보내지 않는다. Embedding 001은 기존 `taskType` 방식을 유지한다. 두 경우 모두 글마다 별도 배치 항목을 만들어 여러 기억을 한 벡터로 합치지 않는다. Cohere는 v2 텍스트 임베딩 규약을 사용한다. 독자 인증·다중 벡터 등 별도 규약이 필요한 모든 서비스까지 자동 호환되는 것은 아니다.

프록시는 서비스 선택 후 주소·모델을 바꾸면 된다. 서비스 선택으로 다른 서버의 주소가 채워지면 이전 API 키는 비운다. 같은 Voyage 서버를 쓰는 일반·문맥 간 전환은 키를 유지한다. **자동 선택**으로 전환하면 입력값은 그대로 두고 공식 주소·모델에 맞춰 요청 형식만 판별한다. Gemini 공식 주소는 입력한 모델의 배치 요청 경로로 맞춘다. Voyage 공식 일반 주소에 context 모델을 넣으면 문맥 전용 경로를 사용한다.

주소와 모델 ID가 있으면 선택란에 **자동 선택 · Voyage 문맥**처럼 인식한 형식이 표시되고, 아래에는 실제 요청 주소가 표시된다. 자동 모드는 주소나 모델을 바꾸면 다시 판별한다. 첫 번째·두 번째 연결 모두 같은 방식이다. 기존 버전에서 서비스를 골랐지만 주소·모델을 비워 둔 경우 설정을 열 때 빈 항목을 채운다. 입력값 채우기와 요청 형식 표시는 API를 호출하지 않는다. 실제 검색에는 저장한 설정이 적용된다.

## 검색과 최적화

- 각 모델이 자기 벡터로만 유사도를 계산한다. 서로 다른 차원의 벡터도 별도 검색하므로 함께 사용할 수 있다.
- 둘 다 결과가 있으면 **RRF**로 기억 ID별 순위를 합친다. `k=60`, 동일 비중이며, 합계에 `61 / 사용 가능한 모델 수`를 곱해 0~1 범위로 맞춘다. 0 이하의 유사도는 순위 점수를 주지 않는다. 이 통합 점수는 정답 확률이나 코사인 유사도가 아니다.
- 기존 단어 일치·최근 기억 우선·이름 검색·주입 토큰 한도는 유지한다. 관계의 기존 유사도 자격 기준은 합친 순위 점수가 아닌 각 모델의 유사도로 확인한다. 선택된 기억은 ID 기준으로 한 번만 주입한다.
- 두 연결은 동시에 실행한다. 각 연결은 최대 두 묶음을 병렬로 요청하고, 플러그인 전체의 임베딩 API 요청은 최대 네 개로 제한한다. 검색 자료 저장은 순서대로 처리하고, 저장 중 도착한 새 결과를 다음 저장에서 누락하지 않는다. 최초 색인은 기억량에 따라 오래 걸릴 수 있으며, 일반 검색 시간은 대략 더 느린 모델의 응답과 로컬 처리 시간에 영향을 받는다.
- 기억 벡터는 기존 `:vectors`와 새 `:vectors-secondary`에 분리한다. 모델별로 새로 생기거나 내용이 바뀐 기억만 다시 만든다. 중단 전 완성된 기록은 주기적으로 저장한다.
- 같은 검색 문장의 조각은 실행 중 메모리에 최대 5분 재사용한다. 최대 128개, 계산상 4 MiB 이내이며, 재시작하거나 연결 설정이 바뀌면 비운다. 실패한 요청은 캐시하지 않는다.
- 벡터 길이 계산을 재사용하고, 모든 비교 점수를 담는 임시 배열 생성을 없앴다. 벡터 정밀도나 기억 본문은 줄이지 않는다.
- Gemini 텍스트는 UTF-8 2,000바이트 이하로 나누어 기존 embedding-001에도 맞는 보수적인 입력 크기를 유지한다. 다른 모델은 기존 2,000자 기준을 유지한다. Voyage 문맥 모델은 같은 기억의 연속 조각을 겹침 없이 문서로 묶는다.
- 주소·모델·API 형식이 같은 중복 연결은 한 번만 사용한다. 하나가 실패해도 다른 모델의 결과를 사용하며, 둘 다 쓸 수 없는 기억은 기존 단어 검색으로 찾는다. 실패를 숨기지 않고 메뉴/보관함/미리보기에 표시한다.
- 채팅·기억 revision·설정 변경 검사와 삭제 후 늦게 도착한 응답 차단을 유지한다. 두 모델을 합친 직후에도 revision을 다시 확인한다.
- 진단/관찰 기록에는 모델별 호출 수·캐시·오류, 선택된 기억의 모델별 순위를 연결한다. API 키는 마스킹한다.

## 기존 기억과 비용

내용·대사, 사실, 관계, 줄거리를 다시 생성하지 않는다. 두 번째 모델용 검색 자료는 처음에 한 번 준비해야 한다. 두 모델을 쓰면 같은 기억과 검색 문장이 두 서비스로 전송되어 비용과 벡터 저장량이 늘어난다. 두 번째 연결을 끄면 호출을 멈추고 캐시는 보관한다. 기억 모두 삭제 시 두 캐시 모두 제거한다.

추가 설정은 안정판의 엄격한 설정 스키마와 충돌하지 않도록 `longmemory:v1:embedding-settings`에 따로 저장한다. 기억 스키마 및 Risu 업데이트 주소는 바꾸지 않는다.

## 두 임베딩과 reranker의 차이

현재 구현은 두 임베딩 모델이 각각 검색하고 순위를 합치는 RRF 방식이다. 두 모델이 공통으로 높게 평가한 기억은 올라가며, 한 모델만 잘 찾는 기억도 다른 검색 결과와 함께 경쟁한다. 합치는 단계에서 AI를 다시 부르지 않고 프로그램이 순위로 계산한다. 서로 다른 모델의 벡터나 원래 유사도 점수를 평균내지 않는다.

Reranker는 임베딩 검색으로 먼저 추린 후보의 본문과 검색 문장을 함께 받아 관련성을 다시 평가하는 별도 모델이다. 예를 들어 후보 30개를 찾아온 뒤 그중 8개를 고르는 단계에 쓴다. 후보에 들어오지 못한 기억은 되살릴 수 없고, 사실의 참·거짓을 검증하는 기능도 아니다. 현재 브랜치에는 reranker 호출을 구현하지 않았다. 나중에 하나 또는 두 임베딩 검색 결과 뒤에 붙일 수 있다.

- [Voyage Rerankers](https://docs.voyageai.com/docs/reranker)
- [Voyage 일반 임베딩 모델](https://docs.voyageai.com/docs/embeddings)
- [Gemini 모델별 검색 입력 형식](https://ai.google.dev/gemini-api/docs/embeddings)
- [Cohere 임베딩 모델](https://docs.cohere.com/v2/docs/embeddings)
- [OpenAI text-embedding-3-large](https://developers.openai.com/api/docs/models/text-embedding-3-large)

## 어느 정도 좋아지는가

EACL 2026 산업 트랙 논문 *Retrieval Enhancements for RAG: Insights from a Deployed Customer Support Chatbot*의 표 3은 강한 단일 모델에 두 번째 dense 임베딩을 RRF로 합친 결과를 비교한다. 아래는 Recall@10, 즉 정답 자료를 상위 10개 안에서 얼마나 회수했는지 나타내는 수치다.

| 데이터 / 조합 | Gemini 단독 | 두 모델 | 차이 |
|---|---:|---:|---:|
| FIQA / Gemini + Stella 1.5B | 65.7% | 66.7% | +1.0%p |
| SciDocs / Gemini + Stella 1.5B | 27.1% | 27.5% | +0.4%p |
| Help Articles / Gemini + text-embedding-3-large | 69.4% | 73.2% | +3.8%p |
| HotpotQA / Gemini + text-embedding-3-large | 94.6% | 92.6% | −2.0%p |

논문의 Gemini 모델은 `gemini_large_03_07`이다. 현재 모델들, 한영 RP, NyoruMemory에 그대로 적용되는 수치는 아니다. 같은 조합도 자료에 따라 나빠질 수 있으므로 보편적인 개선율은 제시할 수 없다. 이 비교에는 후속 LLM 재정렬의 이득을 섞지 않았다.

- [논문 표 2·3](https://aclanthology.org/2026.eacl-industry.13/)
- [Gemini 임베딩 API](https://ai.google.dev/api/embeddings)
- [Cohere Embed API v2](https://docs.cohere.com/reference/embed)
- [Voyage 문맥 임베딩](https://docs.voyageai.com/docs/contextualized-chunk-embeddings)

사용자 요청에 따라 테스트·실 API 호출·설치된 RisuAI 실행 확인·GitHub 업로드는 하지 않는다. 실험용 설치 파일만 빌드한다.
