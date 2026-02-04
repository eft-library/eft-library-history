# 📂 목록

- [Minio 구축하기](./minio.md)
- [ClickHouse 구축하기](./clickhouse.md)
- [Redis 구축하기](./reids.md)
- [사용자 알림 및 실시간 데이터 처리](./user_noti.md)
- [Disk Full - Clickhouse 제거](./disk_full.md)

# Minio 구축하기

이미지 관련 저장소가 필요했는데, Amazon의 S3를 쓸까 하다가 요금이 걱정 되어 Minio를 자체적으로 구축하고 사용하기로 했습니다.

> Version: RELEASE.2025-09-07T16-13-09Z (go1.24.6 linux/amd64)

# 특이사항

구버전은 상관 없는데, 최신 버전은 UI에서 할 수 있는 기능이 상당히 한정적으로 변하였습니다.  
S3와 다르게 관리형 스토리지는 단일 root 관리자를 기본으로 목표로 하기 때문입니다.

1. User 생성
2. Access key 생성
3. Bucket Policy 생성
4. Policy 적용
5. 그냥 설정 관련이 전부 사라졌다고 보면 됩니다.

# Minio 구축하기

그럼 시작해봅시다.

## 바이너리 다운로드

```shell
wget https://dl.min.io/server/minio/release/linux-amd64/minio
sudo mv minio /usr/local/bin/
sudo chmod +x /usr/local/bin/minio
```

## 데이터 저장 디렉토리 및 환경변수 파일 생성

```shell
sudo mkdir -p /data/minio
sudo mkdir -p /etc/minio
sudo vi /etc/minio/minio.conf

# 작성 내용
MINIO_ROOT_USER=admin
MINIO_ROOT_PASSWORD=password
MINIO_VOLUMES="/data/minio"
MINIO_OPTS="--address :9000 --console-address :9001"
```

## selinux 비활성화

이걸 안하면 SELinux가 활성화되어 실행을 막는 경우가 발생해 해제 해주었습니다.

```shell
# 확인 - Enforcing 이면 아래 코드를 입력해서 적용
getenforce

# 적용
sudo setenforce 0
```

## systemd 등록

귀찮아서 root로 했는데 별도의 사용자로 해주세요.

```shell
sudo vi /etc/systemd/system/minio.service

[Unit]
Description=MinIO
Documentation=https://min.io/docs
Wants=network-online.target
After=network-online.target

[Service]
User=root
Group=root
EnvironmentFile=/etc/minio/minio.conf
ExecStart=/usr/local/bin/minio server $MINIO_VOLUMES $MINIO_OPTS
Restart=always
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
```

## 실행

실행 후 사이트에 접속해봅니다.

```shell
sudo systemctl daemon-reload
sudo systemctl enable --now minio
sudo systemctl status minio
```

# 중요: 버킷 설정 변경 하기

저는 누구나 사진을 볼 수 있어야 하기에 Bucket을 Public으로 만들어야 했는데,  
생성시에는 자동으로 Private이 되고, 수정할 수 있는 버튼이 없었습니다.  
그래서 minio client를 받아서 적용해주었습니다.

## mc 사용

```shell
# mc 다운로드 및 권한 부여
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc

# 설치 확인
./mc --version

# 서버 등록
./mc alias set myminio http://127.0.0.1:9000 admin password

# 연결 테스트
./mc ls myminio
```

## bucket policy 생성 및 적용

public 용입니다.

```shell
## 파일 생성
vi public-policy.json

## 내용
cat <<EOF > public-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": ["*"]},
      "Action": [
        "s3:GetBucketLocation",
        "s3:ListBucket",
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::eftlibrary",
        "arn:aws:s3:::eftlibrary/*"
      ]
    }
  ]
}
EOF

# 버킷에 적용
./mc anonymous set-json public-policy.json myminio/eftlibrary

# 확인
./mc anonymous get myminio/eftlibrary

```

이후 다시 접속해보면 Public으로 변경된 것을 알 수 있습니다.

# 외부 Upload 활성화 하기

기존에는 Access Key, Secret Key를 사용하여서 했는데 이제는 사라졌고 로그인 시 사용하는 아이디와 패스워드를 사용하면 됩니다.

아직 패키지는 변경된 것 같지 않은데, Access Key 에 아이디를 넣고, Secret Key에 Password를 넣으면 됩니다.
