# 📂 목록

- 🪤 [Airflow 구축하기 (Nas, Ubuntu)](./airflow.md)
- ⚠️ [데이터 불일치](./different_data.md)
- 🌐 [다국어 데이터 매핑 및 처리량 증가 문제](./i18n_mapping.md)
- 🔹 [다국어 원천 데이터의 신뢰도 문제](./untranslated_data.md)
- 📦 [Data Dump 자동화 설정](./data_dump.md)

# 🪤 Airflow 구축하기 (Nas, Ubuntu)

초기에는 자본이 부족했기 때문에 **Synology NAS**를 웹 서버로 활용하였습니다.

Airflow는 24시간 실행되어야 하므로, 클라우드를 사용할 경우 지속적인 비용이 발생합니다. 이를 고려하여 **온프레미스 환경**에서 운영하는 것이 비용 면에서 더 유리하다고 판단했습니다.

Synology NAS는 다양한 서비스를 지원하며, **Docker**도 사용할 수 있기 때문에 이를 활용하여 개발 환경을 구성했습니다.

Airflow 이미지를 직접 설치한 것이 아니라, 필요한 패키지 파일을 `wget` 명령어로 다운로드하여 **수동 설치 방식**으로 구성하였습니다.

NAS 내에 Airflow와 연동할 전용 폴더를 생성하고,  
이를 통해 **DB 데이터 덤프 파일** 등을 수신할 수 있도록 설정했습니다.

# Synology NAS에 Docker 및 Airflow 환경 설정하기

1. **Docker 설치**  
   Synology NAS의 DSM 패키지 센터에서 Docker를 간편하게 설치할 수 있습니다.

2. **Rocky Linux 이미지 설치 및 컨테이너 생성**  
   Docker를 실행한 후, Rocky Linux OS 이미지를 설치하고 컨테이너를 생성합니다.

3. **PostgreSQL 연결 설정**  
   컨테이너를 생성할 때, PostgreSQL과는 `bridge` 네트워크로 연결해줍니다.

4. **공유 폴더 설정**  
   NAS에 공유 폴더를 하나 생성하여, 컨테이너와 연결할 경로로 설정합니다.

5. **터미널 접속 및 루트 권한 획득**  
   NAS의 터미널에 접속한 후, 다음 명령어로 루트 권한을 획득합니다.

   ```bash
   sudo -i
   ```

6. **Airflow 컨테이너 접속**  
   실행 중인 Airflow 컨테이너 ID를 확인합니다.
   ```bash
   docker ps
   ```
   이후, 다음 명령어로 컨테이너 내부에 접속합니다.
   ```bash
   docker exec -it <컨테이너_ID> /bin/bash
   ```

## 패키지 설치

Airflow를 설치하기 위해 필요한 **필수 패키지**들과,  
자주 사용할 명령어에 필요한 패키지들을 먼저 설치합니다.

```bash
# yum update
yum update -y

# 나머지 패키지 설치
yum install ncurses -y
yum install procps -y
yum install make -y
yum install gcc glibc glibc-common gd gd-devel -y
yum install openssl-devel -y
yum install libffi-devel -y
yum install wget -y
yum install findutils -y
yum install sqlite-devel -y
```

## Python 3.9 설치

wget을 사용하여 Python 3.9.19를 설치 합니다. 꼭 이 버전이 아니어도 가능하며 3.8 이상이면 됩니다.

```bash
# python 다운로드
wget https://www.python.org/ftp/python/3.9.19/Python-3.9.19.tgz
tar xvf Python-3.9.19.tgz
cd Python-3.9.19

# 최적화된 옵션으로 빌드 하면서 ssl 활성화
./configure --enable-optimizations --with-ssl

# make install
make
make install

# pip 업그레이드
/usr/local/bin/python3.9 -m pip install
pip3 install --upgrade pip
```

## Airflow 설치

