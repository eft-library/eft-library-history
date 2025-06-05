# 📂 목록

- 🪤 [Airflow 구축하기 (Nas, Ubuntu)](./airflow.md)
- ⚠️ [데이터 불일치](./different_data.md)
- 🌐 [다국어 데이터 매핑 및 처리량 증가 문제](./i18n_mapping.md)
- 🔹 [다국어 원천 데이터의 신뢰도 문제](./untranslated_data.md)
- 📦 [Data Dump 자동화 설정](./data_dump.md)
- 🐹 [시스템 Health Check 구축](./health_check.md)
- 🧠 [아이템 상세 페이지 성능 튜닝 후기](./item_detail.md)

# 🧠 아이템 상세 페이지 성능 튜닝 후기

## 📌 개요

아이템 상세 페이지 조회 시, 응답 시간이 너무 오래 걸리는 문제가 발생하였습니다.  
기존 조회는 5초 이내로 끝났지만, 사용자 입장에서 체감상 충분히 불편할 수 있다고 판단하여 성능 개선 작업을 진행하게 되었습니다.

**기존 아이템 상세 조회 속도**

https://github.com/user-attachments/assets/30fd5cf6-28f2-4333-bba0-1d5f299c1013

---

## 📄 쿼리 설명 및 성능 분석

### ✅ 쿼리 목적

`item_i18n` 테이블을 기준으로 하여, 해당 아이템이 어떤 용도로 사용되는지 또는 어떤 보상으로 제공되는지를 파악하여  
**하나의 JSON 형태로 상세 정보를 반환**하는 목적의 쿼리입니다.

**기존 사용 쿼리**

