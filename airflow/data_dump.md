# 📂 목록

- 🪤 [Airflow 구축하기 (Nas, Ubuntu)](./airflow.md)
- ⚠️ [데이터 불일치](./different_data.md)
- 🌐 [다국어 데이터 매핑 및 처리량 증가 문제](./i18n_mapping.md)
- 🔹 [다국어 원천 데이터의 신뢰도 문제](./untranslated_data.md)
- 📦 [Data Dump 자동화 설정](./data_dump.md)

# 📦 Data Dump 자동화 설정

매일 00:20에 PostgreSQL DB의 데이터를 덤프(Dump)하여 저장하고, **4일이 지난 파일은 자동으로 삭제**하고 있습니다.

pg_dump 명령어를 사용하며, **터미널 명령을 직접 실행하는 방식**으로 구현했습니다.

초기 설정 과정에서 시행착오가 많았기에, 해당 설정 과정을 기록합니다.

이 설정은 **Docker 기반으로 Airflow를 구축한 상태를 전제**로 합니다.

# PG_DUMP 설정

```shell
# Docker 목록 확인하기
docker ps

# Airflow Container Id를 찾아 shell로 접속하기
docker exec -it container_id /bin/bash

# home에 airflow 폴더를 하나 만들고 .pgpass 적용하기
mkdir /home/airflow
echo "192.168.219.102:11111:db:user:pwd" >> ~/.pgpass
chmod 600 /home/airflow/.pgpass

# 테스트 해보기 - 저장할 경로는 마음대로 하면 됩니다. 저는 docker directory를 연결해두어서 해당위치로 했습니다.
pg_dump -h 192.168.219.102 -p 11111 -U user --inserts db > /opt/airflow/latest_data/test_backup.sql
```

Airflow 작성하기

```python
def get_today():
    """
    년, 월, 일 추출
    """
    now = pendulum.now()

    year = now.year
    month = now.month
    day = now.day

    return f"{year}_{month}_{day}"


def dump_script():
    """
    postgresql dump 뜨고 결과 넘기기
    데이터가 많은 중요하지 않은 데이터는 제외
    """
    today = get_today()

    return f"""
        source ~/.bashrc
        echo "Executing PostgreSQL Command - pg_dump"
        pg_dump -h 192.168.219.102 -p 11111 -U user --inserts --exclude-table-data=public.item_price_i18n --exclude-table-data=public.user_footprint --exclude-table=public.item_i18n --exclude-table=public.search_i18n --exclude-table=public.item_price_history_i18n db > /opt/airflow/latest_data/{today}_backup.sql
        echo $?
        """


def remove_old_file_script():
    """
    4일 지난 백업 파일들 제거 하기
    """

    return "find /opt/airflow/latest_data -type f -mtime +3 -delete"
```
