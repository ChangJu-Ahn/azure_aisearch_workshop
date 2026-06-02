# 2️⃣ 키워드 검색 – Azure 포털에서 직접 해보기 (No-Code Guide)

> 🎯 **이 문서의 목적**
> 앞선 노트북 실습은 **코드(Python SDK)** 로 인덱스를 만들고 데이터를 올리고 검색했습니다.
> 하지만 **똑같은 작업을 Azure 포털 화면(UI)에서 클릭만으로도 할 수 있습니다.**
> 이 가이드는 **기획자/비개발자** 가 코드 없이 포털에서 키워드 검색을 구성·테스트할 수 있도록 안내합니다.

---

## 📌 한눈에 보기: 코드 vs 포털

| 단계 | 노트북(코드)에서 한 일 | 포털(화면)에서 하는 방법 |
|------|----------------------|------------------------|
| 인덱스 생성 | `01-create_index.ipynb` 에서 `SearchIndexClient` 로 필드 정의 | **인덱스 > + 인덱스 추가** 화면에서 필드 추가 |
| 데이터 업로드 | `02-upload_data.ipynb` 에서 `upload_documents()` | **데이터 가져오기(Import data) 마법사** 또는 CSV 업로드 |
| 키워드 검색 | `03-keyword_search.ipynb` 에서 `client.search()` | **검색 탐색기(Search explorer)** 에서 쿼리 입력 |

> 💡 즉, **개발 코드 한 줄 없이도** 인덱스 설계 → 데이터 적재 → 검색 테스트까지 화면에서 끝낼 수 있습니다.

---

## 0. 시작 전 준비

