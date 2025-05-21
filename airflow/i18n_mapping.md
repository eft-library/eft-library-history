# 📂 목록

- ⚠️ [데이터 불일치](./different_data.md)
- 🌐 [다국어 데이터 매핑 및 처리량 증가 문제](./i18n_mapping.md)
- 🔹 [다국어 원천 데이터의 신뢰도 문제](./untranslated_data.md)

# 🌐 다국어 데이터 매핑 및 처리량 증가 문제

사이트에서 다국어(en, ko, ja) 지원 기능을 도입하면서 Graphql API 요청 방식에도 변화가 생겼습니다.

## 🧩 문제 상황

기존에는 단일 요청으로 데이터를 받아 처리했었습니다.
하지만 다국어 버전을 도입하면서 **API 규격상 한 번에 세 언어를 동시에 받을 수 없었고**, **각 언어별로 별도의 요청을 보내야 하는 구조**였습니다.

이로 인해 다음과 같은 문제가 발생했습니다

- **데이터 요청량이 3배로 증가**
- Airflow의 XCom으로 데이터를 주고받기에는 **용량 및 성능 이슈 발생**
- **여러 언어 데이터를 통합하는 로직이 복잡해짐**

## 🛠️ 대응 방법

이를 해결하기 위해 다음과 같은 방식으로 구조를 재설계했습니다

    1. API 요청 결과를 언어별로 JSON 파일로 저장
    → 임시 폴더를 사용하여 Task 간 데이터 전달

    2. Task마다 해당 언어의 JSON 파일을 읽어 데이터 사용

    3. 공통 ID 기준으로 데이터를 매핑하여 병합
    → 하나의 객체로 통합한 후 가공 함수에 전달

    4. 최종적으로 단일 객체 안에 en/ko/ja 언어 데이터를 병합

## 🔧 주요 처리 흐름

###📌 언어별 API 요청 JSON 구성 함수

![get_lang](https://github.com/user-attachments/assets/39117784-784d-45a3-aeb8-91b377687aba)

### 📁 언어별 저장소 경로 선언

![define_lang_data](https://github.com/user-attachments/assets/2c46b761-2d51-4d9d-8f08-147fbd9f42cc)

### 💾 API 응답 결과 JSON으로 저장

![save_json](https://github.com/user-attachments/assets/ea648ce7-919b-49d0-a493-c3011c8c2baf)

### 📥 JSON을 읽고 ID 기준 매핑 후 가공 함수로 전달

![send_lang_data](https://github.com/user-attachments/assets/56b23d4d-5526-4a97-8af4-6aad694be06a)

### 🔗 3개 언어 데이터를 매핑하여 단일 객체 생성

![mapping_data](https://github.com/user-attachments/assets/69cb5a02-92bd-4eef-bfe4-b463a9ecb645)
