# Foundry IQ Agentic Retrieval

> **이 실습은 별도 레포지토리로 이동되었습니다.**  
> 👉 [ignite25-LAB511-build-agentic-knowledge-bases-next-level-rag-with-azure-ai-search](https://github.com/ChangJu-Ahn/ignite25-LAB511-build-agentic-knowledge-bases-next-level-rag-with-azure-ai-search)

---

## Agentic Retrieval이란?

**Agentic Retrieval**은 Azure AI Foundry의 차세대 검색 기술로, LLM이 복잡한 사용자 질의를 자동으로 분해하고 최적의 검색 전략을 수립하여 실행하는 방식입니다.

### 기존 RAG와의 차이점

| 구분 | 기존 RAG | Agentic Retrieval |
|------|----------|-------------------|
| **쿼리 처리** | 사용자 질의 그대로 검색 | LLM이 쿼리를 여러 서브 쿼리로 분해 |
| **검색 실행** | 단일 검색 요청 | 병렬/순차 검색 계획 수립 및 실행 |
| **결과 통합** | 단순 Reranking | 여러 검색 결과를 LLM이 종합 분석 |
| **적응성** | 고정된 검색 로직 | 쿼리 의도에 따라 동적으로 전략 변경 |

### 핵심 개념

#### 1. Knowledge Base
- Azure AI Search 인덱스를 LLM이 사용할 수 있는 지식 소스로 등록
- 여러 인덱스를 하나의 Knowledge Base로 통합 가능
- 각 인덱스의 스키마, 필터링 조건 등을 LLM이 이해

#### 2. Query Planning
- 복잡한 질의를 여러 개의 서브 쿼리로 분해
- 예: "2024년 출시된 가벼운 러닝화 중 5만원 이하 제품" 
  - → 서브쿼리1: category='스포츠/레져' AND price le 50000
  - → 서브쿼리2: 벡터 검색으로 "가벼운 러닝화" 의미 유사도
  - → 서브쿼리3: 필터 조건으로 시간 범위 적용

#### 3. Reasoning Effort
검색 품질과 응답 속도 간 균형을 조절하는 설정:

| Effort | 특징 | 사용 시나리오 |
|--------|------|--------------|
| **minimal** | 단순 검색, 빠른 응답 | 단순 키워드 검색 |
| **low** | 기본적인 쿼리 분해 | 일반적인 질의응답 |
| **medium** | 복잡한 쿼리 플래닝 | 다단계 추론 필요 시 |

#### 4. Activity 분석
- LLM이 어떻게 쿼리를 분해했는지 추적
- 각 서브 쿼리의 실행 계획과 결과 확인
- 토큰 사용량, 응답 시간 등 메트릭 수집

### 왜 Agentic Retrieval이 필요한가?

#### 전통적인 RAG의 한계

**1. 고정된 검색 파이프라인**
- 개발자가 검색 로직을 미리 하드코딩해야 함
- 사용자 질의 유형이 바뀌면 코드 수정 필요
- 복잡한 질의를 처리하기 위해 수많은 조건문 필요

**2. 단일 검색 방식의 제약**
- 키워드, 벡터, 하이브리드 중 하나만 선택
- 쿼리 특성에 따라 최적 방식이 다른데 동적 전환 불가
- 여러 검색 조건을 결합하려면 복잡한 로직 구현 필요

**3. 필터링과 메타데이터 활용의 어려움**
- "5만원 이하", "최근 1년 이내" 같은 조건을 수동으로 파싱
- 암묵적인 조건(예: "겨울 자켓" → 계절 필터)을 개발자가 추론해야 함
- 다중 조건 조합 시 복잡도 기하급수적 증가

**4. 멀티턴 대화 처리의 한계**
- 이전 대화 맥락을 유지하면서 검색하려면 상태 관리 복잡
- "그것보다 저렴한 것"처럼 참조가 있는 질의 처리 어려움
- 대화 흐름에 따라 검색 전략을 바꾸기 어려움

#### Agentic Retrieval이 해결하는 문제

**1. 자율적인 검색 계획 수립**
- LLM이 사용자 의도를 파악하여 최적의 검색 전략 자동 선택
- 복잡한 질의를 여러 단계의 서브 쿼리로 분해하여 처리
- 쿼리 유형에 따라 키워드/벡터/하이브리드를 동적으로 선택

**2. 자연어 조건의 자동 변환**
- "저렴한", "최신", "인기 있는" 같은 표현을 자동으로 필터/정렬로 변환
- 날짜, 가격 범위 등을 OData 필터 구문으로 자동 생성
- 도메인 지식을 활용한 암묵적 조건 추론 (예: 겨울 → 방한 카테고리)

**3. 컨텍스트 기반 검색**
- 대화 히스토리를 자동으로 반영하여 검색 정확도 향상
- "그것"처럼 이전 결과를 참조하는 질의도 자연스럽게 처리
- 사용자가 검색 범위를 좁히거나 넓히는 의도 자동 파악

**4. 실시간 전략 조정**
- 첫 검색 결과가 불충분하면 자동으로 다른 방식 시도
- 병렬 검색으로 여러 소스를 동시에 탐색하여 속도 향상
- 검색 품질과 응답 속도 간 균형을 reasoning effort로 조절

### 실무 활용 시나리오

#### 시나리오 1: 전자상거래 상품 검색
**사용자 질의:** "아이 생일 선물로 좋은 유아용품 추천해줘. 안전 인증 받은 것으로"

**LLM의 자동 처리:**
1. 카테고리 필터: `category eq '유아동'`
2. 벡터 검색: "생일 선물로 좋은" 의미 유사도
3. 리뷰 분석: "안전", "인증" 키워드 포함 상품 우선
4. 결과 통합: 종합 평가로 최적 추천

#### 시나리오 2: 기술 문서 검색
**사용자 질의:** "Python에서 Azure AI Search 벡터 검색 구현하는 방법"

**LLM의 자동 처리:**
1. 프로그래밍 언어 식별: Python
2. 기술 키워드 추출: Azure AI Search, vector search
3. 코드 예제 포함 문서 우선 검색
4. 단계별 가이드와 API 레퍼런스 결합

---

## 실습 진행 방법

이 워크샵의 01-07 모듈을 완료하여 Azure AI Search 인덱스와 데이터가 준비된 상태에서, Agentic Retrieval 실습을 진행할 수 있습니다.

**실습 레포지토리로 이동:**
👉 [ignite25-LAB511-build-agentic-knowledge-bases-next-level-rag-with-azure-ai-search](https://github.com/ChangJu-Ahn/ignite25-LAB511-build-agentic-knowledge-bases-next-level-rag-with-azure-ai-search)

### 실습 내용
- Knowledge Base 생성 및 등록
- 복잡한 쿼리 처리 테스트
- Reasoning Effort별 성능 비교
- Activity 분석 및 최적화

---

## 참고 자료

- [Azure AI Foundry - Agentic Retrieval 개요](https://learn.microsoft.com/azure/ai-foundry/agentic-retrieval-overview)
- [Knowledge Base 구성 가이드](https://learn.microsoft.com/azure/ai-foundry/knowledge-base-setup)
- [Ignite 2025 LAB511 세션](https://github.com/microsoft/ignite25-LAB511-build-agentic-knowledge-bases-next-level-rag-with-azure-ai-search)

---

## 심층 기술 보고서 (L500)

이 모듈의 Foundry IQ / Agentic Retrieval을 **Copilot Retrieval API · Work IQ MCP · WorkIQ 플랫폼**과 함께 라이브로 실측·교차검증한 기술 심층 보고서입니다. Knowledge Source `kind`(`remoteSharePoint`·`workIQ`·`indexedSharePoint`)별 백엔드 매핑, 인증/OBO/FIC 체인, cross-lingual·의미검색 실증, `dataSource` 커버리지 등을 다룹니다.

👉 [WorkIQ · Work IQ MCP · Copilot Retrieval API — Graph API 대비 기술 심층 보고서](./WorkIQ-Graph-Retrieval-Technical-Report-L500.md)
