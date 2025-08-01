# AWS Bedrock API 호출 가이드

이 문서는 CMO AI 챗봇 시스템에서 AWS Bedrock의 언어 모델을 호출하는 방법과 주요 설정을 안내합니다.

## 1. 개요 및 주요 특징

- **목적**: 사용자의 질문을 AWS Bedrock의 Claude 3 Sonnet 모델로 보내 의도를 분석하고 적절한 답변을 생성합니다.
- **호출 방식**: 백엔드 API (`/v1/chat/completions`)를 통해 간접적으로 호출됩니다.

- **인증 방식 (매우 중요)**:
  - 본 시스템은 AWS IAM 역할(Role) 기반의 인증을 사용하지 않습니다.
  - 대신, **API Key(Bearer Token)**를 발급받아 코드에 직접 설정하여 인증합니다. 이는 현재 시스템의 인증 구조에 따른 것입니다.

- **호출 대상 (매우 중요)**:
  - 일반적인 `Model ID` (`anthropic.claude-3-sonnet...`)를 사용하지 않습니다.
  - Bedrock의 `On-Demand` 처리량 옵션 사용 시 발생하는 `ValidationException` 오류를 해결하기 위해, **추론 ARN(Inference ARN)**을 모델 식별자로 사용해야 합니다.

## 2. 핵심 설정

Bedrock 연동을 위한 주요 설정은 `/data/deploy/cmo-ai-be/config.py` 와 `/data/deploy/cmo-ai-be/app/config.py` 파일에 정의되어 있습니다.

- **추론 ARN (Inference ARN)**:
  - **설정값**: `arn:aws:bedrock:ap-northeast-2:584778534078:inference-profile/apac.anthropic.claude-3-sonnet-20240229-v1:0`
  - **변수명**: `BEDROCK_MODEL_ID` (config.py), `MODEL_ID` (app/config.py)
  - **사유**: `On-Demand` 처리량(throughput) 옵션 사용 시, `Model ID` 직접 호출이 `ValidationException`을 발생시키므로 ARN 방식 사용이 강제됩니다.

- **AWS 리전 (AWS Region)**:
  - **설정값**: `ap-northeast-2`
  - **변수명**: `AWS_REGION_NAME` (config.py), `AWS_REGION` (app/config.py)

## 3. API 호출 절차

### 1단계: 인증 토큰 발급

API를 호출하기 위해 먼저 로그인하여 Bearer 토큰을 발급받아야 합니다.

```bash
curl -X POST "http://localhost:8000/api/auth/login" \
-H "Content-Type: application/json" \
-d '{
  "email": "admin@company.com",
  "password": "admin123"
}'
```

### 2단계: 챗봇 API 호출

발급받은 `access_token`을 사용하여 챗봇 API에 질문을 전송합니다.

```bash
# YOUR_TOKEN을 1단계에서 발급받은 access_token으로 교체하세요.
export YOUR_TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

curl -X POST "http://localhost:8000/v1/chat/completions" \
-H "Authorization: Bearer $YOUR_TOKEN" \
-H "Content-Type: application/json" \
-d '{
    "model": "cmo-ai-model",
    "messages": [
        {"role": "user", "content": "세금계산서 발행은 어떻게 하나요?"}
    ]
}'
```

## 4. 테스트 방법

`verify_bedrock_call.py` 스크립트를 사용하면 위 절차를 자동으로 실행하여 Bedrock 연동 상태를 간편하게 확인할 수 있습니다.

- **위치**: `/data/deploy/cmo-ai-be/verify_bedrock_call.py`
- **실행 명령어**:
```bash
cd /data/deploy/cmo-ai-be
python3 verify_bedrock_call.py
```
