# 📂 목록

- 🎗️ [폐지된 커뮤니티 기능](./community.md)
- 🎨 [디자인 리뉴얼 이슈 및 요청](./design.md)
- 👍 [로드맵 - 최고의 컨텐츠](./roadmap.md)
- 🍱 [다국어 지원을 위하여](./i18n.md)
- 🗺️ [3D Map 도입 및 성능 개선 과정](./3dmap.md)
- 📊 [Analytics, Search Console, AdSense 도입기 및 경험 공유](./google.md)
- 🔐 [NextAuth 도입기 – 프론트 중심 인증 경험](./auth.md)
- 🛠️ [프론트엔드 개발 비하인드 – 3번의 마이그레이션 여정](./migration.md)

---

# 🛠️ 프론트엔드 개발 비하인드 – 3번의 마이그레이션 여정

---

## 🚀 1차 개발 (React 17 기반 SPA)

**기간:** 2024.03.28 ~ 2024.07.13  
**기술 스택:** React 17, JavaScript

- 처음 프로젝트를 시작할 당시에는 가장 익숙한 React로 개발을 시작했습니다.
- 라우팅이나 전역 상태 관리 등은 기존에 경험했던 방식 그대로 적용했고, 빠르게 화면을 만들고 동작을 확인하는 데 초점을 맞췄습니다.
- 이 시점에서는 성능이나 SEO보다는 MVP 수준의 빠른 완성이 더 중요한 목표였습니다.

---

## 🔄 2차 마이그레이션 (Next.js 14 + TypeScript)

**기간:** 2024.07.13 ~ 2024.12.30  
**기술 스택:** Next.js 14 (App Router), TypeScript, SSG

- 운영을 시작한 후, SEO 측면에서 성능이 부족하고 초기 렌더링 속도가 느리다는 문제를 인식했습니다.
- 서버 사양도 좋지 않았기에 CSR 방식은 비효율적이라 판단하고, **SSG(Static Site Generation)**을 적용하기 위해 Next.js 14로 마이그레이션을 진행했습니다.
- 많은 페이지가 단순히 조회되는 용도였기에, **사전 렌더링 후 캐싱을 통해 빠르게 제공하는 SSG 방식이 적합하다고 판단**했습니다.

### 🧱 기술적 도전

- 기존에 사용하던 `<img>` 태그도 `next/image`로 교체하면서 **크기 지정, lazy loading 등 최적화 작업**을 병행해야 했습니다.
- `server` 컴포넌트 하위에 `client` 컴포넌트는 둘 수 있지만 그 반대가 안 된다는 제약 때문에 구조 설계에 꽤나 고민이 필요했습니다.
- TypeScript도 처음 도입하다 보니 로직이 복잡한 부분에서 **기존 코드가 타입을 완전히 무시하고 있었다는 사실을 체감**하게 되었습니다.
  - 예를 들어, 아이템 필터 로직이 main과 sub로 나뉘어 있었는데 구조가 비슷해 대충 구현했더니, TypeScript 적용 후 에러가 쏟아졌습니다.
  - 타입 정의를 정리하고, 잘못된 함수 로직을 재작성하는 데 상당한 시간이 소요됐습니다.

**완성된 ItemFilter Hook**

