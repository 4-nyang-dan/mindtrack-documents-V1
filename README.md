# MindTrack – Full System

![Architecture Overview](./docs/architecture_V2.png)

> **MindTrack** 은 화면 캡처 → 이미지 샘플링/캐싱 → AI 분석 → 실시간 질문/답변 제공까지 이어지는 **동적 화면 맥락 분석 기반 예측형 AI 에이전트**입니다.  
> 프론트엔드(Electron), 백엔드(Spring Boot), AI 서버(FastAPI)로 구성되어 있으며, PostgreSQL · Redis · OpenAI API 등을 통합적으로 활용합니다.

---

## 📚 목차
- [📌 프로젝트 개요](#프로젝트-개요)
- [🖼 아키텍처 개요](#아키텍처-개요)
- [🧠 핵심 기술](#핵심-기술)
- [🧭 서비스 구성 & UX](#서비스-구성--ux)
- [🔐 보안 & 개인정보 처리](#보안--개인정보-처리)
- [🏢 비즈니스 모델 & 오픈소스 전략](#비즈니스-모델--오픈소스-전략)
- [📂 레포지토리 구성](#레포지토리-구성)
- [⚙️ 실행 흐름](#실행-흐름)
- [🚀 실행 방법](#실행-방법)

---

## 📌 프로젝트 개요

### 왜 MindTrack 인가? (문제 인식)
- **언어 의존적 AI 상호작용의 한계**: 기존 챗봇은 사용자가 화면 상황을 말로 풀어 설명해야 함 → **맥락 왜곡·정보 손실** 발생  
- **디지털 취약계층 접근성 문제**: 시각적 UI를 언어로 설명해야 하는 구조 자체가 **사용 장벽**이 됨  
- **‘행동 데이터’의 부재**: 화면만 이해해서는 부족. 사용자의 **의도/다음 행동**을 데이터화하고 최적화해야 함

### MindTrack의 목표
- **화면을 이해**하고 **행동을 예측/가이드**하여, 누구나 **목표를 효율적으로 수행**하도록 돕는 것
- **일반 사용자 모드**와 **취약계층 모드**를 제공하여, 상황·능력에 맞춘 맞춤형 경험을 지원

---

## 🖼 아키텍처 개요

```mermaid
flowchart TB
  R[Renderer - React]
  P[Preload - contextBridge]
  M[Main - Electron]
  BE[Backend - Spring Boot]
  AI[AI Server - FastAPI]
  DB[(Postgres)]
  REDIS[(Redis)]

  R -->|window.auth / window.api / window.capture| P
  P -->|IPC - ipcRenderer| M

  M -->|HTTP-upload + JWT| BE
  M -->|SSE - EventSource| BE

  BE -->|cache| REDIS
  BE -->|store| DB
  BE -->|call| AI
  AI -->|insert| DB
  DB -->|notify| BE

  BE -->|publish| M
  M -->|IPC - suggestions / heartbeat / error| R

```
---

## 🧠 핵심 기술
![Adaptive Image Sampling overview](docs/Adaptive Image Sampling.png)
### 1) Adaptive Image Sampling (중복 제거·비용 절감)

연속적인 이미지 스트림(Image Stream)이 입력되면, 1차로 Image Hash를 계산하고, 2차로 SSIM 유사도를 비교하여 95% 이상인 중복 프레임을 걸러냅니다. 이후 CNN Clustering을 통해 최종 대표 이미지를 선정하는 과정을 시각화했습니다.

* 구조적 유사도(SSIM)·이미지 해시(dHash) 로 연속 스크린샷의 중복 프레임을 실시간 필터링
* SSIM < 임계값(예: 0.95) 인 경우만 추출·전송 → 분석 비용과 지연 최소화


![Architecture Overview](docs/병렬 수집 분석 시스템.png)
### 2) 수집-분석 비동기 병렬 파이프라인
15초 단위의 [window 1]에서 이미지를 수집(Collection Image)하는 동시에, 수집이 완료된 이전 15초의 데이터는 [analysis queue]로 전달되어 병렬로 처리됩니다. 이 구조 덕분에 수집과 분석이 동시에 진행되어 사용자의 체감 응답성을 높입니다.

* 사용자별 윈도우(예: 15초) 로 프레임 수집 → Redis 큐 기반 분석 파이프라인 병렬 처리
* 수집과 분석의 동시 진행으로 체감 응답성을 확보


![Architecture Overview](docs/Multi Agent System 설계.png)
### 3) Multi-Agent System (화면이해 → 계획 → 액션가이드)

사용자의 목표(Text: Goal)는 Planner Agent로, 샘플링된 이미지는 Image Captioning LLM과 UI Analysis Agent로 전달됩니다. 이 에이전트들이 **Vector DB(Memory)**와 상호작용하며 계획을 수립하고(4. 현재 화면 기반 가이드 생성), 최종적으로 화면에 가이드(5. 화면 행동 가이드)를 제공하는 흐름을 나타냅니다.

```mermaid
flowchart LR
  A[Adaptive Sampler] --> B[UI Analysis Agent]
  B --> C[Image Captioning LLM]
  C --> D[Planner Agent (Ontology)] --> E[Screen Guide Agent]
  E --> F[(Vector DB Memory)]
  F --> D
  D --> G[Workflow Graph / Visualization]
```

* **UI Analysis Agent:** YOLOX(E2E 객체감지), EasyOCR로 UI 요소·텍스트 추출
* **Image Captioning LLM:** 화면 요약·상태 기술
* **Planner Agent:** 온톨로지 기반으로 목표 달성 경로·세부 단계 설계
    * *예시 온톨로지 축:* Medium(매개: 정부24) / Action(행동: 로그인, 이름 입력) / Subject(대상: 간편인증)
* **Screen Guide Agent:** 현재 단계의 다음 행동과 눌러야 할 UI를 제안(바운딩 박스·툴팁)

### 4) 온톨로지 기반 최적화 (일관된 행동 데이터 축적)

* 행동 임베딩과 유사도·클러스터링으로 반복 과업 최적 루트와 병목 지점을 발견
* 축적된 온톨로지 → 조직 지식 자산화 및 개인화 추천 고도화

---

## 🧭 서비스 구성 & UX

### 일반 사용자 모드 (업무 보조 / 목표 수행)

* 현재 화면 상황 자동 이해 → 예상 질문 선제 제시 → 화면 맥락형 Q&A
* 목표 설정/예측 → 세부 계획 생성 → 단계별 가이드(검색증강 + 화면기반 안내)
* 진행 이력 시각화 및 요약 제공

### 디지털 취약계층 모드 (접근성 특화)

* 음성/텍스트로 목표 설정, 시각적 가이드(바운딩 박스) 와 음성 안내(TTS) 제공
* “마우스만 따라가면” 가능한 친절한 단계별 안내 UX

---

## 🔐 보안 & 개인정보 처리

* 로컬(사용자 PC) 에서 우선 PII 탐지 후 블러 처리 → 외부 API 전송 전에 민감정보 최소화
* 사용 OSS 예: presidio-analyzer, pytesseract/EasyOCR, 사내 규칙 기반 마스킹 파이프라인
* 모듈별 라이선스 준수 및 데이터 거버넌스 기준 문서화

---

## 📂 레포지토리 구성

- **[mindtrack-frontend](https://github.com/4-nyang-dan/mindtrack-front)**  
  Electron + React 기반 UI  
  - 로그인/회원가입, 화면 캡처, SSE UI 표시  
  - 프론트에서 1차 SSIM 필터링

- **[mindtrack-backend](https://github.com/4-nyang-dan/mindtrack-backend)**  
  Spring Boot 기반 API 서버  
  - JWT 인증  
  - 스크린샷 샘플링(dHash/SSIM) + Redis 캐시  
  - Postgres NOTIFY + SSE Hub  

- **[mindtrack-ai](https://github.com/4-nyang-dan/mindtrack-ai)**  
  FastAPI 기반 AI 분석 서버  
  - OCR + PII 마스킹  
  - 이미지 설명, Embedding 저장/검색  
  - 행동/질문 예측 및 QA  

---

## ⚙️ 실행 흐름

1. **Frontend**
   - 사용자가 로그인 후 캡처 시작
   - 프론트에서 1차 SSIM 필터링 → 변화가 큰 이미지만 업로드

2. **Backend**
   - 업로드 이미지 샘플링(dHash/SSIM)
   - Redis 캐시 관리, DB 저장
   - 분석 대기 상태(PENDING) → AI 서버 요청

3. **AI Server**
   - OCR/PII, 이미지 설명, Embedding, 행동/질문 예측
   - 결과 DB 저장 → Postgres NOTIFY 발행

4. **Backend → Frontend**
   - PgSuggestionsListener → SSE Hub → SSE publish
   - Electron Main → Renderer → UI 표시

---

## 🔗 세부 문서 링크
- [Frontend README]
- [Backend README]
- [AI Server README]
  
[Frontend README]: https://github.com/4-nyang-dan/mindtrack-front#readme
[Backend README]:  https://github.com/4-nyang-dan/mindtrack-backend#readme
[AI Server README]: https://github.com/4-nyang-dan/mindtrack-ai#readme

---

## 🚀 실행 방법

> 이 프로젝트는 **루트 폴더**에서 AI 서버(FastAPI), Backend(Spring Boot), Redis, Postgres를 **Docker Compose**로 띄우고, **Frontend(Electron)** 는 로컬에서 실행합니다.
> Mindtrack_V1은 도커 + 로컬 환경으로, 추후 원격 서버 + 도커 및 pc 설치 응용 프로그램 .exe로 제공되는 점 양해 부탁드립니다.

### 0) 폴더 구성 (최종 목표)

```
mindtrack/
├─ mindtrack-ai/                 # mindtrack-ai (별도 리포 clone)
├─ mindtrack-backend/            # mindtrack-backend (별도 리포 clone)
├─ docker-compose.yml  # 루트 docker-compose (로컬에서 생성)
└─ .env                # 루트 환경파일 (로컬에서 생성)
```

---

### 1) 리포지토리/프로젝트 준비

1) 루트 폴더 생성  
2) 루트 안에 **AI**와 **Backend** 각각 clone
~~~bash
# 예시
mkdir mindtrack && cd mindtrack
git clone <ai-repo-url> ai
git clone <backend-repo-url> backend
~~~

3) **템플릿 복사 → 실제 파일 생성**  
- 레포에는 `docker-compose.example.yml`, `.env.example`(루트용), `ai/.env.example`(AI용)를 올려두세요.  
- 사용자는 아래처럼 복사해서 **개인 값**을 채웁니다.
~~~bash
cp docker-compose.example.yml docker-compose.yml
cp .env.example .env
cp ai/.env.example ai/.env
~~~

---

### 2) 루트 `.env` 템플릿 (복사 후 값 채우기)

~~~dotenv
# ── 공용 인프라 ──────────────────────────────
POSTGRES_HOST=postgres
POSTGRES_PORT=5432
POSTGRES_DB=mindtrack
POSTGRES_USER=mindtrack
POSTGRES_PASSWORD=super-secure-password

REDIS_HOST=redis
REDIS_PORT=6379

# ── 백엔드 환경 (스프링에서 참조) ─────────────
# application.properties에서 ${...}로 사용
SPRING_JWT_SECRET=secret-jwt-secret
SPRING_JWT_EXP_MS=3600000
~~~

---

### 3) AI용 `ai/.env` 템플릿 (복사 후 값 채우기)

~~~dotenv
# OpenAI
APP_HOST=0.0.0.0
APP_PORT=8000
APP_ENV=development

OPENAI_API_KEY=sk-****************************************

# OCR / PII (예: 시스템 경로 환경)
TESSERACT_PATH=/usr/bin/tesseract

# 벡터DB/파이프라인 파라미터 (필요 시)
VSTORE_PATH=faiss
VECTOR_DB_PATH=./vectorstore/vector_index.faiss

LOG_LEVEL=INFO
~~~

---

### 4) 루트 `docker-compose.yml` 템플릿

> 백엔드와 AI는 **로컬 clone 디렉토리(./backend, ./ai)** 를 **build context** 로 사용합니다.

~~~yaml
services:
  postgres:
    image: postgres:15
    container_name: mindtrack_postgres
    restart: always
    ports: ["5432:5432"]
    env_file:
      - ./.env
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks: [mindtrack_net]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U 4nyangdan -d mindtrack"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7
    container_name: mindtrack_redis
    restart: always
    ports: ["6379:6379"]
    networks: [mindtrack_net]
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5

  spring-backend:
    build: ./mindtrack-backend
    container_name: mindtrack_spring
    ports: ["8080:8080"]
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    env_file:
      - ./.env
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
      SPRING_DATA_REDIS_HOST: ${REDIS_HOST}
      SPRING_DATA_REDIS_PORT: ${REDIS_PORT}
      AUTH_JWT_SECRET: ${SPRING_JWT_SECRET}
      AUTH_JWT_EXP_MS: ${SPRING_JWT_EXP_MS}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8080/actuator/health"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks: [mindtrack_net]

  fastapi-ai:
    build: ./mindtrack-ai
    container_name: mindtrack_ai
    ports: ["8000:8000"]
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    env_file:
      - ./.env
    networks: [mindtrack_net]

volumes:
  pgdata:

networks:
  mindtrack_net:
    driver: bridge
~~~

---

### 5) 컨테이너 기동

~~~bash
# 루트 디렉토리에서
docker compose up --build
# 또는 (Docker Compose v1)
# docker-compose up --build
~~~

> 성공하면:  
> - 백엔드: http://localhost:${BACKEND_PORT} (기본 8080)  
> - AI: http://localhost:${AI_PORT} (기본 8000)  
> - Redis: localhost:6379, Postgres: localhost:5432

---

### 6) 프론트엔드 실행 (별도 clone)

1) 프론트 프로젝트 clone → 의존성 설치  
2) 타입스크립트 메인 빌드(선택) → 개발 서버 + 일렉트론 실행

~~~bash
# 프론트 폴더에서
npm install
npm run build-main   # (선택) main.ts → main.js 트랜스파일

# Windows (cmd)
set BROWSER=none&& npm start
# Windows (PowerShell)
$env:BROWSER="none"; npm start
# macOS/Linux
BROWSER=none npm start

# React 개발 서버가 뜬 뒤
npx electron .
~~~

> Electron 앱이 뜨면 **로그인 → 캡처 시작** 후, 실시간 Q/A UI가 나타납니다.

---
