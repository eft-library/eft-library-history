# 📂 목록

- 🌐 [다국어 지원](./i18n_data.md)
- 🛠️ [ORM 사용 (With SqlAlchemy)](./orm.md)
- 🔐 [Google 로그인 도입기: 사용자 피드백으로 시작된 변화](./token_check.md)

# 🛠️ ORM 사용 (With SqlAlchemy)

이 프로젝트는 **FastAPI와 SQLAlchemy ORM을 처음으로 도입**하여 사용해본 프로젝트 입니다.

Java에서는 **주로 MyBatis를 사용했기 때문에 ORM 자체가 낯설었지만**, Python 환경에서의 유연함을 경험해보고자 선택하게 되었습니다.

Java로 개발할 때에 JPA가 아닌 MyBatis를 주로 사용하면서 ORM을 써본 경험이 없었는데, 확실히 처음 써보니 미숙한 부분이 많았던거 같습니다.

> 점점 복잡한 쿼리의 경우에는 text -> execute로 가게 되었습니다...  
> 그래도 간단한 경우에는 Model을 만들고 ORM을 적용해보려고 노력했습니다.

## ✅ 관계형 조회: 외래키 기반 SubMenu 전체 조회

![스크린샷 2025-05-21 오전 8 24 56](https://github.com/user-attachments/assets/9bf67cb4-8fe0-44b2-a9f7-97cb1a2a25b2)

현재 사이트의 Menu 정보입니다.

Main Group으로 묶여 있고, Main Group의 value를 기준으로 Sub Menu를 연결 짓고 있습니다.

ORM에서 관계형 데이터를 쉽게 조회하는 방법을 몰랐지만,
`relationship` + `subqueryload`를 통해 아래와 같이 해결할 수 있었습니다.

![스크린샷 2025-05-21 오전 10 54 58](https://github.com/user-attachments/assets/2c8bda42-c1cc-4a40-86d3-68d83648e816)

## 🔍 복합 정보 조회: 아이템 상세 및 연관 정보 일괄 조회

![스크린샷 2025-05-21 오전 8 17 50](https://github.com/user-attachments/assets/533576f2-63e0-4c09-b49e-962769116f0d)

FastAPI + SQLAlchemy 환경에서 **ORM만으로 처리하기 어려운 복합 관계의 데이터 조회가 필요했습니다.**

예시:

- 은신처 제작/건설에 사용되는 아이템
- 특정 NPC의 바터 보상 아이템
- 퀘스트 보상/요구 아이템

ORM의 한계로 인해 해당 부분은 `text -> execute()` 방식으로 전환하였습니다.

> 아이템 상세의 경우 성능 개선을 위하여 item_detail_i18n 테이블을 만들어, airflow를 통해 적재를 진행 한 후 조회를 하는 상태로 변경 되었습니다.

<details>
<summary style="
    font-weight: bold;
    background-color: #f0f8ff;
    color: #003366;
    padding: 8px 12px;
    border: 1px solid #ccc;
    border-radius: 4px;
    cursor: pointer;
  ">아이템 상세 조회 쿼리</summary>

```python

@staticmethod
def get_item_detail_query():
"""
아이템 상세 정보 조회 쿼리
"""

return """
WITH target_item AS (SELECT *
                     FROM item_i18n
                     WHERE url_mapping = :url_mapping
                     limit 1),

     -- 📦 바터 정보
     filtered_barters AS (SELECT n.id                      AS npc_id,
                                 n.name,
                                 n.image,
                                 jsonb_build_object(
                                         'level', barter ->> 'level',
                                         'rewardItems', reward,
                                         'requiredItems', barter -> 'requiredItems'
                                 )                         AS matching_barter,
                                 reward -> 'item' ->> 'id' AS reward_item_id
                          FROM npc_i18n n,
                               jsonb_array_elements(n.barter_info) AS barter,
                               jsonb_array_elements(barter -> 'rewardItems') AS reward),

     -- 🛠 은신처 건설에 사용되는 정보
     filtered_hideout AS (SELECT thir.id,
                                 thir.level_id,
                                 thir.name,
                                 thir.quantity,
                                 thir.count,
                                 thir.image,
                                 thir.item_id,
                                 thm.name as master_name,
                                 thm.id   as master_id
                          FROM hideout_item_require_i18n thir
                                   LEFT JOIN hideout_master_i18n thm
                                             ON SPLIT_PART(thir.level_id, '-', 1) = thm.id),

     -- 🛠 은신처 제작에 사용되는 정보
     filtered_crafts AS (SELECT DISTINCT ON (thc.id) thc.*,
                                                     thm.name                as master_name,
                                                     thm.id                  as master_id,
                                                     elem -> 'item' ->> 'id' AS required_item_id
                         FROM hideout_crafts_i18n thc
                                  LEFT JOIN LATERAL jsonb_array_elements(thc.req_item) AS elem on True
                                  LEFT JOIN hideout_master_i18n thm ON SPLIT_PART(thc.level_id, '-', 1) = thm.id),

     -- 🎯 퀘스트 보상으로 사용되는 정보
     filtered_quests AS (SELECT distinct on (qa.id) qa.id                                           AS quest_id,
                                                    qa.name,
                                                    qa.npc_id,
                                                    qa.url_mapping,
                                                    tn.name                                         as npc_name,
                                                    tn.image                                        AS npc_image,
                                                    jsonb_array_elements(finish_rewards -> 'items') AS reward_elem
                         FROM quest_i18n qa
                                  left join npc_i18n tn on qa.npc_id = tn.id
                         WHERE qa.name is not null),

     -- ❗ questItem에 포함된 경우 (예: giveQuestItem, findQuestItem)
     required_quests_by_quest_item AS (SELECT DISTINCT ON (q.id) q.id     AS quest_id,
                                                                 q.name,
                                                                 q.url_mapping,
                                                                 tn.name  AS npc_name,
                                                                 tn.image AS npc_image,
                                                                 obj      AS objective
                                       FROM quest_i18n q
                                                LEFT JOIN LATERAL jsonb_array_elements(q.objectives) AS obj ON TRUE
                                                LEFT JOIN npc_i18n tn ON q.npc_id = tn.id
                                       WHERE obj ->> 'type' IN ('findQuestItem', 'giveQuestItem')
                                         AND q.name IS NOT NULL),

     -- ❗ items 배열에 포함된 경우 (예: giveItem, plantItem, findItem)
     required_quests_by_items_array AS (SELECT distinct on (q.id) q.id     AS quest_id,
                                                                  q.name,
                                                                  q.url_mapping,
                                                                  tn.name  as npc_name,
                                                                  tn.image AS npc_image,
                                                                  obj      AS objective
                                        FROM quest_i18n q
                                                 LEFT JOIN LATERAL jsonb_array_elements(q.objectives) AS obj ON TRUE
                                                 LEFT JOIN npc_i18n tn on q.npc_id = tn.id
                                        WHERE obj ->> 'type' IN ('plantItem', 'giveItem', 'findItem')
                                          AND q.name is not null
                                          AND EXISTS (SELECT 1
                                                      FROM jsonb_array_elements(obj -> 'items') AS item
                                                      WHERE item ->> 'id' = (SELECT id FROM target_item))),

     item_with_details AS (SELECT ti.id,
                                  ti.name,
                                  ti.category,
                                  ti.image,
                                  ti.image_width,
                                  ti.image_height,
                                  ti.info,
                                  ti.update_time,
                                  ti.url_mapping,

                                  -- 은신처 아이템 요구 정보
                                  COALESCE(
                                                  json_agg(
                                                  DISTINCT jsonb_build_object(
                                                          'id', thir.id,
                                                          'level_id', thir.level_id,
                                                          'name', thir.name,
                                                          'quantity', thir.quantity,
                                                          'count', thir.count,
                                                          'image', thir.image,
                                                          'item_id', thir.item_id,
                                                          'master_name', thir.master_name,
                                                          'master_id', thir.master_id
                                                           )
                                                          ) FILTER (WHERE thir.id IS NOT NULL),
                                                  '[]'
                                  ) AS hideout_items,

                                  -- 은신처 제작에 사용
                                  COALESCE(
                                                  json_agg(
                                                  DISTINCT jsonb_build_object(
                                                          'id', thc.id,
                                                          'name', thc.name,
                                                          'level_id', thc.level_id,
                                                          'level', thc.level,
                                                          'duration', thc.duration,
                                                          'req_item', thc.req_item,
                                                          'reward_item_id', thc.reward_item_id,
                                                          'image', thc.image,
                                                          'quantity', thc.quantity,
                                                          'master_name', thc.master_name,
                                                          'master_id', thc.master_id
                                                           )
                                                          ) FILTER (WHERE thc.id IS NOT NULL),
                                                  '[]'
                                  ) AS used_in_crafts,

                                  -- NPC 바터 보상으로 나오는 정보
                                  COALESCE(
                                                  json_agg(
                                                  DISTINCT jsonb_build_object(
                                                          'npc_id', fb.npc_id,
                                                          'npc_image', fb.image,
                                                          'npc_name', fb.name,
                                                          'barter_info', fb.matching_barter
                                                           )
                                                          ) FILTER (WHERE fb.npc_id IS NOT NULL),
                                                  '[]'
                                  ) AS rewarded_by_npcs,

                                  -- 퀘스트 보상으로 나오는 정보
                                  COALESCE(
                                                  json_agg(
                                                  DISTINCT jsonb_build_object(
                                                          'quest_id', fq.quest_id,
                                                          'name', fq.name,
                                                          'npc_name', fq.npc_name,
                                                          'npc_image', fq.npc_image,
                                                          'url_mapping', fq.url_mapping,
                                                          'reward', fq.reward_elem
                                                           )
                                                          ) FILTER (WHERE fq.quest_id IS NOT NULL),
                                                  '[]'
                                  ) AS rewarded_by_quests,

                                  -- 📌 questItem에 들어 있는 퀘스트
                                  COALESCE(
                                                  json_agg(
                                                  DISTINCT jsonb_build_object(
                                                          'quest_id', rqi.quest_id,
                                                          'name', rqi.name,
                                                          'npc_name', rqi.npc_name,
                                                          'npc_image', rqi.npc_image,
                                                          'url_mapping', rqi.url_mapping,
                                                          'objective', rqi.objective
                                                           )
                                                          ) FILTER (
                                                      WHERE rqi.objective -> 'questItem' ->> 'id' = ti.id
                                                      ),
                                                  '[]'
                                  ) AS required_by_quest_item,

                                  -- 📌 items 배열에 들어 있는 퀘스트
                                  COALESCE(
                                                  json_agg(
                                                  DISTINCT jsonb_build_object(
                                                          'quest_id', rqa.quest_id,
                                                          'name', rqa.name,
                                                          'npc_name', rqa.npc_name,
                                                          'npc_image', rqa.npc_image,
                                                          'url_mapping', rqa.url_mapping,
                                                          'objective', rqa.objective
                                                           )
                                                          ) FILTER (WHERE rqa.quest_id IS NOT NULL),
                                                  '[]'
                                  ) AS required_by_quest_item_array

                           FROM target_item ti
                                    LEFT JOIN filtered_hideout thir ON ti.id = thir.item_id
                                    LEFT JOIN filtered_crafts thc ON thc.required_item_id = ti.id
                                    LEFT JOIN filtered_barters fb ON fb.reward_item_id = ti.id
                                    LEFT JOIN filtered_quests fq ON fq.reward_elem -> 'item' ->> 'id' = ti.id
                                    LEFT JOIN required_quests_by_quest_item rqi
                                              ON rqi.objective -> 'questItem' ->> 'id' = ti.id
                                    LEFT JOIN required_quests_by_items_array rqa ON TRUE
                           GROUP BY ti.id, ti.name, ti.name, ti.category, ti.image,
                                    ti.image_width, ti.image_height, ti.info, ti.update_time, ti.url_mapping)

SELECT *
FROM item_with_details
"""
```

</details>
