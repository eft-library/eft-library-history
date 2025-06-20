# 📂 목록

- 🎗️ [폐지된 커뮤니티 기능](./community.md)
- 🎨 [디자인 리뉴얼 이슈 및 요청](./design.md)
- 👍 [로드맵 - 최고의 컨텐츠](./roadmap.md)
- 🍱 [다국어 지원을 위하여](./i18n.md)
- 🗺️ [3D Map 도입 및 성능 개선 과정](./3dmap.md)
- 📊 [Analytics, Search Console, AdSense 도입기 및 경험 공유](./google.md)
- 🔐 [NextAuth 도입기 – 프론트 중심 인증 경험](./auth.md)
- 🛠️ [프론트엔드 개발 비하인드 – 3번의 마이그레이션 여정](./migration.md)
- 🚀 [사이트 통계 대시보드 개발기](./dashboard.md)

---

# 📊 Analytics, Search Console, AdSense 도입기 및 경험 공유

## Google Analytics 도입

- **도입 난이도**: 매우 쉬움 ✅
- 생성된 추적 코드를 받아서 [`@next/third-parties`](https://nextjs.org/docs/app/building-your-application/optimizing/third-party-scripts) 라이브러리를 사용하여 간단히 적용 가능합니다.
- 별다른 이슈 없이 **바로 적용 및 데이터 수집 성공** 했습니다.

**Google Analytics**

![analytics](https://github.com/user-attachments/assets/1c77a1e9-c3ed-4a4f-96c3-a33832b23829)

---

## Google AdSense 도입 및 문제점

- **도입 초기**: `next/script`를 사용하여 적용을 시도했습니다.
- 그러나 운영 환경에서 아래와 같은 **경고 메시지가 발생** 했습니다.

```
Google AdSense script load error:
TagError: adsbygoogle.push() error: No slot size for availableWidth=0
```

- 다양한 시도 (e.g., `useEffect` 수정 등)를 했으나 해결되지 않았습니다.
- 최종적으로 **`layout.tsx`의 `<head>` 영역에 직접 스크립트 삽입**하여 문제를 해결했습니다.

## ❗ 수익 관련 문제 발생

- 적용은 성공했으나, **예상보다 매우 낮은 수익**이 발생했습니다.
- 현재 원인을 명확히 파악하지 못했으며,  
  **네트워크 품질이나 광고 송출 관련 이슈**일 가능성을 고려 하고 있습니다.
- 수익 관련 이슈는 현재 **보류 상태** 입니다.

---

## Google Search Console 도입 및 색인 문제

- Google Analytics 와 비슷하게 도입 자체는 간단 합니다.  
  마찬가지로 `@next/third-parties`를 통해 생성된 코드를 적용하여 바로 사용 가능합니다.

**Google Search Console**

![스크린샷 2025-05-28 오전 9 33 31](https://github.com/user-attachments/assets/b9343f94-deac-4d1f-b1b8-ca94137589a1)

## 색인(Indexing) 문제

- **사이트 색인이 제대로 되지 않는 문제** 발생했습니다.
- 사이트 운영 경험이 부족한 상태에서 색인 요청을 **무분별하게 진행**한 것이 원인 중 하나로 추정 하고 있습니다.
- 초기에는 사이트 내에서 `query parameter`를 사용하여 **아이템 바로 가기 기능**을 제공했고,  
  이를 sitemap에 포함시켜 **불필요한 페이지들까지 색인** 되어버렸습니다.

**Sitemap.xml**

![스크린샷 2025-05-28 오전 9 24 12](https://github.com/user-attachments/assets/fbc86e71-0f0d-4513-9bec-11ade93116a3)

## 🔄 해결을 위한 조치

- 쿼리 파라미터 기반 기능 제거하였습니다.
- 대신 **아이템별 상세 페이지를 새로 만들어 제공**했습니다.
- 그럼에도 불구하고, 이미 색인된 수많은 페이지는 **삭제되지 않고 그대로 유지되고 있어 곤란한 상황**입니다.

## 📉 색인 수 변화

- 색인 수가 **초기에 500개까지 증가했다가**,  
  이후 **10개 정도로 급감**하였습니다.

## 📌 현재 고민

- 색인이 **광고 수익과도 연관**이 있는 것으로 보이지만,  
  **정확한 해결 방법은 아직 찾지 못한 상황**입니다.
- 혹시 관련 경험이 있는 분의 **조언이 있다면 큰 도움이 될 것** 같습니다.

---

## ✅ 현재까지의 상태 요약

| 항목           | 도입 성공 여부 | 주요 문제                       | 현재 상태          |
| -------------- | -------------- | ------------------------------- | ------------------ |
| Analytics      | ✅ 적용 완료   | 없음                            | 안정 운영 중       |
| AdSense        | ✅ 적용 완료   | 수익 저조, 경고 메시지 발생     | 임시 해결, 보류 중 |
| Search Console | ✅ 적용 완료   | 색인 누락, 불필요한 페이지 색인 | 개선 중, 해결 미완 |
