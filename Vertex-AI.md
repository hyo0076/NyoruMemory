# Vertex AI 연결

1. 모델 및 설정 → 모델 연결 → API 형식에서 **Google Vertex AI · 서비스 계정 JSON**을 선택합니다.
2. Google Cloud에서 받은 서비스 계정 JSON 전체를 붙여넣거나 **JSON 파일 불러오기**로 선택합니다.
3. 사용할 **Gemini 모델 ID**를 입력합니다. 프로젝트 ID는 비워두면 JSON의 `project_id`를 사용합니다.
4. 지역은 기본 `global`입니다. 특정 지역을 사용할 경우 해당 모델이 지원하는 지역을 입력합니다.
5. **저장** 후 **연결 확인**을 누릅니다. 이 버튼은 실제 모델을 한 번 호출합니다. PDF를 켰다면 PDF 읽기도 확인합니다.

‘다른 검수 모델’을 켜면 검수 모델에도 별도 서비스 계정과 모델을 설정할 수 있습니다. RisuAI의 메인·보조 모델 연결은 사용하지 않습니다. 기존 API 키 방식 Gemini도 그대로 선택할 수 있습니다.

## 인증과 요청

JSON의 `type`은 `service_account`여야 하며 `project_id`, `client_email`, `private_key`가 필요합니다. 다른 계정 JSON(예: `authorized_user`)은 지원하지 않습니다. 서비스 계정은 대상 프로젝트에서 모델을 사용할 권한과 API 활성화가 필요합니다. [Google 인증 안내](https://developers.google.com/identity/protocols/oauth2/service-account), [Vertex AI REST 인증](https://docs.cloud.google.com/docs/authentication/rest).

개인 키로 서명한 JWT를 Google 공식 인증 주소에 보내 액세스 토큰을 받습니다. 이후 Gemini 요청에는 토큰만 사용합니다. 토큰은 플러그인 메모리에서 재사용하고 만료 전에 갱신합니다. JSON은 연결 설정에 저장되지만 기억 백업이나 추출 프롬프트에는 들어가지 않습니다. 인증 실패 응답은 원문을 표시하지 않고, 모델 오류에 포함된 키·토큰은 가립니다.

`global`은 `aiplatform.googleapis.com`, 개별 지역은 `{지역}-aiplatform.googleapis.com`을 사용합니다. `us`·`eu`는 공식 다중 지역 주소로 연결합니다. API 주소를 직접 입력하지 않습니다. [공식 엔드포인트 안내](https://cloud.google.com/vertex-ai/generative-ai/docs/learn/locations).

티어를 비우면 서버 기본값입니다. `standard`는 shared 요청, `flex`·`priority`는 shared 요청에 해당 티어 헤더를 추가합니다. 지원하지 않는 조합은 서버 오류를 표시하며 다른 티어로 바꾸거나 자동 재시도하지 않습니다. [Flex PayGo](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/flex-paygo), [Priority PayGo](https://docs.cloud.google.com/gemini-enterprise-agent-platform/models/priority-paygo).

## 연결 실패 시

- 인증 실패: 서비스 계정 키가 유효한지, 기기 시간이 맞는지 확인합니다.
- HTTP 403: 대상 프로젝트의 API 활성화와 서비스 계정 권한을 확인합니다.
- HTTP 404: 프로젝트·지역·모델 ID를 확인합니다.
- HTTP 401: 캐시된 토큰을 비웁니다. 다음 연결 확인에서 다시 인증하며 실패한 모델 요청은 자동 반복하지 않습니다.

검증 범위: 테스트용 RSA 키와 모의 Google 응답으로 서명, 캐시, 만료, 병렬 요청, 취소, 오류 가림을 검증합니다. 실제 사용자 계정의 IAM·모델 접근 권한과 설치된 RisuAI에서의 Google 호출은 별도 연결 확인이 필요합니다.
