# 📂 목록

- 🛠️ [Kafka 환경 구축](./kafka_system_development.md)
- 🚀 [ClickHouse와 PostgreSQL 비교 및 테이블 설계](./clickhouse_postgresql.md)
- [사용자 방문 통계](./user_footprint.md)

# 🚀 ClickHouse와 PostgreSQL 비교 및 테이블 설계

Kafka를 활용해 페이지 방문 히스토리를 수집하면서 처음에는 PostgreSQL만을 고려했으나,

대시보드 및 통계 처리 용도로는 ClickHouse가 더 적합하다고 판단하여 함께 도입하게 되었습니다.

이번 기회에 두 저장소의 성능 비교도 함께 진행해보았습니다.

## 💡 ClickHouse란?

ClickHouse는 Yandex에서 개발한 **오픈소스 컬럼 지향(Column-oriented) DBMS**입니다.

**OLAP(Online Analytical Processing) 용도로 설계**되어, **대용량 데이터의 빠른 집계 및 분석에 특화**되어 있습니다.

> 📊 대시보드나 통계 분석에 적합한 DB를 찾는다면 ClickHouse가 좋은 선택입니다.

- 데이터는 컬럼 단위로 저장됨
- 실시간 분석 쿼리 성능에 강점
- 분산 처리 및 병렬 쿼리에 최적화
- SQL 지원 (표준 SQL과 유사)
- Druid와 유사한 분산처리 구조, Shard와 Replica 구조로 노드를 나눌 수 있음
- Distributed Table을 사용하면 여러 Shard에 분산된 테이블을 추상적으로 하나로 보이게 할 수 있음

## ✅ ClickHouse 특징 요약

    컬럼 기반 저장: 필요한 컬럼만 디스크에서 읽기 때문에 성능 우수
    고속 집계 성능: 수십억 행도 빠르게 처리
    수평 확장성: 분산 처리 및 병렬 쿼리 가능
    다양한 테이블 엔진: MergeTree, SummingMergeTree 등
    SQL 호환성: 표준 SQL과 유사한 쿼리 언어 지원
    클러스터링 구조: Shard/Replica 구조 가능 + Distributed Table 지원

## ClickHouse VS PostgreSQL

Kafka를 통해서 페이지 방문 History 관련 대시보드와 통계를 표현할 것이기에 해당 관점에서 비교한 내용입니다.

| 항목       | **ClickHouse**                        | **PostgreSQL**                               |
| ---------- | ------------------------------------- | -------------------------------------------- |
| 처리 시간  | 0.3 \~ 2초 (분산 환경일 경우 더 빠름) | 5 \~ 30초 (인덱스 최적화 되어 있어도 느림)   |
| CPU 사용량 | 높음 (멀티코어 활용)                  | 보통 (싱글 프로세스 위주)                    |
| RAM 사용량 | 비교적 효율적                         | 집계 시 메모리 사용 많음                     |
| 병렬 처리  | 자동 병렬 실행                        | 병렬 처리 옵션 제한적 (`parallel` 설정 필요) |
| I/O 처리   | 컬럼만 디스크에서 읽음                | 전체 행 읽기 필요                            |
| 확장성     | 노드 추가 시 성능 향상                | 수직 확장에 의존                             |

## 🗂 PostgreSQL 테이블 설계

PostgreSQL USER_FOOTPRINT 테이블 생성

```sql
CREATE TABLE USER_FOOTPRINT
(
  ID SERIAL PRIMARY KEY,
  LINK TEXT,
  REQUEST TEXT,
  FOOTPRINT_TIME timestamp with time zone,
  EXECUTE_TIME timestamp with time zone default now()
);
COMMENT ON COLUMN USER_FOOTPRINT.ID IS '사용자 기록 아이디';
COMMENT ON COLUMN USER_FOOTPRINT.REQUEST IS '사용자 기록 요청 타입';
COMMENT ON COLUMN USER_FOOTPRINT.LINK IS '사용자 기록 주소';
COMMENT ON COLUMN USER_FOOTPRINT.FOOTPRINT_TIME IS '사용자 기록 전송 시간';
COMMENT ON COLUMN USER_FOOTPRINT.EXECUTE_TIME IS '사용자 기록 적재 시간';
```

**TMI**

PostgreSQL에서 VARCHAR가 아닌 TEXT 타입의 장점

    1. VARCHAR 길이 제한은 실효성이 낮음
    → 대부분 의미 없이 VARCHAR(255) 등을 사용하는 경우가 많음

    2. TEXT와 성능 차이 없음
    → PostgreSQL 내부적으로는 TEXT, VARCHAR(n), VARCHAR 모두 동일하게 처리됨

    3. 스키마 유지보수에 유리
    → TEXT는 길이 제한이 없어 스키마 변경 없이 유연하게 대응 가능

## 🗂 ClickHouse 테이블 생성

ClickHouse USER_FOOTPRINT 테이블 생성

**ClickHouse는 PostgreSQL과 다르게 Serial로 자동 생성을 할 수 없고 UUID로 해야 합니다.**

```sql
CREATE TABLE user_footprint
(
    id UUID DEFAULT generateUUIDv4(),
    link String,
    request String,
    footprint_time DateTime64(3),
    execute_time DateTime64(3) DEFAULT now()
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(footprint_time)
ORDER BY (footprint_time);
```
