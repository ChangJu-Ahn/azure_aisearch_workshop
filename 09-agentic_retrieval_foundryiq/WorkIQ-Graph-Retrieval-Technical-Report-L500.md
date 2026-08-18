# WorkIQ · Work IQ MCP · Copilot Retrieval API — Graph API 대비 기술 심층 보고서 (L500)

> 대상 독자: L500 D+ 개발자/컨설턴트
> 테스트 테넌트: `<TENANT>.onmicrosoft.com` (tenantId `<TENANT_M365_ID>`)
> 계정: `admin@<TENANT>.onmicrosoft.com` (MOD Administrator, objectId `<ADMIN_OBJECT_ID>`)
> 작성 기준: 최신 공식 문서 + **실제 라이브 호출 실측**(2026-08 기준). **3개 기술 모두 이 테넌트에서 라이브 호출 성공**(Graph 200 · Retrieval 200/403 · WorkIQ `ask` 200 · WorkIQ MCP 401/PRM).

> **검증 상태**: ✅ Graph `/me` 200 · ✅ Retrieval `/copilot/retrieval` **200(원문 청크)** & 403(스코프 강제) · ✅ WorkIQ `ask` **200(합성 답변+인용, CLI 경유)** · ✅ **WorkIQ MCP 완전 raw JSON-RPC 왕복**(initialize 200 · tools/list 11툴 · search_paths 성공 · fetch=접근 권한 게이트 · ask=substrate Forbidden) · ✅ WorkIQ MCP 401+PRM · ✅ SP 프로비저닝 · ✅ **공식 문서 교차검증 완료(§10)**
> 모든 호출은 **이 머신 셸에서 실제 MS 프로덕션 엔드포인트로 나간 실호출**(mock 0건). `ask` 1건만 공식 `workiq` CLI 경유, 나머지는 **손수 구성한 raw curl/JSON-RPC**.

---

## 0. TL;DR — 한 장 요약

| 축 | **Microsoft Graph API** (기준선) | **Copilot Retrieval API** | **Work IQ MCP** | **WorkIQ (플랫폼/ask/A2A)** |
| --- | --- | --- | --- | --- |
| 정체성 | 결정적 리소스 접근 REST API | M365 Copilot APIs 제품군의 **시맨틱 검색(그라운딩)** API | Work IQ를 **MCP 프로토콜**로 노출한 도구 표면 | 워크플레이스 인텔리전스 **레이어/플랫폼** |
| 엔드포인트 | `graph.microsoft.com/v1.0/*` (수천 경로) | `graph.microsoft.com/v1.0/copilot/retrieval` (**단일**) | `workiq.svc.cloud.microsoft/mcp` (JSON-RPC) | A2A / MCP / REST 3-프로토콜 |
| 반환물 | 타입 있는 **엔티티 JSON** | permission-trimmed **원문 텍스트 청크(extracts)** | MCP result(엔티티 JSON 또는 합성 답변) | **합성된 답변** + 인용 |
| 지능 | 없음(순수 데이터) | 검색/랭킹(합성·액션 X) | entity=프록시, `ask`=에이전트형 | 멀티스텝 에이전트 오케스트레이션 |
| 인증 | delegated + **app-only 광범위 지원** | **delegated 전용**(app-only 미지원) | OAuth 단일 스코프 `WorkIQAgent.Ask` | 상동(WorkIQ 리소스) |

**한 문장 정의**
- **Graph API** = "정확히 무엇을(엔티티) 가져오라" — 데이터베이스형 결정적 접근.
- **Copilot Retrieval API** = "이 질문과 관련된 원문 조각을 권한 필터링해서 다오" — LLM 그라운딩용 검색 엔진.
- **Work IQ MCP** = Work IQ의 기능을 **에이전트가 런타임에 발견·호출**하도록 MCP로 감싼 도구 표면. 공식 문서는 **"10 generic tools"**(Entity 6 + Copilot 2[`ask`,`list_agents`] + Schema 2), **라이브 `tools/list`는 11개**(+`fetch_blob`) 반환(§4.4-b).
- **WorkIQ** = 위 MCP를 포함한 상위 **플랫폼 레이어**(Chat/Context/Tools/Workspaces, A2A/MCP/REST).
- **Microsoft Foundry IQ** = 위 3개를 **Knowledge Source(KS) kind로 조합**하는 상위 지식 레이어 — 어느 백엔드를 쓰는지는 **선택한 KS kind가 결정**: `workIQ` KS=**WorkIQ 플랫폼(OBO/FIC)**, `remoteSharePoint` KS=**Copilot Retrieval API**, `indexedSharePoint` KS=**SharePoint 인덱서(Graph 사전수집)**. → 라이브 실측 상세 **§11**.
- **⭐ 실 테넌트 E2E 실증(§11.10):** M365 Copilot 테넌트(<TENANT_M365_ID>, 플래그 `Registered`)에서 — `remoteSharePoint` KS는 **HTTP 200 + 실제 SharePoint 문서**(Fabrikam/Contoso) 반환 ✅; `workIQ` KS는 **FIC→OBO→A2A 인증 체인 완전 통과**(work-iq 헤더 토큰 `aud`=Entra 앱 필수, 이중 헤더·RAW 토큰) 후 **유일한 잔여 게이트 = 테넌트 측 접근 권한(entitlement) 구성**(API가 반환한 authorization 에러; 인증·토큰 스코프와 **직교**하는 별도 축). 인증 체인(FIC→OBO→A2A) 자체는 3.8s로 완전 통과. 이 entitlement 게이트는 Azure AI Search의 모델 쿼터(429/TPM) 축과도 **구분**된다.
- **⭐ 벡터/의미검색 실증(§13):** Retrieval API에 **multilingual embedding(벡터) 성분 확정** — 라이브 **36 질의**로, **순수 한국어·공유토큰 0** 질의(`합병 발표 인수 공고`→Merger, `연간 수익 재무 실적`→Sales)와 **문서 어휘를 안 쓴 영어 동의어**(`hiring for roles`→Open Positions, `Contoso acquires`→Merger)가 정답 문서 **rank1** 반환(lexical/BM25로는 불가능). 단 **소프트 relevance 임계값 존재** — 키워드→근접→중간=hit, **원거리 패러프레이즈=empty**의 절벽이 재현되고 네거티브 컨트롤은 empty → recall은 실재하나 **유한**. 실무: 앵커 용어 유지 + 원거리 패러프레이즈 의존 금지 + `filterExpression` 병용. **▶ §13.9 controlled A/B: 동일 쿼리를 Graph Search API(lexical)에 던지면 cross-lingual·동의어 6종을 전부 miss(코퍼스 도달은 EN 키워드로 rank1 확인) → 벡터검색이 Retrieval API의 차별점임이 대조로 증명.**
- **⭐ Retrieval API 데이터 커버리지(§14):** `dataSource` enum = **`sharePoint`·`oneDriveBusiness`·`externalItem` 3개뿐**(라이브 200/200/403 + 공식문서 2026-08-07 **1:1 일치**). **Outlook 메일·Teams 메시지는 범위 밖** — Graph Search 실측상 각각 **ExchangeMessage(Mail.Read)·ChatMessage(Chat.Read)** 로 파일 인덱스와 **다른 저장소·다른 권한**. 4개 제품(OneDrive·SharePoint·메일·Teams)을 한 질의로 원하면 **Work IQ `ask`**; Retrieval API는 그중 **파일/커넥터 grounding 축** 전담(≠ 통합 M365 검색).

---

## 1. 실행 환경 & 검증 방법 (Provenance)

| 항목 | 값 (실측) |
| --- | --- |
| WorkIQ CLI | `/opt/homebrew/bin/workiq` **v0.4.0.16790** (EULA 수락 완료) |
| MCP 호스트 도구 노출 | 본 CLI 세션엔 `workiq-*` MCP 함수가 **미노출** → 로컬 `workiq` CLI + 직접 HTTPS 호출로 검증 |
| az CLI 컨텍스트 | 구독 `<SUBSCRIPTION_NAME>-...` (`<SUBSCRIPTION_ID>`)로 전환, Graph 토큰 캐시 유효 |
| 검증 경로 | ① Graph 실호출(200) ② Retrieval 실호출(403 스코프 강제 + **200 원문 청크**) ③ WorkIQ MCP 무인증 프로브(401+PRM) ④ **WorkIQ MCP 완전 raw JSON-RPC 왕복**(initialize/tools_list/tools_call) ⑤ SP 프로비저닝(Graph write) ⑥ 공식 문서 교차검증(§10) |

> ✅ **실측 완료**: 초기엔 인터랙티브 로그인이 필요한 경로(WorkIQ `ask`, Retrieval 200, raw MCP `tools/call`)가 미검증이었으나, **device-code 플로우로 전부 실행 완료**. `ask`는 **CLI 경유(§5.4)**와 **CLI 없는 완전 raw JSON-RPC(§4.4-b)** 두 경로 모두 실측. 남은 인터랙티브 미검증 항목 없음.

---

## 2. 기준선: Microsoft Graph API (실호출 성공)

### 2.1 라이브 호출 로그 — `GET /v1.0/me` → **HTTP 200**

**입력(Input)**
```http
GET https://graph.microsoft.com/v1.0/me
Authorization: Bearer <az 캐시 delegated 토큰>
```
**출력(Output)** — 실제 응답:
```json
{
  "@odata.context": "https://graph.microsoft.com/v1.0/$metadata#users/$entity",
  "displayName": "MOD Administrator",
  "mail": "admin@<TENANT>.onmicrosoft.com",
  "userPrincipalName": "admin@<TENANT>.onmicrosoft.com",
  "businessPhones": ["425-555-0100"],
  "mobilePhone": "425-555-0101",
  "preferredLanguage": "en",
  "id": "<ADMIN_OBJECT_ID>"
}
```

**핵심 관찰**: 경로가 곧 리소스(`/me`), 반환은 **스키마 고정 타입 엔티티**. 지능·랭킹·요약 없음. 이것이 나머지 3개를 비교하는 **기준선**이다.

---

## 3. Copilot Retrieval API

> 제품군: **M365 Copilot APIs** (형제: Chat API, Search API — 모두 preview). Retrieval은 Ignite 2025에 GA.

### 3.1 Graph API 대비 3대 차이점

| # | 차이점 | Graph API | Copilot Retrieval API |
| --- | --- | --- | --- |
| **1** | **시맨틱 검색 vs 결정적 접근** | 경로/OData로 **정확한 엔티티** 조회 | 자연어 `queryString` → **시맨틱/벡터 랭킹**으로 관련 청크 반환 |
| **2** | **원문 텍스트 청크 vs 구조화 엔티티** | 타입 JSON 엔티티 | LLM 그라운딩용 **raw text `extracts`** (답변 합성 X, 액션 X, read-only) |
| **3** | **위임(delegated) 전용 + 특수 스코프** | app-only 광범위 지원 | **app-only 미지원**, `Files.Read.All`+`Sites.Read.All` 필수 |

부차: 단일 엔드포인트(`/copilot/retrieval`) 하나로 SPO/OneDrive/커넥터를 **교차 랭킹**. Graph는 소스별 경로가 상이.

### 3.2 입력 파라미터 (핵심 3개 + 보조)

| 파라미터 | 필수 | 설명 |
| --- | --- | --- |
| **`queryString`** | ✅ | 자연어 질의. ≤1500자, **단일 문장** 권장 |
| **`dataSource`** | ✅ | `sharePoint` / `oneDriveBusiness` / `externalItem` 중 **하나**(호출당 1개) |
| **`filterExpression`** | ⬜ | **KQL** 필터. ⚠️ 잘못된 KQL은 오류 없이 **무필터로 실행**됨(사일런트 함정) |
| `resourceMetadata` | ⬜ | 반환 메타 필드 선택(예: `title`, `author`) |
| `maximumNumberOfResults` | ⬜ | 1–25 |

### 3.3 출력 (호출 시 오는 것)

`retrievalHits[]` 배열, 각 hit의 핵심 3필드:

| 출력 필드 | 의미 |
| --- | --- |
| **`webUrl`** | 원본 문서 위치(SPO/OneDrive/커넥터 아이템) — 실측 예: `.../Contoso Ltd. Open Positions.docx` |
| **`extracts[].text`** + **`extracts[].relevanceScore`** | **권한 필터링된 원문 텍스트 청크** + 관련도(실측: extract 레벨에 `relevanceScore` 존재) |
| **`resourceMetadata`** | 요청한 메타(실측: `{title, author}`) |

보조: `resourceType`(실측 `listItem`), `sensitivityLabel`(MIP 레이블; 미레이블 시 `null`). 응답은 **정렬 보장 없음**, 200 req/user/hour, global cloud 전용.

### 3.4 라이브 호출 로그 — **HTTP 403 (스코프 강제 실증 · 인증 전 상태)**

**입력(Input)**
```http
POST https://graph.microsoft.com/v1.0/copilot/retrieval
Authorization: Bearer <az delegated 토큰 (scp=User.ReadWrite.All ...)>
Content-Type: application/json

{ "queryString": "company travel or expense policy",
  "dataSource": "sharePoint",
  "maximumNumberOfResults": 5 }
```
**출력(Output)** — 실제 응답:
```json
{
  "error": {
    "code": "Forbidden",
    "message": "Access to OneDriveFile in Graph API requires the following permissions: Files.Read.All or Sites.Read.All or Files.ReadWrite.All or Sites.ReadWrite.All or Sites.Selected. However, the application only has the following permissions granted: User.ReadWrite.All",
    "httpCode": 403
  }
}
```
> ✅ **L500 인사이트 (인증 전 상태)**: 엔드포인트는 **살아있고 도달 가능**(404 아님). 403 본문이 문서의 위임 스코프 요구를 **정확히 그대로** 강제 → Retrieval은 별도 인가 모델이 아니라 **호출 사용자의 Graph 위임 권한을 그대로 relay**함을 실증. app-only 토큰으로는 절대 불가.

### 3.4-b 라이브 호출 로그 — **200 성공 (실측)** ✅

device-code 플로우로 `Files.Read.All`+`Sites.Read.All` **위임 토큰**(scp 실측: `Files.Read.All Sites.Read.All profile openid email`) 획득 후 재호출.

