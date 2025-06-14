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

# 🗺️ 3D Map 도입 및 성능 개선 과정

## 도입 초기의 어려움

- **Three.js에 대한 인식은 있었지만** 실제 사용해본 적은 없었습니다.
- "언젠가 써보고 싶다"는 생각은 있었으나, 막상 도입하려 하니 **새로운 언어를 배우는 듯한 어려움**을 느껴서 많이 어려웠습니다.
- 다양한 시행착오를 겪으며, **초기에는 Collada(.dae) 파일 렌더링**에 성공하는 수준까지 도달했었습니다.

**Three.js**

![스크린샷 2025-05-28 오전 8 57 19](https://github.com/user-attachments/assets/18172fd5-3c6f-4b0e-bff5-457428b77334)

---

## 초기 도입: Collada 렌더링 기반

- 선택지: `gltf` vs `collada`
  - `gltf`는 테두리 정보를 저장할 수 없었기 때문에 `collada`를 선택했습니다.
  - 처음에 잘 알아보지 못하고 선택했었는데, 테두리가 없으니 건물들이 도형이 아니라 2D 처럼 보여서 테두리가 가능한 Collada를 선택했습니다.
- Collada 기반 로딩 후, 지도가 크더라도 **성능이 좋고 부드러운 렌더링**이 가능해 만족스러웠음.

## 문제 발생

- 알아본 것과 다르게 건축물에 **테두리가 자동으로 그려지지 않았습니다**.
- **초기 렌더링 속도가 20초**가 넘어갔습니다.
- Collada 파일 자체가 **용량이 너무 컸습니다** (최대 80MB 이상).
- 테두리 문제 해결 방안:
  - 데이터를 불러온 후, **모든 건축물에 테두리를 수작업으로 추가하는 프로세스**를 개발하여 적용.
- 하지만 프로세스가 추가 되니 **초기 렌더링 속도가 30초 이상** 소요되는 심각한 지연이 발생했습니다.

**불러온 후 테두리 추가하는 코드**

```js
export const useLoadMap = (map_three_path: string, isMap: boolean) => {
  const [map, setMap] = useState<ColladaData | null>(null);

  useEffect(() => {
    const loadMapData = async () => {
      try {
        const loadedColladaData = await new Promise<Collada>(
          (resolve, reject) => {
            const loader = new ColladaLoader();
            loader.load(
              formatImage(map_three_path),
              (collada) => resolve(collada),
              undefined,
              (error) => reject(error)
            );
          }
        );

        // 모델의 모든 자식 노드를 반복하여 선을 입력값으로 설정
        loadedColladaData.scene.traverse((child) => {
          if ((child as THREE.Line).isLine) {
            // 선의 재질을 설정
            const lineMaterial = new LineBasicMaterial({
              color: ALL_COLOR.MAP_BLACK,
            });
            (child as THREE.Line).material = lineMaterial;
          }
        });
        // 상태를 업데이트
        setMap({ colladaData: loadedColladaData });
      } catch (e) {
        console.log(e);
      }
    };

    if (map_three_path !== null) {
      loadMapData();
    }
  }, [map_three_path]);
  return map;
};
```

---

## 최적화 시도 및 성능 개선

- 다양한 방법을 고민하던 중, `pmndrs`의 [`gltfjsx`](https://github.com/pmndrs/gltfjsx) 도구를 발견했습니다.
  - `gltf` 파일을 **`glb`로 변환**한 뒤, **React용 JSX 코드로 자동 변환**해주는 도구.
- 해당 방식 적용 후 **눈에 띄는 개선 효과**를 얻었습니다.
  - **파일 용량이 10MB 이하로 감소**.
  - **렌더링 속도 1~2초로 획기적으로 개선**.
- 테두리 문제는 건축물에 조명을 주어 해결했습니다.
  - 이런 설정을 미리 찾아봤어야 했는데, 너무 급하게 만들기만 했던 거 같습니다.
- 현재까지도 해당 방식을 **안정적으로 사용 중**이며,  
  성능과 유지보수 측면에서 매우 만족스러운 결과를 얻고 있습니다.

**조명 옵션**

```js
<pointLight position={[0, 0, 0]} intensity={2} />
```

## gltf에서 테두리를 그리지 못햇던 이유

검색을 해도 그릴 수 있다고 나오는데 개발하면서 테두리가 그려지지 않아 검색을 해보았습니다.

그리고 제가 자체적으로 내린 결론은 gltfjsx를 사용해서 용량을 압축하고, gltf 파일을 glb로 변환하는데요, 이 과정에서 테두리 구역이 좀 뭉쳐졌습니다.

기존에는 명확하게 건축물이 분리되어있었다면, 변환후에는 명확하지가 않았습니다.

그래도 명암을 사용해서 해결하는 방법을 찾아서 다행이라고 생각합니다.

## 현재

현재는 상당히 만족스럽게 운영되고 있습니다.

**대화형 지도**

<img width="1189" alt="스크린샷 2025-05-28 오전 8 54 50" src="https://github.com/user-attachments/assets/d868a634-6f68-47f3-a1d4-f4e5d60baead" />
