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

# 🎗️ 폐지된 커뮤니티 기능

사용자 기능 추가와 함께 커뮤니티를 도입하자는 논의가 있었고, 실제로 짧은 기간 동안 운영되었던 커뮤니티 기능입니다.

---

# 📄 게시글 기능

## 📌 주요 기능

1. **카테고리별 게시판 분류**
2. **좋아요 / 싫어요**
3. **조회수**
4. **작성 / 수정 / 삭제**
5. **인기글 시스템**
6. **신고 기능**

---

## 🗂️ 게시판 카테고리

총 6개의 카테고리로 구성되어 있으며,  
**각 카테고리마다 별도의 테이블**에 데이터를 저장하는 구조입니다.

- `Arena`
- `PVE`
- `PVP`
- `Tip`
- `Forum`
- `Question`

**Tip Table**

<img width="488" alt="board_schema" src="https://github.com/user-attachments/assets/8b0baf7c-e66f-4df3-aa21-d2120403f26a" />

---

## 👍 좋아요 / 👎 싫어요

- **좋아요만 DB에 저장**되며,
- 싫어요는 별도 저장하지 않고, **좋아요 수를 감소시키는 방식**으로 처리합니다.
- **싫어요 개수는 UI에 표시하지 않습니다.** (사용자 반감을 줄이기 위함)

---

## 🔥 인기글 시스템

### ✅ 인기글 선정 조건

- **좋아요 수 ≥ 10**일 경우 인기글로 지정합니다.
- **카테고리 구분 없이 통합 관리**합니다.
- 인기글은 **별도의 테이블에서 관리**합니다.
- 사용자에게는 **인기글 전용 페이지** 또는 **인기글 필터**로 제공합니다.
- **Airflow를 통해 1시간 주기로 통계 산출**

**Issue Table**

<img width="503" alt="board_issue" src="https://github.com/user-attachments/assets/b2f74b8d-7000-4c46-8b1d-3ff8f6f57754" />

---

## 👁️ 조회수 / ✍ 작성·수정·삭제 / 🚨 신고 기능

### 👁️ 조회수 처리

- **중복 조회 방지를 위해 `localStorage` 사용**
- 조회 이력이 없는 경우에만 FastAPI에 요청하여 조회수 증가

```js
import { MAX_UUID_COUNT, LOCAL_STORAGE_KEY } from "@/util/consts/libraryConsts";
import { useEffect } from "react";
import USER_API_ENDPOINTS from "@/config/userEndPoints";
import { useSession } from "next-auth/react";

export default function useViewCount(boardId: string, boardType: string) {
  const { data: session } = useSession();
  useEffect(() => {
    const addBoardViewCount = async () => {
      try {
        const res = await fetch(`${USER_API_ENDPOINTS.ADD_BOARD_VIEW_COUNT}`, {
          method: "POST",
          headers: {
            Authorization:
              session && session.accessToken
                ? `Bearer ${session.accessToken}`
                : "Bearer ",
            "Content-Type": "application/json",
          },
          body: JSON.stringify({
            board_id: boardId,
            board_type: boardType,
          }),
        });
        const response = await res.json();
      } catch (error) {
        console.log(error);
      }
    };

    if (boardId && session) {
      // 로컬 스토리지에서 UUID 목록 가져오기
      const storedUuids =
        JSON.parse(localStorage.getItem(LOCAL_STORAGE_KEY)) || [];

      // 새로 들어온 board_id가 이미 목록에 존재하는지 확인
      if (!storedUuids.includes(boardId)) {
        // UUID 목록이 10만 개를 초과할 경우 목록을 비움
        if (storedUuids.length >= MAX_UUID_COUNT) {
          localStorage.removeItem(LOCAL_STORAGE_KEY);
        }

        // 새로운 게시판 id 추가
        const updatedUuids = [
          ...(JSON.parse(localStorage.getItem(LOCAL_STORAGE_KEY)) || []),
          boardId,
        ];

        // 업데이트된 UUID 목록을 로컬 스토리지에 저장
        localStorage.setItem(LOCAL_STORAGE_KEY, JSON.stringify(updatedUuids));

        // 조회수 통신
        addBoardViewCount();
      }
    }
  }, [boardId]);
}
```

---

### ✍ 작성 / 수정 / 삭제

- **사용자 인증은 Header에 담긴 JWT Token으로 처리**합니다.
- 게시글 작성자와 Token 정보가 일치할 경우에만 수정/삭제 가능합니다.
- react-quill을 Custom 하여 사용했습니다.

**이미지 처리**

게시글 내용중 이미지를 클릭할 경우 확대하여 보여주고 싶었습니다.

