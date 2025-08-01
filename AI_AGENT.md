# AI Agent 챗봇 설계 문서 (최신 반영본)

## 1. 사용자 질문 처리 흐름

- **시작점**: `/data/deploy/cmo-ai-be/backend/api.py`의 `@app.post("/v1/chat/completions")`
- **메인 로직**: `/data/deploy/cmo-ai-be/backend/llm_service.py`의 `generate_response`

`generate_response` 함수는 아래와 같은 흐름으로 여러 에이전트를 순차적으로 호출합니다.

### 전체 처리 순서

1.  **[1단계: 카테고리 분류 및 보안 감사]**
    - `reflection_agent.py`의 `categorize_query` 함수를 실행합니다.
    - LLM을 이용하여 질문의 카테고리, 신뢰도, 보안(감사) 여부 등 구조화된 데이터를 받아옵니다. (현재는 안정성 문제로 규칙 기반으로 동작)

2.  **[2단계: FAQ 문서 검색 (RAG)]**
    - `rag_service.py`를 호출하여, 사용자의 질문과 관련된 FAQ 문서를 검색합니다.
    - **임베딩 방식**: 백엔드 서비스에 내장된 한국어 특화 모델(**`jhgan/ko-sroberta-multitask`**)을 사용하여 질문을 직접 벡터로 변환합니다. (외부 LLM 서버 의존성 제거)
    - **검색**: 생성된 벡터를 사용하여 ChromaDB에서 코사인 유사도 기반으로 가장 관련성 높은 문서를 찾습니다.

3.  **[3단계: 핵심 답변 생성]**
    - `inquiry_agent.py`의 `process` 함수를 호출합니다.
    - **컨텍스트 주입**: RAG 검색 결과를 **사용자 프롬프트(User Prompt)**에 명확하게 포함시켜, LLM이 검색된 정보를 바탕으로 질문에 답변하도록 지시합니다.
    - 1단계에서 얻은 카테고리, 신뢰도를 종합하여 구체적이고 전문가적인 답변을 생성합니다.

4.  **[4단계: 답변 명확성 확인]**
    - `clarification_agent.py`의 `process` 함수를 호출합니다.
    - **호출 조건**: RAG 검색 결과가 없거나 신뢰도가 낮은 경우.
    - 위 조건에 부합하면, `ClarificationAgent`가 추가 설명 요청 또는 명확화 질문을 생성하여 기존 답변을 덮어씁니다.

5.  **[최종 반환]**
    - 최종 생성된 답변을 `api.py`로 반환합니다.

---

## 2. 각 에이전트 설명

### 2.1 ReflectionAgent (카테고리 분류/보안 감사 에이전트)
- **파일 위치**: `/data/deploy/cmo-ai-be/backend/reflection_agent.py`
- **주요 역할**: AWS Bedrock LLM에게 카테고리 분류, 신뢰도, 보안 이슈를 질의하여 구조화된 데이터를 생성합니다.
- **반환값 (예시)**:
  ```json
  {
    "category": "회계",
    "confidence": 0.85,
    "reasoning": "사용자가 '세금계산서'와 '발행'이라는 단어를 사용했기 때문에 회계 관련 질문일 확률이 높음.",
    "security_flags": {
      "sensitive_data_detected": false
    }
  }
  ```
- **중요성**: 반환된 구조화 데이터는 이후 `InquiryAgent`와 `ClarificationAgent`의 동작을 결정하는 핵심 기준이 됩니다.

### 2.2 InquiryAgent (문의 에이전트)
- **파일 위치**: `/data/deploy/cmo-ai-be/backend/inquiry_agent.py`
- **주요 역할**: `ReflectionAgent`의 분석 결과(카테고리, 신뢰도)와 `RAGService`의 검색 결과를 바탕으로 구체적이고 전문가 수준의 답변을 생성합니다.

### 2.3 ClarificationAgent (명확화/추가 질문 에이전트)
- **파일 위치**: `/data/deploy/cmo-ai-be/backend/clarification_agent.py`
- **주요 역할**: 답변의 확신도가 낮거나 사용자의 질문이 불명확할 때 추가 질의 또는 설명 요청을 생성합니다.
- **호출 조건**:
    - RAG 검색 결과의 메타데이터에 `"클라리피케이션":"Y"`가 포함된 경우
    - 카테고리 신뢰도가 60% 이하인 경우
- **행동**: 기존 답변 대신, "조금 더 구체적으로 질문해 주세요." 와 같이 되묻는 메시지를 생성합니다.

---

## 3. 에이전트 동작 흐름 요약

```mermaid
flowchart TD
    A[사용자 질문 수신\n(api.py)] --> B[ReflectionAgent:\n카테고리/보안 분석];
    B --> C[RAGService:\nFAQ 문서 검색];
    C --> D[InquiryAgent:\n전문가 답변 생성\n(카테고리 + FAQ 사용)];
    D --> E{ClarificationAgent 호출 조건?};
    E --"Yes\n(신뢰도 < 60% 또는\n클라리피케이션='Y')"--> F[ClarificationAgent:\n명확화/추가 질문 생성];
    F --> Z[최종 반환];
    E --No--> Z;
```

---

## 4. 에이전트 판단 및 실행 기준 테이블

| 에이전트 (Agent) | 판단/호출 기준 | 설명 |
| :--- | :--- | :--- |
| **ReflectionAgent** | **가장 먼저 실행**<br/>문장 전체 의미 분석 | AWS Bedrock LLM을 통해 카테고리, 신뢰도, 보안 관련 정보 추출 |
| **RAGService** | `ReflectionAgent` 실행 후 | 질문과 연관된 FAQ 문서를 Vector DB에서 검색 |
| **InquiryAgent** | `ReflectionAgent` 및 `RAGService` 실행 후 | 구체적이고 전문가 수준의 답변 제공 |
| **ClarificationAgent**| - RAG FAQ 메타데이터에 `"클라리피케이션":"Y"`<br/>- 또는 카테고리 신뢰도 60% 이하 | 불확실한 상황에서 명확화/추가 질문 생성
