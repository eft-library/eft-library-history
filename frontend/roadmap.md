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

# 👍 로드맵 - 최고의 컨텐츠

기능을 만든 초창기부터 지금까지 가장 많은 사랑을 받고 있는 **퀘스트 로드맵** 기능입니다.

🔗 [퀘스트 로드맵 바로가기](https://eftlibrary.com/roadmap)  
🔗 [참고 사이트 - Quest Roadmap](https://tarkov-market.com/progression/quests-interactive)

> **사용 기술**: `react-xyflow`  
> **성과**: 매우 만족스러운 사용자 반응

**Quest Roadmap**

<img width="1050" alt="roadmap" src="https://github.com/user-attachments/assets/c9460567-86bc-4083-8285-78f5ade0a223" />

---

## 💡 개발 동기

- 기존에 비슷한 로드맵이 있는 사이트가 있었지만 아쉬움이 많았습니다.
  - 불편한 사용성
  - 유료 서비스
- 👉 그래서 **우리가 직접 만들고 무료로 제공**하자는 기획이 시작되었습니다.
- 개발 당시부터 **인기 있을 것이라는 확신**
- 실제로도 **가장 인기 있는 기능**으로 자리 잡았습니다.

---

## 🧩 Node & Edge 구성

- Node와 Edge를 **자동 배치하는 기능**이 있었지만
  - 자동 디자인 결과가 **매우 비효율적이고 보기 어려웠습니다.**
- 🔧 그래서 **기획자가 약 500개의 퀘스트를 직접 수동 배치**
  - 각 Node 좌표를 일일이 지정했습니다.

---

## 🖱️ Node 클릭 이벤트 처리

사용자가 Node의 체크박스를 클릭할 때 처리 로직에 고민이 많았습니다.

### ✅ 체크 동작 규칙

1. **역순 체크 허용**

   - 앞 단계 체크 없이도 뒷 단계 체크 시,  
     → **앞 단계가 자동으로 모두 체크**되어야 합니다.

2. **중간 해제 시 뒷 단계 해제**
   - 체크된 상태에서 중간 단계를 해제하면  
     → **그 이후 단계는 자동으로 체크 해제**되어야 합니다.

> 📌 그냥 구현하면 쉬울 수 있지만,  
> 사용 중인 라이브러리(`react-xyflow`)에 맞추다 보니 난이도 있었습니다.

---

## 🖼️ 상태 예시

**초기 상태**

<img width="852" alt="default" src="https://github.com/user-attachments/assets/ec23dc53-56d5-4a84-826e-ad333d818789" />

**활성화 상태**

<img width="844" alt="check" src="https://github.com/user-attachments/assets/0d169faf-c685-42b7-b98d-e4e5cbfac4aa" />

**비활성화 상태**

<img width="832" alt="uncheck" src="https://github.com/user-attachments/assets/0038462a-539f-4f43-baae-2fe29ac15e88" />

---

이처럼 기획과 개발이 유기적으로 협업하며 만들어낸 결과물이 바로 이 로드맵 기능이며,  
현재도 많은 유저들에게 **핵심 기능**으로 사랑받고 있습니다.