1. 브라우저에서 [Azure Portal](https://portal.azure.com) 에 로그인합니다.
2. 상단 검색창에 **AI Search** 를 입력하고, 사용 중인 **Azure AI Search 리소스**를 클릭합니다.

> 📸 **[이미지 첨부]** Azure 포털 홈에서 AI Search 리소스를 검색/선택하는 화면

---

## 1. 인덱스 만들기 (코드: `01-create_index.ipynb` 대체)

키워드 검색은 **어떤 컬럼을 검색 대상으로 할지** 정의하는 인덱스 설계가 핵심입니다.
노트북에서 정의했던 것과 **동일한 필드**를 화면에서 그대로 만듭니다.

### 1.1 인덱스 추가 화면 열기

1. 좌측 메뉴에서 **검색 관리 > 인덱스(Indexes)** 를 클릭합니다.
2. 상단 **+ 인덱스 추가(Add index)** 버튼을 클릭합니다.
3. **인덱스 이름**에 `products-index` 를 입력합니다. (노트북의 `SEARCH_INDEX_NAME` 과 동일)

> 📸 **[이미지 첨부]** 인덱스 목록 화면 + "인덱스 추가" 버튼 위치

### 1.2 필드 정의하기

**+ 필드 추가(Add field)** 를 눌러 아래 표와 똑같이 필드를 만듭니다.
각 컬럼의 체크박스(검색 가능 / 필터 가능 / 정렬 가능 / 패싯 가능 / 검색 결과 표시)를 표에 맞게 켜면 됩니다.

| 필드명 | 데이터 형식 | 키 | 검색 가능 (Searchable) | 필터 가능 (Filterable) | 정렬 가능 (Sortable) | 패싯 가능 (Facetable) | 검색 결과 표시 (Retrievable) |
|--------|------------|----|:---:|:---:|:---:|:---:|:---:|
| `id` | String | ✅ | | ✅ | | | ✅ |
| `title` | String | | ✅ | | | | ✅ |
| `brand` | String | | ✅ | ✅ | | ✅ | ✅ |
| `category` | String | | | ✅ | | ✅ | ✅ |
| `normal_price` | Int32 | | | ✅ | ✅ | | ✅ |
| `image_link` | String | | | | | | ✅ |

> ⚠️ **꼭 확인**
> - `id` 는 반드시 **키(Key)** 로 지정해야 합니다. (문서를 구분하는 고유값)
> - 검색어로 찾고 싶은 컬럼(`title`, `brand`)은 **검색 가능(Searchable)** 을 켭니다.
> - 필터/정렬/집계에 쓸 컬럼은 각각 **필터 가능 / 정렬 가능 / 패싯 가능** 을 켭니다.

> 📸 **[이미지 첨부]** 필드 추가 패널에서 `title` 필드를 Searchable 로 설정하는 화면

### 1.3 한국어 분석기(ko.lucene) 지정 — 한글 검색 품질의 핵심

노트북에서 `analyzer_name="ko.lucene"` 로 지정했던 부분입니다.
화면에서는 **검색 가능(Searchable)** 으로 설정한 `title`, `brand` 필드의
**분석기(Analyzer)** 항목을 **`Korean - Lucene` (ko.lucene)** 으로 선택합니다.

> 💡 한국어 분석기를 쓰면 "노트북가방" → "노트북", "가방" 처럼 형태소 단위로 잘라 검색 정확도가 올라갑니다.

> 📸 **[이미지 첨부]** 필드 설정에서 Analyzer 를 `Korean - Lucene` 로 선택하는 화면

### 1.4 저장

모든 필드를 추가했으면 **저장(Save)** / **만들기(Create)** 를 클릭해 인덱스를 생성합니다.

> 📸 **[이미지 첨부]** 필드 6개가 모두 추가된 최종 인덱스 정의 화면

---

## 2. 데이터 업로드 (코드: `02-upload_data.ipynb` 대체)

노트북에서는 `sample_data.csv` 를 코드로 올렸지만, 포털에서는 **데이터 가져오기 마법사**로 올릴 수 있습니다.

> ℹ️ 포털의 **데이터 가져오기(Import data)** 마법사는 보통 **Azure Blob Storage, Azure SQL, Cosmos DB** 등
> Azure 데이터 원본에 연결합니다. 따라서 `sample_data.csv` 를 먼저 **Blob Storage 컨테이너에 업로드**한 뒤 연결하는 방식이 가장 간단합니다.

### 2.1 CSV를 Blob Storage에 올리기

1. Azure Portal에서 **스토리지 계정 > 컨테이너** 로 이동합니다.
2. 컨테이너를 하나 만들고 (`products` 등), `../00-data/sample_data.csv` 파일을 **업로드** 합니다.

> 📸 **[이미지 첨부]** Blob 컨테이너에 sample_data.csv 를 업로드한 화면

### 2.2 데이터 가져오기 마법사 실행

1. 다시 **Azure AI Search 리소스**로 돌아갑니다.
2. 상단 **데이터 가져오기(Import data)** 를 클릭합니다.
3. **데이터 원본 연결** 에서 방금 만든 Blob 컨테이너를 선택하고, **구문 분석 모드(Parsing mode)** 를 **Delimited text(CSV)** 로 설정합니다. (첫 행을 헤더로 사용 체크)
4. **인덱스 대상**을 1단계에서 만든 `products-index` 로 지정합니다.
5. **제출(Submit)** 을 누르면 CSV의 각 행이 문서로 인덱스에 적재됩니다.

> 📸 **[이미지 첨부]** 데이터 가져오기 마법사 – 데이터 원본/CSV 파싱 설정 화면

### 2.3 업로드 확인

**인덱스 > products-index > 개요** 에서 **문서 수(Document count)** 가 늘어났는지 확인합니다.

> 📸 **[이미지 첨부]** 인덱스 개요에서 문서 수가 표시된 화면

---

## 3. 키워드 검색 테스트 (코드: `03-keyword_search.ipynb` 대체)

이제 코드 없이 **검색 탐색기(Search explorer)** 에서 바로 검색해 봅니다.

1. 좌측 메뉴 **검색 탐색기(Search explorer)** 를 클릭합니다.
2. 상단에서 인덱스가 `products-index` 로 선택됐는지 확인합니다.

### 3.1 기본 키워드 검색

검색창(쿼리 문자열)에 검색어를 입력하고 **검색(Search)** 을 누릅니다.

```
압소바
```

- 노트북의 `client.search(search_text="압소바")` 와 동일한 결과입니다.
- 결과 JSON의 `@search.score` 값이 **관련도 점수(BM25)** 입니다. 높을수록 관련성이 높습니다.

> 📸 **[이미지 첨부]** 검색 탐색기에서 "압소바" 검색 결과가 나온 화면

### 3.2 필터링 검색

검색 탐색기의 **쿼리 보기(Query view / JSON 보기)** 로 전환하면 필터를 함께 줄 수 있습니다.

```json
{
  "search": "선물",
  "filter": "category eq '유아동' and normal_price le 50000",
  "top": 5
}
```

- "유아동" 카테고리이면서 5만원 이하인 상품만 검색됩니다.
- 노트북의 `filtered_search()` 함수와 동일한 동작입니다.

> 📸 **[이미지 첨부]** 필터를 적용한 JSON 쿼리와 결과 화면

### 3.3 정렬 검색 (가격 낮은 순)

```json
{
  "search": "가방",
  "orderby": "normal_price asc",
  "top": 5
}
```

> 📸 **[이미지 첨부]** 가격 오름차순 정렬 결과 화면

### 3.4 패싯(집계) 검색 — 카테고리/브랜드 분포 보기

```json
{
  "search": "*",
  "facets": ["category", "brand"],
  "top": 0
}
```

- `category`, `brand` 별로 **몇 개의 상품이 있는지** 집계가 함께 반환됩니다.
- 노트북의 `faceted_search()` 와 동일합니다.

> 📸 **[이미지 첨부]** 패싯 집계 결과(JSON)에 카테고리별 개수가 표시된 화면

---

## ✅ 기획자용 체크리스트

화면(포털)만으로 키워드 검색을 끝까지 구성했는지 확인하세요:

- [ ] 인덱스 추가 화면에서 6개 필드를 직접 정의함
- [ ] `title`, `brand` 에 한국어 분석기(ko.lucene) 적용
- [ ] CSV를 Blob에 올리고 데이터 가져오기 마법사로 적재
- [ ] 인덱스 개요에서 문서 수 확인
- [ ] 검색 탐색기에서 기본 검색 / 필터 / 정렬 / 패싯 테스트

---

## 💡 정리

| 결론 | 설명 |
|------|------|
| **코드로도, 화면으로도 가능** | 키워드 검색은 SDK 코드뿐 아니라 **포털 UI 만으로도** 동일하게 구성할 수 있습니다. |
| **기획자 친화적** | 필드 설계(검색/필터/정렬/패싯)는 **체크박스**로, 검색 테스트는 **검색 탐색기**로 즉시 확인 가능합니다. |
| **빠른 프로토타이핑** | 개발 일정 전에 기획자가 직접 검색 시나리오를 검증해볼 수 있습니다. |

---

## 📚 참고 (Microsoft 공식 문서)

- [포털에서 인덱스 만들기 (Create an index)](https://learn.microsoft.com/azure/search/search-how-to-create-search-index)
- [데이터 가져오기 마법사 (Import data wizard)](https://learn.microsoft.com/azure/search/search-import-data-portal)
- [검색 탐색기 사용 (Search explorer)](https://learn.microsoft.com/azure/search/search-explorer)
- [언어 분석기 추가 (Language analyzers)](https://learn.microsoft.com/azure/search/index-add-language-analyzers)
