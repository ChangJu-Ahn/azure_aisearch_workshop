# 3️⃣ 벡터 검색 – Azure 포털에서 직접 해보기 (No-Code Guide)

> 🎯 **이 문서의 목적**
> 앞선 노트북 실습은 **코드**로 벡터 필드를 추가하고, OpenAI 임베딩을 생성해 업로드하고, 벡터로 검색했습니다.
> 같은 작업을 **Azure 포털 화면(UI)** 에서 클릭만으로 할 수 있습니다.
> 특히 포털의 **"데이터 가져오기 및 벡터화(Import and vectorize data)" 마법사**를 쓰면
> **임베딩 생성 코드 없이도** 텍스트가 자동으로 벡터로 변환되어 인덱스에 들어갑니다.

---

## 📌 한눈에 보기: 코드 vs 포털

| 단계 | 노트북(코드)에서 한 일 | 포털(화면)에서 하는 방법 |
|------|----------------------|------------------------|
| 벡터 필드 추가 | `01-update_index.ipynb` 에서 `SearchField(vector_search_dimensions=3072 …)` | 인덱스 편집 화면에서 벡터 필드 추가 **또는** 벡터화 마법사가 자동 생성 |
| 임베딩 생성·업로드 | `02-upload_vectors.ipynb` 에서 OpenAI `embeddings.create()` | **데이터 가져오기 및 벡터화 마법사**가 자동으로 임베딩 생성 |
| 벡터 검색 | `03-vector_search.ipynb` 에서 `VectorizedQuery` | **검색 탐색기**에서 벡터/텍스트 쿼리 실행 |

> 💡 핵심: 노트북에서는 **임베딩을 직접 코드로 만들었지만**, 포털 마법사는 **통합 벡터화(Integrated Vectorization)** 로 이 과정을 자동화합니다.

---

## 0. 시작 전 준비

- **Azure AI Search 리소스** (키워드 실습에서 쓰던 것과 동일)
- **Azure OpenAI 리소스** 와 임베딩 모델 배포
  - 노트북과 동일하게 **`text-embedding-3-large`** (3072차원) 모델을 배포해 둡니다.
- `../00-data/sample_data.csv` 가 **Blob Storage 컨테이너**에 업로드되어 있어야 합니다. (2번 키워드 가이드 2.1 참고)

> 📸 **[이미지 첨부]** Azure OpenAI 에 text-embedding-3-large 모델이 배포된 화면

---

## 1. 데이터 가져오기 및 벡터화 마법사 (코드: `01`+`02` 노트북 대체)

이 마법사 하나가 **벡터 필드 생성 + 임베딩 자동 생성 + 데이터 적재**를 한 번에 처리합니다.

### 1.1 마법사 시작

1. **Azure AI Search 리소스** 로 이동합니다.
2. 상단 **데이터 가져오기 및 벡터화(Import and vectorize data)** 를 클릭합니다.

> 📸 **[이미지 첨부]** 리소스 개요 상단의 "데이터 가져오기 및 벡터화" 버튼 위치

### 1.2 데이터 원본 연결

1. 데이터 원본으로 **Azure Blob Storage** 를 선택합니다.
2. `sample_data.csv` 가 들어있는 컨테이너를 지정합니다.
3. 파싱 모드를 **CSV(구분된 텍스트)** 로 설정하고 첫 행을 헤더로 사용합니다.

> 📸 **[이미지 첨부]** 데이터 원본으로 Blob 컨테이너를 선택한 화면

### 1.3 벡터로 만들 컬럼(텍스트) 선택 — 가장 중요

노트북에서는 **`content_vector`(제품 정보 임베딩)**, **`review_vector`(리뷰 임베딩)** 두 벡터 필드를 3072차원으로 만들었습니다.
포털에서는 **어떤 텍스트 컬럼을 벡터화할지** 고르면, 마법사가 벡터 필드를 자동으로 만들어 줍니다.

- **벡터화할 콘텐츠 필드**: `title` (또는 제품 정보) → `content_vector` 역할
- 리뷰까지 의미 검색하려면 `review` 컬럼도 함께 벡터화 → `review_vector` 역할

> 💡 노트북의 `content_vector` 는 "제품명 + 브랜드 + 카테고리"를, `review_vector` 는 "리뷰 텍스트"를 임베딩한 것입니다. 포털에서도 같은 의도로 컬럼을 고르면 됩니다.

> 📸 **[이미지 첨부]** 벡터화할 텍스트 필드를 선택하는 마법사 화면

### 1.4 임베딩 모델(벡터라이저) 연결

1. **임베딩 모델(Vectorizer)** 단계에서 **Azure OpenAI** 를 선택합니다.
2. 준비해 둔 **`text-embedding-3-large`** 배포를 지정합니다.
   - 이 모델은 **3072차원** 벡터를 생성합니다 (노트북의 `vector_search_dimensions=3072` 와 동일).
