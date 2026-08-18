# Azure AI Search 핸즈온 워크샵

Azure AI Search의 검색 기술을 단계별로 직접 구현해보는 실습 워크샵입니다.

## � 목차

- [🚀 시작하기 (처음부터 따라하기)](#-시작하기-처음부터-따라하기)
  - [1단계: 필수 소프트웨어 설치](#1단계-필수-소프트웨어-설치)
  - [2단계: Azure 리소스 생성](#2단계-azure-리소스-생성)
  - [3단계: 프로젝트 설정](#3단계-프로젝트-설정)
  - [4단계: 연결 테스트](#4단계-연결-테스트)
  - [💡 유용한 팁](#-유용한-팁)
  - [🔧 자주 발생하는 문제 해결](#-자주-발생하는-문제-해결)
- [빠른 시작 (이미 환경이 준비된 경우)](#빠른-시작-이미-환경이-준비된-경우)
- [이 워크샵에서 배우는 것](#이-워크샵에서-배우는-것)
- [실습 순서](#실습-순서)
- [검색 방식별 특성 비교](#검색-방식별-특성-비교)
- [폴더 구조](#폴더-구조)

---

## �🚀 시작하기 (처음부터 따라하기)

> **전제 조건**: Visual Studio Code 설치, Azure 구독 활성화

이 가이드는 완전히 처음부터 시작하는 분들을 위한 단계별 설정 가이드입니다.

### 1단계: 필수 소프트웨어 설치

#### 1-1. Python 3.10 이상 설치

**Windows:**
1. https://www.python.org/downloads/ 접속
2. "Download Python 3.12.x" 클릭
3. 설치 시 **"Add Python to PATH"** 체크박스 필수 선택
4. 설치 완료 후 확인:
   ```powershell
   python --version
   ```

**macOS:**
```bash
# Homebrew로 설치 (권장)
brew install python@3.12

# 또는 공식 사이트에서 다운로드
# https://www.python.org/downloads/macos/

# 설치 확인
python3 --version
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt update
sudo apt install python3.12 python3.12-venv python3-pip
python3 --version
```

#### 1-2. Visual Studio Code 설치

1. https://code.visualstudio.com/ 에서 다운로드
2. 설치 완료 후 실행

#### 1-3. VS Code 확장 설치

VS Code에서 다음 확장을 설치하세요:

1. **Python** (Microsoft) - 필수
   - 확장 ID: `ms-python.python`
2. **Jupyter** (Microsoft) - 필수
   - 확장 ID: `ms-toolsai.jupyter`
3. **Pylance** (Microsoft) - 권장
   - 확장 ID: `ms-python.vscode-pylance`

**설치 방법:**
- VS Code에서 `Ctrl+Shift+X` (macOS: `Cmd+Shift+X`)
- 검색창에 확장 이름 입력 후 "Install" 클릭

### 2단계: Azure 리소스 생성

#### 2-1. Azure AI Search 생성

1. [Azure Portal](https://portal.azure.com) 로그인
2. "리소스 만들기" → "Azure AI Search" 검색
3. 다음 정보 입력:
   - **구독**: 본인의 Azure 구독 선택
   - **리소스 그룹**: 새로 만들기 (예: `rg-aisearch-workshop`)
   - **서비스 이름**: 고유한 이름 (예: `search-workshop-yourname`)
   - **위치**: `Korea Central` 또는 `East US`
   - **가격 책정 계층**: `Basic` (Semantic ranker 사용 시 필수)
4. "검토 + 만들기" → "만들기"
5. 배포 완료 후:
   - "리소스로 이동" 클릭
   - 좌측 메뉴 "키" 클릭
   - **URL**과 **기본 관리 키** 복사 (나중에 사용)

#### 2-2. Azure OpenAI 리소스 생성

1. Azure Portal에서 "리소스 만들기" → "Azure OpenAI" 검색
2. 다음 정보 입력:
   - **구독**: 본인의 Azure 구독
   - **리소스 그룹**: 위에서 만든 것 선택
   - **지역**: `Sweden Central` (추천) 또는 `East US`
   - **이름**: 고유한 이름 (예: `openai-workshop-yourname`)
   - **가격 책정 계층**: `Standard S0`
3. "검토 + 만들기" → "만들기"
4. 배포 완료 후:
   - "리소스로 이동" 클릭
   - 좌측 메뉴 "키 및 엔드포인트" 클릭
   - **엔드포인트**와 **KEY 1** 복사 (나중에 사용)

#### 2-3. Azure OpenAI 모델 배포

1. Azure OpenAI 리소스에서 "모델 배포" 클릭 (또는 [Azure AI Studio](https://ai.azure.com) 접속)
2. **임베딩 모델 배포:**
   - "새 배포 만들기" 클릭
   - 모델: `text-embedding-3-large`
   - 배포 이름: `text-embedding-3-large` (그대로 사용 권장)
   - "배포" 클릭
3. **채팅 모델 배포:**
   - "새 배포 만들기" 클릭
   - 모델: `gpt-4.1-mini`
   - 배포 이름: `gpt-4.1-mini` (그대로 사용 권장)
   - "배포" 클릭

### 3단계: 프로젝트 설정

#### 3-1. 레포지토리 다운로드

**방법 1: Git 사용 (권장)**
```bash
git clone https://github.com/ChangJu-Ahn/azure_aisearch_workshop.git
cd azure_aisearch_workshop
```

**방법 2: ZIP 다운로드**
1. GitHub 레포지토리 페이지에서 "Code" → "Download ZIP"
2. 압축 해제 후 해당 폴더로 이동

#### 3-2. VS Code에서 프로젝트 열기

```bash
code .
```

또는 VS Code에서 "File" → "Open Folder" → 프로젝트 폴더 선택

#### 3-3. 가상환경 생성 및 활성화

> **중요**: 운영체제에 따라 명령어가 다릅니다!

**Windows (PowerShell):**
```powershell
# 가상환경 생성
python -m venv .venv

# 가상환경 활성화
.venv\Scripts\Activate.ps1

# 만약 보안 오류가 발생하면 먼저 실행:
# Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

**Windows (Command Prompt):**
```cmd
# 가상환경 생성
python -m venv .venv

# 가상환경 활성화
.venv\Scripts\activate.bat
```

**macOS / Linux:**
```bash
# 가상환경 생성
python3 -m venv .venv

# 가상환경 활성화
source .venv/bin/activate
```

**가상환경 활성화 확인:**
- 터미널 프롬프트 앞에 `(.venv)` 표시
- Windows: `(.venv) PS C:\Users\...>`
- macOS/Linux: `(.venv) user@machine:~$`

#### 3-4. 패키지 설치

가상환경이 활성화된 상태에서:

```bash
pip install --upgrade pip
pip install -r requirements.txt
```

**설치 확인:**
```bash
pip list
```

다음 패키지들이 보이면 성공:
- `azure-search-documents`
- `openai`
- `pandas`
- `python-dotenv`
- 등등

#### 3-5. .env 파일 생성

**Windows:**
```powershell
copy .env.sample .env
```

**macOS / Linux:**
```bash
cp .env.sample .env
```

VS Code에서 `.env` 파일을 열고 다음 값을 입력:

```env
# Azure AI Search 설정 (2-1 단계에서 복사한 값)
SEARCH_ENDPOINT=https://your-service-name.search.windows.net
SEARCH_ADMIN_KEY=your-admin-key-here
SEARCH_INDEX_NAME=products-index

# Azure OpenAI 설정 (2-2 단계에서 복사한 값)
AZURE_OPEN_AI_ENDPOINT=https://your-openai-name.openai.azure.com
AZURE_OPEN_AI_KEY=your-openai-key-here

# 모델 배포 이름 (2-3 단계에서 설정한 이름)
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-3-large
AZURE_OPENAI_CHAT_DEPLOYMENT=gpt-4o-mini

# API 버전 (변경하지 마세요)
AZURE_OPENAI_API_VERSION=2024-06-01
```

**중요:**
- `SEARCH_ENDPOINT`는 `https://`로 시작하고 `.search.windows.net`으로 끝남
- `AZURE_OPEN_AI_ENDPOINT`는 `https://`로 시작하고 `.openai.azure.com`으로 끝남
- 키 값에는 따옴표 없이 그대로 붙여넣기
- 모든 값 입력 후 저장 (`Ctrl+S` 또는 `Cmd+S`)

### 4단계: 연결 테스트

1. VS Code에서 `01-setup/test_connection.ipynb` 파일 열기
2. 상단에서 Python 인터프리터 선택:
   - "Select Kernel" 클릭
   - "Python Environments..." 선택
   - `.venv` (가상환경) 선택
3. 첫 번째 셀부터 순서대로 실행 (Shift+Enter)
4. 모든 테스트가 ✅로 표시되면 성공!

**문제 해결:**

- **"No module named 'azure'"**: 가상환경이 활성화되지 않았거나 패키지 설치 안 됨
  → 3-3, 3-4 단계 다시 확인
- **"❌ 환경 변수가 설정되지 않았습니다"**: .env 파일 확인
  → 3-5 단계에서 모든 값을 올바르게 입력했는지 확인
- **"❌ Azure AI Search 연결 실패"**: SEARCH_ENDPOINT, SEARCH_ADMIN_KEY 확인
  → Azure Portal에서 값을 다시 복사하여 .env에 붙여넣기
- **"❌ Embedding 모델 연결 실패"**: 모델 배포 이름 확인
  → Azure AI Studio에서 배포 이름이 정확히 일치하는지 확인

### 다음 단계

✅ 연결 테스트가 성공하면 아래 "실습 순서"를 따라 진행하세요!

### 💡 유용한 팁

**가상환경 비활성화:**
```bash
deactivate
```

**VS Code에서 Python 인터프리터 변경:**
1. 노트북 파일 열기
2. 우측 상단 "Select Kernel" 클릭
3. "Python Environments..." 선택
4. `.venv` 환경 선택

**패키지 재설치가 필요한 경우:**
```bash
pip install --upgrade pip
pip install -r requirements.txt --force-reinstall
```

### 🔧 자주 발생하는 문제 해결

<details>
<summary><b>가상환경 활성화 오류 (Windows PowerShell)</b></summary>

**증상:**
```
.venv\Scripts\Activate.ps1 : 이 시스템에서 스크립트를 실행할 수 없으므로...
```

**해결:**
PowerShell을 관리자 권한으로 실행 후:
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```
그 다음 다시 가상환경 활성화 시도.
</details>

<details>
<summary><b>Python을 찾을 수 없음 (Windows)</b></summary>

**증상:**
```
'python'은(는) 내부 또는 외부 명령... 아닙니다.
```

**해결:**
1. Python 재설치 시 "Add Python to PATH" 체크
2. 또는 환경 변수에 수동으로 Python 경로 추가

**확인 방법:**
```powershell
python --version
```
</details>

<details>
<summary><b>pip install 오류: No matching distribution</b></summary>

**증상:**
```
ERROR: Could not find a version that satisfies the requirement...
```

**해결:**
Python 버전 확인 (3.10 이상 필요):
```bash
python --version
```
3.10 미만이면 Python 업그레이드 필요.
</details>

<details>
<summary><b>VS Code에서 .env 파일을 읽지 못함</b></summary>

**증상:**
노트북 실행 시 "환경 변수가 설정되지 않았습니다"

**해결:**
1. `.env` 파일이 프로젝트 루트에 있는지 확인
2. VS Code를 완전히 종료 후 재시작
3. 노트북 커널 재시작: "Restart Kernel"
4. `.env` 파일 내용에 따옴표가 없는지 확인:
   ```env
   # 올바른 예
   SEARCH_ENDPOINT=https://mysearch.search.windows.net
   
   # 잘못된 예 (따옴표 사용 금지)
   SEARCH_ENDPOINT="https://mysearch.search.windows.net"
   ```
</details>

<details>
<summary><b>Azure 연결 오류: Unauthorized (401)</b></summary>

**증상:**
```
❌ Azure AI Search 연결 실패: (401) Unauthorized
```

**해결:**
1. Azure Portal에서 키를 다시 복사
2. `.env` 파일의 `SEARCH_ADMIN_KEY` 값 확인
3. 키 앞뒤에 공백이 없는지 확인
4. 쿼리 키가 아닌 **관리 키**를 사용하는지 확인
</details>

<details>
<summary><b>모듈을 찾을 수 없음: No module named 'azure'</b></summary>

**증상:**
```
ModuleNotFoundError: No module named 'azure'
```

**해결:**
1. 가상환경이 활성화되어 있는지 확인 (터미널에 `(.venv)` 표시)
2. 가상환경에 패키지 설치:
   ```bash
   pip install -r requirements.txt
   ```
3. VS Code에서 올바른 Python 인터프리터 선택 (.venv 환경)
</details>

<details>
<summary><b>Azure OpenAI 모델 배포 이름 오류</b></summary>

**증상:**
```
❌ Embedding 모델 연결 실패: The API deployment for this resource does not exist
```

**해결:**
1. [Azure AI Studio](https://ai.azure.com) 접속
2. "배포" 탭에서 실제 배포 이름 확인
3. `.env` 파일의 배포 이름을 정확히 일치시키기:
   ```env
   AZURE_OPENAI_EMBEDDING_DEPLOYMENT=실제배포이름
   ```
</details>

<details>
<summary><b>macOS에서 SSL 인증서 오류</b></summary>

**증상:**
```
SSL: CERTIFICATE_VERIFY_FAILED
```

**해결:**
Python 3.x 폴더에서 "Install Certificates.command" 실행:
```bash
/Applications/Python\ 3.12/Install\ Certificates.command
```

또는:
```bash
pip install --upgrade certifi
```
</details>

<details>
<summary><b>Jupyter Kernel이 시작되지 않음</b></summary>

**해결:**
1. VS Code에서 Jupyter 확장 재설치
2. VS Code 재시작
3. 가상환경에 ipykernel 설치:
   ```bash
   pip install ipykernel
   ```
4. Python 인터프리터 다시 선택
</details>

---

## 이 워크샵에서 배우는 것

| 모듈 | 핵심 기술 | 구현 내용 |
|------|----------|----------|
| 02-keyword_search | BM25, OData 필터, Facet | 인덱스 스키마 정의, `ko.lucene` 한국어 분석기, 필터/정렬/집계 |
| 03-vector_search | HNSW, KNN, 임베딩 | 벡터 필드 추가, `text-embedding-3-large`로 3072차원 벡터 생성 |
| 04-hybrid_search | RRF (Reciprocal Rank Fusion) | 키워드+벡터 결합, 점수 계산 원리 이해 |
| 05-score_tuning | VectorizedQuery weight, Scoring Profile | 벡터/하이브리드/키워드 가중치, 비즈니스 로직 부스팅 |
| 06-re_ranking | Semantic L2 모델, Captions/Answers | 하이브리드 결과 재정렬, 답변 추출 |
| 07-enriched_dataset | GPT Vision | 이미지 분석 → 텍스트 증강 → 벡터 재생성 |
| 08-skillsets (개념) | Indexer, Skillset, Knowledge Store | OCR/Entity Recognition 등 Built-in Skills, Custom Skills, 실무 시나리오 5가지 |
| 09-foundryiq (별도 레포) | Agentic Retrieval, Knowledge Base | LLM이 쿼리 분해/병렬 실행, Reasoning Effort, Activity 분석 |

## 기술 스택

```
Python 3.10+
azure-search-documents >= 11.6.0  (권장: 12.0.0 GA)
openai >= 1.12.0
python-dotenv
pandas, numpy, scikit-learn, tqdm
```

## 빠른 시작 (이미 환경이 준비된 경우)

> **처음 시작하시는 분은 위의 "🚀 시작하기" 섹션을 참조하세요!**

### 필수 항목 체크리스트
- ✅ Python 3.10 이상 설치
- ✅ VS Code + Python/Jupyter 확장 설치
- ✅ Azure AI Search 리소스 (Basic 이상)
- ✅ Azure OpenAI 리소스 + 모델 배포
  - `text-embedding-3-large`
  - `gpt-4o-mini`

### 빠른 설정

**1. 가상환경 생성 및 활성화**

Windows (PowerShell):
```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

macOS / Linux:
```bash
python3 -m venv .venv
source .venv/bin/activate
```

**2. 패키지 설치**
```bash
pip install -r requirements.txt
```

**3. 환경변수 설정**

Windows:
```powershell
copy .env.sample .env
```

macOS / Linux:
```bash
cp .env.sample .env
```

`.env` 파일을 열고 실제 값 입력:
```env
SEARCH_ENDPOINT=https://<your-service>.search.windows.net
SEARCH_ADMIN_KEY=<admin-key>
SEARCH_INDEX_NAME=products-index
AZURE_OPEN_AI_ENDPOINT=https://<your-endpoint>.openai.azure.com
AZURE_OPEN_AI_KEY=<api-key>
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-3-large
AZURE_OPENAI_CHAT_DEPLOYMENT=gpt-4o-mini
AZURE_OPENAI_API_VERSION=2024-06-01
```

**4. 연결 테스트**
```
01-setup/test_connection.ipynb 실행
```

## 샘플 데이터

`00-data/sample_data.csv` - 한국어 이커머스 상품 247건

| 필드 | 타입 | 용도 |
|------|------|------|
| product_id | string | 문서 키 |
| title | string | 검색 대상 (SearchableField) |
| brand | string | 필터/패싯 |
| category | string | 필터/패싯 |
| price | int | Magnitude 함수용 |
| review | string | 검색 대상, Semantic 분석 |
| image_link | string | Vision 분석용 |

## 실습 순서

### Phase 1: 기본 검색 파이프라인

```
01-setup/test_connection.ipynb     → 리소스 연결 확인
02-keyword_search/01-create_index  → 인덱스 스키마 정의
02-keyword_search/02-upload_data   → 문서 업로드
02-keyword_search/03-keyword_search → BM25, 필터, 패싯
```

**이 단계에서 익히는 API:**
- `SearchIndexClient.create_or_update_index()`
- `SearchClient.upload_documents()`
- `SearchClient.search(search_text, filter, facets, order_by)`

### Phase 2: 벡터 검색

```
03-vector_search/01-update_index   → 벡터 필드, HNSW 프로필 추가
03-vector_search/02-upload_vectors → 임베딩 생성 및 업로드
03-vector_search/03-vector_search  → 벡터 검색 실행
```

**이 단계에서 익히는 개념:**
- HNSW (Hierarchical Navigable Small World) vs Exhaustive KNN
- `VectorizedQuery(vector, k_nearest_neighbors, fields)`
- 임베딩 차원과 유사도 메트릭 (cosine)

### Phase 3: 하이브리드 검색

```
04-hybrid_search/01-hybrid_search  → 키워드 + 벡터 결합
```

**핵심 코드:**
```python
results = search_client.search(
    search_text="가벼운 운동화",
    vector_queries=[VectorizedQuery(vector=embedding, k_nearest_neighbors=10, fields="content_vector")]
)
# RRF 점수 = Σ(1 / (k + rank)), k=60
```

### Phase 4: 검색 품질 튜닝

```
05-score_tuning/01-scoring_profile    → 벡터/하이브리드/키워드 가중치 튜닝
06-re_ranking/01-semantic_reranking   → L2 모델로 재정렬
```

**Scoring Profile 예시:**
```python
ScoringProfile(
    name="lowPriceFirst",
    functions=[
        MagnitudeScoringFunction(
            field_name="price",
            boost=2,
            parameters=MagnitudeScoringParameters(
                boosting_range_start=0,
                boosting_range_end=50000,
                should_boost_beyond_range_by_constant=False
            )
        )
    ]
)
```

**Semantic Re-ranking:**
```python
results = search_client.search(
    search_text="따뜻한 겨울 자켓",
    query_type=QueryType.SEMANTIC,
    semantic_configuration_name="my-semantic-config",
    query_caption=QueryCaptionType.EXTRACTIVE,
    query_answer=QueryAnswerType.EXTRACTIVE
)
```

### Phase 5: 데이터 증강

```
07-enriched_dataset/01-enrich_with_vision → GPT Vision으로 이미지 분석
```

**파이프라인:**
```
이미지 URL → GPT Vision 분석 → 텍스트 특징 추출 → content_text에 병합 → 벡터 재생성 → 인덱스 업데이트
```

### 개념 학습 (실습 없음)

```
08-skillsets_aienrichment/README.md → Indexer + Skillset 개념
```

- OCR, Entity Recognition, Key Phrase Extraction 등
- Knowledge Store 활용
- 실제 구현 시 별도 Azure AI Services 리소스 필요

### Phase 6: Agentic Retrieval (별도 레포)

```
09-agentic_retrieval_foundryiq/ → Foundry IQ Agentic Retrieval 개념
```

**Agentic Retrieval이란?**
LLM이 복잡한 사용자 질의를 자동으로 분해하고 최적의 검색 전략을 수립하여 실행하는 차세대 RAG 기술

**해결하는 문제:**
- 고정된 검색 파이프라인 → LLM이 쿼리에 맞춰 동적으로 전략 선택
- 자연어 조건 파싱 → "5만원 이하", "최신" 같은 표현을 자동으로 필터로 변환
- 복잡한 질의 처리 → 여러 서브 쿼리로 분해하여 병렬 실행
- 멀티턴 대화 → 이전 맥락을 자동으로 반영

**실습 진행:**
이 워크샵의 01-07 모듈로 인덱스를 준비한 후, 아래 레포에서 실습
👉 [ignite25-LAB511](https://github.com/ChangJu-Ahn/ignite25-LAB511-build-agentic-knowledge-bases-next-level-rag-with-azure-ai-search)

## 검색 방식별 특성 비교

| 방식 | 장점 | 단점 | 적합한 케이스 |
|------|------|------|--------------|
| Keyword (BM25) | 정확한 용어 매칭 | 동의어, 의미 파악 불가 | 제품코드, 브랜드명 검색 |
| Vector | 의미 기반 유사도 | 정확한 키워드 놓칠 수 있음 | 자연어 질의, 유사 상품 |
| Hybrid (RRF) | 두 방식의 장점 결합 | 단순 순위 결합 | 일반적인 검색 |
| Hybrid + Semantic | 의미 기반 재정렬 | 추가 지연 시간 | 높은 품질 요구 시 |

## 폴더 구조

```
azure_aisearch_workshop/
├── README.md                          # 🎯 메인 가이드 (여기)
├── requirements.txt                   # Python 패키지 목록
├── .env.sample                        # 환경설정 템플릿
├── .env                               # 환경설정 파일 (직접 생성, Git 제외)
├── .gitignore                         # Git 제외 파일 목록
│
├── 00-data/
│   ├── README.md
│   └── sample_data.csv                # 247개 한국어 상품 데이터
│
├── 01-setup/
│   └── test_connection.ipynb          # 🔍 환경 연결 테스트 (시작점)
│
├── 02-keyword_search/
│   ├── README.md
│   ├── 01-create_index.ipynb         # 인덱스 스키마 정의
│   ├── 02-upload_data.ipynb          # 데이터 업로드
│   └── 03-keyword_search.ipynb       # BM25, 필터, 패싯
│
├── 03-vector_search/
│   ├── README.md
│   ├── 01-update_index.ipynb         # 벡터 필드 추가
│   ├── 02-upload_vectors.ipynb       # 임베딩 생성 및 업로드
│   └── 03-vector_search.ipynb        # 벡터 검색 실습
│
├── 04-hybrid_search/
│   ├── README.md
│   └── 01-hybrid_search.ipynb        # RRF (키워드+벡터)
│
├── 05-score_tuning/
│   ├── README.md
│   └── 01-scoring_profile.ipynb      # 벡터/하이브리드/키워드 가중치 & 스코어 튜닝
│
├── 06-re_ranking/
│   ├── README.md
│   ├── 01-semantic_reranking.ipynb   # Semantic L2 재정렬
│   └── 02-semantic_preview_features.ipynb
│
├── 07-enriched_dataset/
│   ├── README.md
│   └── 01-enrich_with_vision.ipynb   # GPT Vision 이미지 분석
│
├── 08-skillsets_aienrichment/
│   └── README.md                      # 개념 설명 (실습 없음)
│
└── 09-agentic_retrieval_foundryiq/
    └── README.md                      # 별도 레포 안내
```

**주요 파일 설명:**
- `README.md` (이 파일): 전체 워크샵 가이드
- `.env.sample`: 환경설정 템플릿 (복사하여 사용)
- `.env`: 실제 환경설정 (직접 생성, Git에 포함 안 됨)
- `requirements.txt`: Python 패키지 목록
- `00-data/sample_data.csv`: 실습용 한국어 데이터
- `01-setup/test_connection.ipynb`: **가장 먼저 실행해야 할 파일**

## 참고 문서

- [Azure AI Search REST API](https://learn.microsoft.com/rest/api/searchservice/)
- [azure-search-documents Python SDK](https://learn.microsoft.com/python/api/overview/azure/search-documents-readme)
- [Vector Search 개요](https://learn.microsoft.com/azure/search/vector-search-overview)
- [Semantic Ranking](https://learn.microsoft.com/azure/search/semantic-search-overview)
- [Azure OpenAI Service](https://learn.microsoft.com/azure/ai-services/openai/)

## 🤝 지원 및 기여

### 문제 발생 시

1. **먼저 확인해보세요**: [🔧 자주 발생하는 문제 해결](#-자주-발생하는-문제-해결) 섹션
2. **연결 테스트**: `01-setup/test_connection.ipynb`를 실행하여 환경 확인
3. **이슈 등록**: [GitHub Issues](https://github.com/ChangJu-Ahn/azure_aisearch_workshop/issues)에서 문제 보고

### 기여 방법

이 워크샵을 개선하는 데 도움을 주고 싶으시다면:

1. 이 레포지토리를 Fork
2. 개선 사항 구현
3. Pull Request 생성

### 피드백

워크샵에 대한 피드백이나 제안사항이 있으시면:
- GitHub Issues에 의견 남기기
- Pull Request로 직접 수정 제안

## ⚖️ 고지사항

- 이 워크샵은 Azure AI Search와 Azure OpenAI의 기능을 학습하기 위한 교육 자료입니다.
- 샘플 데이터는 학습 목적으로만 사용하세요.

## 📧 문의

워크샵 관련 문의사항이 있으시면 GitHub Issues를 통해 연락 주세요.

---

**Happy Learning! 🎉**