Click Event를 감지하여 해당 html tag가 img인 경우 Modal과 연계하여 표출해주었습니다.

```js
export default function ImgWithZoom({ content }) {
  const { isOpen, onOpen, onClose } = useDisclosure();
  const [selectedImage, setSelectedImage] = useState("");

  const handleImageClick = (event) => {
    const img = event.target;
    if (img.tagName === "IMG") {
      setSelectedImage(img.src);
      onOpen();
    }
  };

  return (
    <>
      <Text
        className="ql-editor"
        dangerouslySetInnerHTML={{ __html: content }}
        onClick={handleImageClick}
      />
      <Modal isOpen={isOpen} onClose={onClose} size="xl">
        <ModalOverlay />
        <ModalContent
          bg={ALL_COLOR.EDITOR_IMAGE_SHADOW}
          maxW="80vw"
          minW="20vw"
        >
          <ModalBody>
            <Image
              src={selectedImage}
              w={"100%"}
              alt="Selected Image"
              objectFit="contain"
            />
          </ModalBody>
        </ModalContent>
      </Modal>
    </>
  );
}
```

---

### 🚨 신고 기능

- **게시글 ID 및 신고 사유를 수신하여 DB에 저장하는 간단한 CRUD API**

신고 요청 예시:

```js
const reportPost = async () => {
  try {
    let finalReason = "";
    if (selectReson !== "other") {
      finalReason = selectReson;
    } else {
      finalReason = reason;
    }

    const response = await fetchUserData(
      USER_API_ENDPOINTS.REPORT_POST,
      "POST",
      { board_id: post.id, board_type: post.type, reason: finalReason },
      session
    );
    if (response.status === 200) {
      alert("해당 게시글이 신고되었습니다.");
      onClose();
    } else {
      alert("잠시후 다시 시도해주세요");
      onClose();
    }
  } catch (error) {
    console.error(error);
  }
};
```

**Report Table**

<img width="507" alt="board_report" src="https://github.com/user-attachments/assets/2aa596e8-9f41-4874-ade5-39894731e1f9" />

---

# 💬 댓글 기능

## 주요 기능

1. **좋아요 / 싫어요**
2. **작성 / 수정 / 삭제**
3. **신고**
4. **이모티콘**
5. **대댓글**

---

## 👍 좋아요 / 👎 싫어요

- 댓글의 좋아요/싫어요는 사용자에게 **모두 노출**되며,
- 사용자 반감이 적을 것으로 판단하여 UI에 표시하게 되었습니다.

---

## ✍ 작성·수정·삭제 / 🚨 신고 기능

- **기본적으로 게시글과 동일한 방식**으로 처리됩니다.

---

## 😀 이모티콘 기능

- 댓글에 **이모티콘 기능은 꼭 넣고 싶었던 요소**였습니다.
- 댓글을 보다 **다채롭고 유쾌하게 만드는 효과**가 있다고 판단하였습니다.
- 기존에는 textarea를 통한 입력을 받았었는데, 이모티콘요소를 추가하기 위해서 게시글 작성과 동일한 react-quill을 사용했습니다.
- 만들고나니 생각한 대로 나오긴 했는데 댓글이 마치 작은 게시글 처럼 동영상, 이미지 등 너무 많은 자유도가 생겨버렸습니다.

> 추후에 개발하게 된다면, 이모티콘만 넣을수 있게 Custom 해서 적용할 예정입니다.

---

## 💬 대댓글 기능

- **가장 구현이 복잡했던 기능 중 하나**
- 처음에는 단순할 것 같았지만, 생각보다 **구조적으로 많은 고민이 필요했습니다.**
- `Closure Table`, `Adjacency List`를 학습하며 구조를 설계했습니다.
- **최종적으로는 Adjacency List + Recursive 쿼리** 방식으로 구현했습니다.

> 구현 이후 쿼리가 복잡해지는 문제로 인해 다음에는 유지보수가 더 쉬운 구조로 새롭게 만들 예정입니다.

<details>
<summary style="font-weight: bold">댓글 조회 쿼리</summary>

