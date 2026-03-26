# Local Development Setup with Docker PostgreSQL

Lakebase 대신 로컬 Docker PostgreSQL을 사용하여 Builder App을 실행하는 가이드입니다.

---

## Prerequisites

- [uv](https://github.com/astral-sh/uv) - Python package manager
- [Node.js 18+](https://nodejs.org/)
- [Docker](https://www.docker.com/) - PostgreSQL 컨테이너용
- [Databricks CLI](https://docs.databricks.com/aws/en/dev-tools/cli/) - 인증 설정 완료 상태

---

## 변경 사항 요약

이 저장소에는 원본 대비 다음 3가지 수정이 포함되어 있습니다.

### 1. API 경로 불일치 수정 (`client/src/lib/api.ts`)

| 항목 | 내용 |
|------|------|
| **문제** | 프론트엔드가 `GET /api/me`를 호출하지만, 백엔드의 config 라우터는 `/api/config` prefix로 마운트되어 실제 경로는 `/api/config/me` |
| **증상** | 페이지 로드 시 `404 Not Found` |
| **수정** | `fetchUserInfo()` 경로를 `/me` → `/config/me`로 변경 |

### 2. claude-agent-sdk 버전 업그레이드 (`pyproject.toml`, `uv.lock`)

| 항목 | 내용 |
|------|------|
| **문제** | SDK 0.1.29가 최신 Claude의 `tool_use_summary` 메시지 타입을 파싱하지 못함 |
| **증상** | Agent 실행 시 `Unknown message type: tool_use_summary` 에러로 대화 중단 |
| **수정** | `claude-agent-sdk>=0.1.19` → `>=0.1.50`으로 업그레이드, `uv.lock` 갱신 |

### 3. 로컬 PostgreSQL 설정 (`.env.local` — gitignore 대상)

| 항목 | 내용 |
|------|------|
| **문제** | 기본 설정이 Databricks Lakebase를 가정하여, 로컬 환경에서 DB 연결 불가 |
| **증상** | DB 관련 500 에러, 테이블이 존재하지 않는 에러 |
| **수정** | `LAKEBASE_PG_URL`에 로컬 PostgreSQL URL 설정 + `search_path` 지정 |

---

## 설치 및 실행

### Step 1: Docker PostgreSQL 실행

```bash
docker run -d \
  --name builder-postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=builder_app \
  -p 5432:5432 \
  postgres:16
```

DB가 실행 중인지 확인:

```bash
psql postgresql://postgres:postgres@localhost:5432/builder_app -c "SELECT 1;"
```

### Step 2: MCP Server 설정

```bash
cd databricks-mcp-server
bash setup.sh
cd ..
```

### Step 3: Builder App 설정

```bash
cd databricks-builder-app
bash scripts/setup.sh
```

### Step 4: 환경 변수 설정

`.env.local` 파일을 편집합니다 (setup.sh가 `.env.example`에서 복사해 생성):

```bash
vi databricks-builder-app/.env.local
```

아래 값들을 수정합니다:

```env
# Databricks 인증 (본인 워크스페이스에 맞게 수정)
DATABRICKS_HOST=https://your-workspace.cloud.databricks.com
DATABRICKS_TOKEN=dapi_your_token_here

# 로컬 Docker PostgreSQL (Lakebase 대신)
# 기존 LAKEBASE_ENDPOINT, LAKEBASE_DATABASE_NAME 줄은 주석 처리하고 아래 추가
LAKEBASE_PG_URL=postgresql://postgres:postgres@localhost:5432/builder_app?options=-csearch_path%3Dbuilder_app,public
```

> **`search_path` 설정이 필요한 이유:**
> Alembic 마이그레이션이 `builder_app` 스키마에 테이블을 생성하지만,
> SQLAlchemy 런타임은 기본적으로 `public` 스키마를 조회합니다.
> `search_path`를 지정하지 않으면 `relation "projects" does not exist` 에러가 발생합니다.

### Step 5: 실행

```bash
cd databricks-builder-app
bash scripts/start_dev.sh
```

- Backend: http://localhost:8000
- Frontend: http://localhost:3000

---

## 트러블슈팅

### `404 Not Found` on page load

프론트엔드의 `/api/me` 호출이 404를 반환하는 경우, `client/src/lib/api.ts`의 `fetchUserInfo`가 `/config/me`를 호출하는지 확인하세요.

### `Unknown message type: tool_use_summary`

`claude-agent-sdk` 버전이 0.1.50 미만인 경우 발생합니다:

```bash
# 확인
.venv/bin/python -c "import claude_agent_sdk; print(claude_agent_sdk.__version__)"

# 수동 업그레이드 (start_dev.sh가 다운그레이드한 경우)
uv pip install --python .venv/bin/python --upgrade claude-agent-sdk
```

> **주의:** `scripts/start_dev.sh`가 sibling package를 재설치하면서 SDK를 다운그레이드할 수 있습니다.
> `pyproject.toml`과 `uv.lock`이 0.1.50+를 지정하고 있는지 확인하세요.

### `relation "projects" does not exist` (500 Error)

`LAKEBASE_PG_URL`에 `search_path` 옵션이 빠져있으면 발생합니다.
URL 끝에 `?options=-csearch_path%3Dbuilder_app,public`을 추가하세요.

### `/api/config/me` 타임아웃

`.env.local`의 `DATABRICKS_HOST`와 `DATABRICKS_TOKEN`이 유효하지 않으면 발생합니다.
Databricks CLI로 인증 상태를 확인하세요:

```bash
databricks auth env
```