```js
/**
 * 3D 맵에서 화면에 표시할 아이템을 필터링 해주는 함수
 */
export const useItemFilter = (mapItem: JpgItemPath[]) => {
  const { newItemFilter, setItemFilter } = useAppStore((state) => state);
  const [viewItemList, setViewItemList] = useState<string[]>(
    extractValues(newItemFilter)
  );

  // itemFilter가 비어있을 때만 데이터를 가져옵니다.
  useEffect(() => {
    const getItem = async () => {
      try {
        const response = await fetch(`${API_ENDPOINTS.GET_ITEM_FILTER}`, {
          next: { revalidate: 14400 },
        });

        if (!response.ok) {
          throw new Error("Network response was not ok");
        }

        const result: FetchSchema = await response.json();

        setItemFilter(result.data);
      } catch (error) {
        console.error("Fetch error:", error);
        return [];
      }
    };

    if (newItemFilter.length === 0) {
      getItem();
    }
  }, [newItemFilter, setItemFilter]);

  // mapItem이 변경될 때마다 viewItemList를 업데이트합니다.
  const valuesList = useMemo(() => {
    const valuesSet = new Set<string>();
    mapItem.forEach((item) => {
      valuesSet.add(item.childValue);
      valuesSet.add(item.motherValue);
    });
    return [...valuesSet];
  }, [mapItem]);

  useEffect(() => {
    if (valuesList.length > 0) {
      setViewItemList(valuesList);
    }
  }, [valuesList]);

  const updateViewItemList = (
    newItems: string[],
    removeItems: string[] = []
  ) => {
    setViewItemList((prev) => {
      const updatedSet = new Set(prev);
      newItems.forEach((item) => updatedSet.add(item));
      removeItems.forEach((item) => updatedSet.delete(item));
      return [...updatedSet];
    });
  };

  /**
   * 아이템 클릭 이벤트
   */
  const onClickItem = (clickValue: string) => {
    const rootValueList = newItemFilter.map((item) => item.value);

    if (rootValueList.includes(clickValue)) {
      handleRootItemClick(clickValue);
    } else {
      handleChildItemClick(clickValue);
    }
  };

  /**
   * 아이템 전체 선택 또는 해제
   */
  const onClickAllItem = (isAll: boolean) => {
    setViewItemList(isAll ? [] : valuesList);
  };

  /**
   * 상위 값 클릭 시
   * viewItemList child가 전부 있는지 확인
   * 전부 있으면 모두 제거, 전부 있지 않으면 모두 추가
   */
  const handleRootItemClick = (clickValue: string) => {
    const rootList: ItemType = newItemFilter.find(
      (item) => item.value === clickValue
    )!;
    const childList = rootList.sub.map((childItem) => childItem.value);
    const shouldRemoveAllChild = checkAllChild(viewItemList, childList);
    const newItems = shouldRemoveAllChild ? [] : [...childList, clickValue];
    const removeItems = shouldRemoveAllChild ? [clickValue, ...childList] : [];
    updateViewItemList(newItems, removeItems);
  };

  /**
   * 하위 값 클릭 시
   * viewItemList 해당 값 있는지 확인
   * 있으면 item 제거, 없으면 item 추가
   * root의 모든 아이템 제거 시 root 제거, 모두 추가될 경우 root 추가
   */
  const handleChildItemClick = (clickValue: string) => {
    const rootList: ItemType = newItemFilter.find((item) =>
      findObjectWithValue(item, clickValue)
    )!;

    const childList = rootList.sub.map((childItem) => childItem.value);

    let updatedItemList = viewItemList.includes(clickValue)
      ? viewItemList.filter((item) => item !== clickValue)
      : [...viewItemList, clickValue];

    const isHaveAllChild = checkAllChild(updatedItemList, childList);
    const isHaveAnyMissingChild = checkSomeChild(updatedItemList, childList);

    // 전부 있거나, 몇 개만 있을 경우 root 추가
    if (isHaveAllChild || isHaveAnyMissingChild) {
      updatedItemList.push(rootList.value);
    }

    // 전부 없을 경우 root 제거
    if (!isHaveAnyMissingChild) {
      updatedItemList = updatedItemList.filter(
        (item) => item !== rootList.value
      );
    }

    const result = [...new Set(updatedItemList)];
    setViewItemList(result);
  };

  return { viewItemList, onClickItem, onClickAllItem };
};
```

### 📉 후일담

- SEO 개선을 위해 SSG 방식으로 구축했지만, 실제로는 대부분의 화면이 CSR 방식으로 렌더링되고 있다는 것을 나중에야 알게 됐습니다.
- `LightHouse` 점수는 좋아졌지만, **사용자 입장에서 SSG의 체감 효과는 크지 않았고**, SSG 설계가 제대로 되지 않았다는 것을 3차 마이그레이션에서 알게 되었습니다.

---

## 🔁 3차 마이그레이션 (React 19 + Next.js 15 + 패키지 최신화)

**기간:** 2024.12.30 ~ 현재  
**기술 스택:** React 19, Next.js 15, 최신 라이브러리 전체 업데이트

- 사이트 이용이 활발해지고 사용자 피드백이 많아지면서 전체적인 리팩토링과 보수가 필요하다는 점을 실감했습니다.
- 이 기회에 최신 버전인 **React 19와 Next.js 15로 업그레이드**를 진행했습니다.
  - RSC 최적화, React Compiler 등 새로운 기능 도입을 위한 준비 목적도 있었습니다.
- 그 과정에서 과거에 "SSG 기반으로 만들었다"고 생각했던 코드들이 사실상 대부분 CSR로 동작하고 있다는 사실을 명확히 인지하게 되었습니다.
  - `useEffect`로 데이터를 fetch하거나 동적으로 import한 부분들이 많았기 때문입니다.

### ⚠️ 현재의 이슈

- 전역 상태 관리를 위해 사용 중인 `zustand`가 아직 React 19를 정식으로 지원하지 않아, 개발 중 콘솔에 warning이 출력되고 있습니다.
- 다른 상태 관리 라이브러리(e.g. Recoil, Jotai, Redux Toolkit 등)로의 전환을 검토 중입니다.

---

## 🧭 회고

| 마이그레이션  | 주요 이유                   | 성과                                         | 한계/이슈                               |
| ------------- | --------------------------- | -------------------------------------------- | --------------------------------------- |
| 1차 (SPA)     | 익숙한 환경에서 빠른 개발   | MVP 완성 및 빠른 피드백 확보                 | SEO 미지원, CSR로 인한 초기 렌더링 문제 |
| 2차 (Next.js) | SEO, 서버 부담 줄이기       | SSG 도입, TypeScript 정착                    | 잘못된 CSR 구조, 타입 문제 노출         |
| 3차 (최신화)  | 트래픽 대응, 최신 기술 적용 | 성능 개선, 구조 개선, Next.js 기능 정착 시도 | zustand 비호환, SSG/CSR 혼재 구조       |

---

이번 마이그레이션들을 통해 단순히 버전만 올리는 것이 아니라  
**프로젝트의 구조, 방향성, 기술 선택의 기준**을 하나씩 점검하고 다듬을 수 있었습니다.

앞으로는 React 19와 Next.js의 서버 컴포넌트 최적화를 좀 더 잘 활용하는 방향으로 리팩토링을 이어나갈 계획입니다.