```python
    def get_comment_query():
        """
        특정 게시글 댓글 전체 조회 쿼리
        """

        return """
                WITH RECURSIVE comment_tree AS (
                SELECT
                    tkl_comments.ID,
                    tkl_comments.BOARD_ID,
                    tkl_comments.USER_EMAIL,
                    tkl_comments.BOARD_TYPE,
                    tkl_comments.PARENT_ID,
                    CASE
                        WHEN tkl_comments.is_delete_by_admin THEN '<p>해당 댓글은 관리 규정에 따라 삭제되었습니다.</p>'
                        WHEN tkl_comments.is_delete_by_user THEN '<p>해당 댓글은 작성자에 의해 삭제되었습니다.</p>'
                        ELSE tkl_comments.CONTENTS
                    END AS CONTENTS,
                    tkl_comments.DEPTH,
                    tkl_comments.CREATE_TIME,
                    tkl_comments.UPDATE_TIME,
                    tkl_comments.is_delete_by_admin,
                    tkl_comments.is_delete_by_user,
                    tkl_comments.like_count,
                    tkl_comments.dislike_count,
                    tkl_comments.ID AS root_id,
                    ARRAY[tkl_comments.ID] AS path,
                    tkl_comments.CREATE_TIME AS root_create_time,
                    a.nick_name,
                    a.icon,
                    b.nick_name as parent_nick_name
                FROM TKL_COMMENTS
                LEFT JOIN TKL_USER a ON tkl_comments.user_email = a.email
                LEFT JOIN TKL_USER b ON tkl_comments.parent_user_email = b.email
                WHERE DEPTH = 1 AND board_id = :board_id
                UNION ALL
                SELECT
                    c.ID,
                    c.BOARD_ID,
                    c.USER_EMAIL,
                    c.BOARD_TYPE,
                    c.PARENT_ID,
                    CASE
                        WHEN c.is_delete_by_admin THEN '<p>해당 댓글은 관리 규정에 따라 삭제되었습니다.</p>'
                        WHEN c.is_delete_by_user THEN '<p>해당 댓글은 작성자에 의해 삭제되었습니다.</p>'
                        ELSE c.CONTENTS
                    END AS CONTENTS,
                    c.DEPTH,
                    c.CREATE_TIME,
                    c.UPDATE_TIME,
                    c.is_delete_by_admin,
                    c.is_delete_by_user,
                    c.like_count,
                    c.dislike_count,
                    ct.root_id,
                    ct.path || c.ID,
                    ct.root_create_time,
                    a.nick_name,
                    a.icon,
                    b.nick_name as parent_nick_name
                FROM TKL_COMMENTS c
                INNER JOIN comment_tree ct ON c.PARENT_ID = ct.ID
                INNER JOIN TKL_USER a ON c.user_email = a.email
                INNER JOIN TKL_USER b ON c.parent_user_email = b.email
                WHERE c.board_id = :board_id
            )
            SELECT
                ct.*,
                CASE
                    WHEN tcl.comment_id IS NOT NULL THEN true
                    ELSE false
                END AS is_liked_by_user,
                CASE
                    WHEN tcdl.comment_id IS NOT NULL THEN true
                    ELSE false
                END AS is_disliked_by_user,
                b.ban_reason,
                b.ban_start_time,
                b.ban_end_time
            FROM
                comment_tree ct
            LEFT JOIN
                tkl_comment_like tcl
                ON ct.id = tcl.comment_id AND tcl.user_email = :user_email
            LEFT JOIN
                tkl_comment_dislike tcdl
                ON ct.id = tcdl.comment_id AND tcdl.user_email = :user_email
            LEFT JOIN
                tkl_user_ban b
                ON ct.user_email = b.user_email
            {where_clause}
            """
```

</details>

# 개발, 운영, 끝

커뮤니티 기능은 내부 관계자 기준으로 완성되었다는 판단과 함께 약 한 달간 운영했지만, 다음과 같은 이유로 **폐지되었습니다.**

1. **커뮤니티 관리자의 부재**  
   24시간 상주하며 관리할 인력이 없었습니다.  
   (신고가 많을 경우 알림을 받아 처리하는 방안도 고려했지만, 확실한 방법이 아니라 판단했습니다.)

2. **기능의 불안정성**  
   제가 기능을 충분히 안정적으로 만들지 못했던 것 같습니다.  
   (모든 기능에 예외 처리는 되어 있지만, UX 경험을 위해서는 보다 고도화된 예외 처리가 필요하다고 느꼈습니다.)

3. **프로세스 확립의 부족**  
   개발 인력이 저 혼자였기 때문에, 다양한 유스케이스를 충분히 커버하지 못했습니다.

아직 커뮤니티 기능에 대한 논의가 재개되지는 않았습니다.  
하지만 개인적으로는 해당 기능에 대해 계속해서 고민하고 있습니다.

만약 사용자 여러분의 피드백을 통해 커뮤니티 기능에 대한 수요가 확인된다면,  
**다시 기능을 기획하고 개발할 것입니다.**

**🙏 초기 버전보다 훨씬 더 발전된 형태로 돌아오겠습니다.**