```sql
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
                 filtered_crafts AS (SELECT thc.*,
                                            thm.name                as master_name,
                                            thm.id                  as master_id,
                                            elem -> 'item' ->> 'id' AS required_item_id
                                     FROM hideout_crafts_i18n thc
                                              LEFT JOIN LATERAL jsonb_array_elements(thc.req_item) AS elem on True
                                              LEFT JOIN hideout_master_i18n thm ON SPLIT_PART(thc.level_id, '-', 1) = thm.id),

                 -- 🎯 퀘스트 보상으로 사용되는 정보
                 filtered_quests AS (SELECT qa.id                                           AS quest_id,
                                            qa.name,
                                            qa.npc_id,
                                            qa.url_mapping,
                                            tn.name                                         as npc_name,
                                            tn.image                                        AS npc_image,
                                            jsonb_array_elements(finish_rewards -> 'items') AS reward_elem
                                     FROM quest_i18n qa
                                              left join npc_i18n tn on qa.npc_id = tn.id),

                 -- 🎯 offer unlock으로 사용되는 정보
                 filtered_quests_offer_unlock AS (SELECT qa.id AS quest_id,
                                                         qa.name,
                                                         qa.npc_id,
                                                         qa.url_mapping,
                                                         tn.name as npc_name,
                                                         tn.image AS npc_image,
                                                         jsonb_array_elements(finish_rewards -> 'offerUnlock') AS reward_elem
                                                  FROM quest_i18n qa
                                                           left join npc_i18n tn on qa.npc_id = tn.id),

                 -- 🧩 craftUnlock.rewardItems[*].item.id에 포함된 경우 (finish_rewards 안)
                 filtered_quests_craft_unlock AS (
                     SELECT qa.id AS quest_id,
                            qa.name,
                            qa.npc_id,
                            qa.url_mapping,
                            tn.name as npc_name,
                            tn.image AS npc_image,
                            reward_item
                     FROM quest_i18n qa
                              LEFT JOIN npc_i18n tn ON qa.npc_id = tn.id
                              LEFT JOIN LATERAL jsonb_array_elements(qa.finish_rewards -> 'craftUnlock') AS craft_unlock ON TRUE
                              LEFT JOIN LATERAL jsonb_array_elements(craft_unlock -> 'rewardItems') AS reward_item ON TRUE
                     WHERE reward_item -> 'item' ->> 'id' = (SELECT id FROM target_item)
                 ),

                 -- ❗ questItem에 포함된 경우 (예: giveQuestItem, findQuestItem)
                 required_quests_by_quest_item AS (SELECT q.id     AS quest_id,
                                                          q.name,
                                                          q.url_mapping,
                                                          tn.name  AS npc_name,
                                                          tn.image AS npc_image,
                                                          obj      AS objective
                                                   FROM quest_i18n q
                                                            LEFT JOIN LATERAL jsonb_array_elements(q.objectives) AS obj ON TRUE
                                                            LEFT JOIN npc_i18n tn ON q.npc_id = tn.id
                                                   WHERE obj ->> 'type' IN ('findQuestItem', 'giveQuestItem')),

                 -- ❗ items 배열에 포함된 경우 (예: giveItem, plantItem, findItem)
                 required_quests_by_items_array AS (SELECT q.id     AS quest_id,
                                                           q.name,
                                                           q.url_mapping,
                                                           tn.name  as npc_name,
                                                           tn.image AS npc_image,
                                                           obj      AS objective
                                                    FROM quest_i18n q
                                                             LEFT JOIN LATERAL jsonb_array_elements(q.objectives) AS obj ON TRUE
                                                             LEFT JOIN npc_i18n tn on q.npc_id = tn.id
                                                    WHERE obj ->> 'type' IN ('plantItem', 'giveItem', 'findItem')
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

                                              -- 퀘스트 보상으로 나오는 정보
                                              COALESCE(
                                                              json_agg(
                                                              DISTINCT jsonb_build_object(
                                                                      'quest_id', fqon.quest_id,
                                                                      'name', fqon.name,
                                                                      'npc_name', fqon.npc_name,
                                                                      'npc_image', fqon.npc_image,
                                                                      'url_mapping', fqon.url_mapping,
                                                                      'reward', fqon.reward_elem
                                                                       )
                                                                      ) FILTER (WHERE fqon.quest_id IS NOT NULL),
                                                              '[]'
                                              ) AS rewarded_by_quests_offer_unlock,

                                              -- 퀘스트 craftUnlock의 rewardItems에 포함된 정보
                                              COALESCE(
                                                              json_agg(
                                                              DISTINCT jsonb_build_object(
                                                                      'quest_id', fqc.quest_id,
                                                                      'name', fqc.name,
                                                                      'npc_name', fqc.npc_name,
                                                                      'npc_image', fqc.npc_image,
                                                                      'url_mapping', fqc.url_mapping,
                                                                      'reward', fqc.reward_item
                                                                       )
                                                                      ) FILTER (WHERE fqc.quest_id IS NOT NULL),
                                                              '[]'
                                              ) AS rewarded_by_quests_craft_unlock,

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
                                                LEFT JOIN required_quests_by_quest_item rqi ON rqi.objective -> 'questItem' ->> 'id' = ti.id
                                                LEFT JOIN filtered_quests_offer_unlock fqon ON fqon.reward_elem -> 'item' ->> 'id' = ti.id
                                                LEFT JOIN filtered_quests_craft_unlock fqc ON TRUE
                                                LEFT JOIN required_quests_by_items_array rqa ON TRUE
                                       GROUP BY ti.id, ti.name, ti.name, ti.category, ti.image,
                                                ti.image_width, ti.image_height, ti.info, ti.update_time, ti.url_mapping)

            SELECT *
            FROM item_with_details
```

### 🧩 주요 서브쿼리 설명

| CTE 이름                         | 설명                                                                               |
| -------------------------------- | ---------------------------------------------------------------------------------- |
| `target_item`                    | `item_i18n` 전체                                                                   |
| `filtered_barters`               | NPC 바터 보상 중 해당 아이템을 포함하는 JSON 구조 해석                             |
| `filtered_hideout`               | 은신처 건설에 필요한 아이템 정보                                                   |
| `filtered_crafts`                | 은신처 제작(crafts)에 필요한 아이템 정보                                           |
| `filtered_quests`                | 퀘스트 보상으로 나오는 아이템                                                      |
| `filtered_quests_offer_unlock`   | 퀘스트의 `offerUnlock` 보상에 포함된 아이템                                        |
| `filtered_quests_craft_unlock`   | 퀘스트의 `craftUnlock.rewardItems`에 포함된 아이템                                 |
| `required_quests_by_quest_item`  | 퀘스트의 `questItem` 항목에 포함된 아이템                                          |
| `required_quests_by_items_array` | 퀘스트의 `items` 배열(`giveItem`, `plantItem`, `findItem`)에 포함된 아이템         |
| `item_with_details`              | 위 서브쿼리 결과를 모두 LEFT JOIN 및 `json_agg`로 통합하여 아이템별 상세 정보 생성 |

---

## 🚨 성능 이슈 원인 분석

1. **`jsonb_array_elements` 사용**

   - JSON 배열을 행 단위로 풀어내기 때문에 데이터가 많을수록 **비용이 기하급수적으로 증가**합니다.

