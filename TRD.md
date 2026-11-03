# 쇼핑몰 문의 응대 에이전트 TRD

## 1. 문서 목적

[PRD.md](PRD.md)에 정의한 쇼핑몰 문의 응대 에이전트의 기술 구성, 에이전트 설계, Azure 인프라, 보안, 오류 처리, 테스트를 정리한다.

## 2. 시스템 구성

```
브라우저 ── SSE 스트리밍 ──▶ FastAPI 앱 (app/main.py)
 (static/app.js)                  │
 JS 없으면 일반 폼 제출           ├─ 사전 점검 (app/guard.py, app/pii.py) ─ 개인정보 가림, 인젝션 신호 탐지
                                  ├─ Microsoft Agent Framework 순차 워크플로 (app/agents.py)
                                  │    ScreeningExecutor → 분류 에이전트 → 조사 에이전트 → 답변 에이전트
                                  │                         └ 도구: get_order, search_policy, get_inquiry_history
                                  │    각 에이전트 = GitHubCopilotAgent, 앱 전체가 CopilotClient 하나를 공유
                                  ├─ 근거 검증 (app/grounding.py) ─ 답장 문장별 사실 대조
                                  ├─ 생산성 측정 (app/metrics.py)
                                  └─ 저장소 (app/storage.py)
                                        │ Managed Identity (DefaultAzureCredential)
        ┌───────────────────────────────┼─────────────────────────────┐
  Azure Table Storage           Azure Key Vault              Application Insights
  (orders, inquiries)     (GITHUB-TOKEN, ADMIN-PASSWORD)       + Log Analytics
```

## 3. 모듈 구성

| 파일 | 역할 |
|---|---|
| `app/main.py` | 앱 팩토리(`create_app`), 라우트, 세션 쿠키·보안 헤더 미들웨어, 입력 검증, 횟수 제한, SSE 응답 |
| `app/agents.py` | 지시문, 도구 3개, 권한 핸들러, 공유 Copilot 클라이언트, 워크플로 구성, 스트리밍 실행, 재시도, 응답 해석 |
| `app/guard.py` | 사전 점검: 개인정보 마스킹 호출, 인젝션 신호 8종 탐지, 에이전트용 점검 메모 생성 |
| `app/pii.py` | 주민등록번호·카드번호·전화번호·이메일·계좌번호 마스킹 |
| `app/grounding.py` | 답장 문장별 사실 추출과 근거 대조 |
| `app/policies.py` | 정책 조항 10개(REFUND-1~4, EXCHANGE-1~2, SHIP-1~3, CS-1)와 검색 |
| `app/storage.py` | 저장소 인터페이스, `TableRepository`(Azure), `InMemoryRepository`(로컬·테스트), 샘플 주문 |
| `app/secrets.py` | Key Vault 비밀 읽기 (재시도 3회), 로컬에서는 환경변수 사용 |
| `app/metrics.py` | 유형별 평균 처리 시간과 직접 처리 기준값 비교 |
| `app/config.py` | 환경변수 설정, PRD 4장 실측 기준값 |
| `templates/`, `static/` | 화면 템플릿 8개, SSE 클라이언트(`app.js`), 반응형·다크 모드 스타일 |
| `infra/main.bicep` | Azure 인프라 전체 정의 |
| `scripts/deploy.ps1` | Bicep 배포 스크립트 |
| `tests/` | pytest 테스트 43개 |
| `.github/workflows/` | `ci.yml`(테스트·Bicep 컴파일), `build.yml`(테스트 통과 후 이미지 빌드·푸시) |

## 4. 기술 스택

| 구분 | 내용 |
|---|---|
| 언어 | Python 3.11 |
| 웹 | FastAPI, Jinja2, SSE(`text/event-stream`), 바닐라 JavaScript |
| 에이전트 프레임워크 | Microsoft Agent Framework (`agent-framework-core` 1.23.0, `agent-framework-orchestrations` 1.4.0, `agent-framework-github-copilot` 2.1.0) |
| 모델 연결 | GitHub Copilot SDK (`github-copilot-sdk` 1.0.26), Copilot CLI (Node.js 22) |
| 데이터 | Azure Table Storage |
| 비밀 정보 | Azure Key Vault (RBAC 모드) |
| 인증 | 시스템 할당 Managed Identity |
| 관측 | Application Insights(`azure-monitor-opentelemetry`), Log Analytics, 메트릭 경보 |
| 인프라 | Bicep, PowerShell 배포 스크립트 |
| CI/CD | GitHub Actions, GitHub Container Registry |
| 테스트 | pytest, httpx |

