# NDHS API

> 남도인(https://ndhs.app) Flask 기반 RESTful API

## 🚀 주요 기능

- **게시판 시스템**: 게시글 작성/조회 및 댓글 기능
- **관리자 승인 시스템**: 게시글/댓글 승인/반려 기능
- **좋아요 기능**: IP 기반 중복 방지
- **건조기 현황 조회**: 실시간 건조기 사용 현황 제공
- **키셋 페이지네이션**: 효율적인 목록 조회
- **Azure Cosmos DB**: NoSQL 데이터베이스 사용
- **AWS Lambda**: 서버리스 배포

## 📋 API 명세

### 게시판 API

| 기능 | 메서드 | 엔드포인트 | 설명 |
|------|--------|------------|------|
| 글 작성 | POST | `/boards/<board_id>` | 게시글 작성 (기본 미승인) |
| 글 목록 조회 | GET | `/boards/<board_id>` | 게시글 목록 조회 (페이징) |
| 글 상세 조회 | GET | `/boards/<board_id>/<post_id>` | 특정 게시글 상세 조회 |
| 댓글 작성 | POST | `/boards/<board_id>/<post_id>/comments` | 댓글 작성 (기본 미승인) |
| 댓글 목록 조회 | GET | `/boards/<board_id>/<post_id>/comments` | 댓글 목록 조회 (페이징) |
| 좋아요 | POST | `/boards/<board_id>/<post_id>/like` | 게시글 좋아요 (IP당 1회) |

### 관리자 API

| 기능 | 메서드 | 엔드포인트 | 인증 |
|------|--------|------------|------|
| 글 승인/반려 | POST | `/admin/boards/<board_id>/<post_id>/accept` | `X-Admin-Token` |
| 댓글 승인/반려 | POST | `/admin/boards/<board_id>/<post_id>/comments/<comment_id>/accept` | `X-Admin-Token` |
| 대기 글 목록 | GET | `/admin/boards/<board_id>/pending` | `X-Admin-Token` |
| 특정 글의 대기 댓글 | GET | `/admin/boards/<board_id>/<post_id>/comments/pending` | `X-Admin-Token` |
| 게시판의 모든 대기 댓글 | GET | `/admin/boards/<board_id>/comments/pending` | `X-Admin-Token` |

### 기타 API

| 기능 | 메서드 | 엔드포인트 | 설명 |
|------|--------|------------|------|
| 건조기 현황 | GET | `/laundry/<sex>` | 건조기 사용 현황 조회 (m/f) |
| 내 정보 조회 | GET | `/info/my` | 사용자 정보 조회 |

## 📝 요청/응답 예시

### 게시글 작성

```http
POST /boards/free
Content-Type: application/json

{
  "title": "제목",
  "content": "내용",
  "user_id": "사용자명",
  "tag": "일반"
}
```

**응답:**
```json
{
  "message": "Post created",
  "post_id": "123"
}
```

### 게시글 목록 조회

```http
GET /boards/free?last=100&last_created_at=2025-12-09T10:00:00.000Z
```

**응답:**
```json
{
  "posts": [
    {
      "id": "100",
      "post_id": "100",
      "board_id": "free",
      "title": "제목",
      "content": "내용",
      "user_id": "작성자",
      "created_at": "2025-12-09T10:00:00.000Z",
      "isAccept": true,
      "likes": 5
    }
  ],
  "last": "99",
  "last_created_at": "2025-12-09T09:50:00.000Z"
}
```

### 관리자 글 승인

```http
POST /admin/boards/free/123/accept
X-Admin-Token: your-admin-token
Content-Type: application/json

{
  "accept": true
}
```

**응답:**
```json
{
  "post_id": "123",
  "isAccept": true
}
```

## 🗄️ 데이터베이스 구조

### Azure Cosmos DB 컨테이너

| 컨테이너 | 파티션 키 | 문서 ID | 설명 |
|----------|-----------|---------|------|
| `posts` | `/board_id` | `post_id` | 게시글 저장 |
| `comments` | `/post_id` | `comment_id` | 댓글 저장 |
| `counters` | `/board_id` | `board_id` | 게시판별 글번호 카운터 |
| `likes` | `/post_id` | `ip` | 좋아요 IP 중복 방지 |

### 게시글 문서 구조

```json
{
  "id": "post_id",
  "post_id": "123",
  "board_id": "free",
  "title": "제목",
  "content": "내용",
  "user_id": "작성자",
  "created_at": "2025-12-09T10:00:00.000Z",
  "ip": "1.2.3.4",
  "isAccept": false,
  "likes": 0,
  "tag": "일반"
}
```

### 댓글 문서 구조

```json
{
  "id": "comment_id",
  "comment_id": "uuid",
  "post_id": "123",
  "board_id": "free",
  "content": "댓글 내용",
  "user_id": "작성자",
  "created_at": "2025-12-09T10:05:00.000Z",
  "ip": "1.2.3.4",
  "isAccept": false
}
```

## 🔐 인증 및 보안

### 공지 게시판 (`notice`)
- 글 작성 시 `password` 필드 필수
- 환경변수 `NOTICE_PW`와 일치해야 작성 가능
- 작성 즉시 승인 처리 (`isAccept=true`)

### 일반 게시판
- 글/댓글 작성 시 `isAccept=false`로 저장
- 관리자 승인 후 노출
- XSS 방지를 위한 HTML 이스케이프 처리

### 관리자 인증
- 헤더: `X-Admin-Token: <ADMIN_TOKEN>`
- 또는 쿼리: `?adminToken=<ADMIN_TOKEN>`

### CORS
- 허용 도메인: `https://ndhs.app`

## 📄 페이지네이션

### 키셋 페이지네이션 방식
- **게시글**: `created_at` 기준 내림차순 (최신순)
- **댓글**: `created_at` 기준 오름차순 (오래된순)

### 사용법
```http
# 첫 페이지
GET /boards/free

# 다음 페이지 (last와 last_created_at 사용)
GET /boards/free?last=100&last_created_at=2025-12-09T10:00:00.000Z
```

## 🛠️ 기술 스택

- **언어**: Python 3.13
- **프레임워크**: Flask
- **ASGI 어댑터**: Mangum (AWS Lambda 호환)
- **데이터베이스**: Azure Cosmos DB for NoSQL
- **배포**: AWS Lambda + Serverless Framework
- **주요 라이브러리**:
  - `flask-cors`: CORS 처리
  - `azure-cosmos`: Cosmos DB SDK
  - `requests`: HTTP 요청
  - `python-dotenv`: 환경변수 관리

## 🚀 로컬 개발

### 필수 요구사항
- Python 3.13+
- Azure Cosmos DB 계정
- Node.js (Serverless Framework)

### 설치 및 실행

1. **의존성 설치**
```bash
pip install -r requirements.txt
npm install
```

2. **환경변수 설정** (`.env`)
```env
COSMOS_URI=https://your-account.documents.azure.com:443/
COSMOS_KEY=your-cosmos-primary-key
COSMOS_DB_NAME=ndhs
ADMIN_TOKEN=your-admin-token
NOTICE_PW=your-notice-password

# 건조기 API (선택)
LAUNDRY_API=https://laundry-api-url
LAUNDRY_AGENT=user-agent-string
LAUNDRY_REFERER=https://referer-url
LAUNDRY_AUTH=access-token
LAUNDRY_REFRESH_TOKEN=refresh-token
LAUNDRY_CACHE_TTL=60
```

3. **로컬 서버 실행**
```bash
python app.py
```

서버가 `http://localhost:5000`에서 실행됩니다.

## 📦 배포 (AWS Lambda)

### Serverless Framework 배포

```bash
# 배포
serverless deploy

# 특정 스테이지 배포
serverless deploy --stage production

# 로그 확인
serverless logs -f api -t
```

### Lambda 환경변수 설정

AWS Lambda 콘솔 또는 `serverless.yml`에서 환경변수를 설정하세요:
- `COSMOS_URI`
- `COSMOS_KEY`
- `COSMOS_DB_NAME`
- `ADMIN_TOKEN`
- `NOTICE_PW`
- 기타 필요한 환경변수

### 아키텍처
```
Client (https://ndhs.app)
    ↓
API Gateway (HTTP API)
    ↓
AWS Lambda (Python 3.13)
    ↓
Azure Cosmos DB
```

## 🔧 주요 기능 상세

### 1. 게시글 카운터
- 게시판별로 자동 증가하는 글번호
- ETag를 이용한 낙관적 동시성 제어
- 최대 5회 재시도

### 2. 좋아요 시스템
- IP 기반 중복 방지
- 트랜잭션 보장 (좋아요 기록 + 카운터 증가)
- ETag를 통한 동시성 처리

### 3. 건조기 현황
- 외부 API 연동
- 인메모리 캐시 (TTL: 60초)
- 토큰 자동 갱신

### 4. 관리자 승인 시스템
- 글/댓글 일괄 승인/반려
- 대기 목록 조회
- 반려 시간 기록

## 📊 에러 처리

API는 다음과 같은 HTTP 상태 코드를 반환합니다:

- `200`: 성공
- `201`: 리소스 생성 성공
- `400`: 잘못된 요청
- `403`: 권한 없음
- `404`: 리소스 없음
- `500`: 서버 오류
- `502`: 외부 API 오류

## 🤝 기여

이 프로젝트는 남도인 (https://ndhs.app) 을 위한 백엔드 API입니다.