2. **`LEFT JOIN LATERAL` 반복 사용**

   - 조인 순서 최적화가 어렵고, 내부에서 다시 JSON 파싱이 수행되므로 **인덱스 활용이 어려움**.

3. **조건 없는 JOIN**

   - `LEFT JOIN ... ON TRUE` 방식은 **모든 Row를 조인**하여 매우 비효율적입니다.

4. **중첩 JSON 필드 접근**
   - 예: `reward_elem -> 'item' ->> 'id'` 같은 방식은 **인덱스를 사용할 수 없음**.

**explain 결과**

```txt
+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|QUERY PLAN                                                                                                                                                                                                                                                                                                                                                            |
+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
|Subquery Scan on item_with_details  (cost=313261324586346151936.00..314316243974285426688.00 rows=1 width=288)                                                                                                                                                                                                                                                        |
|  CTE target_item                                                                                                                                                                                                                                                                                                                                                     |
|    ->  Limit  (cost=0.00..484.48 rows=1 width=422)                                                                                                                                                                                                                                                                                                                   |
|          ->  Seq Scan on item_i18n  (cost=0.00..484.48 rows=1 width=422)                                                                                                                                                                                                                                                                                             |
|                Filter: (url_mapping = 'bolts'::text)                                                                                                                                                                                                                                                                                                                 |
|  ->  GroupAggregate  (cost=313261324586346151936.00..314316243974285426688.00 rows=1 width=520)                                                                                                                                                                                                                                                                      |
|        Group Key: ti.id, ti.name, ti.category, ti.image, ti.image_width, ti.image_height, ti.info, ti.update_time, ti.url_mapping                                                                                                                                                                                                                                    |
|        InitPlan 2                                                                                                                                                                                                                                                                                                                                                    |
|          ->  CTE Scan on target_item  (cost=0.00..0.02 rows=1 width=32)                                                                                                                                                                                                                                                                                              |
|        ->  Sort  (cost=313261324586346151936.00..313297701116964765696.00 rows=14550612247438080000 width=3520)                                                                                                                                                                                                                                                      |
|              Sort Key: ti.id, ti.name, ti.category, ti.image, ti.image_width, ti.image_height, ti.info, ti.update_time, ti.url_mapping, (jsonb_build_object('id', thir.id, 'level_id', thir.level_id, 'name', thir.name, 'quantity', thir.quantity, 'count', thir.count, 'image', thir.image, 'item_id', thir.item_id, 'master_name', thm.name, 'master_id', thm.id))|
|              ->  Nested Loop Left Join  (cost=30891.77..182628648502135744.00 rows=14550612247438080000 width=3520)                                                                                                                                                                                                                                                  |
|                    ->  Nested Loop Left Join  (cost=30891.77..745995409084647.00 rows=15031624222560000 width=3246)                                                                                                                                                                                                                                                  |
|                          ->  Hash Left Join  (cost=30890.50..4471425008.97 rows=310570748400 width=2972)                                                                                                                                                                                                                                                             |
|                                Hash Cond: (ti.id = ((reward.value -> 'item'::text) ->> 'id'::text))                                                                                                                                                                                                                                                                  |
|                                ->  Merge Left Join  (cost=24061.48..5703272.83 rows=282337044 width=2800)                                                                                                                                                                                                                                                            |
|                                      Merge Cond: (ti.id = (((fqon.reward_elem -> 'item'::text) ->> 'id'::text)))                                                                                                                                                                                                                                                     |
|                                      ->  Merge Left Join  (cost=12913.11..36512.47 rows=1166682 width=2526)                                                                                                                                                                                                                                                          |
|                                            Merge Cond: (ti.id = (((fq.reward_elem -> 'item'::text) ->> 'id'::text)))                                                                                                                                                                                                                                                 |
|                                            ->  Merge Left Join  (cost=1764.74..1873.31 rows=4821 width=2252)                                                                                                                                                                                                                                                         |
|                                                  Merge Cond: (ti.id = (((obj.value -> 'questItem'::text) ->> 'id'::text)))                                                                                                                                                                                                                                           |
|                                                  ->  Merge Left Join  (cost=619.21..627.62 rows=498 width=1978)                                                                                                                                                                                                                                                      |
|                                                        Merge Cond: (ti.id = thir.item_id)                                                                                                                                                                                                                                                                            |
|                                                        ->  Sort  (cost=574.51..574.98 rows=188 width=1627)                                                                                                                                                                                                                                                           |
|                                                              Sort Key: ti.id                                                                                                                                                                                                                                                                                         |
|                                                              ->  Hash Right Join  (cost=3.62..567.41 rows=188 width=1627)                                                                                                                                                                                                                                            |
|                                                                    Hash Cond: (((elem.value -> 'item'::text) ->> 'id'::text) = ti.id)                                                                                                                                                                                                                                |
|                                                                    ->  Nested Loop  (cost=3.59..448.00 rows=18800 width=1395)                                                                                                                                                                                                                                        |
|                                                                          ->  Hash Left Join  (cost=3.58..71.99 rows=188 width=1363)                                                                                                                                                                                                                                  |
|                                                                                Hash Cond: (split_part(thc.level_id, '-'::text, 1) = thm_1.id)                                                                                                                                                                                                                        |
|                                                                                ->  Seq Scan on hideout_crafts_i18n thc  (cost=0.00..67.88 rows=188 width=1263)                                                                                                                                                                                                       |
|                                                                                ->  Hash  (cost=3.26..3.26 rows=26 width=100)                                                                                                                                                                                                                                         |
|                                                                                      ->  Seq Scan on hideout_master_i18n thm_1  (cost=0.00..3.26 rows=26 width=100)                                                                                                                                                                                                  |
|                                                                          ->  Function Scan on jsonb_array_elements elem  (cost=0.00..1.00 rows=100 width=32)                                                                                                                                                                                                         |
|                                                                    ->  Hash  (cost=0.02..0.02 rows=1 width=264)                                                                                                                                                                                                                                                      |
|                                                                          ->  CTE Scan on target_item ti  (cost=0.00..0.02 rows=1 width=264)                                                                                                                                                                                                                          |
|                                                        ->  Sort  (cost=44.70..45.48 rows=315 width=351)                                                                                                                                                                                                                                                              |
|                                                              Sort Key: thir.item_id                                                                                                                                                                                                                                                                                  |
|                                                              ->  Hash Left Join  (cost=3.58..31.63 rows=315 width=351)                                                                                                                                                                                                                                               |
|                                                                    Hash Cond: (split_part(thir.level_id, '-'::text, 1) = thm.id)                                                                                                                                                                                                                                     |
|                                                                    ->  Seq Scan on hideout_item_require_i18n thir  (cost=0.00..27.15 rows=315 width=251)                                                                                                                                                                                                             |
|                                                                    ->  Hash  (cost=3.26..3.26 rows=26 width=100)                                                                                                                                                                                                                                                     |
|                                                                          ->  Seq Scan on hideout_master_i18n thm  (cost=0.00..3.26 rows=26 width=100)                                                                                                                                                                                                                |
|                                                  ->  Sort  (cost=1145.54..1147.96 rows=968 width=274)                                                                                                                                                                                                                                                                |
|                                                        Sort Key: (((obj.value -> 'questItem'::text) ->> 'id'::text))                                                                                                                                                                                                                                                 |
|                                                        ->  Nested Loop  (cost=1.25..1097.53 rows=968 width=274)                                                                                                                                                                                                                                                      |
|                                                              ->  Hash Left Join  (cost=1.25..361.85 rows=484 width=582)                                                                                                                                                                                                                                              |
|                                                                    Hash Cond: (q.npc_id = tn.id)                                                                                                                                                                                                                                                                     |
|                                                                    ->  Seq Scan on quest_i18n q  (cost=0.00..358.84 rows=484 width=492)                                                                                                                                                                                                                              |
|                                                                    ->  Hash  (cost=1.11..1.11 rows=11 width=140)                                                                                                                                                                                                                                                     |
|                                                                          ->  Seq Scan on npc_i18n tn  (cost=0.00..1.11 rows=11 width=140)                                                                                                                                                                                                                            |
|                                                              ->  Function Scan on jsonb_array_elements obj  (cost=0.00..1.50 rows=2 width=32)                                                                                                                                                                                                                        |
|                                                                    Filter: ((value ->> 'type'::text) = ANY ('{findQuestItem,giveQuestItem}'::text[]))                                                                                                                                                                                                                |
|                                            ->  Materialize  (cost=11148.37..11390.37 rows=48400 width=274)                                                                                                                                                                                                                                                           |
|                                                  ->  Sort  (cost=11148.37..11269.37 rows=48400 width=274)                                                                                                                                                                                                                                                            |
|                                                        Sort Key: (((fq.reward_elem -> 'item'::text) ->> 'id'::text))                                                                                                                                                                                                                                                 |
|                                                        ->  Subquery Scan on fq  (cost=1.25..1092.69 rows=48400 width=274)                                                                                                                                                                                                                                            |
|                                                              ->  ProjectSet  (cost=1.25..608.69 rows=48400 width=306)                                                                                                                                                                                                                                                |
|                                                                    ->  Hash Left Join  (cost=1.25..361.85 rows=484 width=643)                                                                                                                                                                                                                                        |
|                                                                          Hash Cond: (qa_1.npc_id = tn_3.id)                                                                                                                                                                                                                                                          |
|                                                                          ->  Seq Scan on quest_i18n qa_1  (cost=0.00..358.84 rows=484 width=553)                                                                                                                                                                                                                     |
|                                                                          ->  Hash  (cost=1.11..1.11 rows=11 width=140)                                                                                                                                                                                                                                               |
|                                                                                ->  Seq Scan on npc_i18n tn_3  (cost=0.00..1.11 rows=11 width=140)                                                                                                                                                                                                                    |
|                                      ->  Materialize  (cost=11148.37..11390.37 rows=48400 width=274)                                                                                                                                                                                                                                                                 |
|                                            ->  Sort  (cost=11148.37..11269.37 rows=48400 width=274)                                                                                                                                                                                                                                                                  |
|                                                  Sort Key: (((fqon.reward_elem -> 'item'::text) ->> 'id'::text))                                                                                                                                                                                                                                                     |
|                                                  ->  Subquery Scan on fqon  (cost=1.25..1092.69 rows=48400 width=274)                                                                                                                                                                                                                                                |
|                                                        ->  ProjectSet  (cost=1.25..608.69 rows=48400 width=306)                                                                                                                                                                                                                                                      |
|                                                              ->  Hash Left Join  (cost=1.25..361.85 rows=484 width=643)                                                                                                                                                                                                                                              |
|                                                                    Hash Cond: (qa_2.npc_id = tn_4.id)                                                                                                                                                                                                                                                                |
|                                                                    ->  Seq Scan on quest_i18n qa_2  (cost=0.00..358.84 rows=484 width=553)                                                                                                                                                                                                                           |
|                                                                    ->  Hash  (cost=1.11..1.11 rows=11 width=140)                                                                                                                                                                                                                                                     |
|                                                                          ->  Seq Scan on npc_i18n tn_4  (cost=0.00..1.11 rows=11 width=140)                                                                                                                                                                                                                          |
|                                ->  Hash  (cost=2338.02..2338.02 rows=110000 width=204)                                                                                                                                                                                                                                                                               |
|                                      ->  Nested Loop  (cost=0.02..2338.02 rows=110000 width=204)                                                                                                                                                                                                                                                                     |
|                                            ->  Nested Loop  (cost=0.00..23.11 rows=1100 width=172)                                                                                                                                                                                                                                                                   |
|                                                  ->  Seq Scan on npc_i18n n  (cost=0.00..1.11 rows=11 width=154)                                                                                                                                                                                                                                                     |
|                                                  ->  Function Scan on jsonb_array_elements barter  (cost=0.00..1.00 rows=100 width=32)                                                                                                                                                                                                                               |
|                                            ->  Memoize  (cost=0.01..1.01 rows=100 width=32)                                                                                                                                                                                                                                                                          |
|                                                  Cache Key: barter.value                                                                                                                                                                                                                                                                                             |
|                                                  Cache Mode: binary                                                                                                                                                                                                                                                                                                  |
|                                                  ->  Function Scan on jsonb_array_elements reward  (cost=0.01..1.00 rows=100 width=32)                                                                                                                                                                                                                               |
|                          ->  Materialize  (cost=1.27..4756.10 rows=48400 width=274)                                                                                                                                                                                                                                                                                  |
|                                ->  Nested Loop  (cost=1.27..2717.10 rows=48400 width=274)                                                                                                                                                                                                                                                                            |
|                                      ->  Nested Loop Left Join  (cost=1.25..1329.85 rows=48400 width=274)                                                                                                                                                                                                                                                            |
|                                            ->  Hash Left Join  (cost=1.25..361.85 rows=484 width=643)                                                                                                                                                                                                                                                                |
|                                                  Hash Cond: (qa.npc_id = tn_1.id)                                                                                                                                                                                                                                                                                    |
|                                                  ->  Seq Scan on quest_i18n qa  (cost=0.00..358.84 rows=484 width=553)                                                                                                                                                                                                                                               |
|                                                  ->  Hash  (cost=1.11..1.11 rows=11 width=140)                                                                                                                                                                                                                                                                       |
|                                                        ->  Seq Scan on npc_i18n tn_1  (cost=0.00..1.11 rows=11 width=140)                                                                                                                                                                                                                                            |
|                                            ->  Function Scan on jsonb_array_elements craft_unlock  (cost=0.01..1.00 rows=100 width=32)                                                                                                                                                                                                                               |
|                                      ->  Memoize  (cost=0.01..1.77 rows=1 width=32)                                                                                                                                                                                                                                                                                  |
|                                            Cache Key: craft_unlock.value                                                                                                                                                                                                                                                                                             |
|                                            Cache Mode: binary                                                                                                                                                                                                                                                                                                        |
|                                            ->  Function Scan on jsonb_array_elements reward_item  (cost=0.01..1.76 rows=1 width=32)                                                                                                                                                                                                                                  |
|                                                  Filter: (((value -> 'item'::text) ->> 'id'::text) = (InitPlan 2).col1)                                                                                                                                                                                                                                              |
|                    ->  Materialize  (cost=0.00..75118.62 rows=968 width=274)                                                                                                                                                                                                                                                                                         |
|                          ->  Nested Loop Left Join  (cost=0.00..75113.78 rows=968 width=274)                                                                                                                                                                                                                                                                         |
|                                Join Filter: (q_1.npc_id = tn_2.id)                                                                                                                                                                                                                                                                                                   |
|                                ->  Nested Loop  (cost=0.00..74965.02 rows=968 width=184)                                                                                                                                                                                                                                                                             |
|                                      ->  Seq Scan on quest_i18n q_1  (cost=0.00..358.84 rows=484 width=492)                                                                                                                                                                                                                                                          |
|                                      ->  Function Scan on jsonb_array_elements obj_1  (cost=0.00..154.13 rows=2 width=32)                                                                                                                                                                                                                                            |
|                                            Filter: (((value ->> 'type'::text) = ANY ('{plantItem,giveItem,findItem}'::text[])) AND EXISTS(SubPlan 4))                                                                                                                                                                                                                |
|                                            SubPlan 4                                                                                                                                                                                                                                                                                                                 |
|                                              ->  Function Scan on jsonb_array_elements item  (cost=0.03..1.52 rows=1 width=0)                                                                                                                                                                                                                                        |
|                                                    Filter: ((value ->> 'id'::text) = (InitPlan 3).col1)                                                                                                                                                                                                                                                              |
|                                                    InitPlan 3                                                                                                                                                                                                                                                                                                        |
|                                                      ->  CTE Scan on target_item target_item_1  (cost=0.00..0.02 rows=1 width=32)                                                                                                                                                                                                                                    |
|                                ->  Materialize  (cost=0.00..1.17 rows=11 width=140)                                                                                                                                                                                                                                                                                  |
|                                      ->  Seq Scan on npc_i18n tn_2  (cost=0.00..1.11 rows=11 width=140)                                                                                                                                                                                                                                                              |
|JIT:                                                                                                                                                                                                                                                                                                                                                                  |
|  Functions: 182                                                                                                                                                                                                                                                                                                                                                      |
|  Options: Inlining true, Optimization true, Expressions true, Deforming true                                                                                                                                                                                                                                                                                         |
+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+

```

