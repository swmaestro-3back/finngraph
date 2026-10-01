# 로컬 전체 스택

각 서비스 레포의 `compose.yaml` 을 `include` 로 묶어 한 번에 띄운다. 서비스 정의는 각 레포에 있고, 이 폴더는 목록과 레포를 넘나드는 의존 관계(`overrides/`)만 갖는다.

## 폴더 구조

서비스 레포를 이 레포와 같은 상위 폴더에 clone 한다.

```
<작업 폴더>/
├── finngraph/                       ← 이 레포
├── finngraph-etl/
├── finngraph-backend-api-server/
├── finngraph-ai-server/
└── finngraph-api-gateway-GO/
```

폴더 이름이 다르면 `local/.env` 의 `ETL_DIR` · `BACKEND_DIR` · `AI_DIR` · `GATEWAY_DIR` 로 바꾼다.

## 처음 한 번

```bash
cd finngraph/local
cp .env.example .env
```

각 서비스 레포의 `.env` 는 그 레포의 `.env.example` 을 보고 만든다. 비밀값·API 키는 각 레포 `.env` 에 둔다.

- 백엔드는 `JWT_SIGNING_KEY`, `JWT_PUBLIC_KEY` 가 필수다. ES256(P-256) 키를 base64 DER 로 넣는다.

```bash
openssl ecparam -name prime256v1 -genkey -noout > /tmp/jwt.pem
echo "JWT_SIGNING_KEY=$(openssl pkcs8 -topk8 -nocrypt -outform DER -in /tmp/jwt.pem | base64 | tr -d '\n')"
echo "JWT_PUBLIC_KEY=$(openssl ec -pubout -outform DER -in /tmp/jwt.pem 2>/dev/null | base64 | tr -d '\n')"
```

- 백엔드 이미지는 미리 빌드한 jar 를 복사하므로, 처음과 코드를 바꾼 뒤에 백엔드 레포에서 `./gradlew :app:bootJar -x test` 를 실행한다.

## 실행

```bash
docker compose up -d --build          # Airflow 제외
COMPOSE_PROFILES=airflow docker compose up -d --build   # Airflow 포함
docker compose ps
docker compose down                   # 데이터 유지
docker compose down -v                # 데이터까지 삭제
```

`.env` 에 `COMPOSE_PROFILES=airflow` 를 적어 두면 매번 붙이지 않아도 된다.

| 주소 | 서비스 |
|---|---|
| http://localhost:8090 | gateway — 배포 환경과 같은 경로(`/api/*`, `/kg/*`)로 확인할 때 |
| http://localhost:8081 | backend |
| http://localhost:8000 | ai |
| http://localhost:8080 | Airflow (`airflow` 프로필) |
| http://localhost:7474 | Neo4j Browser |
| localhost:15432 · 15433 · 16379 | ETL DB · app-db · Redis |

클라이언트는 각자 `npm run dev`(http://localhost:5173)로 띄운다. Vite 프록시 기본값은 백엔드 `localhost:8080`, AI `localhost:8000` 이다. 이 스택의 백엔드는 8081 이므로 `BACKEND_PROXY_TARGET=http://localhost:8081 npm run dev` 로 띄운다.

## 값의 우선순위

- 서비스 간 주소(`db`, `neo4j`, `redis`, `backend`, `ai`)는 각 레포 `compose.yaml` 에 고정돼 있다.
- 여러 서비스가 같이 쓰는 값(ETL DB 계정, Neo4j 계정, `INTERNAL_API_TOKEN`)은 이 폴더 `.env` 에 한 번만 적는다. 같은 키가 서비스 레포 `.env` 에 있어도 이 폴더 `.env` 가 우선한다.

## 서비스 하나만 띄울 때

각 서비스 레포에서 `docker compose up` 으로 그 레포 서비스만 띄울 수 있다. 다만 다른 레포 서비스(예: AI 가 쓰는 `db`·`neo4j`)는 없으므로 연결이 필요한 기능은 이 전체 스택에서 확인한다.

## 기존 로컬 데이터

이 스택의 볼륨은 프로젝트 이름 `finngraph-local` 아래에 새로 만들어진다. 서비스 레포에서 따로 띄우던 기존 볼륨(`finngraph-etl_etl_pgdata` 등)은 그대로 남고, 이 스택에서는 보이지 않는다.
