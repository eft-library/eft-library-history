# 📂 목록

- [Minio 구축하기](./minio.md)
- [ClickHouse 구축하기](./clickhouse.md)
- [Redis 구축하기](./reids.md)
- [사용자 알림 및 실시간 데이터 처리](./user_noti.md)
- [Disk Full - Clickhouse 제거](./disk_full.md)

# ClickHouse 구축하기

Kafka를 활용해 페이지 방문 히스토리를 수집하면서 처음에는 PostgreSQL만을 고려했으나,

대시보드 및 통계 처리 용도로는 ClickHouse가 더 적합하다고 판단하여 함께 도입하게 되었습니다.

두 DB의 성능을 비교해보고 추후에는 하나로 갈 예정입니다.

> Version: 25.11.2.24

## 구축 과정

```shell
# RPM 설정
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://packages.clickhouse.com/rpm/clickhouse.repo

# 설치
sudo yum install -y clickhouse-server clickhouse-client

# 실행
sudo systemctl enable clickhouse-server
sudo systemctl start clickhouse-server
sudo systemctl status clickhouse-server

# 결과 확인
clickhouse-client

# 포트 및 외부 접속 허용 - 나올때는 w!
sudo vi /etc/clickhouse-server/config.xml
tcp port 9000 => 9232
<listen_host>::</listen_host>
<listen_host>0.0.0.0</listen_host>

# 재실행
sudo systemctl restart clickhouse-server

# 확인 - 0.0.0.0~ 으로 나와야 함
sudo ss -tulnp | grep clickhouse

# DB, 사용자 생성 및 권한 부여
create database prd;
create user tkl IDENTIFIED With plaintext_password by 'TKL@2635!';
GRANT SELECT, INSERT, CREATE, UPDATE, DELETE, TRUNCATE, DROP ON prd.* TO tkl;
```
