# 📂 목록

- 🌐 [다국어 지원](./i18n_data.md)
- 🛠️ [ORM 사용 (With SqlAlchemy)](./orm.md)
- 🔐 [Google 로그인 도입기: 사용자 피드백으로 시작된 변화](./token_check.md)

# 🔐 Google 로그인 도입기: 사용자 피드백으로 시작된 변화

## 🧭 로그인 기능의 시작

처음 개발을 시작할 때는 **로그인 기능이 필요하다고 생각하지 않았습니다.**

그러나 사용자들의 피드백을 받으면서,

**사용자 개인 데이터를 저장할 수 있는 기능의 필요성**을 느끼게 되었고, 로그인 기능을 도입하게 되었습니다.

## 💾 사용자 데이터 저장 항목

로그인 기능을 통해 사용자들은 다음과 같은 정보를 저장하고 관리할 수 있습니다.

- user_info : 사용자 기본 정보
- user_hideout : 사용자 은신처 정보
- user_quest : 사용자 퀘스트 플래너 정보
- user_roadmap : 사용자 로드맵 정보

이 데이터를 기반으로 **자신의 게임 내 진행 상황을 손쉽게 저장 및 확인**할 수 있으며,

**목표 설정 및 진로 탐색에도 도움이 되는 경험을 제공**한다고 생각합니다.

# 🌍 Google && Naver 인증

## 📌 초기에는 두 가지 인증 방식 도입

개발 초기에는 **Google과 Naver, 총 두 가지 OAuth 인증 방식을 동시에 도입**했습니다.

최근에 다국어를 지원하면서 **해외에는 naver가 없으니 결론적으로는 좋은 선택**이었다 생각합니다.

## 📊 사용자 선택과 인증 방식 통합

하지만 실제 운영 데이터를 통해 확인한 결과, **대다수의 사용자들이 Google 로그인을 선택**하고 있었으며,

두 인증 시스템을 동시에 유지하는 것이 **관리 측면에서 비효율적**이라 판단했습니다.

따라서 인증 방식을 Google로 단일화하였습니다.

## 🌐 다국어 및 글로벌 지원을 위한 좋은 선택

최근 다국어(한국어, 영어, 일본어 등)를 지원하기 시작하면서, **해외 사용자 입장에서 Naver 인증은 익숙하지 않다**는 점도 고려하게 되었습니다.

결과적으로 Google 하나로 통일한 결정은 **글로벌 확장성과 관리 효율성 측면에서 올바른 방향이었다고 판단**합니다.

**초기**

```python
import requests
from fastapi import FastAPI, HTTPException


NAVER_USER_INFO_URL = "https://openapi.naver.com/v1/nid/me"
GOOGLE_TOKEN_INFO_URL = "https://www.googleapis.com/oauth2/v1/tokeninfo"


class UserUtil:

    @staticmethod
    def verify_token(provider: str, access_token: str):
        """
        1차 토큰 분류
        """
        if provider == "naver":
            res = UserUtil.verify_naver_token(access_token)
            if res:
                return res
            else:
                return False
        else:
            res = UserUtil.verify_google_token(access_token)
            if res:
                return res
            else:
                return False

    @staticmethod
    def verify_naver_token(access_token: str):
        """
        naver 검증
        """
        headers = {
            "Authorization": f"Bearer {access_token}",
        }
        response = requests.get(NAVER_USER_INFO_URL, headers=headers)

        if response.status_code != 200:
            raise HTTPException(status_code=401, detail="Invalid Naver access token")
        data = response.json()
        return data["response"]["email"]

    @staticmethod
    def verify_google_token(access_token: str):
        """
        구글 검증
        """
        response = requests.get(f"{GOOGLE_TOKEN_INFO_URL}?access_token={access_token}")

        if response.status_code != 200:
            raise HTTPException(status_code=401, detail="Invalid Google access token")
        data = response.json()
        return data["email"]
```

**수정**

```python
import requests
import os
from dotenv import load_dotenv


load_dotenv()


class UserUtil:
    @staticmethod
    def verify_google_token(access_token: str):
        """
        구글 검증
        """
        response = requests.get(
            f"{os.getenv('GOOGLE_TOKEN_INFO_URL')}?access_token={access_token}"
        )

        if response.status_code != 200:
            return False
        data = response.json()
        return data["email"]

```

# 🪪 Google Token With Header

구글에서 발급한 **OAuth Token을 클라이언트에서 백엔드로 전달할 때**,
처음에는 **HTTP Header를 통해 전달**하려고 했습니다.

하지만 생각처럼 잘 동작하지 않아 많은 시간을 들여 삽질을 하게 되었습니다.

결국 문제를 해결하긴 했지만,
**해결 방법이 예상 외로 너무 단순하고 명확해서 허탈했던 경험**이었습니다.

```python
from fastapi import APIRouter, Depends
from api.response import CustomResponse
from fastapi.security import OAuth2PasswordBearer
from util.constants import HTTPCode
from api.constants import Message
from api.user.service import UserService
from api.user.user_req_models import AddUserReq
from api.user.util import UserUtil

router = APIRouter(tags=["User"])

# JWT를 헤더에서 추출하는 의존성 함수
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

@router.post("/get")
def get_user(token: str = Depends(oauth2_scheme)):

    user_email = UserUtil.verify_google_token(access_token=token)
    if user_email:
        user = UserService.get_user(user_email)
        if user is None:
            return CustomResponse.response(None, HTTPCode.OK, Message.INVALID_USER)
        return CustomResponse.response(user, HTTPCode.OK, Message.SUCCESS)
    else:
        return CustomResponse.response(None, HTTPCode.OK, Message.INVALID_USER)
```
