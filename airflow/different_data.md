# 📂 목록

- [Airflow 3.1.5 구축하기](./airflow.md)
- [데이터 불일치](./different_data.md)
- [다국어 데이터 매핑 및 처리량 증가 문제](./i18n_mapping.md)
- [다국어 원천 데이터의 신뢰도 문제](./untranslated_data.md)
- [Data Dump 자동화 설정](./data_dump.md)
- [시스템 Health Check 구축](./health_check.md)
- [아이템 상세 페이지 성능 튜닝 후기](./item_detail.md)

# 데이터 불일치

현재 Airflow에서는 [Tarkov Dev](https://tarkov.dev/api/) 사이트의 데이터를 주기적으로 수집하고 있습니다.

대부분의 정보는 실제 게임 내 데이터와 잘 일치하지만, **일부 항목은 전혀 다른 값이 들어오는 경우**가 있었습니다.

## 수동 보정

정확한 정보를 제공하기 위해, **코드 내에서 해당 데이터를 수동으로 조정**하여 게임 내 정보와 일치시키는 작업을 진행했습니다.

![different_data](https://github.com/user-attachments/assets/a1e838ea-22ce-4c0d-a235-bf7f45e85d0c)

## 현재의 한계

다만 이 방식에는 다음과 같은 문제가 있습니다

값이 변경되더라도 이를 즉시 감지하기 어렵다는 점입니다.

외부 데이터가 변경될 경우, 기존에 수동으로 조정한 값이 다시 불일치하게 되며, 이를 실시간으로 인지하거나 자동으로 감지하는 기능은 아직 없습니다.

그래서 방법을 모색하고 있는 중입니다.
