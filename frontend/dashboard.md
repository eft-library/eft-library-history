# 📂 목록

- 🎗️ [폐지된 커뮤니티 기능](./community.md)
- 🎨 [디자인 리뉴얼 이슈 및 요청](./design.md)
- 👍 [로드맵 - 최고의 컨텐츠](./roadmap.md)
- 🍱 [다국어 지원을 위하여](./i18n.md)
- 🗺️ [3D Map 도입 및 성능 개선 과정](./3dmap.md)
- 📊 [Analytics, Search Console, AdSense 도입기 및 경험 공유](./google.md)
- 🔐 [NextAuth 도입기 – 프론트 중심 인증 경험](./auth.md)
- 🛠️ [프론트엔드 개발 비하인드 – 3번의 마이그레이션 여정](./migration.md)
- 🚀 [사이트 통계 대시보드 개발기](./dashboard.md)

---

# 🚀 사이트 통계 대시보드 개발기

사이트가 어느 정도 안정화됨에 따라 사용자 동향 및 사이트 주요 지표를 실시간으로 파악할 수 있는 **대시보드**를 개발하게 되었습니다.  
본 대시보드는 운영 중인 다양한 서비스의 상태와 API 요청 현황을 한눈에 확인할 수 있도록 구성하였습니다.

👉 실제 대시보드 확인: [https://eftlibrary.com/dashboard](https://eftlibrary.com/dashboard)

**사용자 통계**

![스크린샷 2025-06-15 081657](https://github.com/user-attachments/assets/b244a60f-9a10-4a3e-93d0-e6ca5067622e)
![스크린샷 2025-06-15 081704](https://github.com/user-attachments/assets/22c76fc9-6c00-4164-afe1-d137d11b3388)

<br />

## ✅ 데이터 수집 방식

API 호출 로그는 **Kafka**를 통해 스트리밍 처리되어 데이터 웨어하우스로 적재되며,  
자세한 Kafka 처리 과정은 아래 GitHub 저장소에서 확인하실 수 있습니다.

🔗 GitHub: [eft-library-history - Kafka 처리 구조](https://github.com/eft-library/eft-library-history/tree/main/kafka)

서비스 상태는 기존에 Airflow를 통해서 5분마다 점검하는 것이 있었는데, 결과를 DB에 적재하는 부분을 추가했습니다.

평균 응답 시간은 각 서비스의 health를 호출하여 시간을 측정하여 DB에 적재하고 있습니다.

<br />

## 📊 주요 지표

대시보드는 다음과 같은 **6가지 핵심 지표**로 구성되어 있습니다:

---

### 1. 총 요청 수

- 선택한 기간 동안의 전체 API 요청 수 집계

```sql
WITH current_period AS (
  SELECT COUNT(*) AS request_count
  FROM USER_FOOTPRINT
  WHERE FOOTPRINT_TIME AT TIME ZONE 'Asia/Seoul' >= :start_date
    AND FOOTPRINT_TIME AT TIME ZONE 'Asia/Seoul' < :end_date
)
SELECT
  c.request_count AS current_requests
FROM current_period c
```

---

### 2. 상위 10개 API 엔드포인트

- 요청 수 기준으로 가장 많이 호출된 API 10개 목록
- 각 API별 요청 수 및 상세 정보 제공

```sql
SELECT
  REQUEST,
  LINK,
  COUNT(*) AS request_count
FROM USER_FOOTPRINT
WHERE (EXECUTE_TIME AT TIME ZONE 'Asia/Seoul') BETWEEN :start_date AND :end_date
  AND LINK NOT LIKE '%health%'
GROUP BY REQUEST, LINK
ORDER BY request_count DESC
LIMIT 10
```

---

### 3. 시간대별 요청 분포

- 요청이 집중되는 **시간대 트렌드** 시각화
- 피크 시간 파악 및 성능 튜닝 참고

```sql
SELECT
  TO_CHAR(
    date_trunc('hour', EXECUTE_TIME AT TIME ZONE 'Asia/Seoul') +
    INTERVAL '1 minute' * (FLOOR(EXTRACT(MINUTE FROM EXECUTE_TIME AT TIME ZONE 'Asia/Seoul') / 15) * 15),
    'HH24:MI'
  ) AS time,
  COUNT(*) AS requests
FROM USER_FOOTPRINT
WHERE (EXECUTE_TIME AT TIME ZONE 'Asia/Seoul') BETWEEN :start_date AND :end_date
GROUP BY time
ORDER BY time
```

---

### 4. 서비스 평균 응답 시간

- **FastAPI**, **Next.js** 기준 평균 응답 시간

```sql
SELECT
    service_name,
    ROUND(AVG(response_ms::DOUBLE PRECISION) * 1000) AS avg_response_ms
FROM
    response_time
WHERE
    (checked_time AT TIME ZONE 'Asia/Seoul') BETWEEN :start_date AND :end_date
  AND response_ms IS NOT NULL
GROUP BY
    service_name
```

---

### 5. 사용자 정보

- 선택 기간 내 **활성 사용자 수**
- 전체 가입 사용자 수

```sql
SELECT COUNT(*) AS user_total_count
FROM USER_INFO

SELECT COUNT(*) AS active_user
FROM USER_INFO
WHERE (attendance_time AT TIME ZONE 'Asia/Seoul') BETWEEN :start_date AND :end_date
```

---

### 6. 서비스 상태 모니터링

- 주요 서비스별 실시간 상태 확인 가능

| 서비스명            | 설명                       |
| ------------------- | -------------------------- |
| Airflow             | 배치 및 워크플로 관리      |
| Clickhouse          | 로그 및 통계 데이터 저장소 |
| FastAPI             | 백엔드 API 서버            |
| Kafka Topic         | 실시간 로그 수집           |
| Minio               | 오브젝트 스토리지          |
| Next.js             | 프론트엔드 서버            |
| Nginx Proxy Manager | 리버스 프록시 및 SSL 관리  |
| PostgreSQL          | 운영 데이터베이스          |

```sql
SELECT
    service_name,
    COUNT(*) AS total,
    SUM(CASE WHEN status = 'OK' THEN 1 ELSE 0 END) AS ok_count,
    SUM(CASE WHEN status = 'FAIL' THEN 1 ELSE 0 END) AS fail_count,
    ROUND(SUM(CASE WHEN status = 'OK' THEN 1 ELSE 0 END)::numeric / COUNT(*) * 100, 2) AS ok_percentage,
    ROUND(SUM(CASE WHEN status = 'FAIL' THEN 1 ELSE 0 END)::numeric / COUNT(*) * 100, 2) AS fail_percentage
FROM health_check
WHERE (checked_time AT TIME ZONE 'Asia/Seoul') BETWEEN :start_date AND :end_date
GROUP BY service_name
ORDER BY service_name
```

---

## 🛠️ 개발 후기

- DB에 데이터를 쌓는 시간이 UTC라서 약간 불편했습니다. 현재는 Asia/Seoul로 캐스팅 후 진행합니다.
- Kafka와 Clickhouse 기반의 로그 수집 및 분석 구조를 설계하면서 **실시간 데이터 흐름에 대한 이해**가 높아졌습니다.
- 기존에 Airflow를 통해 health check와 같은 서비스들을 만들어두니 금방 적용할 수 있어서 편했습니다.
- FastAPI 및 Next.js 서비스의 응답 시간을 지속적으로 모니터링하면서 **성능 개선 포인트**를 도출할 수 있었습니다.
- 사용자 피드백을 기반으로 대시보드 기능을 계속 확장할 예정입니다.

---

## 🔍 앞으로의 계획

- 사용자 행동 기반의 세부 분석 기능 추가
- 특정 API 오류 정보 수집 하기 - 아직 별도로 수집하지 않고 있습니다.

---

대시보드는 단순한 지표 표시 도구를 넘어서, 서비스 운영 전반에 대한 인사이트를 제공하는 **중요한 관제 시스템**으로 활용될 예정입니다.