## 5. 화면과 API

| 메서드 | 경로 | 기능 |
|---|---|---|
| GET | `/` | 문의 입력, 예시 문의 버튼, 진행 상황 패널, 최근 문의 |
| GET | `/api/orders/{order_number}` | 주문 요약 미리보기 (JSON, 연락처 가림) |
| POST | `/api/inquiries/stream` | 문의 처리 SSE 스트림 |
| POST | `/inquiries` | 문의 처리 (자바스크립트 없는 폼 제출) |
| GET | `/inquiries/{id}` | 문의 상세: 점검 결과, 답장 편집, 문장별 근거·검증, 확정, 환불 승인/반려 |
| POST | `/inquiries/{id}/confirm` | 수정한 답장을 다시 검증한 뒤 확정 |
| POST | `/inquiries/{id}/refund/approve` | 환불 승인 |
| POST | `/inquiries/{id}/refund/reject` | 환불 반려 |
| GET | `/history?status=` | 문의 목록 (불만도 순, 상태 필터) |
| GET | `/dashboard` | 생산성 대시보드 |
| GET | `/about` | 작동 방식, 데이터 처리, 한계 안내 |
| POST | `/demo/reset` | 내 브라우저의 데모 데이터 초기화 |
| GET, POST | `/admin` | 전체 데모 데이터 정리 (관리자 비밀번호) |
| GET | `/healthz` | 상태 확인 (App Service 헬스 체크) |
| GET | `/readyz` | 준비 상태: Storage 연결, AI 설정 |

### 5.1 SSE 이벤트

| `type` | 내용 | 화면 표시 |
|---|---|---|
| `step` | 단계(screening, triage, investigator, writer, verify)와 상태(running, done) | 진행 상황 목록 |
| `tool` | 호출한 도구 이름 | 도구 호출 목록 |
| `token` | 에이전트가 쓴 텍스트 | 답장 작성 과정 |
| `warning` | 개인정보 가림, 인젝션 신호, 재시도 안내 | 경고 목록 |
| `result` | 저장된 문의 ID | 상세 화면으로 이동 |
| `error` | 사용자용 오류 문구 | 오류 표시, 입력값 유지 |
| `done` | 스트림 종료 | 버튼 다시 활성화 |

### 5.2 입력 검증과 오류 응답

| 상황 | 상태 코드 | 안내 |
|---|:-:|---|
| 주문번호 형식 오류 또는 없는 주문 | 400 | 샘플 주문번호 범위 안내 |
| 문의 내용 없음 / 2,000자 초과 | 400 | 입력 조건 안내 |
| 시간당 처리 한도(8건) 초과 | 429 | 한도 안내 |
| AI 설정 준비 안 됨 | 503 | 잠시 후 다시 시도 |
| 근거 없음 문장을 확인하지 않고 확정 | 400 | 확인 체크 요구 |
| 환불 대기가 아닌 문의의 승인·반려 | 409 | 상태 안내 |
| 다른 브라우저의 문의 접근 | 404 | 문의를 찾을 수 없음 |
| 관리자 비밀번호 불일치 | 403 | 비밀번호 오류 |

오류가 나도 입력한 주문번호와 문의 내용은 화면에 그대로 남는다.

## 6. 에이전트 워크플로

### 6.1 구성

`SequentialBuilder(participants=[ScreeningExecutor, triage, investigator, writer], intermediate_output_from=["triage", "investigator"]).build()`로 워크플로 하나를 만들고 `workflow.run(prompt, stream=True)`로 실행한다.

| 순서 | 참가자 | 종류 | 도구 | 역할 | 출력 |
|:-:|---|---|---|---|---|
| 1 | `screening` | 코드 실행기(`Executor`) | | 사전 점검 결과(가린 개인정보 종류, 인젝션 신호)를 대화에 덧붙임 | 점검 메모 |
| 2 | `triage` | GitHubCopilotAgent | 없음 | 유형(배송/환불/교환/기타), 불만도(1~5), 핵심 요청 판단 | 판단 이유 + JSON |
| 3 | `investigator` | GitHubCopilotAgent | 3개 | 주문·정책·이전 문의를 조회하고 확인된 사실만 `[근거ID] 사실` 형식으로 정리 | 사실 목록 |
| 4 | `writer` | GitHubCopilotAgent | 없음 | 확인된 사실로만 답장 작성, 문장별 근거 ID, 환불 필요 여부 제안 | 답장 + JSON |