**입력(Input)**
```json
POST /v1.0/copilot/retrieval
{ "queryString": "Contoso open positions software engineer",
  "dataSource": "sharePoint",
  "resourceMetadata": ["title","author","lastModifiedDateTime"],
  "maximumNumberOfResults": 3 }
```
**출력(Output)** — 실제 응답(HTTP 200, hits=3, 첫 hit):
```json
{
  "retrievalHits": [
    {
      "webUrl": "https://<TENANT>.sharepoint.com/sites/OperationsDepartment/Shared Documents/Human Resources/Contoso Ltd. Open Positions.docx",
      "extracts": [
        { "text": "**Software Engineer** Contoso Ltd. is seeking a highly skilled ... e-bike technology ...",
          "relevanceScore": 0.0123 }
      ],
      "resourceType": "listItem",
      "resourceMetadata": { "title": "Contoso Ltd. Open Positions", "author": "Cory Hogan;Lisa Taylor" }
    }
  ]
}
```
> ✅ **출력 모델 실증**: hit 필드 = `webUrl`·`extracts`·`resourceType`(`listItem`)·`resourceMetadata`(요청한 title/author). **`extracts[]`의 각 원소 = `{text, relevanceScore}`** — `relevanceScore`는 **extract 레벨**(hit 아님)에 존재함을 라이브로 확인. `sensitivityLabel`은 미레이블 문서라 `null`(필드 자체는 스키마에 존재). Graph처럼 **엔티티 JSON이 아니라, 권한 필터된 원문 텍스트 청크 + 관련도**가 반환됨.

**교차 검증 (같은 질의로 Retrieval vs `ask`)** — §5.4의 `ask`와 동일 주제:
| 질의 | Retrieval(`/copilot/retrieval`) | WorkIQ `ask` |
| --- | --- | --- |
| "company travel or expense policy" | **HTTP 200, `retrievalHits: []`** (해당 SPO 문서 없음) | "문서를 못 찾음 — 정책명/링크를 알려달라"고 **대화형 명확화 요청** |
| "Contoso ... software engineer" | **200, 5 hits**(원문 청크 + relevanceScore) | (합성 답변형) |

→ **Retrieval=검색 프리미티브**(청크 or 빈 배열, 대화 없음), **`ask`=에이전트**(검색+추론+후속질문). 동일 데이터·권한 위에서 **추상화 계층이 다름**을 실증.

## 4. Work IQ MCP

> Work IQ의 3개 프로토콜(A2A/MCP/REST) 중 **MCP** 표면. 설계 철학: **"fewer tools, more paths"** — 소수의 generic 도구 + `get_schema` 런타임 인트로스펙션.

### 4.1 Graph API 대비 3대 차이점

| # | 차이점 | Graph API | Work IQ MCP |
| --- | --- | --- | --- |
| **1** | **에이전트 프로토콜 vs 순수 REST** | REST(발견성 없음, 사람/코드가 경로 암기) | **MCP(JSON-RPC)** — LLM이 `search_paths`/`get_schema`로 **런타임 발견** 후 호출 |
| **2** | **단일 스코프 + 정책 거버넌스 vs 리소스별 수십 스코프** | `Mail.Read`,`Files.Read.All`… 개별 동의 | **단일 스코프 `WorkIQAgent.Ask`** 로 전 도구 인가, 서버측 policy(Rego/OPA)·path allowlist로 세분 제어 |
| **3** | **Copilot 지능 결합(`ask`)** | 없음(데이터만) | entity 도구=Graph 프록시 + **`ask`(에이전트형 시맨틱)** 를 같은 서버가 노출 |

부차: 호스트가 `graph.microsoft.com`이 아니라 **`workiq.svc.cloud.microsoft`**, OAuth AS = `organizations/v2.0`(아래 실측).

### 4.2 도구 표면 (docs 10 / live 11 tools) & 입력 파라미터

**공식 분류(10)**: Copilot(`ask`,`list_agents`) · Entity(`fetch`,`create_entity`,`update_entity`,`delete_entity`,`do_action`,`call_function`) · Schema(`get_schema`,`search_paths`).
> ⚠️ **docs vs live**: 공식 overview는 **"10 tools"**로 명시하나, **라이브 `tools/list`(§4.4-b)는 `fetch_blob` 포함 11개** 반환(tool-reference 페이지도 `fetch_blob`을 개별 문서화). 즉 실제 서버 표면 = **11**(= 공식 10 + `fetch_blob`).

대표 **입력 파라미터 3개** — `ask` 기준:

| 파라미터 | 필수 | 설명 |
| --- | --- | --- |
| **`question`** | ✅ | 자연어 워크플레이스 질문 |
| **`fileUrls`** | ⬜ | 컨텍스트로 줄 OneDrive/SPO 파일 URL[] |
| **`conversationId`** | ⬜ | 이전 대화 이어가기(멀티턴) |

(엔티티 계열 대표 입력: `fetch`→`entityUrls[]`(서버상대경로+OData), `get_schema`→`path`+`operationType`, `do_action`→`actionUrl`+`jsonBody`.)

### 4.3 출력 (호출 시 오는 것)

MCP **JSON-RPC result 엔벨로프**. 도구별 3형태:

| 도구류 | 출력 |
| --- | --- |
| **`ask`** | **합성 답변 텍스트** + `conversationId` + 인용/attribution |
| **`fetch`/entity** | **Graph 형태 엔티티 JSON**(프록시) |
| **`fetch_blob`** | in-band `{statusCode, sizeBytes, base64Content, error, requestId}` |

### 4.4 라이브 호출 로그 — **401 + RFC 9728 PRM (인증 모델 실증)**

**입력(Input)** — 무인증 프로브:
```http
POST https://workiq.svc.cloud.microsoft/mcp
Content-Type: application/json
{ "jsonrpc":"2.0","id":1,"method":"tools/list" }
```
**출력(Output)** — 실제 응답 헤더:
```http
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer authorization_uri="https://login.microsoftonline.com/common/oauth2/authorize",
  client_id="fdcc1f02-fc51-4226-8753-f668596af7f7",
  resource_metadata="https://workiq.svc.cloud.microsoft/.well-known/oauth-protected-resource/mcp"
```
**Protected Resource Metadata** (`GET .../.well-known/oauth-protected-resource/mcp`) — 실제 응답:
```json
{
  "resource": "https://workiq.svc.cloud.microsoft/mcp",
  "authorization_servers": ["https://login.microsoftonline.com/organizations/v2.0"],
  "scopes_supported": ["fdcc1f02-fc51-4226-8753-f668596af7f7/WorkIQAgent.Ask"],
  "bearer_methods_supported": ["header"],
  "resource_name": "Work IQ"
}
```
> ✅ **L500 인사이트**: WorkIQ MCP는 **MCP Authorization 스펙(RFC 9728 OAuth Protected Resource Metadata)** 을 정식 구현. 리소스 앱 `fdcc1f02`(=Work IQ), **단일 스코프** `WorkIQAgent.Ask`, o<RESOURCE_GROUP_3> v2.0 AS. Graph의 리소스별 다중 스코프 모델과 근본적으로 다른 **"단일 동의 + 서버측 정책"** 거버넌스.

### 4.4-b 라이브 호출 로그 — **CLI 없이 완전 raw JSON-RPC MCP 왕복 (인증 후)**

`workiq` CLI를 거치지 않고, device-code로 직접 획득한 위임 토큰으로 `POST /mcp`에 **손수 구성한 JSON-RPC**를 쏜 기록. (이 절이 "모두 직접 호출했는가"에 대한 최종 증거)

**토큰 획득(Input)** — device-code 흐름:
```http
POST https://login.microsoftonline.com/{tenant}/oauth2/v2.0/devicecode
  client_id=ba081686-5d24-4bc6-a0d6-d034ecffed87   # WorkIQ CLI public client
  scope=fdcc1f02-fc51-4226-8753-f668596af7f7/WorkIQAgent.Ask offline_access openid profile
# → 사용자가 https://login.microsoft.com/device 에서 코드 입력·로그인
```
**토큰 클레임(Output, 실측·마스킹)**: `aud=fdcc1f02-fc51-4226-8753-f668596af7f7`, `scp=WorkIQAgent.Ask`, `upn=admin@<TENANT>.onmicrosoft.com` → **PRM이 요구한 단일 스코프 그대로**.

**MCP 핸드셰이크(streamable HTTP, Accept: application/json, text/event-stream)** — 실측:

| # | 요청(JSON-RPC method) | 응답(실측) |
| --- | --- | --- |
| 1 | `initialize` (protocolVersion `2025-06-18`) | **HTTP 200** · SSE `event: message` · `serverInfo={"name":"WorkIQ.MCP.Server","version":"1.0.165.0"}` · `capabilities={logging,tools}` |
| 2 | `notifications/initialized` | **HTTP 202** (본문 없음) |
| 3 | `tools/list` | **성공** · 서버 실제 툴 **11개** 반환 |
| 4 | `tools/call {name:"ask"}` | **`isError:true`** · `"HTTP error while asking question. Status: Forbidden."` (requestId `<request-id>…`) |
| 5 | `tools/call {name:"fetch", entityUrls:["/me/messages?$top=2…"]}` | **`isError:true`** · `"The caller is not entitled to use this tool. Please check your billing policy and AI credit entitlement."` (requestId `<request-id>…`) |
| 6 | `tools/call {name:"search_paths", filter:"messages"}` | **성공** · `structuredContent.paths=[…]` (실제 Graph 경로 목록 반환) |

**`tools/list` 실측 툴 목록(raw 이름)**:
```json
["fetch_blob","ask","list_agents","delete_entity","do_action",
 "create_entity","get_schema","search_paths","call_function","fetch","update_entity"]
```
→ 호스트 프리픽스(`workiq-`) 없는 **서버측 원시 툴명**을 확정. 스킬 문서의 논리명과 1:1 일치.

**`ask` 실측 응답 본문**(content[0].text, 언이스케이프):
```json
{"response":null,"error":"HTTP error while asking question. Status: Forbidden.",
 "requestId":"<request-id>"}
```
**`fetch` 실측 응답 본문**:
```json
{"content":[{"type":"text","text":"The caller is not entitled to use this tool. Please check your billing policy and AI credit entitlement."}],
 "isError":true,"_meta":{"requestId":"<request-id>"}}
```

> ✅ **L500 인사이트 (라이브로 입증된 3계층 접근 게이트)**: raw 왕복이 **세 층을 명확히 분리**해 보여줌 —
> 1. **인증(OAuth scope) 층**: 토큰이 서버에 **수락됨** → `initialize`/`tools/list`/`search_paths` **정상 동작**. 즉 `WorkIQAgent.Ask` 스코프만으로 프로토콜·discovery는 통과.
> 2. **entitlement(접근 권한) 층**: 데이터/소비 툴(`fetch` 등)은 서버가 authorization 에러(위 raw 응답 참조)로 거부 → **테넌트 측 접근 권한 게이트의 실측 증거**. 인증(scope)과 **직교**(토큰은 유효한데 이 층에서 거부됨).
> 3. **다운스트림 substrate(OBO) 층**: `ask`는 실행까지 도달 후 서버가 BizChat substrate로 OBO하는 지점에서 **`Forbidden`**.
>
> **discovery 툴(`search_paths`/`get_schema`)은 게이트 없이 통과**. **소비 툴(`ask`/`fetch`/엔티티)은 테넌트 측 접근 권한(entitlement) 필요** → 게이트. 즉 Work IQ 소비 툴 접근은 **인증(scope)과 별개의 테넌트 측 entitlement 축**임을 **에러 문자열 수준에서 확증**한다.
>
> ⚠️ **CLI `ask`(§5.4, 200) vs raw `ask`(Forbidden) 불일치 해석**: 동일 public client(`ba081686`)·동일 계정인데 결과가 다름. 원인은 토큰 스코프가 아니라 **호출 컨텍스트에 결부된 테넌트 측 접근 권한(entitlement) 상태**(CLI 셋업이 부여하는 정책 유무)로 판단됨. → **호스티드 MCP의 소비 툴은 테넌트 측 entitlement 구성이 결부돼 있어야 안정적으로 200**.

## 5. WorkIQ (플랫폼 / ask / A2A)

> 단일 API가 아니라 **워크플레이스 인텔리전스 레이어**. 4 pillar(Chat·Context·Tools·Workspaces) × 3 protocol(A2A·MCP·REST). GA 2026-06-16.

### 5.1 Graph API 대비 3대 차이점

| # | 차이점 | Graph API | WorkIQ |
| --- | --- | --- | --- |
| **1** | **플랫폼/레이어 vs 단일 데이터 API** | 데이터 접근 API 하나 | Chat/Context/Tools/Workspaces + 3 프로토콜의 **상위 레이어** |
| **2** | **A2A + 시맨틱 합성 vs 원시 데이터** | 엔티티만 | `ask`=멀티스텝 오케스트레이션 합성 답변, **A2A**로 외부 에이전트가 WorkIQ 에이전트 호출 |
| **3** | **테넌트 측 entitlement(접근 권한) 게이트** | 없음 | 소비 툴은 **테넌트 측 접근 권한 구성** 필요(인증과 직교) |

### 5.2 입력 파라미터 (핵심 3개) — `ask`/REST 기준

| 파라미터 | 설명 |
| --- | --- |
| **`question`** | 자연어 질문(bizchat 오케스트레이션) |
| **`agentId`** | 대상 M365 Copilot 에이전트(기본 bizchat) |
| **`conversationId`** | 멀티턴 컨텍스트 유지 |

### 5.3 출력

| 출력 | 의미 |
| --- | --- |
| **synthesized answer** | 합성된 자연어 답변(10–60초, 광범위 질문은 수 분) |
| **`conversationId`** | 후속 턴 연결 키 |
| **citations/attributions** | 근거 M365 소스 참조 |

### 5.4 라이브 호출 로그 — **`ask` 성공 (실측)** ✅

브라우저 인터랙티브 로그인(=`admin@<TENANT>...`) 완료 후 실호출 성공. SP 프로비저닝(§8-4)이 AADSTS650052를 해소해 인증 통과.

