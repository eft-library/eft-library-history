# 📂 목록

- 🌐 [다국어 지원](./i18n_data.md)
- 🛠️ [ORM 사용 (With SqlAlchemy)](./orm.md)
- 🔐 [Google 로그인 도입기: 사용자 피드백으로 시작된 변화](./token_check.md)

# 🌐 다국어 지원

초기 개발 당시에는 **다국어 지원에 대한 고려가 전혀 없었습니다.**

하지만 점차 더 넓은 사용자를 대상으로 서비스를 제공하고 싶다는 목표가 생기면서, **영어(en), 한국어(ko), 일본어(ja)를 지원**하기로 결정하였습니다.

기존에는 간헐적으로 영어 데이터가 필요할 경우를 대비해 DB에

**name_en, name_kr 같은 별도 컬럼**으로 다국어 데이터를 관리했었습니다.

그러나 다국어 전면 지원을 결정하면서, **Jsonb 타입을 활용해 언어별 데이터를 하나의 컬럼에 통합하는 방식으로 구조를 변경**하였습니다.

이 구조로 전환하면서 **기존 테이블들을 모두 새로 설계하고 재구성**하게 되었습니다.

**초기**

```sql
CREATE TABLE TKL_ITEM
(
    ID TEXT NOT NULL PRIMARY KEY,
    NAME_EN TEXT,
    NAME_KR TEXT,
    CATEGORY TEXT,
    IMAGE TEXT,
    IMAGE_WIDTH NUMERIC,
    IMAGE_HEIGHT NUMERIC,
    URL_MAPPING TEXT,
    INFO JSONB,
    UPDATE_TIME timestamp with time zone default now()
);
COMMENT ON COLUMN TKL_ITEM.ID IS '아이템 아이디';
COMMENT ON COLUMN TKL_ITEM.NAME_EN IS '아이템 이름 영문';
COMMENT ON COLUMN TKL_ITEM.NAME_KR IS '아이템 이름 한글';
COMMENT ON COLUMN TKL_ITEM.CATEGORY IS '아이템 카테고리';
COMMENT ON COLUMN TKL_ITEM.IMAGE IS '아이템 사진';
COMMENT ON COLUMN TKL_ITEM.IMAGE_WIDTH IS '아이템 사진 가로 크기';
COMMENT ON COLUMN TKL_ITEM.IMAGE_HEIGHT IS '아이템 사진 세로 크기';
COMMENT ON COLUMN TKL_ITEM.URL_MAPPING IS '아이템 주소';
COMMENT ON COLUMN TKL_ITEM.INFO IS '아이템 상세 정보';
COMMENT ON COLUMN TKL_ITEM.UPDATE_TIME IS '아이템 업데이트 날짜';
```

**수정**

```sql
CREATE TABLE ITEM_I18N
(
    ID TEXT NOT NULL PRIMARY KEY,
    NAME JSONB,
    CATEGORY TEXT,
    IMAGE_WIDTH NUMERIC,
    IMAGE_HEIGHT NUMERIC,
    URL_MAPPING TEXT,
    IMAGE TEXT,
    INFO JSONB,
    UPDATE_TIME timestamp with time zone default now()
);
COMMENT ON COLUMN ITEM_I18N.ID IS '아이템 아이디';
COMMENT ON COLUMN ITEM_I18N.NAME IS '아이템 이름';
COMMENT ON COLUMN ITEM_I18N.CATEGORY IS '아이템 카테고리';
COMMENT ON COLUMN ITEM_I18N.IMAGE IS '아이템 사진';
COMMENT ON COLUMN ITEM_I18N.IMAGE_WIDTH IS '아이템 사진 가로 크기';
COMMENT ON COLUMN ITEM_I18N.IMAGE_HEIGHT IS '아이템 사진 세로 크기';
COMMENT ON COLUMN ITEM_I18N.URL_MAPPING IS '아이템 주소';
COMMENT ON COLUMN ITEM_I18N.INFO IS '아이템 상세 정보';
COMMENT ON COLUMN ITEM_I18N.UPDATE_TIME IS '아이템 업데이트 날짜';
```