3. 마법사가 벡터 필드의 차원을 **자동으로 채워줍니다.**

> 📸 **[이미지 첨부]** 벡터라이저로 Azure OpenAI / text-embedding-3-large 를 연결하는 화면

### 1.5 벡터 검색 알고리즘 확인 (HNSW)

마법사는 기본적으로 **HNSW 알고리즘**과 **코사인 유사도(cosine)** 로 벡터 프로필을 구성합니다.
이는 노트북의 `hnsw-config` + `hnsw-profile`(metric: cosine) 과 동일한 설정입니다.

| 노트북 설정 | 포털 기본값 |
|------------|------------|
| HNSW 알고리즘 (`hnsw-config`) | HNSW (빠른 근사 검색) |
| 유사도 측정 = cosine | cosine |
| 차원 = 3072 | 모델에 맞춰 자동 |

> 💡 노트북에는 정확도 우선의 **Exhaustive KNN** 프로필도 있었습니다. 포털 기본은 HNSW이며, 인덱스 편집 화면에서 KNN 프로필을 추가로 만들 수도 있습니다.

> 📸 **[이미지 첨부]** 벡터 프로필(HNSW/cosine) 설정이 표시된 화면

### 1.6 인덱스 이름 지정 후 만들기

인덱스 이름을 지정(예: `products-index`)하고 **만들기(Create)** 를 누르면:

1. 인덱스에 벡터 필드가 생성되고
2. 선택한 텍스트가 자동으로 임베딩되어
3. 문서가 인덱스에 적재됩니다.

> 📸 **[이미지 첨부]** 마법사 마지막 요약/만들기 화면

---

## 2. 벡터 검색 테스트 (코드: `03-vector_search.ipynb` 대체)

### 2.1 검색 탐색기에서 의미 기반 검색

1. 좌측 **검색 탐색기(Search explorer)** 를 엽니다.
2. 통합 벡터화로 인덱스를 만들었다면, **자연어 텍스트를 입력**하기만 해도
   포털이 그 텍스트를 자동으로 임베딩하여 **벡터 검색**을 수행합니다.

예시 쿼리(노트북과 동일한 의미 검색 시나리오):

```
가벼운 외출복
```

```
선물하기 좋은 제품
```

- 키워드가 정확히 일치하지 않아도 **의미적으로 유사한 상품**(바람막이, 기프트 세트 등)이 검색됩니다.
- 노트북의 `VectorizedQuery(... fields="content_vector")` 와 동일한 효과입니다.

> 📸 **[이미지 첨부]** 검색 탐색기에서 자연어로 벡터 검색한 결과 화면

### 2.2 (참고) JSON 쿼리로 직접 벡터 검색

검색 탐색기의 **JSON / 쿼리 보기**에서는 벡터 쿼리를 직접 줄 수도 있습니다. (통합 벡터화 사용 시 `text` 만으로 충분)

```json
{
  "search": null,
  "vectorQueries": [
    {
      "kind": "text",
      "text": "운동할 때 입기 좋은 옷",
      "fields": "content_vector",
      "k": 5
    }
  ]
}
```

- `k` 값은 노트북의 `k_nearest_neighbors` 와 동일하게 **가져올 결과 개수**를 의미합니다.

> 📸 **[이미지 첨부]** JSON 벡터 쿼리와 결과 화면

---

## ✅ 기획자용 체크리스트

- [ ] Azure OpenAI 에 `text-embedding-3-large` 배포 확인
- [ ] CSV를 Blob에 업로드
- [ ] "데이터 가져오기 및 벡터화" 마법사로 벡터 필드 자동 생성
- [ ] 벡터라이저(Azure OpenAI)와 모델 연결 → 3072차원 자동 설정
- [ ] 검색 탐색기에서 자연어로 의미 검색 테스트

---

## 💡 정리

| 결론 | 설명 |
|------|------|
| **임베딩 코드 불필요** | 포털의 통합 벡터화가 임베딩 생성을 자동 처리합니다. |
| **노트북과 동일한 구성** | HNSW + cosine, 3072차원 등 노트북 설정을 화면에서 그대로 재현합니다. |
| **의미 검색을 즉시 체험** | 기획자가 자연어 쿼리로 벡터 검색 품질을 직접 확인할 수 있습니다. |

---

## 📚 참고 (Microsoft 공식 문서)

- [데이터 가져오기 및 벡터화 마법사](https://learn.microsoft.com/azure/search/search-get-started-portal-import-vectors)
- [통합 벡터화(Integrated Vectorization) 개요](https://learn.microsoft.com/azure/search/vector-search-integrated-vectorization)
- [벡터 검색 개요](https://learn.microsoft.com/azure/search/vector-search-overview)
- [검색 탐색기에서 벡터 쿼리](https://learn.microsoft.com/azure/search/search-explorer)
