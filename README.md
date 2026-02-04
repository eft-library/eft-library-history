# eft-library-history

> 이 문서는 해당 프로젝트의 **개발 및 인프라 구축 과정에 대한 History와 회고**를 담고 있습니다.  
> 현재 총 5개의 영역으로 구성되어 있으며, **지속적으로 업데이트될 예정입니다.**

## 📂 구성 항목

- [UI 개발 (Frontend)](./frontend/README.md)
  - [폐지된 커뮤니티 기능](./frontend/community.md)
  - [디자인 리뉴얼 이슈 및 요청](./frontend/design.md)
  - [로드맵 - 최고의 컨텐츠](./frontend/roadmap.md)
  - [다국어 지원을 위하여](./frontend/i18n.md)
  - [3D Map 도입 및 성능 개선 과정](./frontend/3dmap.md)
  - [Analytics, Search Console, AdSense 도입기 및 경험 공유](./frontend/google.md)
  - [NextAuth 도입기 – 프론트 중심 인증 경험](./frontend/auth.md)
  - [프론트엔드 개발 비하인드 – 3번의 마이그레이션 여정](./frontend/migration.md)
  - [사이트 통계 대시보드 개발기](./frontend/dashboard.md)
  - [미니게임 - 칸성비 운빨 망겜 개발기](./frontend/minigame-rng-item.md)
- [API 개발 (Backend)](./backend/README.md)
  - [다국어 지원](./backend/i18n_data.md)
  - [ORM 사용 (With SqlAlchemy)](./backend/orm.md)
  - [Google 로그인 도입기: 사용자 피드백으로 시작된 변화](./backend/token_check.md)
  - [사이트 공격 대비](./backend/defence.md)
- [실시간 데이터 처리 (Kafka)](./kafka/README.md)
  - [Kafka 환경 구축](./kafka/kafka_system_development.md)
  - [사용자 방문 통계](./kafka/user_footprint.md)
- [스케줄링 및 배치 (Airflow)](./airflow/README.md)
  - [Airflow 3.1.5 구축하기](./airflow/airflow.md)
  - [데이터 불일치](./airflow/different_data.md)
  - [다국어 데이터 매핑 및 처리량 증가 문제](./airflow/i18n_mapping.md)
  - [다국어 원천 데이터의 신뢰도 문제](./airflow/untranslated_data.md)
  - [Data Dump 자동화 설정](./airflow/data_dump.md)
  - [시스템 Health Check 구축](./airflow/health_check.md)
  - [아이템 상세 페이지 성능 튜닝 후기](./airflow/item_detail.md)
- [인프라 구축 (Rocky Linux 10)](./infra/README.md)
  - [Minio 구축하기](./infra/minio.md)
  - [ClickHouse 구축하기](./infra/clickhouse.md)
  - [Redis 구축하기](./infra/reids.md)
  - [사용자 알림 및 실시간 데이터 처리](./infra/user_noti.md)