에이전트마다 역할에 필요한 도구만 준다. 분류와 답변 에이전트는 도구를 쓰지 않는다.

### 6.2 공유 Copilot 클라이언트

- `CopilotClientHolder`가 앱 전체에서 `CopilotClient` 하나를 공유한다. 처음 쓸 때 시작하고, 연결 오류가 나면 종료 후 다시 만든다.
- 에이전트별 옵션: `on_permission_request`(권한 핸들러), `timeout`(90초), `model`(`COPILOT_MODEL` 설정 시).
- 동시 실행은 세마포어로 2건까지 허용한다.

### 6.3 도구

도구는 요청마다 현재 문의의 주문번호와 세션에 묶어서 만든다.

| 도구 | 반환 근거 ID | 동작 |
|---|---|---|
| `get_order(order_number)` | `ORDER` | 현재 문의의 주문만 조회. 다른 주문번호는 오류로 돌려줌. 이름·연락처는 가림. 금액 문자열, 오늘 날짜, 주문·배송완료 후 경과 일수를 함께 반환 |
| `search_policy(query, category)` | 조항 ID | 정책 조항 검색. 응대 원칙 CS-1은 항상 포함 |
| `get_inquiry_history(order_number)` | `HISTORY` | 같은 세션·같은 주문의 이전 문의 최대 5건 |

도구 결과는 모두 근거 목록에 기록되고, 근거 검증과 상세 화면 표시에 쓰인다.

### 6.4 권한 제어

`permission_handler`는 위 도구 3개의 호출만 `PermissionDecisionApproveOnce`로 허용한다. 셸 실행, 파일 읽기·쓰기 등 나머지 요청은 `PermissionDecisionReject`로 거부하고 경고 로그를 남긴다.

### 6.5 응답 해석

