# 📂 목록

- [Minio 구축하기](./minio.md)
- [ClickHouse 구축하기](./clickhouse.md)
- [Redis 구축하기](./reids.md)
- [사용자 알림 및 실시간 데이터 처리](./user_noti.md)
- [Disk Full - Clickhouse 제거](./disk_full.md)

# Disk Full - Clickhouse 제거

네 Disk Full이 발생했습니다...

저희는 분명히 Nvme m.2 SSD 500GB를 사용 중인데, Full 이 발생했습니다.

운영체제를 설치할 때 /root 의 크기를 더 늘려주었어야 하는데, 그점을 간과하고 다음만 누르고 설치하다 보니 70GB만 할당 되었습니다.

사실 처음이라서 잘 몰랐는데, yum install을 통해 설치된 것들은 전부 /root에 할당 되기에 충분히 크기를 늘려주었어야 합니다.

추후에 운영체제를 설치할 때는 [root 크기 늘리면서 설치하기](https://yongeekd01.tistory.com/15) 여기를 참고해서 하려합니다.

## 사건 발생

하루만에 Kafka, Airflow가 4번 터졌습니다.

로그를 확인해보니 Clickhouse에 넣으려고 하는데 용량이 부족하다는 에러가 떴습니다.

타르코프 정식 출시 후 사용자가 급격하게 늘었는데 그에 대한 반동의 여파가 약간 있는 것 같습니다.

**Kafka Error Log**

<img width="3141" height="362" alt="스크린샷 2026-02-04 225826" src="https://github.com/user-attachments/assets/72c75f2e-34a2-4f20-b73e-35a5e51401bf" />

바로 df -h 를 사용해서 보니 실제로 /root에 용량이 없었습니다.

docker 또한 /root 를 공유하여 사용하는데 용량이 부족하니 Airflow 또한 맛이 가버린 것입니다.

**시스템 용량 현황**

<img width="1660" height="440" alt="스크린샷 2026-02-04 225806" src="https://github.com/user-attachments/assets/f78f727a-2cea-4f2e-a27a-cceecb208571" />

> 이것도 docker 로그를 전부 지워서 1%를 마련한 것입니다...

## Disk Full 원인

일단 /root의 공간이 너무 작게 잡혀 있었습니다.

그리고 Clickhouse를 사용하고 있었는데, 이게 꽤나 많은 용량을 차지했습니다.

Clickhouse가 주요 원인인데, 적재 데이터와 로그 데이터 포함 52GB를 사용하고 있었습니다.

**/var/lib/clickhouse du 현황**

<img width="1022" height="652" alt="스크린샷 2026-02-04 230511" src="https://github.com/user-attachments/assets/c50b9c11-5b08-44df-b15c-7c8c87f9576c" />


**Clickhouse 데이터 현황**

<img width="2476" height="1555" alt="스크린샷 2026-02-04 230038" src="https://github.com/user-attachments/assets/add67652-fbca-47be-aad4-0972bf5317f4" />

## 해결 과정

/root가 LVM이 아니기에 지금 뒤늦게 /root를 늘리는 방법은 없었습니다.

docker의 문제도 아니기에 docker를 /home으로 옮기고 심볼릭링크를 거는 것도 의미가 없습니다.

/root를 늘리려면 포멧후 재설정하는 방법밖에 없었고, clickhouse를 옮겨야 하는데, 그냥 제거하기로 했습니다.

애초에 성능 비교를 위해서 구축한 것인데, 비교도 못하고 끝나버렸습니다.

**Clickhouse 삭제**

## Clickhouse 삭제

삭제는 간단합니다.

```shell
# 서비스 중지
sudo systemctl stop clickhouse-server
sudo systemctl disable clickhouse-server

# 패키지 삭제
sudo dnf remove -y clickhouse-server clickhouse-client

# 데이터 디렉토리 삭제
sudo rm -rf /var/lib/clickhouse

# 설정 삭제
sudo rm -rf /etc/clickhouse-server
sudo rm -rf /etc/clickhouse-client

# 로그 삭제
sudo rm -rf /var/log/clickhouse-server

# 사용자 삭제
sudo userdel clickhouse
```

끝입니다. 홀가분하게 끝났습니다.

**시스템 용량 현황**

<img width="1227" height="365" alt="스크린샷 2026-02-05 오전 8 23 04" src="https://github.com/user-attachments/assets/96ac3c40-e749-4c36-a576-fffc4e99009e" />

## Clickhouse 데이터 저장 방식

PostgreSQL도 동일한 양의 데이터가 있는데, 왜 Clickhouse와 차이가 이렇게 큰지 GPT 한테 물어봤습니다.

1. **칼럼형(Columnar Storage)**                       
   ClickHouse는 칼럼 단위로 데이터를 저장함                       
   장점: 집계/검색 빠름                       
   단점: 각 컬럼마다 인덱스/메타데이터가 생성 → 작은 테이블도 디스크 사용량이 커질 수 있음                       

2. **MergeTree 엔진**                       
   기본 엔진(MergeTree 계열) 사용 시 part 단위로 데이터 저장                       
   각 part는 압축 전과 압축 후 파일, primary key index를 별도로 가짐                       
   그래서 row 수가 적어도 part가 많으면 용량이 크게 나옴                       

3. **압축 설정**                      
   ClickHouse는 데이터 타입에 따라 다양한 압축 방법 사용 가능                       
   기본 압축(LZ4)도 좋지만, 정수·카테고리 컬럼은 더 효율적 압축 가능                       
   반대로 텍스트, UUID, low-cardinality 컬럼은 압축 효율 낮음 → 용량 증가                       

| 데이터 타입            | 특징               | 용량 영향              |
| ---------------------- | ------------------ | ---------------------- |
| String / FixedString   | 텍스트 저장        | 압축 안 되면 크게 증가 |
| UUID                   | 16바이트 고정      | row 수 많으면 증가     |
| Date / DateTime        | 4~8바이트          | 압축 효율 좋음         |
| LowCardinality(String) | 반복 문자열 최적화 | 용량 절약 가능         |

Clickhouse에 저장은 나중에 더욱더 커지면 시도해봐야겠습니다.
