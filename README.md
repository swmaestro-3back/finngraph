# finngraph

finngraph 서비스 전체의 입구 레포. 각 서비스 코드는 서비스 레포에 있고, 이 레포는 레포 지도와 로컬 전체 스택을 갖는다.

## 레포

| 레포 | 역할 | 로컬 서비스 |
|---|---|---|
| [finngraph-client](https://github.com/swmaestro-3back/finngraph-client) | 웹 클라이언트 (Vite) | — |
| [finngraph-api-gateway-GO](https://github.com/swmaestro-3back/finngraph-api-gateway-GO) | API 게이트웨이. 라우팅·CORS·레이트 리밋 | `gateway`, `gw-redis` |
| [finngraph-backend-api-server](https://github.com/swmaestro-3back/finngraph-backend-api-server) | 백엔드 API (Spring Boot) | `backend`, `app-db`, `app-db-migrate`, `redis` |
| [finngraph-ai-server](https://github.com/swmaestro-3back/finngraph-ai-server) | 지식 그래프 API (FastAPI) | `ai` |
| [finngraph-etl](https://github.com/swmaestro-3back/finngraph-etl) | 수집·적재 파이프라인 (Airflow) | `db`, `neo4j`, `neo4j-init`, `airflow-*` |
| [finngraph-core](https://github.com/swmaestro-3back/finngraph-core) | 뉴스 기업 관계 추출 파이프라인 (LangGraph) | — |
| [finngraph-theme-etl](https://github.com/swmaestro-3back/finngraph-theme-etl) | 테마 ETL | — |
| [finngraph-infra](https://github.com/swmaestro-3back/finngraph-infra) | AWS 인프라 (Terraform) | — |
| [github-actions](https://github.com/swmaestro-3back/github-actions) | 공용 CI/CD 워크플로·action | — |

## 요청 흐름

```
브라우저 ── dev.finngraph.com (CloudFront + S3) ── 클라이언트
   │
   └─ api.dev.finngraph.com (ALB) ── gateway ─┬─ /api/*  → backend ─┬─ db (ETL Postgres)
                                              │                     ├─ app-db
                                              │                     └─ redis
                                              └─ /kg/*   → ai ──────┬─ neo4j
                                                                    └─ db
ETL (Airflow) ── db · neo4j 적재, backend 내부 API 호출
```

## 환경

| 환경 | 위치 | 배포 정의 |
|---|---|---|
| local | 개발자 PC | 이 레포 `local/` |
| dev | AWS (ap-northeast-2) | 각 서비스 레포 `deploy/dev/` |
| staging · prod | 예정 | 각 서비스 레포 `deploy/<env>/` |

## 로컬 전체 스택

[local/README.md](local/README.md)