```bash
# 가상환경 생성 및 적용
python3.9 -m venv airflow_venv
source airflow_venv/bin/activate

# pip 업그레이드
pip install --upgrade pip

# 환경 변수 설정
export AIRFLOW_HOME=/home/airflow
export AIRFLOW_VERSION=2.9.1

# 아래의 내용은 그냥 명시적으로 적어도 됩니다. ex) export PYTHON_VERSION=3.9
export PYTHON_VERSION="$(python -c 'import sys; print(f"{sys.version_info.major}.{sys.version_info.minor}")')"
export CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-${AIRFLOW_VERSION}/constraints-${PYTHON_VERSION}.txt"

# Airflow 설치
pip install "apache-airflow==${AIRFLOW_VERSION}" --constraint "${CONSTRAINT_URL}"

# 사용할 DB에 맞는 driver 설치 - 여기서는 postgresql을 사용
pip install psycopg2-binary
```

## Airflow 환경 설정

초기에 설정 파일을 얻기 위해 airflow db migrate를 진행한 뒤, 설정을 바꾼 후 다시 migrate를 진행합니다.

```shell
# 초기 migrate
airflow db migrate

# 필요 없는 db 정보 제거
rm -f airflow.db

# 생성된 config 파일 수정
vi airflow.cfg

# 동시에 실행할 수 있는 최대 task - 서버 환경에 맞춰서 사용하면 됩니다.
parallelism = 8

# web port - 사용할 port를 사용하면 됩니다.
web_server_port = 13000

# 기본은 SequentialExecutor 이며 단일 스레드 입니다. 병렬처리를 할 수 없으니, LocalExecutor로 변경해줍니다. - 별도의 DB가 연결되어야 합니다.
executor = LocalExecutor

# 예시 파일 없음
load_examples = False

# DB 연결 - postgresql
sql_alchemy_conn = postgresql+psycopg2://{user}:{password}@{host}:{port}/{database}

# 위의 내용을 수정하여 저장하였으면 적용을 위해 migrate 해줍니다.
airflow db migrate

# airflow에 Admin Role을 가진 계정을 추가합니다.
airflow users create --username {user} --firstname {firstname} --lastname {lastname} --role Admin --email {email}
```

## Airflow 실행 및 확인

```shell
# webserver 실행 및 확인
nohup airflow webserver > webserver.log 2>&1 &
tail -f webserver.log

# scheduler 실행 및 확인
nohup airflow scheduler > scheduler.log 2>&1 &
tail -f scheduler.log
```

잘 접속 되면 설정한 Port로 접속하면 됩니다.

# Trouble Shooting

진행하며 발생했던 문제 입니다.

## 환경 변수 초기화의 문제

초기에 Docker에 Airflow를 구축했었는데, Docker에서 구축했을 경우, bash로 재접속 하면 환경 변수가 사라지는 문제가 있습니다. Container의 정상적인 동작이며 저희가 해결해야 합니다.

위에서 export 하는 변수들을 container를 구축할 때 정해야 합니다. 이 방식은 **airflow와 python의 버전을 특정했을때 사용가능합니다.**

    예시1)
    docker run -e AIRFLOW_HOME=/home/airflow -e AIRFLOW_VERSION=2.9.1 -e PYTHON_VERSION=3.9 -e CONSTRAINT_URL="https://raw.githubusercontent.com/apache/airflow/constraints-2.9.1/constraints-3.9.txt"

```shell
# 예시 2)
services:
  my_service:
    image: my_image
    environment:
      AIRFLOW_HOME=/home/airflow
      AIRFLOW_VERSION=2.9.1
      PYTHON_VERSION=3.9
      CONSTRAINT_URL=https://raw.githubusercontent.com/apache/airflow/constraints-2.9.1/constraints-3.9.txt
```

```shell
# 예시 3)
# vi ~/.bashrc # 이것들도 가능 ~/.bash_profile, ~/.profile

export AIRFLOW_HOME=/home/airflow
export AIRFLOW_VERSION=2.9.1
export PYTHON_VERSION=3.9
export CONSTRAINT_URL=https://raw.githubusercontent.com/apache/airflow/constraints-2.9.1/constraints-3.9.txt
```

# Ubuntu 22.04.5 LTS 버전에 Airflow 설치하기

Ubuntu 에서는 Nas와 다르게 Docker Image로 한번에 설치했습니다.

