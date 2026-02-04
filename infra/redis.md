# 📂 목록

- [Minio 구축하기](./minio.md)
- [ClickHouse 구축하기](./clickhouse.md)
- [Redis 구축하기](./reids.md)
- [사용자 알림 및 실시간 데이터 처리](./user_noti.md)
- [Disk Full - Clickhouse 제거](./disk_full.md)

# Redis 구축하기

사용자 알림 데이터를 바로 소비하기 보다는 여유를 두고 보관 및 삭제를 하고 싶어 Redis를 구축하였습니다.

설치는 되게 간단합니다. Rocky 10 기반이니 주의해주세요.

> Version: redis:remi-8.4

## 구축과정

```shell
# EPEL 설치
sudo dnf install epel-release -y
sudo dnf update -y

# Remi Repository 설치
sudo dnf install https://rpms.remirepo.net/enterprise/remi-release-10.rpm -y

# Redis 모듈 검색
dnf module list redis

# 설치
sudo dnf module enable redis:remi-8.4 -y
sudo dnf install redis -y

# 외부 접속 허용
vi /etc/redis/redis.conf
bind 0.0.0.0

# 실행
sudo systemctl enable --now redis

# 확인
redis-server --version
```
