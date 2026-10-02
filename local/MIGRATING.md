# 기존 로컬 DB 데이터 옮기기

각 레포(`finngraph-etl`, `finngraph-backend-api-server`)에서 `docker compose` 로 DB 를 띄워 쓰던 경우,
그 데이터를 중앙 스택(`finngraph/local`)으로 옮기는 방법이다.

중앙 스택은 프로젝트 이름이 `finngraph-local` 이라 볼륨도 새로 만든다. 예전 볼륨은 그대로 남고,
아래 절차는 예전 볼륨을 **복사만** 하므로 원본은 바뀌지 않는다.

## 무엇을 옮기나

| 데이터 | 예전 볼륨 | 새 볼륨 | 옮기기 |
|---|---|---|---|
| ETL DB (Postgres) | `<ETL 폴더명>_etl_pgdata` | `finngraph-local_etl_pgdata` | 생성일에 따라 다름 (아래) |
| Neo4j | `<ETL 폴더명>_etl_neo4j_data` | `finngraph-local_etl_neo4j_data` | 가능 |
| 백엔드 app-db | `finngraph-backend_app-db-data` | `finngraph-local_app-db-data` | 가능 |
| Airflow 메타DB | `<ETL 폴더명>_etl_airflow_pgdata` | `finngraph-local_etl_airflow_pgdata` | 선택. DAG 실행 이력뿐이라 새로 시작해도 된다 |
| Redis | — | — | 옮기지 않는다. 핫테마·레이트 리밋 같은 임시 데이터다 |

ETL 볼륨 이름의 앞부분은 ETL 레포를 clone 한 **폴더 이름**이다. 기본 clone 이면 `finngraph-etl_etl_pgdata` 다.

```bash
docker volume ls | grep -E 'etl_pgdata|etl_neo4j_data|app-db-data|etl_airflow_pgdata'
```

### ETL DB 는 볼륨을 만든 날짜를 먼저 확인한다

```bash
docker volume inspect <ETL 폴더명>_etl_pgdata --format '{{.CreatedAt}}'
```

| 생성일 | 옮기기 | 이유 |
|---|---|---|
| 2026-09-16 이후 | 가능 | 지금의 `V1`~`V6` 마이그레이션으로 만들어졌다 |
| 2026-09-08 ~ 09-15 | 새로 시작 | 그 뒤 정리된 예전 마이그레이션 파일로 만들어져 스키마가 다를 수 있다 |
| 2026-09-08 이전 | 새로 시작 | TimescaleDB 이미지 시절 볼륨이라 지금 이미지로는 시작되지 않는다 (`could not access file "timescaledb"`) |

새로 시작하는 경우 ETL DB 는 옮기지 않고 넘어가면 된다. Neo4j·app-db 는 날짜와 상관없이 옮길 수 있다.

## 절차

### 1. 예전 스택을 내린다

실행 중인 DB 의 파일을 복사하면 깨질 수 있다. 각 레포에서 **`-v` 없이** 내린다.

```bash
cd <ETL 레포> && docker compose down
cd <백엔드 레포> && docker compose down
```

### 2. 중앙 `.env` 의 Neo4j 값을 예전 값과 맞춘다

Neo4j 는 비밀번호와 기본 DB 이름을 **데이터를 처음 만들 때 데이터 안에 저장**한다.
`finngraph/local/.env` 의 `NEO4J_USERNAME`, `NEO4J_PASSWORD`, `NEO4J_DATABASE` 를 ETL 레포 `.env` 와 같은 값으로 둔다.

ETL DB·app-db 계정(`ETL_DB_*`, `APP_DB_*`)도 예전에 바꿔 쓰던 값이 있으면 똑같이 맞춘다. 기본값을 썼다면 그대로 두면 된다.

### 3. 새 볼륨을 만들고 복사한다

`finngraph/local` 에서 실행한다. 새 볼륨이 이미 있고 데이터가 들어 있으면 덮어쓰게 되니, 처음 한 번만 한다.

```bash
cd finngraph/local

docker compose --profile airflow up --no-start db neo4j app-db airflow-db

copy() {
  docker run --rm -v "$1":/from:ro -v "$2":/to alpine \
    sh -c 'find /to -mindepth 1 -delete && cp -a /from/. /to/'
}

copy <ETL 폴더명>_etl_neo4j_data        finngraph-local_etl_neo4j_data
copy finngraph-backend_app-db-data      finngraph-local_app-db-data
copy <ETL 폴더명>_etl_pgdata            finngraph-local_etl_pgdata
copy <ETL 폴더명>_etl_airflow_pgdata    finngraph-local_etl_airflow_pgdata
```

옮기지 않을 항목은 그 줄을 빼면 된다.

`up --no-start` 로 볼륨을 만드는 이유는 compose 가 관리하는 볼륨으로 만들기 위해서다.
`find … -delete` 는 컨테이너를 만들 때 이미지가 볼륨에 미리 채워 넣는 기본 파일(Neo4j)을 지운다.

### 4. ETL DB 만 최초 1회: 기준선 0 으로 마이그레이션

예전 방식은 Flyway 이력 없이 SQL 을 직접 실행했기 때문에, 그대로 두면 Flyway 가 V1 부터 다시 적용하다 실패한다.
기준선을 0 으로 잡고 V1 부터 다시 실행시킨다. `V1`~`V6` 은 모두 다시 실행해도 안전하게
(`IF NOT EXISTS`, `ON CONFLICT DO NOTHING` 등) 작성돼 있어 기존 데이터는 그대로 남고, 빠진 마이그레이션만 채워진다.

```bash
docker compose up -d --wait db
docker compose run --rm \
  -e FLYWAY_BASELINE_ON_MIGRATE=true -e FLYWAY_BASELINE_VERSION=0 \
  db-migrate
```

`Successfully baselined schema with version: 0` 과 `now at version v6` 이 나오면 된다.
ETL DB 를 옮기지 않았다면 이 단계는 건너뛴다. app-db 는 원래 Flyway 이력이 있어 필요 없다.

### 5. 평소처럼 띄운다

```bash
docker compose up -d
```

## 확인

```bash
docker compose exec db psql -U threeback -d finngraph \
  -c "select version, type from flyway_schema_history order by installed_rank"

docker compose exec neo4j cypher-shell -u "$NEO4J_USERNAME" -p "$NEO4J_PASSWORD" -d "$NEO4J_DATABASE" \
  'MATCH (n) RETURN count(n)'
```

ETL DB 이력이 `0 BASELINE`, `1`~`6 SQL` 순서로 보이고, Neo4j 노드 수가 예전과 같으면 끝이다.

## 되돌리기 · 정리

- 잘못됐으면 `docker compose down -v` 로 **새 볼륨만** 지우고 3단계부터 다시 한다. 예전 볼륨은 그대로다.
- 중앙 스택에서 데이터를 확인한 뒤 예전 볼륨을 지운다.

  ```bash
  docker volume rm <ETL 폴더명>_etl_pgdata <ETL 폴더명>_etl_neo4j_data <ETL 폴더명>_etl_airflow_pgdata finngraph-backend_app-db-data
  ```