- `extract_json`은 ```json 블록을 먼저 찾고, 없으면 본문의 마지막 JSON 객체를 쓴다.
- 분류 결과: 유형이 네 가지 밖이면 `기타`, 불만도는 1~5 범위로 맞춘다.
- 답변 결과: `reply`가 비어 있으면 해석 실패로 처리한다. `needs_refund`는 불리언 `true` 또는 문자열 `"true"`일 때만 참으로 읽는다. `"false"` 같은 문자열은 환불 제안으로 처리하지 않는다.

### 6.6 실행 보증

- 조사 에이전트가 `get_order`를 호출하지 않았으면 답장을 저장하지 않고 오류로 처리한다.
- `search_policy`를 호출하지 않았으면 검증 결과에 "정책 문서를 참고하지 않은 답변" 경고를 추가한다.
- 전체 실행 시간은 180초로 제한한다.
- 연결 오류(`AgentException`, `ConnectionError`, `OSError`, `RuntimeError`)가 나면, 화면에 아직 텍스트가 나오지 않은 경우에 한해 클라이언트를 다시 만들고 한 번 재시도한다.
- 예상하지 못한 오류도 사용자에게는 안내 문구만 보내고, 상세 내용은 로그에만 남긴다.

## 7. 근거 검증 (환각 표시)

1. 답장을 문장으로 나눈다.
2. 문장마다 확인 가능한 사실을 뽑는다: 금액, 날짜(`YYYY-MM-DD`, `N월 N일`), 영업일, 일수, 주문번호, 배송 상태, 정책 조항.
3. 각 사실이 도구 결과에 있는지 대조한다. `10월 26일`은 `-10-26`으로도 대조한다.
4. 환불이 승인되지 않았는데 "환불이 완료되었습니다"처럼 쓴 문장도 표시한다.
5. 문장 상태는 `supported`(근거 있음), `unsupported`(근거 없음), `general`(확인할 사실 없음)이다.
6. 운영자가 답장을 고쳐서 확정할 때 다시 검증한다. 근거 없음 문장이 있으면 확인 체크를 해야 확정된다.

## 8. 사람의 결정

| 동작 | 처리 |
|---|---|
| AI 환불 제안 | 문의와 주문을 "환불 대기"로만 표시. AI는 환불을 실행하지 않음 |
| 환불 승인 | 서버에서 현재 상태가 "환불 대기"인지 다시 확인한 뒤 "환불완료"로 변경 |
| 환불 반려 | 같은 확인 후 "반려"로 변경, 주문은 "없음"으로 되돌림 |
| 답장 확정 | 수정한 답장을 재검증, 상태를 "확정"으로 변경 |

승인·반려·확정 시점에 처리 시간(문의 생성부터 결정까지 + AI 처리 시간)을 기록한다.

## 9. 데이터 구조 (Azure Table Storage)

모든 데이터는 브라우저 세션 ID(PartitionKey)로 나뉜다. 다른 브라우저의 문의는 조회되지 않는다.

### 9.1 `orders` 테이블

샘플 주문 20건은 코드에서 만들고(주문일은 오늘 기준 상대 날짜), 테이블에는 세션별 환불 상태 변경만 저장한다.

| 키·필드 | 내용 |
|---|---|
| PartitionKey | 세션 ID |
| RowKey | 주문번호 |
| `refund_status` | 없음 / 환불 대기 / 환불완료 |

### 9.2 `inquiries` 테이블

| 키·필드 | 내용 |
|---|---|
| PartitionKey / RowKey | 세션 ID / 문의 ID |
| `order_number`, `inquiry_masked` | 주문번호, 개인정보를 가린 문의 |
| `category`, `anger`, `summary` | 분류 결과 |
| `reply_draft`, `reply_final` | 초안, 확정본 |
| `needs_refund`, `refund_reason`, `refund_status` | 환불 제안과 상태 |
| `status` | 초안 / 확정 |
| `citations`, `checks`, `evidence`, `screening` | 문장별 근거, 검증 결과, 도구 결과, 점검 결과 (JSON 문자열) |
| `ai_seconds`, `handling_seconds`, `created_at`, `decided_at` | 처리 시간 기록 |

Table Storage는 빈 값을 저장하지 않으므로, 읽을 때 기본값으로 빈 필드를 채운다. 문의는 24시간 뒤 자동 삭제한다(앱 시작 시, 이후 10분마다 확인).

## 10. Azure 인프라

`infra/main.bicep` 하나로 전체를 만든다.

| 리소스 | 설정 |
|---|---|
| App Service Plan | Linux, B1 |
| Web App | GHCR 컨테이너 이미지, HTTPS 전용, TLS 1.2 이상, FTPS 꺼짐, Always On, 헬스 체크 `/healthz`, 시스템 할당 Managed Identity |
| Storage Account | Standard_LRS, 공유 키 접근 꺼짐(`allowSharedKeyAccess: false`), `orders`·`inquiries` 테이블 |
| Key Vault | RBAC 권한 모델, 소프트 삭제 7일 |
| 역할 할당 | 웹앱 → Storage Table Data Contributor, Key Vault Secrets User (배포자에게는 비밀 등록용 Secrets Officer 선택 부여) |
| Log Analytics | 보존 30일, 일일 수집 1GB 상한 |
| Application Insights | Log Analytics 연동 |
| 진단 설정 | 웹앱 HTTP 로그·콘솔 로그·메트릭을 Log Analytics로 전송 |
| 경보 | 5분 동안 5xx 응답 5건 이상, 헬스 체크 상태 100% 미만. 이메일 지정 시 Action Group으로 알림 |
| 예산 | 리소스 그룹 월 예산(기본 30달러) 80% 도달 시 알림 |

앱 설정에는 비밀 값이 없고 주소만 있다: `KEY_VAULT_URL`, `STORAGE_TABLE_URL`, `APPLICATIONINSIGHTS_CONNECTION_STRING`, `COPILOT_MODEL`, `WEBSITES_PORT`.

## 11. 보안과 개인정보

| 항목 | 구현 |
|---|---|
| 비밀 값 | `GITHUB-TOKEN`, `ADMIN-PASSWORD`는 Key Vault에만 저장. 앱 시작 시 Managed Identity로 읽음 (실패 시 3회 재시도) |
| 개인정보 | 주민등록번호·카드번호·전화번호·이메일·계좌번호를 AI 전달 전에 가리고, 가린 문의만 저장 |
| 도구 결과 | 고객 이름과 연락처를 가려서 반환 |
| 로그 | 도구 이름과 오류 종류만 남기고, 문의 본문과 AI 응답 원문은 남기지 않음 |
| 프롬프트 인젝션 | 문의를 `<customer_inquiry>` 태그 안의 데이터로만 다루도록 모든 에이전트에 지시. 사전 점검이 8종 신호(지시 무시 요구, 역할 변경, 시스템 프롬프트 요구, 처리 직접 지시, 관리자 사칭, 명령 실행 요구, 태그 삽입 등)를 탐지해 경고하고 에이전트에도 알림. 꺾쇠괄호는 다른 문자로 바꿔 태그 탈출을 막음 |
| 세션 분리 | 브라우저마다 무작위 세션 ID 쿠키(`HttpOnly`, `SameSite=Lax`, HTTPS에서 `Secure`). 문의·주문 상태는 세션별로 저장 |
| 남용 방지 | 세션당 시간당 8건, 동시 실행 2건 |
| 보안 헤더 | CSP(`script-src 'self'`, 인라인 스크립트 없음), `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy` |
| 관리 기능 | 관리자 비밀번호는 상수 시간 비교(`hmac.compare_digest`) |
| AI 표시 | 답장에 "AI가 작성한 초안" 표시, 작동 방식 화면에 한계 안내 |

## 12. 화면 처리

- 진행 상황 패널이 단계 5개(사전 점검 → 분류 → 조사 → 답장 작성 → 근거 검증)의 상태, 호출한 도구, 경고, 답장 작성 과정을 실시간으로 보여 준다.
- 주문번호를 입력하면 주문 요약을 미리 보여 준다.
- 예시 문의 버튼 3개로 바로 시험할 수 있다.
- 상세 화면에서 근거 없음 문장을 강조하고, 문장별 근거 ID와 도구 결과를 펼쳐 볼 수 있다.
- 자바스크립트가 꺼져 있으면 일반 폼 제출로 같은 처리를 한다.
- 반응형 레이아웃, 다크 모드, 키보드 조작, `aria-live` 영역을 지원한다.

## 13. 테스트와 CI/CD

pytest 테스트 43개. 실제 Copilot을 호출하지 않고 가짜 에이전트로 실제 Agent Framework 워크플로를 실행한다.

| 파일 | 검증 내용 |
|---|---|
| `tests/test_workflow.py` | 워크플로 단계 순서, 에이전트별 도구·점검 메모 전달, 다른 주문 조회 거부, 권한 핸들러, `get_order` 미호출·답변 JSON 누락 시 오류, 근거 없음 문장 표시, 환불 플래그 엄격 해석 |
| `tests/test_api.py` | SSE 응답, 폼 제출, 다른 브라우저 접근 차단, 확정 시 재검증·확인 체크, 환불 승인·반려와 중복 승인 거부(409), 연락처 미저장, 인젝션 경고, 입력 검증, 횟수 제한, 보안 헤더 |
| `tests/test_grounding.py` | 사실 추출, 근거 대조, 날짜 표기 대조, 미승인 환불 완료 문장, 처리 시간 표시, Table 빈 필드 기본값 |
| `tests/test_guard.py` | 개인정보 마스킹, 인젝션 신호 탐지 |

| 워크플로 | 내용 |
|---|---|
| `ci.yml` | 푸시·PR마다 테스트, Bicep 컴파일 |
| `build.yml` | 테스트 통과 후 이미지를 GHCR에 `latest`와 커밋 해시 태그로 푸시 |

## 14. 배포

1. GitHub Actions가 이미지를 GHCR에 올린다.
2. `scripts/deploy.ps1`이 리소스 그룹을 만들고 `infra/main.bicep`을 배포한다. 매개변수는 이미지, 지역, 경보 이메일, 모델, 예산 사용 여부.
3. Key Vault에 `GITHUB-TOKEN`, `ADMIN-PASSWORD`를 등록하고 앱을 다시 시작한다.
4. `/readyz`가 `{"storage":"ok","ai":"ok"}`를 돌려주면 준비 완료다.

배포 링크: https://app-shopreply-zj2ite.azurewebsites.net

## 15. 로컬 실행

```powershell
pip install -r requirements-dev.txt
npm install -g @github/copilot
$env:GITHUB_TOKEN="..."; $env:ADMIN_PASSWORD="..."
uvicorn app.main:app --reload
```

`KEY_VAULT_URL`, `STORAGE_TABLE_URL`이 없으면 환경변수 비밀과 메모리 저장소로 실행된다.

## 16. 한계

- App Service B1 인스턴스 하나와 단일 지역으로 운영한다. 비용을 고려한 데모 구성이며 자동 확장은 없다.
- 문의 1건에 에이전트 3개가 차례로 실행되어 30초~1분 걸린다. 첫 요청에는 Copilot 연결 준비 시간이 더해진다.
- 샘플 주문만 지원하며 실제 쇼핑몰·결제·발송 연동은 없다.
- 근거 검증은 금액·날짜·상태 같은 확인 가능한 사실만 대조하며, 문장의 의미까지 판단하지는 않는다.