---

## ⚙ 실행 및 테스트

성능 개선을 위해 쿼리를 수정한 뒤, 테스트를 진행하였습니다.

### 초기 테스트 결과

- `item_detail_i18n` 데이터는 약 4200여개
- `LIMIT`을 설정하지 않은 상태에서 실행했으나, **커넥션이 끊기는 문제**가 발생하였습니다.
- `LIMIT 1000`만 적용했을 때도 **10분 이상 소요**되었습니다.
- `EXPLAIN` 결과, 과도한 리소스 소모가 확인되었습니다.

---

## 🌀 Airflow 적용 및 배치 처리

매일 아이템 데이터가 업데이트되므로, Airflow DAG를 통해 **상세 정보를 통계 낸 후 UPSERT 처리**하는 방식으로 리팩터링하였습니다.

- 처음에는 `LIMIT / OFFSET`을 사용해 **순차 실행 방식**으로 DAG를 구성하였고,
- 약 **4200개 데이터를 처리하는 데 1시간 이상 소요**되었습니다.

**Airflow 실행 결과**

![long_time](https://github.com/user-attachments/assets/b03c6255-98a6-4e2b-88fe-ffb6ee1e50c3)

---

## 🧯 문제 인지 및 원인

- 특정 데이터가 잘못 들어가 있었고, 쿼리 자체에 문제가 있었음을 뒤늦게 인지하게 되었습니다.
- 성능 병목은 아래와 같은 **문제 쿼리**에서 발생하였습니다
- 단일 값을 조회할 때는 잘되었는데, 전체 조회로 바꾸는 부분에서 수정을 잘못하여 문제가 발생하였습니다.

**문제가 된 부분**

```sql
-- 예시 문제 쿼리 영역
WITH target_item AS (SELECT *
                     FROM item_i18n
                     offset %s limit %s),

     -- 🧩 craftUnlock.rewardItems[*].item.id에 포함된 경우 (finish_rewards 안)
     filtered_quests_craft_unlock AS (SELECT qa.id    AS quest_id,
                                             qa.name,
                                             qa.npc_id,
                                             qa.url_mapping,
                                             tn.name  as npc_name,
                                             tn.image AS npc_image,
                                             reward_item
                                      FROM quest_i18n qa
                                               LEFT JOIN npc_i18n tn ON qa.npc_id = tn.id
                                               LEFT JOIN LATERAL jsonb_array_elements(qa.finish_rewards -> 'craftUnlock') AS craft_unlock
                                                         ON TRUE
                                               LEFT JOIN LATERAL jsonb_array_elements(craft_unlock -> 'rewardItems') AS reward_item
                                                         ON TRUE
                                      WHERE reward_item -> 'item' ->> 'id' IN (SELECT id FROM target_item)),

     -- ❗ items 배열에 포함된 경우 (예: giveItem, plantItem, findItem)
     required_quests_by_items_array AS (SELECT q.id     AS quest_id,
                                               q.name,
                                               q.url_mapping,
                                               tn.name  as npc_name,
                                               tn.image AS npc_image,
                                               obj      AS objective
                                        FROM quest_i18n q
                                                 LEFT JOIN LATERAL jsonb_array_elements(q.objectives) AS obj ON TRUE
                                                 LEFT JOIN npc_i18n tn on q.npc_id = tn.id
                                        WHERE obj ->> 'type' IN ('plantItem', 'giveItem', 'findItem')
                                          AND EXISTS (SELECT 1
                                                      FROM jsonb_array_elements(obj -> 'items') AS item
                                                      WHERE item ->> 'id' in (SELECT id FROM target_item))),

     item_with_details AS (SELECT ti.id,

                                  -- 퀘스트 craftUnlock의 rewardItems에 포함된 정보
                                  COALESCE(
                                                  json_agg(
                                                  DISTINCT jsonb_build_object(
                                                          'quest_id', fqc.quest_id,
                                                          'name', fqc.name,
                                                          'npc_name', fqc.npc_name,
                                                          'npc_image', fqc.npc_image,
                                                          'url_mapping', fqc.url_mapping,
                                                          'reward', fqc.reward_item
                                                           )
                                                          ) FILTER (WHERE fqc.quest_id IS NOT NULL),
                                                  '[]'
                                  ) AS rewarded_by_quests_craft_unlock,

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
                                    LEFT JOIN filtered_quests_craft_unlock fqc ON TRUE
                                    LEFT JOIN required_quests_by_items_array rqa ON TRUE
                           GROUP BY ti.id, ti.name, ti.name, ti.category, ti.image,
                                    ti.image_width, ti.image_height, ti.info, ti.update_time, ti.url_mapping)

SELECT id,
       rewarded_by_quests_craft_unlock,
       required_by_quest_item_array
FROM item_with_details
```

## ✅ 최종 개선 결과 요약

- Airflow DAG에서 아이템 상세정보를 upsert 처리하는 구조로 변경
- 처음에는 `LIMIT + OFFSET` 기반 병렬 처리 DAG으로 구성했으나, 오히려 병목 발생
- 쿼리 오류 수정 후, 단일 쿼리로 전체 데이터를 일괄 처리 → 성능 급향상
  - 4200여 건 처리에 1시간에서 30초 이내로 변경

## 🧠 회고: 아이템 상세정보 성능 개선기

### 📌 배경

- 복잡한 JSON 데이터를 파싱하여 아이템별 사용처/보상 정보를 종합 제공하는 쿼리가 필요
- 초기에는 여러 CTE와 JSON 파싱을 사용하여 데이터를 구성

---

### 🛠 문제점

- `jsonb_array_elements` 및 LATERAL JOIN의 과도한 사용 → 성능 저하
- `LEFT JOIN ON TRUE` 등 무조건적인 조인으로 인해 불필요한 계산량 발생
- EXPLAIN 결과, Nested Loop Join 과다 발생 및 비효율적인 실행 계획
- 처음에는 문제의 원인을 모른 채 DAG을 병렬로 구성 + LIMIT/OFFSET 전략
  - 실행은 잘 되지만 너무 오랜 시간이 걸렸습니다 (1시간).

---

### 🔍 원인 분석

- 쿼리 내 특정 필드 조건 누락으로 인해 **전체 데이터를 과하게 JOIN**하고 있었습니다.
- 성능 병목의 주 원인이 **불필요한 조인**과 **필터 조건 부재**였습니다.
- DAG 병렬 처리보다 먼저 쿼리 로직 자체 최적화가 선행되어야 했습니다.

---

### ✅ 개선 및 결과

- 문제 구간의 쿼리를 수정 → 조건 추가 및 불필요한 JOIN 제거
- 쿼리 Task 병렬 처리 수행
- 실행 시간: **기존 1시간 → 최적화 후 30초 이내**
- Airflow를 통한 일일 upsert 구조로 안정성 확보

**현재 아이템 상세 조회 속도**



https://github.com/user-attachments/assets/30ed75af-9ed6-4b8e-bb64-6332a7341c19


---

### 🤔 배운 점

- 성능 이슈는 대부분 **쿼리 설계의 문제**에서 시작되었다는 점
- **병렬 처리나 DAG 구성은 근본 원인 해결 후에 고려해야 할 보조 수단**이라는 점
- `EXPLAIN (ANALYZE)`는 필수 도구, 문제가 생기면 가장 먼저 봐야 하고 cost와 row가 비정상적이라면 쿼리를 확인해봐야 한다는 점
- Airflow에서 동적으로 Mapped Task를 만들 수 있다는 것 - 현재 데이터 총 개수와 limit, offset 기준으로 동적으로 생성
- 결국 성능은 코드보다 **데이터 모델링과 쿼리 로직 설계**가 핵심이라는 교훈을 얻었습니다.

---

### 📝 다음에는 제발~~

- JSON 필드는 가능하면 사전 정규화하거나, ETL 단계에서 파싱 처리하기
- 복잡한 쿼리는 처음부터 성능 측정 후 설계하기
- DAG 설계는 "성능 병목 제거 후" 도입하기
- 쿼리는 꼭 여러번 확인하기

### Dag Mapped Task Code

현재 적용중인 아이템 상세 Dynamic Task 코드 입니다.

```python
from airflow.decorators import dag, task
from airflow.providers.postgres.hooks.postgres import PostgresHook
from contextlib import closing
import pendulum
from custom_module.psql_func import read_sql

BATCH_SIZE = 500


@dag(
    dag_id="dags_item_detail_upsert",
    start_date=pendulum.datetime(2024, 5, 1, tz="Asia/Seoul"),
    schedule_interval="50 0 * * *",
    catchup=False,
    max_active_tasks=10,
    default_args={
        "owner": "airflow",
        "retries": 1,
        "retry_delay": pendulum.duration(minutes=5),
    },
    tags=["postgresql", "tarkov-dev-api"],
)
def item_detail_upsert():

    @task
    def get_total_count(postgres_conn_id: str) -> int:
        hook = PostgresHook(postgres_conn_id)
        sql = read_sql("item_count.sql")
        with closing(hook.get_conn()) as conn, closing(conn.cursor()) as cur:
            cur.execute(sql)
            return cur.fetchone()[0]

    @task
    def make_ranges(total_count: int) -> list[dict]:
        """
        XCom(리스트) → process_batch 에 매핑됨.
        각 dict가 하나의 Task 인스턴스(Worker Slot)를 생성.
        """
        return [
            {"start": start, "end": min(start + BATCH_SIZE, total_count)}
            for start in range(0, total_count, BATCH_SIZE)
        ]

    @task
    def process_batch(batch: dict, postgres_conn_id: str):
        start, end = batch["start"], batch["end"]
        hook = PostgresHook(postgres_conn_id)
        sql = read_sql("item_detail_upsert.sql")
        with closing(hook.get_conn()) as conn, closing(conn.cursor()) as cur:
            cur.execute(sql, (start, end))
            conn.commit()

    total = get_total_count(postgres_conn_id="tkl_db")
    ranges = make_ranges(total)
    process_batch.expand(batch=ranges, postgres_conn_id=["tkl_db"])


dag = item_detail_upsert()
```