Nas를 사용할 때는 환경변수 입력 방법을 몰라서 하지 못했었는데, 사용하면서 어느정도 숙달이 되니 눈에 보이게 되면서 할 수 있게 되었습니다.

## 방화벽 해제 및 Git clone

UI를 사용할 port를 외부에서 접속할 수 있게 열어줍니다.

```shell
# 방화벽 해제 및 확인
sudo ufw allow 13003
sudo ufw status

# Git clone
mkdir /home/airflow
git clone airflow_project_link

# data dump 폴더 생성
mkdir /home/airflow/latest_data

# 권한 부여
chmod -R 777 /home/airflow
```

# Airflow 설치

yaml을 Pull 해줍니다.

```shell
# yaml 설치
docker pull apache/airflow:latest-python3.9

# yaml 수정
docker run -d \
  --name airflow \
  -p '바꿀port':8080 \
  -v /home/airflow:/opt/airflow \
  -e AIRFLOW_UID=50000 \
  -e AIRFLOW_GID=0 \
  -e AIRFLOW__CORE__DAGS_FOLDER=/opt/airflow/eft-library-airflow/dags \
  -e AIRFLOW__CORE__DEFAULT_TIMEZONE=Asia/Seoul \
  -e AIRFLOW__CORE__EXECUTOR=LocalExecutor \
  -e AIRFLOW__CORE__LOAD_EXAMPLES=False \
  -e AIRFLOW__CORE__PLUGINS_FOLDER=/opt/airflow/eft-library-airflow/plugins \
  -e AIRFLOW__DATABASE__SQL_ALCHEMY_CONN=postgresql+psycopg2://user:password@192.168.219.102:dbport/airflow_meta \
  -e AIRFLOW__WEBSERVER__RBAC=True \
  -e AIRFLOW__WEBSERVER__SECRET_KEY=$(openssl rand -hex 32) \
  -e _AIRFLOW_WWW_USER_USERNAME=root \
  -e _AIRFLOW_WWW_USER_PASSWORD=password \
  apache/airflow:latest-python3.9 \
  bash -c "airflow db migrate && airflow webserver"

# -v 는 공유 디렉토리 설정으로 data dump시 docker에 접속하는게 아닌 ubuntu 환경에서 편하게 받기 위한 설정 입니다.

# LocalExecutor를 사용한 이유는 Stand-alone 환경이기에 선택했습니다.

# 나머지 설정을 입력합니다.
```

> 마지막에 **bash -c "airflow db migrate && airflow webserver" 명령어를 추가**해 바로 실행하게 해줍시다 - **하지 않을 경우 컨테이너가 바로 종료** 됩니다.

이제 접속후 Airflow를 설정해줍시다.

```shell
# container 접속
docker exec -it --user root '컨테이너 아이디' /bin/bash

# airflow 유저 생성
airflow users create \
    --username user \
    --password password \
    --firstname FirstName \
    --lastname LastName \
    --role Admin \
    --email user@gmail.com

# 추가를 했다면 터미널을 나와 Ubuntu로 돌아온 뒤 스케줄러를 백그라운드로 실행해줍니다.
docker exec -d airflow airflow scheduler
```

## PG Dump 설정하기 (pgpass)

PostgreSQL 데이터를 Dump 뜨기 위해 Airflow Dag를 만들어도 권한 문제와 비밀번호를 묻는 것이 나와서 막히게 됩니다.

Airflow Dag를 실행하는 주체는 Docker의 Container이니 내부에서 .pgpass를 설정하여 비밀번호를 묻지 않게 설정해줘야 합니다.

```shell
# container 접속
docker exec -it --user root '컨테이너 아이디' /bin/bash

# pgpass 작성
cd /home/airflow
echo "192.168.219.102:dbport:db:user:password" >> ~/.pgpass
chmod 600 /home/airflow/.pgpass

# test
pg_dump -h 192.168.219.102 -p dbport -U user --inserts db > /opt/airflow/latest_data/test_backup.sql
```

이렇게 하면 모든 설정이 완료 됩니다.