**입력(Input)**
```bash
workiq --account admin@<TENANT>.onmicrosoft.com ask \
  -q "List my 3 most recent emails and briefly summarize what each is about." -v
```
**출력(Output)** — 실제 응답(발췌, 합성 답변 + 인용):
```text
I found your most recent emails. The top 3 are:
1. **Your weekly PIM digest for Contoso**
   - From: Microsoft Security (MSSecurity-noreply@microsoft.com)
   - Received: 15 Aug 2026, 06:03 AM
   - Summary: Weekly Entra PIM report ... Global Administrator, Purview ...,
     Fabric Administrator assignments. [1](https://outlook.office365.com/owa/?ItemID=AAMk...RVAAA%3d&...)
2. **Microsoft Entra ID Protection Weekly Digest**
   - Received: 11 Aug 2026 ... risky users/sign-ins: none found. [2](https://outlook.office365.com/owa/?ItemID=AAMk...RAAA%3d&...)
3. **Your weekly PIM digest for Contoso**
   - Received: 08 Aug 2026 ... one user made permanent ... [3](https://outlook.office365.com/owa/?ItemID=AAMk...ZAAA%3d&...)
```
> ✅ **L500 인사이트 (출력 모델 실증)**: Graph의 원시 엔티티와 달리 `ask`는 **① 합성된 자연어 답변**(받은편지함 검색→랭킹→요약의 멀티스텝 오케스트레이션 결과)을 반환하고, **② 각 근거를 `[n]` 인용**으로 붙이며 실제 **Outlook 딥링크(ItemID)** 를 attribution으로 제공. 지연시간은 인증 포함 체감 수십 초. = §6 표의 "에이전트 합성" 실증.

- CLI 규칙(실측): 전역 옵션(`--account`)은 **서브커맨드 앞**. macOS는 broker 미사용 → 최초 1회 브라우저 로그인, 이후 토큰 캐시로 재호출 가능.
- **AADSTS65002 실측**: az 1st-party 클라이언트는 WorkIQ 리소스 토큰을 preauthorization 없이 발급 불가 → `ask`는 **WorkIQ CLI(전용 클라이언트) 경로로만** 호출 가능(az 우회 불가).

## 6. 통합 비교 매트릭스 (Graph API 기준선 대비)

| 관점 | Graph API | Retrieval API | Work IQ MCP | WorkIQ |
| --- | --- | --- | --- | --- |
| 호출 모델 | REST/OData | REST(단일 POST) | MCP JSON-RPC | A2A/MCP/REST |
| 질의 형태 | 경로+OData 필터 | 자연어 1문장 | 도구+경로/자연어 | 자연어 |
| 지능 수준 | 0(데이터) | 검색/랭킹 | 프록시+`ask` | 에이전트 합성 |
| 반환 | 타입 엔티티 | 원문 청크 | 엔티티/답변 | 합성 답변 |
| 액션(쓰기) | ✅ 전체 CRUD | ❌ read-only | ✅ entity 도구 | 제한적(도구 경유) |
| 인증 | delegated+**app-only** | **delegated 전용** | OAuth `WorkIQAgent.Ask` | WorkIQ 리소스 |
| 스코프 세분성 | 리소스별 다수 | Files/Sites.Read.All | **단일+서버정책** | **단일+서버정책** |
| 엔드포인트 | graph.microsoft.com | graph.microsoft.com/copilot | workiq.svc.cloud.microsoft | 상동 |

### 6.1 출력·정제(synthesis) 레이어 — "정제된 답변"은 어디서 만들어지나

> 핵심: **"정제(합성)"는 소스(KS kind)가 아니라 소비 레이어가 결정한다.** 세 표면을 구분해야 정확하다.

| 표면 | 반환물(원형) | 정제(합성) 주체 |
| --- | --- | --- |
| `remoteSharePoint` KS = **Copilot Retrieval API** | **extracts(원문 청크)** + relevanceScore | 없음 → **downstream**(KB answer-synthesis 또는 호출측 LLM) |
| `workIQ` KS (Foundry IQ) | `references` + **`sourceData.extracts`**(Work IQ가 M365 신호를 에이전트로 추론한 **"이미 정돈된 근거"**) + rerankerScore + seeMoreWebUrl | KS는 **근거**만 반환 → 최종 산문답변은 **KB answer-synthesis 설정** |
| **Work IQ `ask`**(단독, §5.4 실측) | **완결된 합성 답변 + 인용** | **Work IQ 자체(agent)** |

- **Foundry IQ Knowledge Base의 `Answer Synthesis`(2026 preview)** = LLM이 근거를 받아 인용 포함 산문답변을 생성하는 KB-레벨 옵션이며 **KS kind와 직교(orthogonal)**: `remoteSharePoint`라도 켜면 산문답변, `workIQ`라도 끄면 extracts만 나온다. (learn *Enable Answer Synthesis*, agentic-retrieval, 2026)
- 따라서 **"Foundry IQ가 workIQ 호출 = 정제된 답변"은 부분적으로만 참**: workIQ가 KB에 넘기는 것도 (산문답변이 아니라) **근거(extracts)**다. 다만 그 근거가 **raw 청크가 아닌 Work IQ의 에이전트 추론 산물**이라는 점이 `remoteSharePoint`와의 본질적 차이.
- **진짜 축 = 지능(추론)의 위치:** `remoteSharePoint`=**순수 검색**(추론은 전부 downstream) / `workIQ`=**에이전트형 소싱**(M365 신뢰경계 내 upstream 추론) / `ask`=**완결 답변**. ⇒ 원문 청크만 필요하면 `remoteSharePoint`(빠름), 크로스-시그널로 추론된 근거가 필요하면 `workIQ`(느림 40–60s+·데이터 M365 밖 불출·테넌트 측 entitlement 게이트).

---

> *§7은 원본에서 상용 조건·운영 정책을 다루던 절로, 본 기술 공개판에서는 제외되었습니다. 이하 절 번호(§8~)와 문서 전반의 상호 참조는 원본 번호를 그대로 유지합니다.*

## 8. 기술 실행 로그 (실제 진행 과정 — 입력/출력/에러/조치 전량 기록)

시간순. 각 단계의 **입력 명령 → 실제 출력/에러 → 조치**.

1. **환경 프로브** — `workiq --version` → `0.4.0.16790`; MCP `workiq-*` 함수 **미노출** 확인 → **로컬 CLI + 직접 HTTPS**로 검증 결정.
2. **EULA** — `workiq accept-eula` → 수락 완료(ref: https://github.com/microsoft/work-iq-mcp).
3. **`workiq ask` 1차** — 인자 순서 오류 발견 → **전역 옵션은 서브커맨드 앞** 규칙 확정. 브라우저 로그인 후:
    - 에러 **AADSTS650052**: "Agent Tools" SP(`ea9ffc3e-...`)가 테넌트 <TENANT_M365_ID>에 **미프로비저닝**.
4. **근본원인 진단 & 조치(FIX)** — Graph write로 SP 2개 생성/확인:
    - `az ad sp create --id fdcc1f02-...` → **Work IQ**(objId `<SP_OBJID>`)
    - `az ad sp create --id ea9ffc3e-...` → **Agent Tools**(objId `<SP_OBJID_2>`)
    - ✅ 실측 확인(테이블):
```
    AppId                                 Name         ObjId
     fdcc1f02-fc51-4226-8753-f668596af7f7  Work IQ      <SP_OBJID>
     ea9ffc3e-8a23-4a7d-836d-234d7c7565c1  Agent Tools  <SP_OBJID_2>
```
- 📌 **문서 밖 실전 함정**: `enable-work-iq` 문서는 `fdcc1f02`만 언급하나, CLI `ask`는 **`ea9ffc3e`도 필수**. (라이브 AAD displayName은 `Agent Tools`이나, MS 공식 `Enable-WorkIQToolsForTenant.ps1`은 이 appId를 **"Work IQ Tools"**로 명명하며 실제로는 **Work IQ Tools + 9개 sub-server SP + CLI 클라이언트(`ba081686`)** 등 ~10개 SP를 프로비저닝함 — 본 라이브 경로는 최소 2개만으로 `ask` 성공.)
5. **WorkIQ 토큰 우회 시도** — `az account get-access-token`(WorkIQ 리소스) → **AADSTS65002**(1st-party preauthorization 필요) → az 경유 WorkIQ 호출 **불가** 확정.
6. **Graph 베이스라인** — `GET /v1.0/me` → **200**(MOD Administrator). [§2.1]
7. **Retrieval 스코프 토큰 시도** — `az ... --scope "https://graph.microsoft.com/Files.Read.All Sites.Read.All"` → **AADSTS65002**(az는 해당 Graph 스코프 preauth 없음) → 비대화형 200 **불가** 확정.
8. **Retrieval 라이브** — `POST /v1.0/copilot/retrieval` → **403**, 본문이 위임 스코프 요구를 정확히 강제. [§3.4]
9. **WorkIQ MCP 라이브** — 무인증 `POST /mcp` → **401 + WWW-Authenticate**(client_id `fdcc1f02`) → **PRM** JSON 확보(단일 스코프 `WorkIQAgent.Ask`, org v2.0 AS). [§4.4]
10. **테넌트 구성 실측** — `GET /me/licenseDetails` + `/subscribedSkus` 호출로 계정/테넌트 서비스 플랜 목록을 조회(라이브 200). Retrieval 403은 **토큰 스코프(Files/Sites.Read.All) 문제**로 확정(구성 전제와 무관).
11. **WorkIQ `ask` 라이브 (성공)** ✅ — 브라우저 로그인 통과(SP 수정 효과 확인) → `ask`가 **합성 답변 + `[n]` 인용(Outlook 딥링크)** 반환. `ask` 출력 모델 실증. [§5.4]
12. **Retrieval 라이브 (200 성공)** ✅ — **device-code 플로우**(client `14d82eec` Graph CLI Tools)로 `Files.Read.All`+`Sites.Read.All` 위임 토큰 획득 → `Contoso` 질의에 **200/5 hits**(원문 청크 + `relevanceScore` + `resourceType=listItem` + `resourceMetadata`). `travel policy` 질의는 **200/빈 배열**(= `ask`의 "못 찾음"과 교차 일치). Retrieval 출력 모델·`ask` 대비 실증. [§3.4-b]
13. **WorkIQ MCP 완전 raw 왕복 (CLI 없이)** ✅ — device-code(client `ba081686`, scope `fdcc1f02/WorkIQAgent.Ask`)로 위임 토큰 획득(`aud=fdcc1f02`,`scp=WorkIQAgent.Ask` 실측) → 손수 구성한 JSON-RPC로 `POST /mcp` 호출: `initialize` **200**(serverInfo `WorkIQ.MCP.Server v1.0.165.0`) → `notifications/initialized` **202** → `tools/list` **성공(11개 툴, raw명 확정)** → `tools/call`: `search_paths` **성공**, `fetch`는 서버 authorization 에러로 **거부**(테넌트 측 접근 권한 게이트 실측), `ask`는 실행 후 substrate **`Forbidden`**. 인증/소비/OBO **3계층 분리**를 에러 문자열로 확증. [§4.4-b]

**결론**: **3개 기술 전부 라이브 실측 완료** — Graph 200, **Retrieval 200(원문 청크+relevanceScore) & 403(스코프 강제)**, **WorkIQ `ask` 200(합성 답변+인용, CLI 경유) + WorkIQ MCP 완전 raw JSON-RPC 왕복(initialize/tools/list/search_paths 성공, fetch=접근 권한 게이트, ask=substrate Forbidden)**, WorkIQ MCP 401/PRM, SP 프로비저닝까지 모두 실제 호출로 검증. `ask`는 **CLI(§5.4)와 raw(§4.4-b)** 두 경로 모두 실측.

---

## 9. 재현 절차 (모두 라이브 완료 ✅ — 아래는 재실행용 레시피)

**A. WorkIQ `ask` 라이브 — 완료 ✅** (결과 §5.4)
```bash
workiq --account admin@<TENANT>.onmicrosoft.com ask \
  -q "List my 3 most recent emails and briefly summarize what each is about." -v
```
- SP 프로비저닝(§8-4) 반영 후 브라우저 로그인으로 정상 동작. 테넌트 측 entitlement 미구성 시 `403(스코프 에러 아님)`가 유일 변수였으나, 이 테넌트에선 **정상 200**.

**B. Retrieval 200 라이브 — 완료 ✅** (결과 §3.4-b)
```bash
# device-code로 위임 토큰(Files.Read.All+Sites.Read.All) 획득 (client=Graph CLI Tools 14d82eec)
curl -s -X POST "https://login.microsoftonline.com/<tenant>/oauth2/v2.0/devicecode" \
  -d "client_id=14d82eec-204b-4c2f-b7e8-296a70dab67e" \
  --data-urlencode "scope=https://graph.microsoft.com/Files.Read.All https://graph.microsoft.com/Sites.Read.All offline_access"
# → 코드로 로그인/동의 후 token 폴링 → POST /v1.0/copilot/retrieval 호출
```
- 대안: **Graph Explorer**(aka.ms/ge) 로그인 → 스코프 동의 → §3.4 페이로드 실행.

**C. 비교 실험(Retrieval vs `ask`) — 완료 ✅**: §3.4-b 표에 실측 반영(동일 질의에서 Retrieval=빈 배열/청크, `ask`=대화형 명확화/합성).

---

## 10. 공식 문서 대조 검증 (Doc Cross-Verification)

> 본 섹션은 §0–§9의 라이브 실측 결과를 **최신 공식 문서(learn.microsoft.com) + MS 블로그**와 1:1 대조한 결과다. **방법**: 3개 백그라운드 리서치 에이전트(Retrieval / Work IQ MCP / Work IQ 플랫폼) + 필자의 직접 `web_fetch`/`web_search` 병렬 수행. **판정 범례**: ✅ CONFIRMED(문서 일치) · 🟠 CORRECTED(교정 반영) · 🔴 OUTDATED(문서가 갱신되어 기존 서술 대체) · 🟡 INFERENCE(문서 미명시, 추론으로 표기) · 🔵 LIVE>DOC(문서에 없으나 라이브 증거가 authoritative).

### 10.1 검증 소스 (문서 날짜)

| 기술 | 핵심 공식 소스 (learn.microsoft.com, 별도 표기 외) | 문서 기준일 |
| --- | --- | --- |
| Retrieval | `.../api/ai-services/retrieval/overview`, `.../copilotroot-retrieval`, `.../resources/retrievalextract`, `.../concepts/retrieval-api-paygo` | 2026-08-07 |
| Work IQ MCP | `.../extensibility/work-iq/mcp/overview`·`/tool-reference`·`/policy-governance-mcp`; GitHub `microsoft/work-iq`(`Enable-WorkIQToolsForTenant.ps1`, `.mcp.json`) | 2026-08 |
| Work IQ 플랫폼 | `.../extensibility/work-iq/`; blog `microsoft.com/.../2026/06/02/announcing-the-new-work-iq-apis/`; `devblogs.microsoft.com/microsoft365dev/work-iq-production-ready-intelligence-for-every-agent/` | GA 2026-06-16 |

### 10.2 Retrieval API — 8개 핵심 주장 검증 (7 CONFIRMED / 1 OUTDATED)

| # | 보고서 주장 | 판정 | 비고 |
| --- | --- | --- | --- |
| 1 | 엔드포인트 `/v1.0/copilot/retrieval`(GA)+`/beta`(preview) | ✅ | 문서 verbatim 일치 |
| 2 | **app-only 미지원**(delegated 전용) | ✅ | 문서 "Not supported" 명시 |
| 3 | 스코프 `Files.Read.All`+`Sites.Read.All`(최소), `ExternalItem.Read.All`(커넥터) | ✅ | least-priv/higher-priv 구분 일치 |
| 4 | `queryString` ≤1500자·단문 | ✅ | 문서 일치 |
| 5 | `maximumNumberOfResults` 1–25 | ✅ | 문서 일치 |
| 6 | `filterExpression` KQL, **invalid-KQL는 무필터 silent** | ✅ | 문서 일치(에러 아님) |
| 7 | `relevanceScore`=**extract 레벨** cosine 0–1, 부재 가능(특히 커넥터) | ✅ | `retrievalExtract` 리소스 문서 일치 |
| 8 | 200 req/user/hr, 순서 무보장, **global 클라우드 전용** | ✅ | 문서 일치 |
| — | `dataSource` enum `sharePoint/oneDriveBusiness/externalItem`; 원문 청크 반환(합성 X) | ✅ | 문서 일치 |

**신규 반영 세부(문서 대조로 추가 확인)**: `$batch` 최대 20건 · **호출당 dataSource 1종**(인터리빙 불가) · 파일 크기 한계(>512MB docx/pptx/pdf, >150MB 기타는 미검색) · 시맨틱 대상 확장자(.doc/.docx/.pptx/.pdf/.aspx/.one, 그 외 lexical) · `dataSourceConfiguration`(externalItem connectionId 스코핑) 선택 파라미터.

### 10.3 Work IQ MCP — 검증 (핵심 6 CONFIRMED + 3 CORRECTED + 2 LIVE>DOC)

| 항목 | 보고서 주장 | 판정 | 비고 |
| --- | --- | --- | --- |
| 표면 | MCP(streamable HTTP) 도구 표면, "fewer tools, more paths"+`get_schema` 인트로스펙션 | ✅ | 문서 verbatim |
| 엔드포인트 | `workiq.svc.cloud.microsoft/mcp` | ✅ | `.mcp.json` 일치 |
| 스코프 | 단일 `WorkIQAgent.Ask` | ✅ | 문서 일치 |
| CLI 클라이언트 | public client `ba081686-...` | ✅ | `.mcp.json` 일치 |
| 거버넌스 | **관리자 정책(policy) 별도 필요**; Rego/OPA 정책 엔진 | ✅ | 문서 일치 → **§4.4-b 라이브 접근 권한 게이트와 정합** |
| **앱 명칭** | `ea9ffc3e`=**"Agent Tools"** | 🟠 | 공식명 **"Work IQ Tools"**(MS `Enable-WorkIQToolsForTenant.ps1`). 단 **라이브 AAD displayName은 `Agent Tools`** → 양자 병기(부록 A·§8 반영) |
| **`fdcc1f02` 역할** | 프로비저닝 대상 SP | 🟠 | **리소스/오디언스 앱(MS 관리, 토큰 aud)**; enable 스크립트는 `ea9ffc3e`+9 sub-server+`ba081686` 등 ~10 SP 프로비저닝. (라이브선 최소 `fdcc1f02`+`ea9ffc3e` 2개로 `ask` 성공) |
| **툴 개수** | 11개(라이브) | 🟠 | 문서 overview=**"10 tools"**(fetch_blob 제외), tool-reference+**라이브=11**(+`fetch_blob`). §0·§4.2 반영 |
| RFC 9728 경로 | `/.well-known/oauth-protected-resource/mcp` | 🔵 | 문서는 표준 접미어 無 경로 인용, **라이브 401 `WWW-Authenticate`가 `/mcp` 접미어 PRM URL을 직접 제공** → 라이브 우선 |
| AS | `login.microsoftonline.com/organizations/v2.0` | 🔵 | 문서 미명시, **라이브 PRM JSON에서 관측** → 라이브 우선 |

### 10.4 Work IQ (플랫폼) — 7개 주장 검증 (7 CONFIRMED)

| # | 보고서 주장 | 판정 | 비고 |
| --- | --- | --- | --- |
| 1 | GA 2026-06-16(발표 06-02) | ✅ | 블로그 일치 |
| 2 | 4개 pillar: Chat/Context/Tools/Workspaces | ✅ | 문서 일치 |
| 3 | 3개 프로토콜: A2A/MCP/REST | ✅ | 문서 일치 |
| 4 | 명칭="Work IQ"(2단어), **"layer"**(플랫폼과 혼용 주의) | ✅ | MS는 "layer" 표기 — 본문 "플랫폼/레이어" 병기 유지 |

### 10.5 결론 & 교정 요약

- **전체 판정**: 보고서의 라이브 실측은 **최신 공식 문서와 광범위하게 일치**(Retrieval 7/8, Work IQ MCP 핵심 6/6, 플랫폼 핵심 4/4 CONFIRMED). 라이브에서 잡은 값 중 문서에 없던 2건(`/mcp` PRM 경로·`/organizations/v2.0` AS)은 **라이브 증거가 문서보다 최신·authoritative**.
- **적용한 교정(inline 반영 완료)**:
    1. 🟠 `ea9ffc3e` → **"Work IQ Tools"**(공식명) + 라이브 displayName `Agent Tools` 병기 (부록 A, §8).
    2. 🟠 `fdcc1f02` = **리소스/오디언스 앱** 역할 명확화 (§10.3, §8).
    3. 🟠 툴 개수 = **docs 10 / live 11(+fetch_blob)** (§0, §4.2).
    4. 🟡 `M365_COPILOT_INTELLIGENT_SEARCH`-grounds-Retrieval는 **추론**으로 완화.
- **신규 사실 추가**: Retrieval `$batch`(20)·dataSource 1종/호출·파일 크기·시맨틱 확장자·`dataSourceConfiguration`; enable 스크립트의 **~10 sub-server SP** 프로비저닝.
- **미해결/후속**: Work IQ MCP entity 도구의 테넌트 측 접근 권한(entitlement) 게이트 해제는 이 테넌트에서 미검증 — 다음 라이브 검증 후보.

---

## 11. Microsoft Foundry IQ 검증 — "어느 레이어를 쓰는가" (라이브 실측)

> **질문**: Foundry IQ가 실제로 WorkIQ / Work IQ MCP / Copilot Retrieval API 중 무엇을 쓰는가?
> **답(라이브+문서 확정)**: 단일 사실이 아니라 **선택한 Knowledge Source(KS) `kind`가 백엔드를 결정**한다. Foundry IQ는 §2~§5의 세 기술을 **KS로 감싸 조합**하는 상위 지식 레이어다.

### 11.1 Foundry IQ KS `kind` 12종 & M365 접점 3경로

| KS `kind` | 백엔드 | 본 보고서 대응 | 질의 방식 |
| --- | --- | --- | --- |
| **`workIQ`** | **Work IQ 플랫폼** (OBO 토큰교환) | **§5** (플랫폼/`ask`) | 라이브(사용자 위임) |
| **`remoteSharePoint`** | **Copilot Retrieval API** (`/copilot/retrieval`, 문서 명시) | **§3** | 라이브(사용자 위임) |
| **`indexedSharePoint`** | SharePoint Online **인덱서/커넥터**(Graph 사전수집) | §2(Graph) | 인덱스 로컬 질의 |
| `mcpServer` | 임의 HTTPS **MCP 서버** | §4(메커니즘 동일) | 원격 도구호출 |
| 그 외 7종 | searchIndex·azureBlob·azureSql·file·oneLake·fabricDataAgent·fabricOntology·web | — | 인덱스/원격 |

→ **M365-facing 경로는 3개**(`workIQ`/`remoteSharePoint`/`indexedSharePoint`)이며 **각기 다른 백엔드**다. `mcpServer`로 Work IQ MCP(`workiq.svc.cloud.microsoft/mcp`)를 붙이는 것은 **공식 문서에 가이드 없음**(전용 `workIQ` KS가 대체) — 이론상 가능하나 미지원.

### 11.2 라이브 검증 환경 (Provenance)

| 항목 | 값 |
| --- | --- |
| Search 서비스 | `<SEARCH_SERVICE_2>` (eastus, <RESOURCE_GROUP_2>, **SKU basic**, `apiKeyOnly`) |
| 구독 / 테넌트 | `<SUBSCRIPTION_ID_2>-...` / `<TENANT_AZURE_ID>-...` |
| api-version | `2026-05-01-preview`(remoteSharePoint) · **`2026-08-01-preview`(workIQ 필수)** |
| 피처플래그 실측 | `az feature show ... EnableFoundryIQWithWorkIQ` → **`state: "Pending"`** (등록됨·Microsoft 승인 대기) |
| ⚠️ 테넌트 주의 | 이 구독 테넌트(<TENANT_AZURE_ID>)는 **M365 Copilot 테스트 테넌트(<TENANT_M365_ID>/<TENANT>)와 다름** → cross-tenant **retrieve 불가**. 그래서 §11.2~11.9는 **컨트롤플레인(KS 생성) 계약을 라이브 검증**. ➡️ **이후 M365 Copilot 테넌트(<TENANT_M365_ID>) 자체에 Azure 구독이 있음을 발견해 §11.10에서 진짜 데이터플레인 E2E를 수행**(플래그 `Registered`·실 SharePoint 200·workIQ 인증 체인 통과). |

### 11.3 라이브 로그 — `remoteSharePoint` KS 생성 **성공(201)** ✅

```http
PUT {search-url}/knowledgesources/zz-verify-rsp-ks?api-version=2026-05-01-preview
api-key: {admin-key}
Content-Type: application/json

{ "name":"zz-verify-rsp-ks", "kind":"remoteSharePoint",
  "remoteSharePointParameters": { "filterExpression":"filetype:docx", "resourceMetadata":["Author","Title"] } }
```
```jsonc
// HTTP 201 — 저장된 정의를 그대로 반향
{ "name":"zz-verify-rsp-ks", "kind":"remoteSharePoint", "encryptionKey":null,
  "@odata.etag":"\"0x8DEFCC8AE1970DB\"",
  "remoteSharePointParameters":{ "filterExpression":"filetype:docx", "resourceMetadata":["Author","Title"] } }
```
→ `remoteSharePointParameters.filterExpression`(**KQL**)·`resourceMetadata`가 **§3.2 Retrieval API 바디(`resourceMetadata`, KQL `filterExpression`)와 1:1**. 즉 이 KS는 `/copilot/retrieval`의 **얇은 래퍼**임이 라이브로 확인됨.

### 11.4 라이브 로그 — `workIQ` KS 계약 **역설계(progressive probing)**

문서(§11.7)의 **최소 payload는 2026-05-01-preview에서 정상 생성(201)** 된다(A행). 단, **FIC 기반 `entraAppAuthentication` 인증 모델**을 쓰려면 `workIQParameters`가 필요하고, 이 property는 **2026-08-01-preview 이상에서만 허용**된다. 오류 메시지를 단계적으로 따라가 신버전 계약을 완전 복원(2026-08-18 재측정):

| # | 보낸 payload 요지 | api-version | HTTP | 라이브 응답(핵심) |
| --- | --- | --- | --- | --- |
| A | `{name,kind:workIQ,description}` (문서형 최소) | **2026-05-01-preview** | **201** | 생성 성공 `{name,kind:workIQ,description,encryptionKey:null}` — ⭐ **문서대로 동작** |
| A2 | 좌동(최소) | 2026-08-01-preview | 400 | `Property '**workIQParameters**' is required for kind 'workIQ'` |
| A3 | `+workIQParameters` | 2026-05-01-preview | 400 | `Property '**workIQParameters**' is **not supported for kind 'workIQ' before API version 2026-08-01-preview**` |
| 2 | `workIQParameters:{}` | 2026-08-01-preview | 400 | `The '**entraAppAuthentication**' field is required` |
| 3 | `...{clientId,clientSecret}` | 2026-08-01-preview | 400 | `property 'clientId' does not exist on type 'EntraAppAuthentication'` (→ **secret 아님**) |
| 2b | `entraAppAuthentication:{}` | 2026-08-01-preview | 400 | `'entraAppAuthentication.**applicationId**' is required` |
| 4 | `{applicationId}` | 2026-08-01-preview | 400 | `'entraAppAuthentication.**federatedCredentialId**' is required` |
| 5 | `{applicationId, federatedCredentialId}` (더미 GUID) | 2026-08-01-preview | **201** | 생성 성공 — 응답에 `**tenantId**:null` 노출 |

> ⚠️ **자기교정(2026-08-18 재프로브):** 본 표의 초기 초안은 A행을 "`workIQ` **kind**가 2026-08-01-preview 필수(구버전 거부)"로 적었으나, 이는 **오기**였다. 재측정 결과 **kind+최소 payload는 2026-05-01-preview에서 201 정상**(A행)이며, 버전 게이트는 **`workIQParameters` property에만** 걸린다(A3행). 즉 **문서(최소형)는 정확**했고, 우리 초안이 property 게이트를 kind 게이트로 혼동한 것이다. 상세 판정은 **§12** 참조.

**복원된 현재 `workIQ` KS 생성 계약(라이브):**
```http
PUT {search-url}/knowledgesources/{name}?api-version=2026-08-01-preview
{ "name":"...", "kind":"workIQ", "description":"...",
  "workIQParameters": { "entraAppAuthentication": {
      "applicationId":"<Entra 앱 GUID>",           // 필수
      "federatedCredentialId":"<FIC id>",          // 필수 (secretless)
      "tenantId": null } } }                        // 선택(nullable)
```
→ **secretless OBO**: `clientSecret`은 타입에 없고 **연합자격증명(Federated Identity Credential)** 기반. Search의 관리 ID가 Entra 앱으로 federate하여 Work IQ 토큰을 secret 없이 획득.

### 11.5 인증/권한 모델 & 게이트 경계 (라이브로 입증)

- **retrieve 시 OBO**: 최종 사용자 토큰을 `x-ms-query-source-authorization` 헤더로 전달(**`https://search.azure.com/.default` 오디언스**). 엔진이 이를 **Work IQ 스코프 토큰으로 교환**해 사용자 대신 질의. 표준 Search 인증은 **별도로 추가** 필요.
- **게이트는 데이터플레인에만**: Probe 5에서 **더미 GUID로도 KS 정의가 201 생성**됨 → **컨트롤플레인(KS 생성)은 Entra 앱/FIC/피처플래그를 검증하지 않고 통과**. `EnableFoundryIQWithWorkIQ` 승인·Entra 앱 유효성·M365 권한은 **retrieve(질의) 시점에만 강제**됨(라이브 관찰).
- **거버넌스(문서 확정)**: Work IQ는 **M365 신뢰경계 내**에서만 동작, 데이터가 **테넌트를 벗어나지 않음**, M365 권한·민감도 레이블·정보 장벽을 **자동 강제**(권한 상승 불가).

### 11.6 `workIQ` 응답 필드 (문서 확정)

```jsonc
"activity":[{ "type":"workIQ", "knowledgeSourceName":"my-workiq-ks",
             "elapsedMs":1137, "workIQArguments":{"search":"my query"} }],
"references":[{ "type":"workIQ", "rerankerScore":3.5,
               "attributions":[{"seeMoreWebUrl":"https://..."}],
               "sourceData":{"extracts":[{"text":"Have your VPN username and password ready."}]} }]
```
- `sourceData` 수신하려면 retrieve의 `knowledgeSourceParams`에 **`includeReferenceSourceData:true`**.
- Work IQ는 **40–60s+** 소요 → **`maxRuntimeInSeconds ≥ 120`** 필수.
- 출력형(`extracts[].text` + `seeMoreWebUrl` 딥링크)이 **§5.3의 `ask` 합성답변+인용과 동형** — 동일 Work IQ 백엔드임을 방증.

### 11.7 사전 테스트와의 등가 비교 (핵심 산출물)

| Foundry IQ KS | 실제 백엔드 | 본 보고서 **사전 라이브 증거** | 등가성 근거 |
| --- | --- | --- | --- |
| `remoteSharePoint` | **Copilot Retrieval API** | **§3.4 (403 스코프)·§3.4-b (200 extracts)** | 문서가 `/copilot/retrieval` 사용을 **명시** + 파라미터 1:1(§11.3) + **제약 동일**(SharePoint 전용·200req/user/hr·1500자·hybrid 확장자·max25·무순서) |
| `workIQ` | **Work IQ 플랫폼(OBO)** | **§5.4 (`ask` 200 + 인용)** | 동일 리소스앱(`fdcc1f02`)·동일 출력형(extracts+seeMore, §11.6) + KS 생성계약 라이브 검증(§11.4) |
| `mcpServer` | 임의 MCP tool-calling | **§4.4-b (raw JSON-RPC 왕복)** | 메커니즘 동일(MCP `tools/list`→`tools/call`); 단 Work IQ MCP 부착은 **미문서화** |
| `indexedSharePoint` | SharePoint 인덱서(Graph) | §2 (Graph `/me` 200) | Graph 기반 **사전 인덱싱**(ACL/Purview 동기) 후 로컬 질의 |

→ **결론**: "Foundry IQ가 WorkIQ를 쓴다"는 **KS kind에 따라 셋 중 하나**를 의미. `remoteSharePoint`↔Retrieval API, `workIQ`↔Work IQ 플랫폼은 **본 보고서가 이미 라이브로 각각 검증한 백엔드와 동일**하므로, Foundry IQ 경로는 **기존 실측으로 사실상 검증됨**.

### 11.9 문서 vs 라이브 불일치 (L500 주의)

| 항목 | 최신 공식 문서 | **라이브 실측(2026-08-18 재측정)** | 판정 |
| --- | --- | --- | --- |
| `workIQ` kind + 최소 payload | how-to(2026-06-02) & **2026-05-01-preview REST 레퍼런스**: `{name,kind,description}` | 2026-05-01-preview에서 **201 정상** | **문서 정확**(초안 A행 오기 교정) |
| FIC 스키마 `workIQParameters.entraAppAuthentication` | **문서·레퍼런스에 없음**(2026-05-01 레퍼런스 workIQ = kind/name/desc/encryptionKey뿐) | 2026-08-01-preview에서만 허용·필수 | **문서 미반영(신버전·레퍼런스 미발행)** |
| retrieve 헤더 | 단일 `x-ms-query-source-authorization`(RAW, search.azure.com/.default) | FIC 경로는 **이중 헤더** 추가 | **미문서화(FIC 경로 한정)** |

→ 즉 문서의 **최소·OBO 경로는 지금도 유효**(문서 정확). 라이브에서 만난 복잡성(FIC·이중헤더·토큰 aud)은 **2026-08-01-preview의 신규·미문서 경로**로, "문서 오류"가 아니라 **문서 지연(doc-lag)** 이다. 2026-08-01-preview는 **Learn에 REST 레퍼런스가 아직 없음**(요청 시 2026-04-01 stable로 fallback). 참고 레포 `azure-ai-search-foundry-iq-live-knowledge-sources`는 더 오래됨(`workIQ` KS 자체 없음). **상세 근거·판정은 §12.**

### 11.10 실 테넌트(<TENANT_M365_ID>/<TENANT>) **end-to-end retrieve 실증** — 이번 라이브의 핵심 ✅

§11.2~11.9는 **다른 테넌트(<TENANT_AZURE_ID>, 플래그 Pending)** 의 컨트롤플레인 검증이었다. 이후 **M365 Copilot 테넌트 자체에 Azure 구독**이 있음을 발견해 **진짜 데이터플레인 E2E**를 수행했다.

**전환 환경 (Provenance):**

| 항목 | 값 |
| --- | --- |
| 구독 / 테넌트 | `<SUBSCRIPTION_NAME>` (`<SUBSCRIPTION_ID>-…`) / **`<TENANT_M365_ID>-…`** (= admin@<TENANT>) |
| 피처플래그 실측 | `EnableFoundryIQWithWorkIQ` → **`Registered`** (⭐ Microsoft 승인 **이미 완료** — <TENANT_AZURE_ID>의 `Pending`과 대조) |
| Search 서비스 | `<SEARCH_SERVICE>` (<RESOURCE_GROUP>, westus2, basic, **SystemAssigned MI** principalId `<SEARCH_MI_PRINCIPAL>-…`) |
| KB 합성 모델 | `gpt-5.4-mini` @ `fiqliveks-aoai-…openai.azure.com` (keyless, Search MI) |

#### (A) `remoteSharePoint` KS → **진짜 SharePoint 데이터 HTTP 200** ✅

- 생성: KS `zz-rsp-ks`(remoteSharePoint) + KB `zz-rsp-kb` → **201**.
- **retrieve 입력**(단일 헤더):
```http
 POST {ep}/knowledgebases/zz-rsp-kb/retrieve?api-version=2026-08-01-preview
  api-key: {admin-key}
  x-ms-query-source-authorization: {사용자 search 토큰 RAW}   ← ⚠️ "Bearer " 접두어 금지
  { "messages":[{"role":"user","content":[{"type":"text","text":"What documents mention Contoso or Fabrikam? ..."}]}],
    "knowledgeSourceParams":[{"kind":"remoteSharePoint","knowledgeSourceName":"zz-rsp-ks","includeReferences":true,"includeReferenceSourceData":true}],
    "outputMode":"answerSynthesis","includeActivity":true,"maxRuntimeInSeconds":120 }
```
- **retrieve 출력(200, 실측):** 실제 M365 문서 반환 —
    - `references[]`: **"Fabrikam Case Study"** (`Contoso E-Bike Product Planning.docx`), **"Sales Process.vsdx"** — 각 `sourceData.extracts[].text`(본문 발췌)·`webUrl`(M365 딥링크)·`resourceMetadata{Author,Title}` 포함.
    - `activity[].type` = `[modelQueryPlanning, remoteSharePoint, remoteSharePoint, remoteSharePoint, modelAnswerSynthesis, agenticReasoning]`.
- ⚠️ **재캡처 시 429** = `gpt-5.4-mini`(eastus2) **AOAI TPM 초과**(모델 쿼터) — remoteSharePoint 경로 기능과 **무관한 인프라 한도**. 최초 호출은 200 실데이터 확인됨.
- **판정:** Foundry IQ `remoteSharePoint` KS는 **Copilot Retrieval API(`/copilot/retrieval`)로 실제 M365 데이터를 반환**함을 **E2E로 실증**(문서 명시 + §3 등가 + 실데이터 3중 확인).

#### (B) `workIQ` KS → **인증 체인 완전 통과**, 최종 **테넌트 측 접근 권한(entitlement) 게이트**에서만 차단

- 준비: Entra 앱 `foundryiq-workiq-obo-verify`(appId **`<ENTRA_APP_ID>-…`**) + **FIC**(subject=Search MI oid <SEARCH_MI_PRINCIPAL>, issuer=`login.microsoftonline.com/{tenant}/v2.0`, aud=`api://AzureADTokenExchange`) + Work IQ 위임권한 `WorkIQAgent.Ask` **admin-consent**.
- 생성: KS `zz-workiq-ks`(workIQ, **실제 앱+FIC**) + KB `zz-workiq-kb` → **201**.
- **⭐ 토큰 오디언스 역설계 (라이브 3단계 — 오류 코드가 정확히 길을 알려줌):**

  | 시도 | `x-ms-query-work-iq-source-authorization` 토큰 | workIQ task | 결과 |
  |---|---|---|---|
  | 1 | Work IQ 토큰 (aud=`fdcc1f02`, azp=WorkIQ CLI) | ~0.4s | `workIQA2A: Failed to acquire token. **AADSTS50013**`(assertion 서명검증 실패) |
  | 2 | search 토큰 (aud=`https://search.azure.com`) | ~0.4s | `workIQA2A: … **AADSTS500131**`(assertion **audience 불일치**) |
  | 3 | **앱 스코프 토큰 (aud=`api://<ENTRA_APP_ID>`)** | **~3.8s** | ✅ **인증 통과** → `Could not call WorkIQ A2A agent. **AI credits access is not configured for this user.**` |

  → **결정적 사실:** work-iq 헤더 토큰의 **`aud`가 KS의 `entraAppAuthentication.applicationId`(=<ENTRA_APP_ID>)** 여야 한다. 그래야 Search가 **FIC로 그 앱이 되어(MI→앱) 사용자 토큰을 OBO 교환**한다. (획득법: 앱에 `access_as_user` 스코프 노출 + **Azure CLI(`04b07795`) 사전승인** → `az account get-access-token --scope api://<ENTRA_APP_ID>/access_as_user` 로 **무(無)추가로그인** 획득.)
- **⭐ 이중 헤더 요구 (라이브 신규 — 문서 미기재):** retrieve는 `x-ms-query-source-authorization`(search 토큰) **AND** `x-ms-query-work-iq-source-authorization`(aud=앱 토큰) **둘 다** 필요. 후자 누락 시 `header … is null or empty`. (문서 how-to-work-iq 2026-05-01은 **단일 헤더**만 기술 → **라이브가 최신**.)
- **⭐ 최종 게이트 = 테넌트 측 접근 권한(entitlement):** 시도 3에서 workIQ task가 **3.8초** 소요하며 **실제 Work IQ A2A 에이전트 호출에 도달**했고, 인증이 아니라 **엔타이틀먼트(접근 권한)**에서 거부됨:
  > `Could not call WorkIQ A2A agent. AI credits access is not configured for this user. Please request access from your admin.`
    - 이 게이트는 **인증·토큰 스코프와 직교**하는 **테넌트 측 접근 권한(entitlement) 구성** 축이다(§4.4-b의 3계층 분리와 동일). 구성 주체·상용 조건은 본 기술 보고서 범위 밖.

#### (C) 접근 경로별 인증/게이트 요약

| M365 접근 경로 | 인증 모델 | 이번 라이브 결과 |
| --- | --- | --- |
| **대화형 `ask`** (native Work IQ, bizchat) | 사용자 delegated | **§5.4 → 200 + 인용** ✅ |
| **Foundry IQ `workIQ` KS** (Work IQ API / **A2A**) | FIC→OBO→A2A + 피처플래그 승인 + Entra앱/FIC + **동일 테넌트** | 인증 **완전 통과**, 테넌트 측 접근 권한(entitlement) 게이트만 남음 |
| **Foundry IQ `remoteSharePoint` KS** (Retrieval API) | 사용자 delegated(search+source 토큰) | **200 실데이터** ✅ |
| Graph `/me/*` (원천) | delegated / app-only | §2 → 200 |

> **핵심 통찰(기술):** "Foundry IQ가 Work IQ를 쓴다"는 곧 **Work IQ *API*(A2A)** 를 쓴다는 뜻이며, 그 경로는 사용자가 앱에서 직접 하는 `ask`와 달리 **인증과 직교하는 테넌트 측 접근 권한(entitlement) 게이트**를 통과해야 한다. 이 게이트 경계가 **대화형 Work IQ vs 프로그래밍적 Work IQ(Foundry IQ 포함)** 의 가장 실무적인 차이다.

#### (D) 잔여 테스트 리소스 & 완결/정리 방법

라이브 검증으로 생성한 리소스(테넌트 <TENANT_M365_ID>): KS `zz-rsp-ks`·`zz-workiq-ks`, KB `zz-rsp-kb`·`zz-workiq-kb`, Entra 앱 `<ENTRA_APP_ID>-…`(+FIC). `workIQ` E2E는 **테넌트 측 접근 권한(entitlement) 구성만 완료하면 즉시 완결**된다.

```bash
# 공통
az account set --subscription <SUBSCRIPTION_ID>
SVC=<SEARCH_SERVICE>; RG=<RESOURCE_GROUP>; EP="https://$SVC.search.windows.net"; API=2026-08-01-preview
KEY=$(az search admin-key show --service-name $SVC --resource-group $RG --query primaryKey -o tsv)

# ▶ 완결(테넌트 측 접근 권한(entitlement) 구성 완료 후):
STOK=$(az account get-access-token --scope https://search.azure.com/.default --query accessToken -o tsv)
ATOK=$(az account get-access-token --scope api://<ENTRA_APP_ID>/access_as_user --query accessToken -o tsv)
curl -X POST "$EP/knowledgebases/zz-workiq-kb/retrieve?api-version=$API" -H "api-key: $KEY" \
 -H "Content-Type: application/json" -H "x-ms-query-source-authorization: $STOK" \
 -H "x-ms-query-work-iq-source-authorization: $ATOK" --max-time 240 \
 -d '{"messages":[{"role":"user","content":[{"type":"text","text":"What is top of mind across my recent emails and meetings?"}]}],"includeActivity":true,"knowledgeSourceParams":[{"kind":"workIQ","knowledgeSourceName":"zz-workiq-ks","includeReferences":true,"includeReferenceSourceData":true}],"outputMode":"answerSynthesis","maxRuntimeInSeconds":180}'

# ▶ 정리(원할 때; az AD 삭제는 CAE로 az login 갱신 필요):
for n in zz-rsp-kb zz-workiq-kb; do curl -X DELETE "$EP/knowledgebases/$n?api-version=$API" -H "api-key: $KEY"; done
for n in zz-rsp-ks zz-workiq-ks; do curl -X DELETE "$EP/knowledgesources/$n?api-version=$API" -H "api-key: $KEY"; done
az ad app delete --id <ENTRA_APP_ID>
```

### 11.11 결론

1. **Foundry IQ ≠ 단일 백엔드.** M365 지식은 `workIQ`(플랫폼/A2A)·`remoteSharePoint`(Retrieval API)·`indexedSharePoint`(인덱서) **3경로**로 갈리며, 원하는 정체성(에이전트형 vs 검색형 vs 사전인덱스)에 따라 KS kind를 고른다.
2. **데이터플레인 E2E 실증 완료(실 테넌트 <TENANT_M365_ID>).** `remoteSharePoint` KS는 **HTTP 200 + 실제 SharePoint 문서**(Fabrikam/Contoso) 반환; `workIQ` KS는 **FIC→OBO→A2A 인증 체인 완전 통과**(3.8s A2A 도달), 유일한 잔여 게이트는 **테넌트 측 접근 권한(entitlement) 구성**. 컨트롤플레인(§11.2~11.9)에 더해 데이터플레인까지 검증됨.
3. **데이터플레인 등가성 확정.** `remoteSharePoint`↔§3(Retrieval), `workIQ`↔§5(Work IQ 플랫폼)는 본 보고서가 라이브로 검증한 백엔드와 동일 — 이번에 **동일 테넌트에서 실데이터로 재확인**.
4. **게이트 경계 확정.** `workIQ` KS의 잔여 게이트는 **인증과 직교하는 테넌트 측 접근 권한(entitlement) 구성**뿐이며(피처플래그 **`Registered`**·Entra앱(FIC)·동일 테넌트는 모두 충족), 이를 구성하면 즉시 E2E 완결. 상용 조건은 본 보고서 범위 밖.
5. **라이브 신규 사실(문서 미반영, §12 재검증).** ① work-iq 헤더 토큰 **aud=`entraAppAuthentication.applicationId`** 필수, ② **이중 헤더**(`source-authorization` + `work-iq-source-authorization`), ③ 두 헤더 모두 **RAW 토큰**("Bearer " 금지), ④ **FIC 인증(`entraAppAuthentication`) 경로**는 `2026-08-01-preview` 필수 — 단 **문서형 최소 payload는 `2026-05-01-preview`에서 정상 생성(201)** 이며(§12.3 Probe A), 이 4가지는 "문서 오류"가 아니라 **미발행 신버전(2026-08-01-preview)의 doc-lag**.

### 11.12 라이브 재확인 (2026-08-18 14:28 KST) — **어느 Search 서비스에 `workIQ` KS가 있나** (착오 방지)

> 계기: 두 Search 서비스가 존재해 "`<SEARCH_SERVICE_2>`에 workIQ가 안 보인다"는 관찰이 나옴. **라이브 KS 인벤토리**로 확정.

- **✅ `<SEARCH_SERVICE>`** (M365 테넌트 <TENANT_M365_ID>, <RESOURCE_GROUP>) — `GET {ep}/knowledgeSources?api-version=2026-08-01-preview`(admin-key) 실측 **4건**:
  `fabric-ontology-ks`(fabricOntology) · `microsoft-learn-mcp-ks`(mcpServer) · `zz-rsp-ks`(remoteSharePoint) · **`zz-workiq-ks`(workIQ)**.
  - `zz-workiq-ks` 정의(라이브): `applicationId=<ENTRA_APP_ID>-…` · `federatedCredentialId=5b6c99c6-…` · `tenantId=<TENANT_M365_ID>-…` — **부록 A ⭐ 식별자와 1:1** → **§11.10 E2E workIQ KS가 지금도 상주.**
- **❌ `<SEARCH_SERVICE_2>`** (다른 테넌트 <TENANT_AZURE_ID>, <RESOURCE_GROUP_2>) — **`workIQ` KS 없음이 정상.** 이 서비스는 §11.2~11.9의 **컨트롤플레인(생성계약) 역설계 전용**(프로브 KS는 더미 GUID로 만들고 정리)이고, **M365와 다른 테넌트**라 데이터플레인 E2E 대상이 아님. 현재 M365 로그인(<TENANT_M365_ID>)에선 **구독 밖이라 조회 자체가 불가**(`az resource list` 미검색).
- **판정:** "lmwqmhmig3fb6에 workIQ 없음" = **보고서와 모순 아님.** workIQ E2E는 `n5eo36w5mpcew`에서 수행했고 그 KS가 **라이브 상주**. 두 서비스를 **테넌트·용도(컨트롤플레인 vs 데이터플레인)로 구분**해서 볼 것.

## 12. 실측 ↔ 최신 공식문서 정합성 **재검증(adjudication)** — "우리 오류 vs 문서 오류"

> 목표(사용자 요청): 두 실험(§11.10-A `remoteSharePoint`, §11.10-B `workIQ`)을 **최신 공식문서**와 재대조하여, 불일치가 **우리 실험 오류인지 문서 오류인지 근거로 판정**. 2026-08-18 문서 재수집 + **라이브 재프로브(1차 증거)** 로 수행.

### 12.1 판정 방법론
- **버전 정합이 최우선**: 문서가 겨냥한 api-version과 우리가 쓴 버전이 다르면 스키마 차이는 "오류"가 아니라 **버전 진화**. 최종 중재자는 **버전별 REST 레퍼런스**.
- **서비스 자기모순 우선**: 라이브 서비스의 **자체 오류 메시지**가 문서와 어긋나면 강한 doc-lag 증거.
- **1차 증거 우선**: 오늘 재측정한 HTTP 상태/바디 최우선. 각 문서 `ms.date` 병기.

### 12.2 재수집한 최신 문서 소스 (2026-08-18 확인)
| 문서 | ms.date | 핵심 사실 |
| --- | --- | --- |
| `…how-to-sharepoint-remote.md` | **2026-06-02** | rsp KS = **Copilot Retrieval API 사용** 명문; 제약·응답필드 목록 |
| `…agentic-retrieval-how-to-retrieve.md` | **2026-07-21** | **단일** `x-ms-query-source-authorization`(RAW, `search.azure.com/.default`), 서비스 인증과 분리 |
| `…how-to-work-iq.md` | **2026-06-02** | workIQ **최소 payload @2026-05-01**, 단일헤더 OBO |
| `search-query-access-control-rbac-enforcement.md` | **2026-08-08**(최신) | 2부 인증(앱 RBAC + 사용자 토큰); **work-iq 2번째 헤더 언급 없음** |
| REST ref `knowledge-sources/create-or-update` **@2026-05-01-preview** | 레퍼런스 | `WorkIQKnowledgeSource` = `kind/name/description/encryptionKey`뿐(**`workIQParameters` 없음**); `RemoteSharePoint` 정식 등재 |
| REST ref **@2026-08-01-preview** | **미발행** | 요청 시 **2026-04-01(stable)로 fallback**; stable엔 workIQ/rsp kind 자체 없음 |

### 12.3 라이브 재프로브 (1차 증거, 2026-08-18 · `<SEARCH_SERVICE>`)
| Probe | 요청 | 결과 |
| --- | --- | --- |
| A | `PUT workIQ {name,kind,description}` @**2026-05-01-preview** | **201** ✅ (문서형 최소 payload 정상) |
| B | 좌동(최소) @2026-08-01-preview | **400** `Property 'workIQParameters' is required for kind 'workIQ'` |
| C | `+workIQParameters` @2026-05-01-preview | **400** `… 'workIQParameters' is **not supported for kind 'workIQ' before API version 2026-08-01-preview**` |
| D | `+entraAppAuthentication{applicationId,federatedCredentialId(더미)}` @2026-08-01-preview | **201** ✅ 응답 `tenantId:null` (생성 시 FIC 유효성 미검증) |

→ **자기교정**: 초안 §11.4-A행("`workIQ` **kind**가 2026-08-01 필수")은 **오기**. 실제로는 **kind+최소 payload는 2026-05-01에서 유효**(A)하고, **`workIQParameters` property만** 2026-08-01로 버전 게이트(C)됨. 문서(최소형)가 오히려 정확. (임시 probe KS 3종은 즉시 삭제 완료.)

### 12.4 통합 판정표
| # | 실측 주장 | 판정 | 근거(문서 ms.date / 1차증거) | 신뢰도 |
| --- | --- | --- | --- | --- |
| 1 | `remoteSharePoint` 백엔드 = **Copilot Retrieval API** | ✅ **CONFIRMED** | rsp how-to(06-02) 명문 + retrieve 표 라인577 | 높음 |
| 2 | rsp: **단일헤더·RAW 토큰·서비스인증 분리** | ✅ **CONFIRMED** | retrieve how-to(07-21) REST 예시 + RBAC(08-08) 2부 인증 | 높음 |
| 3 | rsp: **제약·응답필드**(200req/hr·1500자·max25·무순서·resourceMetadata·webUrl·sensitivityLabel) | ✅ **CONFIRMED** | rsp how-to 제약·필드 = §3.4-b 실측 1:1 | 높음 |
| 4 | `workIQ` **최소 payload @2026-05-01 유효** | ✅ **문서 정확**(⚠️ 초안 자기교정) | Probe A(201) + 2026-05-01 레퍼런스 + how-to | 높음 |
| 5 | `workIQ` **FIC 스키마 `entraAppAuthentication` @2026-08-01 필수** | 🆕 **UNDOCUMENTED**(신버전 doc-lag) | Probe B/C/D + 2026-08-01 레퍼런스 미발행 | 높음 |
| 6 | `workIQ` **이중 헤더**(`x-ms-query-work-iq-source-authorization`) | 🆕 **UNDOCUMENTED**(FIC 경로 한정) | retrieve(07-21)·RBAC(08-08) **단일헤더만** | 중~높음 |
| 7 | `workIQ` work-iq 토큰 **aud = appId** | 🆕 **UNDOCUMENTED**(설계상·역설계) | 50013→500131→해결(§11.10-B) + FIC OBO 구조 | 중 |
| 8 | **테넌트 측 접근 권한(entitlement)** 게이트 | ◑ **부분 확증** | §11.10-B 라이브 authorization 에러(3.8s A2A 도달 후 차단); 상세 구성은 본 보고서 범위 밖 | 중~높음 |

### 12.5 결론
1. **실험 1(`remoteSharePoint`) = 최신 문서와 100% 일치.** 백엔드·헤더·토큰형·제약·응답필드 모두 문서가 **정확·최신**. **문서 오류 0, 우리 오류 0.**
2. **실험 2(`workIQ`)**: 문서의 **최소·OBO 경로(2026-05-01)는 지금도 유효**(Probe A 201). 우리가 밟은 **FIC·이중헤더·2026-08-01 경로는 실재하나 미문서**(2026-08-01 레퍼런스 미발행) = **문서 오류가 아니라 doc-lag**. 단 **우리 초안 1건(§11.4-A행)은 오기였고 재측정으로 교정**.
3. **순판정**: 최신 공식문서에서 **사실 오류(틀린 서술)는 발견되지 않음.** 불일치의 실체는 (a) **미발행 신버전(2026-08-01-preview) 스키마/헤더의 문서 지연**과 (b) **우리 쪽 1건 자기교정**뿐. 즉 라이브가 "문서를 앞선" 것이지 "문서가 틀린" 것이 아니다. **2026-08-01-preview REST 레퍼런스가 게시되면 #5~#7은 CONFIRMED로 승격** 전망.

---

## 13. Cross-lingual · 의미검색 실증 — Retrieval API에 벡터검색이 포함되는가 (라이브 4배터리)

> **질문(사용자):** `/copilot/retrieval`은 lexical(BM25)만인가, 아니면 multilingual **벡터/semantic**을 포함하는가? "벡터검색을 포함시키는 가장 큰 경로"가 이 API라면, **한국어↔영어 교차질의**와 **키워드가 겹치지 않는 동의어/자연어 질의**로 정답 문서가 나오는지가 결정적 리트머스다.
>
> **가설:** 순수 lexical이면 표면 토큰이 겹치지 않는 질의의 recall ≈ 0. 반대로 **정답 문서가 나오면** multilingual embedding 기반 semantic 성분이 실재한다(lexical 엔진으로는 불가능).

### 13.1 공통 방법 (입력 파라미터 — 4개 배터리 전부 동일, `queryString`만 변화)

| 항목 | 값 |
| --- | --- |
| Endpoint | `POST https://graph.microsoft.com/v1.0/copilot/retrieval` |
| 인증 | **위임(delegated) device-code**, client `14d82eec-…`(Graph CLI Tools), scope `Files.Read.All Sites.Read.All offline_access`, tenant `<TENANT_M365_ID>-…`(admin@<TENANT>) |
| 고정 body | `{"dataSource":"sharePoint","resourceMetadata":["title","author","lastModifiedDateTime"],"maximumNumberOfResults":5}` |
| 변수 | **`queryString`만** 교체 |
| 관측값 | `retrievalHits[*].webUrl`(정답 문서 판정), `extracts[*].relevanceScore`(top score), rank |
| 대상 | semantic-eligible 확장자 문서(`.docx`/`.pptx`) — Contoso/Fabrikam 데모 콘텐츠 |
| 규모 | **4 배터리 · 36 질의, 전건 HTTP 200** |

> **주의:** 이 스코프는 `az account get-access-token`으로 불가(AADSTS65002 1st-party preauth 부재) → **device-code가 유일 경로**. device code는 ~15분 만료(AADSTS70020).

### 13.2 배터리 A — 동일 의미의 언어 교체(EN↔KO)

| # | queryString | 언어 | hits | top 문서 | topScore |
| --- | --- | --- | --- | --- | --- |
| Q1 | `Contoso open positions software engineer` | EN | 5 | Contoso Ltd. Open Positions | **0.734** |
| Q2 | `Contoso 소프트웨어 엔지니어 채용 공고` | KO(Contoso 유지) | 5 | Contoso Ltd. Open Positions | 0.704 |
| Q3 | `콘토소 소프트웨어 엔지니어 채용` | KO(전음역) | 4 | Contoso Ltd. Open Positions | 0.670 |
| Q4 | `developer job openings at Contoso` | EN(패러프레이즈) | 5 | Contoso Ltd. Open Positions | — |
| Q5 | `Fabrikam case study e-bike product planning` | EN(.docx) | 4 | Fabrikam Case Study | 0.772 |
| Q6 | `파브리캄 전기자전거 제품 기획 사례 연구` | KO(.docx) | **0** | — | — |

- **콘토소**(전음역, 공유 토큰 0)로도 정답 문서 **#1** 유지 → lexical 불가. 단 Q6은 0 → 규명 필요(→13.3).

### 13.3 배터리 B — 대조쌍(EN↔KO 동의어쌍) + Q6 규명

| # | queryString | 언어 | hits | top 문서 | topScore |
| --- | --- | --- | --- | --- | --- |
| P1 | `Fabrikam 전기자전거 제품 기획 사례 연구` | KO+`Fabrikam` | 1 | Fabrikam Case Study | 0.674 |
| P2 | `파브리캄 전기자전거` | KO | **0** | — | — |
| P3 | `전기자전거 제품 기획` | KO | **0** | — | — |
| P4 | `e-bike 제품 기획` | KO+`e-bike` | 2 | EBike Brainstorming(0.773), Fabrikam(0.705) | 0.773 |
| P5 | `e-bike product planning` | EN | 5 | (다수) | — |
| P6 | `merger announcement acquisition` | EN | 1 | Merger Announcement | 0.516 |
| P7 | `합병 발표 인수 공고` | **KO(공유토큰 0)** | 1 | Merger Announcement | 0.499 |
| P8 | `annual sales report revenue` | EN | 1 | Annual Sales Report | 0.769 |
| P9 | `연간 매출 보고서 매출액` | **KO(공유토큰 0)** | 1 | Annual Sales Report | 0.686 |

- **P7/P9 = 결정적 증거:** 순수 한국어·영문문서와 **공유 토큰 0**인데 정답 영문 문서를 정확히 반환(matched extract 본문은 영어: P7 → "## Merger Announcement: Contoso Ltd. and Wingtip Toys Unite…"). lexical로는 불가능.
- **Q6=0의 원인:** 하드 lexical 게이트가 아님. P7/P9(순수 KO)는 성공하는데 P2/P3(전기자전거·파브리캄)만 0 → **전음역·복합 KO 용어가 embedding 상 너무 멀어 임계값 미달**. P1/P4에서 영어 앵커(`Fabrikam`/`e-bike`)를 넣자 즉시 복구.

### 13.4 배터리 C — 동의어·패러프레이즈·자연어질문(**문서 어휘 미사용**) + 네거티브 컨트롤

> 문서의 실제 단어(open/positions·merger/acquisition·sales/revenue)를 **쓰지 않고** 동의어·자연어 질문으로만 질의. 이게 진짜 "키워드 비매칭" 테스트.

| # | queryString (의도) | hits | 정답 rank | topScore |
| --- | --- | --- | --- | --- |
| R_EN_syn | `yearly earnings and overall financial performance turnover figures`(=매출, sales/revenue 無) | 1 | **#1** | **0.747** |
| R_KO_syn | `연간 수익과 재무 실적 총액 수치`(순수 KO, 공유토큰 0) | 1 | **#1** | 0.680 |
| H_EN_syn | `career vacancies recruiting new talent to join the team`(=채용) | **0** | — | — |
| H_KO_syn | `인재 모집 입사 지원 구인 공고` | **0** | — | — |
| H_QA_KO | `이 회사에서 일하려면 어떤 직무에 지원할 수 있나요`(자연어) | **0** | — | — |
| M_EN_syn | `two companies combining into a single business consolidation buyout`(=합병) | **0** | — | — |
| M_KO_syn | `두 기업이 하나로 통합 결합하는 내용` | **0** | — | — |
| M_QA_EN | `which firms are joining forces to combine their operations` | **0** | — | — |
| R_QA_KO | `작년에 회사가 돈을 얼마나 벌었나요`(구어체 자연어) | **0** | — | — |
| **NEG_EN** | `employee parental leave reimbursement policy`(없는 주제) | **0** | — | — |
| **NEG_KO** | `출장 여비 정산 규정`(없는 주제) | **0** | — | — |

- **R(매출) 문서**만 순수 동의어로 EN·KO **양쪽 rank1** 적중 → **키워드 없는 의미검색 확정.** 반면 H·M은 원거리 동의어에서 전부 0 → **임계값 존재** 시사(→13.5로 규명).
- **네거티브 컨트롤 2건 모두 0** → 엔진이 "아무거나 top-N 반환"하지 않음. **0 hit은 진짜 '임계값 미달'**이고 hit은 진짜 의미매칭.

### 13.5 배터리 D — 그라데이션(키워드→근접→중간→원거리)으로 **임계값 규명**

| # | queryString | 단계 | 정답 rank | targetScore |
| --- | --- | --- | --- | --- |
| HG1 | `Contoso Ltd. Open Positions` | 키워드(컨트롤) | **#1** | 0.728 |
| HG2 | `job openings at Contoso` | 근접(openings~open) | **#1** | 0.729 |
| HG3 | `hiring for roles at Contoso company` | **중간(open/position 無)** | **#1** | **0.731** |
| HG4 | `career vacancies recruiting talent to join the team` | 원거리 | **0 MISS** | — |
| MG1 | `Merger Announcement Contoso Wingtip Toys` | 키워드(컨트롤) | **#1** | 0.514 |
| MG2 | `Contoso and Wingtip Toys merger deal` | 근접 | **#1** | 0.499 |
| MG3 | `Contoso acquires Wingtip Toys` | **중간(merger 無, acquire)** | **#1** | 0.499 |
| MG4 | `two companies combining into a single business consolidation` | 원거리 | **0 MISS** | — |
| HK | `콘토소 채용 공고`(KO) | 근접·cross-lingual | **#1** | 0.604 |
| MK | `콘토소 윈팁 합병`(KO) | 근접·cross-lingual | **#1** | 0.513 |

- **키워드 컨트롤(HG1·MG1) 적중** → 문서는 **지금도 살아있음**. 즉 13.4의 동의어 miss는 **deindex가 아니라 semantic distance** 때문.
- **HG3**(`hiring for roles`, open/position 단어 없음)·**MG3**(`acquires`, merger 단어 없음) 모두 **rank1** → **키워드 없는 의미매칭을 monolingual에서도 재확인.**
- 두 문서 모두 **키워드→근접→중간=hit(≈0.73/≈0.50), 원거리=0**의 **깨끗한 절벽(cliff).** 원거리 질의는 낮은 rank가 아니라 **행 자체가 비어서 반환**(sub-floor score suppression) → **소프트 임계값**의 존재를 직접 입증. 이는 13.2 Q6·13.3 P2/P3 zero-hit과 **동일 메커니즘**.

### 13.6 정량 지표

- **Jaccard 결과셋 겹침**(배터리 A/B): Q1vsQ3(EN vs 전음역KO)=**0.80**, Q1vsQ2=0.67, Q1vsQ4(EN vs EN패러프레이즈)=**0.67**, P6vsP7(합병 EN vs KO)=**1.00**, P8vsP9(매출 EN vs KO)=**1.00**, 대조군 P5vsP3=0.00, Q5vsQ6=0.00.
  → **한국어 번역에 의한 drift(0.67) ≈ 영어 패러프레이즈 drift(0.67).** 언어 교체가 "또 하나의 패러프레이즈"처럼 취급됨 = 의미공간 검색의 특징.
- **EN ≳ KO score gap(일관):** 채용 0.734(EN)>0.704>0.670(KO), 매출 0.769(EN)>0.686(KO)·0.747(EN)>0.680(KO). 합병만 EN/KO 사실상 동률(0.514≈0.513). → 대체로 영어가 소폭 우세하나 정답 #1은 견고.
- **문서별 score ceiling이 다름:** 매출 ~0.75–0.77, 채용 ~0.73, **합병 ~0.51(현저히 낮음).** 임계값은 **절대 semantic-similarity 기준**이라, ceiling이 낮은 합병 문서는 동의어가 조금만 멀어져도 먼저 탈락.

### 13.7 판정

1. **✅ 벡터/semantic(키워드 독립) 검색 확정.** 서로 독립인 3가지로 입증: **(i) cross-lingual** — 순수 한국어·공유토큰 0 질의가 정확한 영문 문서 반환(P7 합병, P9 매출, R_KO 수익). **(ii) monolingual 동의어 스왑** — 문서 어휘를 안 쓴 영어 질의가 적중(HG3 hiring→open positions, MG3 acquires→merger, R_EN earnings→sales). **(iii) 둘 동시**(R_KO). lexical/BM25는 세 경우 모두 score≈0 → **multilingual embedding 성분을 가진 hybrid retrieval이 경험적으로 확립.** **▶ §13.9에서 동일 쿼리·동일 토큰·동일 코퍼스를 Graph Search API(lexical)에 던진 controlled 대조군으로 인과까지 분리 — semantic-only 6종 확정.**
2. **⚠️ 단, 소프트 relevance 임계값에 의해 recall이 유한.** 그라데이션에서 키워드→근접→중간=hit, **원거리 패러프레이즈=empty**의 절벽이 두 문서에서 재현. 문서별 ceiling(합병 0.51 vs 매출/채용 0.73)이 달라 각자의 floor를 가짐. Q6·P2/P3 zero-hit과 동일 메커니즘(=하드 lexical 게이트 아님).
3. **✅ hit은 유의미, 0은 진짜 미달.** 네거티브 컨트롤(없는 주제)=empty, 키워드 컨트롤=hit → 엔진은 무차별 반환하지 않음.
4. **🧭 컨설팅 함의:** cross-lingual·동의어 recall은 **실재하나 무제한이 아님.** 견고한 검색을 위해 **(a) 고유명사·앵커 용어(Contoso/Wingtip/e-bike) 유지, (b) 원거리 패러프레이즈 의존 금지, (c) `filterExpression`(KQL) 병용 및 자체 re-rank** 권장. Azure AI Search식 **명시적 벡터화가 필요한지**를 가르는 실측 근거가 이 절이다.

### 13.8 재현 / 원자료

- 스크립트: `work/retr_test.py`(배터리 A, device-code 재인증 포함) · `work/probe2.py`(B) · `work/probe3.py`(C) · `work/probe4.py`(D). B/C/D는 `token.json` 재사용(무재인증).
- 원자료 JSON: `retr_raw.json`/`retr_summary.json`, `probe2_*`, `probe3_*`, `probe4_*`.
- 전 배터리 **HTTP 200**. 점수는 절대값이 아닌 **문서·언어 간 상대비교**로 해석(동일 세션·동일 body).

### 13.9 Graph Search API lexical 대조군 — "의미검색이 Retrieval API의 차별점"의 controlled 입증 (2026-08-18)

> **방법론 계기(사용자 지적):** §13.1–13.8은 Retrieval API가 cross-lingual·동의어에 hit함을 보였으나, 이것만으로는 그 hit이 **Retrieval API 고유의 semantic 성분** 때문인지 단정 불가. **동일 delegated 토큰·동일 코퍼스·동일 `queryString`** 을 lexical 엔진인 **Graph Search API**(`POST /v1.0/search/query`, `entityTypes:["driveItem"]`)에 함께 던져 **Graph가 miss**하는 대조(control)가 나와야 인과가 분리된다.

**설계:** 같은 Graph 토큰(scp=`Files.Read.All Sites.Read.All`)으로 각 쿼리를 (a) `/copilot/retrieval`(semantic), (b) `/search/query`(lexical/KQL) **두 엔진에 페어링**. target 문서 3종의 실제 위치(라이브):
- **Hiring** — `…/sites/operationsdepartment/…/human resources/Contoso Ltd. Open Positions.docx`
- **Merger** — `…/sites/leadership/…/merger and acquisitions/Merger Announcement.docx`
- **Sales** — `…/sites/salesandmarketing/…/Annual Sales Report.docx`

**결과 (target_rank; 숫자=hit 위치, —=miss):**

| # | `queryString` | 성격 | Retrieval | Graph Search | 판정 |
|---|---|---|---|---|---|
| X1 | "Contoso open positions software engineer" | EN 키워드 | **1** | **1** | 둘 다 hit(코퍼스 도달 확인) |
| C_SALES | "Contoso annual sales report revenue" | EN 키워드 | **1** | **1** | 둘 다 hit |
| MERG_kw | "merger announcement" | EN 키워드(교정) | **1** | **1** | 둘 다 hit |
| **X2** | "콘토소 채용 공고" | cross-lingual | **1** | **—** | ⭐ semantic-only |
| **X3** | "콘토소 윈팁 합병" | cross-lingual | **1** | **—** | ⭐ semantic-only |
| **X4** | "콘토소 연간 매출 보고서" | cross-lingual | **1** | **—** | ⭐ semantic-only |
| **S1** | "yearly earnings … turnover figures" | EN 동의어(공유토큰 0) | **1** | **—** | ⭐ semantic-only |
| **S2** | "연간 수익 재무 실적 총액 수치" | KO 동의어 | **1** | **—** | ⭐ semantic-only |
| **D2** | "Contoso acquires" | 부분키워드 semantic | **2** | **—** | ⭐ semantic-only |
| **CMERG_x** | "Contoso WinTip merger announcement" | 비매칭토큰 혼입 | **1** | **—** | ⭐ soft-match vs hard-AND |
| S3/S4 | hire/merger 원거리 동의어 | (임계값 밖) | — | — | 둘 다 miss(§13.7-② 정합) |
| N1/N2 | 없는 주제(EN/KO) | 네거티브 | — | — | 둘 다 empty(컨트롤 정상) |

**⭐ controlled 결론 (인과 분리):**
1. **코퍼스 도달성 통제됨.** 3개 target 문서 **전부** Graph Search가 **올바른 EN 키워드로 rank1 hit**(X1·C_SALES·`"merger announcement"`). 따라서 cross-lingual/동의어에서의 Graph miss는 "문서 부재/권한 부족"이 **아니라** 매칭 방식(lexical)의 한계로 귀속됨. (Graph는 사용자 권한 트리밍된 SPO 인덱스를 content까지 검색하므로 공정한 baseline.)
2. **semantic-only 6종 확정.** cross-lingual 3종(X2·X3·X4)·순수 동의어 2종(S1·S2)·부분키워드 1종(D2)에서 **Retrieval hit ∧ Graph miss.** lexical(KQL/BM25)은 **언어 경계(KO 질의→EN 문서)와 어휘 경계(공유토큰 0 동의어)를 넘지 못함**을 실측 → Retrieval API의 **multilingual embedding(vector) 성분이 그 차별의 원인**임이 대조로 증명.
3. **soft-match vs hard-AND (부수 발견 + 방법론 투명성).** `"Contoso WinTip merger announcement"`는 문서에 없는 `Contoso`·`WinTip` 토큰 때문에 Graph가 **implicit AND로 0건**이나 Retrieval은 여전히 rank1 → Retrieval은 **비매칭 토큰에 강건**(semantic soft-scoring). *(최초 merger 컨트롤이 바로 이 confound로 0건이었고, `"merger announcement"`·`merger`·`path:.../sites/leadership`로 재검색해 Graph의 Leadership-site 도달성을 확인 후 컨트롤을 교정함 — 오탐을 근거로 제거한 과정 명기.)*
4. **네거티브·both-miss 정상.** 없는 주제=양쪽 empty; Retrieval이 임계값 밖으로 버린 원거리 동의어(S3·S4)·`"hiring for roles"`(probe4 HIT 0.731→이번 런 miss)는 Graph도 miss → **soft-threshold의 run-to-run 경계 변동**까지 재확인(§13.7-②).

**원자료:** `work/graph_vs_retr.py`(+`graph_vs_retr_summary.json`/`_raw.json`), `work/graph_kw_controls.json`. 전 호출 HTTP 200(양 엔진 정상), 판정은 target 문서 등장/부재의 이진 처리.

> **한 줄 요약:** *동일 쿼리를 던졌을 때 **Graph Search(lexical)는 언어·동의어 경계에서 전멸, Copilot Retrieval(semantic)은 rank1 생존** — "Retrieval API에 벡터검색이 포함되는가?"에 대한 대조 기반 YES.*

---

## 14. Copilot Retrieval API 데이터소스 커버리지 — "SharePoint만인가? OneDrive·메일·Teams는?" (라이브 enum + 공식문서, 2026-08-18)

> **사용자 질문:** "Copilot API"를 쓰는 개발자는 최소 **OneDrive·SharePoint·Outlook 메일·Teams** 4개 제품에서 검색되길 기대한다. Retrieval API는 어디까지 커버하나?

### 14.1 `dataSource` enum 라이브 판정 (동일 토큰, `POST /v1.0/copilot/retrieval`)

| `dataSource` 후보 | HTTP | 판정 |
|---|---|---|
| `sharePoint` | **200** | ✅ 유효 |
| `oneDriveBusiness` | **200** | ✅ 유효 (OneDrive for Business) |
| `externalItem` | **403** `ExternalItem.Read.All` 필요 | ✅ **유효 enum**(권한만 부족) — Copilot connectors 외부데이터 |
| `oneDrive`·`mail`·`message`·`email`·`outlook`·`teams`·`chatMessage`·`teamsMessage`·`copilotConnectors`·`connectors`·`calendar`·`people`·`person` | **400** | ❌ 미지원 값(enum 밖) |

### 14.2 공식문서 확증 (Learn *copilotroot-retrieval*, 갱신 **2026-08-07**)

- 정의: "retrieval of relevant text extracts from **SharePoint, OneDrive, and Copilot connectors** content **that the calling user has access to**."
- `dataSource` Acceptable values: **`sharePoint` · `oneDriveBusiness` · `externalItem`** — **라이브 enum과 1:1 일치**.
- 권한(delegated only): **Files.Read.All + Sites.Read.All**(SP·OneDrive **둘 다 필요**) · **ExternalItem.Read.All**(externalItem). **Application(app-only)·개인 MS 계정=미지원.**
- `filterExpression`(KQL) 지원 속성: `Author`·`FileExtension`·`Filename`·`FileType`·`InformationProtectionLabelId`·`LastModifiedTime`·`ModifiedBy`·`Path`·`SiteID`·`Title`(+커넥터 queryable 속성). **메일/Teams 속성은 없음.**
- 문서 전체에 **Outlook 메일·Teams 메시지 언급 없음** → 명백히 범위 밖.

### 14.3 그럼 메일·Teams는 어디로? — 저장소 분리 라이브 실측

같은 토큰(scp=`Files.Read.All Sites.Read.All`)으로 Graph Search `/search/query`를 `entityTypes` 별로:

| `entityTypes` | HTTP | 저장소(에러가 명시) |
|---|---|---|
| `driveItem` · `site` · `listItem` | **200** | 파일/SharePoint (=Retrieval과 **동일 인덱스**) |
| `message` | **403** `Mail.Read` 필요 | **ExchangeMessage** (메일 substrate) |
| `chatMessage` | **403** `Chat.Read`/`ChannelMessage.Read.All` | **ChatMessage** (Teams substrate) |
| `event` | **403** `Calendars.Read` 필요 | **ExchangeEvent** (캘린더 substrate) |

→ 메일·Teams·캘린더는 **파일 인덱스와 다른 저장소·다른 권한 도메인**(ExchangeMessage/ChatMessage/ExchangeEvent). Retrieval API는 이들을 **구조적으로 다루지 않음**.

### 14.4 ⭐ 커버리지 판정 & 개발자 라우팅

| 원하는 컨텍스트 소스 | 올바른 API | 필요 권한 |
|---|---|---|
| SharePoint 문서 | Retrieval API `dataSource:sharePoint` | Files.Read.All + Sites.Read.All |
| OneDrive for Business | Retrieval API `dataSource:oneDriveBusiness` | 〃 |
| 외부(Copilot connectors) | Retrieval API `dataSource:externalItem`(+`dataSourceConfiguration`) | + ExternalItem.Read.All |
| **Outlook 메일** | ❌ Retrieval 불가 → **Graph Search `message`** 또는 **Work IQ `ask`** | Mail.Read |
| **Teams 메시지/채팅** | ❌ Retrieval 불가 → **Graph Search `chatMessage`** 또는 **Work IQ `ask`** | Chat.Read / ChannelMessage.Read.All |
| **4개 제품 전부를 하나의 semantic 질의로** | **Work IQ `ask`** (substrate 전체: mail·meetings·Teams·files·people 합성) | 사용자 delegated(§5); A2A 경로는 **테넌트 측 접근 권한 게이트**(§11) |

**결론:** **Retrieval API ≠ 통합 M365 검색.** 이것은 **파일·커넥터 grounding(RAG) 전용** API로, 커버리지는 **SharePoint + OneDrive for Business + Copilot connectors**에 한정된다(라이브 enum·공식문서 2026-08-07 동시 확증). "Copilot API 한 방으로 OneDrive·SharePoint·메일·Teams 전부"를 원하는 개발자는 **Work IQ `ask`** (M365 Copilot substrate 전체를 semantic 합성; §5·§6)를 써야 하며, Retrieval API는 그중 **파일/커넥터 축**만 담당한다. 세밀 제어가 필요하면 소스별로 **Retrieval API(파일)·Graph Search(메일/Teams `entityTypes`)·Work IQ(합성)** 를 **조합**한다. → §6 "3-surface" 매트릭스의 **데이터 커버리지 관점 재확인**.

## 부록 A. 핵심 식별자

| 항목 | 값 |
| --- | --- |
| Work IQ 리소스 앱 | `fdcc1f02-fc51-4226-8753-f668596af7f7` (SP objId `<SP_OBJID>-...`) |
| **Work IQ Tools** 앱(공식명; 라이브 AAD displayName은 `Agent Tools`) | `ea9ffc3e-8a23-4a7d-836d-234d7c7565c1` (SP objId `<SP_OBJID_2>-...`) |
| WorkIQ MCP endpoint | `https://workiq.svc.cloud.microsoft/mcp` |
| WorkIQ OAuth scope | `fdcc1f02.../WorkIQAgent.Ask` (단일) |
| WorkIQ CLI OAuth client | `ba081686-5d24-4bc6-a0d6-d034ecffed87` (public client) |
| Retrieval endpoint | `POST https://graph.microsoft.com/v1.0/copilot/retrieval` |
| npm | `@microsoft/workiq` |
| **Foundry IQ** Search 서비스(§11.2, 컨트롤플레인 전용) | `<SEARCH_SERVICE_2>` (eastus, <RESOURCE_GROUP_2>, basic; **테넌트 <TENANT_AZURE_ID> — M365와 다름 · `workIQ` KS 없음이 정상**) |
| **Foundry IQ** KS api-version | `2026-05-01-preview`(일반) · **`2026-08-01-preview`**(`workIQ` KS 필수) |
| **Foundry IQ** Work IQ 게이트 | 플래그 `Microsoft.Search/EnableFoundryIQWithWorkIQ`(**실측 Pending**) + 승인폼 `aka.ms/foundry-iq-work-iq-admin-consent-form` |
| **⭐ 실 E2E 테넌트(§11.10)** | `<TENANT_M365_ID>` (admin@<TENANT>) — 구독 `<SUBSCRIPTION_ID>-…`, 플래그 **`Registered`** |
| **⭐ 실 E2E Search 서비스** | `<SEARCH_SERVICE>` (westus2, <RESOURCE_GROUP>, basic, **SAMI** principalId `<SEARCH_MI_PRINCIPAL>`) — **라이브 상주 KS(2026-08-18): `zz-workiq-ks`(workIQ)·`zz-rsp-ks`(remoteSharePoint)·`fabric-ontology-ks`·`microsoft-learn-mcp-ks`** |
| **⭐ workIQ OBO Entra 앱**(생성) | `<ENTRA_APP_ID>` (identifierUri `api://<ENTRA_APP_ID>-…`, scope `access_as_user`, FIC `5b6c99c6-…`) |
| **⭐ workIQ retrieve 헤더** | `x-ms-query-source-authorization`(search 토큰 RAW) **+** `x-ms-query-work-iq-source-authorization`(**aud=Entra 앱** 토큰 RAW) — 둘 다 필수 |

## 부록 B. 자주 만나는 에러 코드

| 코드 | 의미 | 조치 |
| --- | --- | --- |
| **AADSTS650052** | Work IQ/Agent Tools SP 미프로비저닝 | `az ad sp create --id {fdcc1f02, ea9ffc3e}` |
| **AADSTS65002** | 1st-party preauthorization 부재(az→WorkIQ/추가 Graph 스코프) | 인터랙티브/preauthorized 클라이언트 사용 |
| **403 (Forbidden, scope 명시)** | Retrieval에 Files/Sites.Read.All 부재 | 위임 스코프 재동의 후 재호출 |
| **403 (scope 언급 없음)** | 테넌트 측 접근 권한(entitlement) 미구성 | 테넌트 측 접근 권한 구성 후 재시도(전파 15–30분) |
| **400 `kind 'workIQ' ... only supported with api-version 2026-08-01-preview`** | Foundry IQ `workIQ` KS를 구버전 preview로 생성 시도 | api-version을 **2026-08-01-preview 이상**으로 상향 (§11.4) |
| **400 `Property 'workIQParameters' is required` / `entraAppAuthentication.federatedCredentialId is required`** | `workIQ` KS에 FIC 기반 Entra 앱 인증 누락 | `workIQParameters.entraAppAuthentication{applicationId, federatedCredentialId}` 추가 (§11.4) |
| **400 `header 'x-ms-query-work-iq-source-authorization' is null or empty`** | `workIQ` retrieve에 **이중 헤더** 중 work-iq 토큰 누락 | `x-ms-query-work-iq-source-authorization`(aud=Entra 앱) 추가 (§11.10-B) |
| **502 `workIQA2A … AADSTS50013`** (assertion 서명검증 실패) | work-iq 헤더에 **Work IQ 토큰(aud=fdcc1f02)** 을 넣음 → 앱이 OBO 교환 불가 | 토큰 aud를 **Entra 앱(`entraAppAuthentication.applicationId`)** 으로 (§11.10-B) |
| **502 `workIQA2A … AADSTS500131`** (assertion audience 불일치) | work-iq 헤더에 **search 토큰(aud=search.azure.com)** 을 넣음 | 좌동 — 앱 스코프 토큰 `az … --scope api://{appId}/access_as_user` |
| **502 `Could not call WorkIQ A2A agent. AI credits access is not configured for this user`** | 인증은 통과, **테넌트 측 접근 권한(entitlement) 미구성**(A2A 게이트) | 테넌트 관리자에게 **접근 권한 구성** 요청 (§11.10-B) |
| **429 `model endpoint … TooManyRequests … gpt-5.4-mini`** | KB 합성 모델(AOAI)의 **TPM 쿼터 초과**(인프라) | 대기/쿼터 상향 — KS 경로 기능과 무관 (§11.10-A) |
