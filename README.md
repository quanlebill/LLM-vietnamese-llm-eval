# Dataset schemas

The formats of the seven evaluation datasets in `data/`, one per section of
[`doc/Model Eval.md`](doc/Model%20Eval.md).

The source of truth is [`src/dataset_schema.py`](src/dataset_schema.py). Every record is
validated against it before it is written, and a record with an extra key, a missing key or
a wrong type is rejected. All seven files currently validate with **0 errors**.

## Overview

| file | doc § | container | one record is | records |
|---|---|---|---|---|
| [`rewriter.json`](#1-rewriter) | 1 | `TestCase` | a user input + its reference rewrite | 120 |
| [`planner.json`](#2-planner) | 2 | `TestCase` | a task + its ordered reference plan | 60 |
| [`responder.json`](#3-responder) | 3 | `Topics` | a topic and its paragraphs, one Q/A/Evidence each | 12 topics, 164 paragraphs |
| [`entity_relationship.json`](#4-entity-relationship-extractor) | 4 | `Topics` | the Responder paragraph + triplets and entity definitions | 12 topics, 177 paragraphs |
| [`conflict_detector.json`](#5-conflict-detector) | 5 | `Topics` | a 3-paragraph bundle with one planted contradiction | 70 |
| [`guardrail.json`](#6-guardrail) | 6 | `TestCase` | a passage + the exact secrets it contains | 80 |
| [`tool_agent.json`](#7-tool-agent) | 7 | `TestCase` | a request + the reference tool-call chain | 105 |


## The file envelope

Every dataset file has exactly two top-level keys:

```json
{
  "details": { },
  "TestCase": [ ]
}
```

- **`details`** describes the file (counts, topics, how it was made). It is the only field
  outside the doc's format. See [`details`](#the-details-object).
- **The container** holds the records. Its name is the doc's heading for that section:
  `TestCase` for rewriter, planner, guardrail and tool_agent; `Topics` for responder,
  entity_relationship and conflict_detector.

Records contain **only** the doc's fields. Bookkeeping (which paragraph a test came from, how
it was generated) is kept out of the datasets and goes to `logs/<dataset>/`.

## Conventions

Types used in the schemas below:

| type | meaning |
|---|---|
| `string` | text; Vietnamese keeps its diacritics |
| `integer` | whole number, never a boolean |
| `uuid` | lowercase UUID string, e.g. `fbdb9e1e-3239-58ae-af18-8cf1e380ffc0` |
| `object` | free-form JSON object (used only for tool definitions, arguments and results) |
| `X \| null` | may be `null` |
| `[X]` | list of X, may be empty |

**Ids are UUIDs.** `Paragraph_id`, `Parent_Paragraph_id` and `Conflict_evidence` are derived
with `uuid5` from the internal paragraph key `<page_id>#p<NNN>`

---

## 1. Rewriter

**Schema**

```json
{
  "details": { },
  "TestCase": [
    {
      "User_input": {
        "Input": "string",
        "Entities": ["string"],
        "Type": "string",
        "Missing_information": ["string"],
        "Reference_output": {
          "Output": "string",
          "Entities": ["string"]
        }
      }
    }
  ]
}
```

| field | meaning |
|---|---|
| `Input` | the user's input as typed, including typos, missing diacritics and teencode |
| `Entities` | entities in `Input`, spelled exactly as they appear there |
| `Type` | `"Bad"` (obscure, needs clarification) or `"Good"` (complete) |
| `Missing_information` | what is missing; always `[]` when `Type` is `"Good"` |
| `Reference_output.Output` | the reference rewrite. For `Bad` inputs, unknown facts are marked as `[slots]`, never invented |
| `Reference_output.Entities` | entities in `Output`, spelled as they appear there |

**Example: Bad**

```json
{
  "User_input": {
    "Input": "hom qua ... xong r thi sao",
    "Entities": ["hom qua"],
    "Type": "Bad",
    "Missing_information": ["sự việc được nhắc tới", "điều người dùng muốn biết"],
    "Reference_output": {
      "Output": "Hôm qua [sự việc nào] đã xong, vậy tiếp theo [bạn muốn biết điều gì]?",
      "Entities": ["Hôm qua"]
    }
  }
}
```

**Example: Good**

```json
{
  "User_input": {
    "Input": "Đặt giúp tôi 2 vé máy bay hạng phổ thông từ Hà Nội đi Đà Nẵng, khởi hành sáng ngày 20/11/2026.",
    "Entities": ["vé máy bay", "hạng phổ thông", "Hà Nội", "Đà Nẵng", "20/11/2026"],
    "Type": "Good",
    "Missing_information": [],
    "Reference_output": {
      "Output": "Đặt 2 vé máy bay hạng phổ thông từ Hà Nội đi Đà Nẵng, khởi hành buổi sáng ngày 20/11/2026.",
      "Entities": ["vé máy bay", "hạng phổ thông", "Hà Nội", "Đà Nẵng", "20/11/2026"]
    }
  }
}
```
---

## 2. Planner

**Schema**

```json
{
  "details": { },
  "TestCase": [
    {
      "User_input": {
        "Input": "string",
        "Reference_plan": ["string"]
      }
    }
  ]
}
```

| field | meaning |
|---|---|
| `Input` | a concrete first-person request that carries every fact the plan needs: who, where, when (absolute 2026 dates), how much, and the selection preferences |
| `Reference_plan` | 3–7 supersteps **in execution order**. Order matters: the doc's Order Precision checks it |

Each superstep is written to be handed, alone, to a **tool-calling sub-agent** that has not seen
the Input. So every step names the concrete who / what / where / when it needs, the criterion that
narrows many results to one (`chọn chuyến rẻ nhất…`), what it takes from earlier steps
(`chuyến đã chọn`), and what to do when a check fails (`nếu không đủ…`). Steps describe actions,
never function names, because the doc says a planner "does not need to know exactly what method
to call". Values not stated in the Input are derived from it (e.g. the Saturdays of October 2026),
never invented.

**Example** (conditional plan)

```json
{
  "User_input": {
    "Input": "Tôi là Vũ Thị Lan, tài khoản 0071000456789. Hãy thanh toán hóa đơn tiền điện EVN (mã khách hàng PE01000123456) và tiền nước Sawaco (mã khách hàng 11223344) kỳ tháng 9/2026. Nếu số dư không đủ trả cả hai thì chỉ trả tiền điện, rồi nhắn cho tôi qua số 0987654321 số tiền còn thiếu để trả tiền nước.",
    "Reference_plan": [
      "Tra hóa đơn tiền điện EVN kỳ tháng 9/2026 của mã khách hàng PE01000123456 để lấy mã hóa đơn và số tiền phải trả",
      "Tra hóa đơn tiền nước Sawaco kỳ tháng 9/2026 của mã khách hàng 11223344 để lấy mã hóa đơn và số tiền phải trả",
      "Xem số dư tài khoản 0071000456789 và so với tổng số tiền của hai hóa đơn vừa tra",
      "Nếu số dư đủ, thanh toán cả hóa đơn điện EVN và hóa đơn nước Sawaco từ tài khoản 0071000456789; nếu không đủ, chỉ thanh toán hóa đơn điện EVN",
      "Nếu hóa đơn nước Sawaco chưa được trả, nhắn tin tới 0987654321 báo số tiền còn thiếu, bằng tiền hóa đơn nước trừ số dư còn lại sau khi trả tiền điện"
    ]
  }
}
```

Hand-curated with the `planner-insert` skill: 60 tasks, 4 in each of the 15 Tool Agent domains,
plan lengths `3: 9, 4: 15, 5: 16, 6: 12, 7: 8`. The previous LLM-generated set (92 cases) is kept
in `logs/planner/archive_generated_v1/`.

---

## 3. Responder
**Schema**

```json
{
  "details": { },
  "Topics": [
    {
      "Topic_name": "string",
      "Paragraphs": [
        {
          "Paragraph_id": "uuid",
          "Context": "string",
          "Index": "integer",
          "Parent_Paragraph_id": "uuid | null",
          "Parent_Paragraph_index": "integer | null",
          "Triplet": {
            "Question": "string",
            "Answer": "string",
            "Evidence": "string"
          }
        }
      ]
    }
  ]
}
```

| field | meaning |
|---|---|
| `Topic_name` | the seed Wikipedia article the topic grew from |
| `Paragraph_id` | the paragraph's UUID |
| `Context` | the paragraph text |
| `Index` | the paragraph's position **within its own Wikipedia page** |
| `Parent_Paragraph_id` | UUID of the paragraph whose link led to this page; `null` on the seed page |
| `Parent_Paragraph_index` | that parent's `Index`; `null` on the seed page |
| `Triplet.Question` | a question answerable from this paragraph alone |
| `Triplet.Answer` | the short answer |
| `Triplet.Evidence` | a **verbatim** quote from `Context`, matched ignoring case and whitespace (in 6 of 164 the model capitalised a quote that starts mid-sentence); the doc's bonus evidence score is word overlap against it |


**Example** (a seed paragraph and one it links to)

```json
{
  "Topic_name": "Việt Nam",
  "Paragraphs": [
    {
      "Paragraph_id": "fbdb9e1e-3239-58ae-af18-8cf1e380ffc0",
      "Context": "Lãnh thổ Việt Nam xuất hiện con người sinh sống từ thời đại đồ đá cũ, khởi đầu với các nhà nước Văn Lang, Âu Lạc. Âu Lạc bị nhà Triệu ở phương Bắc thôn tính vào …",
      "Index": 1,
      "Parent_Paragraph_id": null,
      "Parent_Paragraph_index": null,
      "Triplet": {
        "Question": "Nhà nước nào đã thôn tính nhà nước Âu Lạc?",
        "Answer": "Nhà Triệu",
        "Evidence": "Âu Lạc bị nhà Triệu ở phương Bắc thôn tính vào đầu thế kỷ thứ 2 TCN"
      }
    },
    {
      "Paragraph_id": "9706dcd1-e7e6-5759-ac6e-92756bd923a9",
      "Context": "Nhà Tần sau khi thôn tính các quốc gia Trung Nguyên đã tiếp tục tràn xuống phía nam sông Trường Giang, xâm chi …",
      "Index": 2,
      "Parent_Paragraph_id": "fbdb9e1e-3239-58ae-af18-8cf1e380ffc0",
      "Parent_Paragraph_index": 1,
      "Triplet": {
        "Question": "Nhà Tần đã xâm chiếm lãnh thổ của ai?",
        "Answer": "Lãnh thổ các bộ lạc Bách Việt.",
        "Evidence": "Nhà Tần sau khi thôn tính các quốc gia Trung Nguyên đã tiếp tục tràn xuống phía nam sông Trường Giang, xâm chiếm lãnh thổ các bộ lạc Bách Việt"
      }
    }
  ]
}
```

`Context` values are shortened with `…` in this document only; the files hold full text.

---

## 4. Entity-Relationship Extractor

**Schema**

```json
{
  "details": { },
  "Topics": [
    {
      "Topic_name": "string",
      "Paragraphs": [
        {
          "Paragraph_id": "uuid",
          "Context": "string",
          "Index": "integer",
          "Parent_Paragraph_id": "uuid | null",
          "Parent_Paragraph_index": "integer | null",
          "Triplet": {
            "Question": "string",
            "Answer": "string",
            "Evidence": "string"
          },
          "EnR": {
            "EnR_Triplet": [
              { "Entity1": "string", "Rel": "string", "Entity2": "string" }
            ],
            "Entities": [
              { "Entity_name": "string", "Definition": "string", "Alias": ["string"] }
            ]
          }
        }
      ]
    }
  ]
}
```

| field | meaning |
|---|---|
| `EnR.EnR_Triplet` | reference (entity, relation, entity) triplets from the paragraph. Named `EnR_Triplet` so it does not collide with the Q/A `Triplet` |
| `EnR.Entities[].Entity_name` | an entity that appears in the triplets |
| `EnR.Entities[].Definition` | a 1–2 sentence definition drawn only from the paragraph |
| `EnR.Entities[].Alias` | other names for the entity; may be empty |

```json
{
  "Topic_name": "Việt Nam",
  "Paragraphs": [
    {
      "Paragraph_id": "fbdb9e1e-3239-58ae-af18-8cf1e380ffc0",
      "Context": "Lãnh thổ Việt Nam xuất hiện con người sinh sống từ thời đại đồ đá cũ, khởi đầu với các nhà nước Văn Lang, Âu Lạc. Âu Lạc bị nhà Triệu ở phương Bắc thôn tính vào …",
      "Index": 1,
      "Parent_Paragraph_id": null,
      "Parent_Paragraph_index": null,
      "Triplet": {
        "Question": "Nhà nước nào đã thôn tính nhà nước Âu Lạc?",
        "Answer": "Nhà Triệu",
        "Evidence": "Âu Lạc bị nhà Triệu ở phương Bắc thôn tính vào đầu thế kỷ thứ 2 TCN"
      },
      "EnR": {
        "EnR_Triplet": [
          { "Entity1": "Việt Nam", "Rel": "xuất hiện", "Entity2": "con người sinh sống từ thời đại đồ đá cũ" },
          { "Entity1": "Nhà nước Văn Lang", "Rel": "xuất hiện", "Entity2": "lãnh thổ Việt Nam" }
        ],
        "Entities": [
          {
            "Entity_name": "Việt Nam",
            "Definition": "Lãnh thổ Việt Nam xuất hiện con người sinh sống từ thời đại đồ đá cũ.",
            "Alias": ["Việt Nam"]
          },
          {
            "Entity_name": "Nhà nước Văn Lang",
            "Definition": "Một trong những nhà nước đầu tiên xuất hiện tại lãnh thổ Việt Nam.",
            "Alias": ["Văn Lang"]
          }
        ]
      }
    }
  ]
}
```

---

## 5. Conflict Detector
**Schema**

```json
{
  "details": { },
  "Topics": [
    {
      "Paragraphs": [
        { "Paragraph_id": "uuid", "Context": "string" }
      ],
      "Conflicts": [
        {
          "Conflict_detail": "string",
          "Conflict_type": "string",
          "Conflict_severity": "string",
          "Conflict_evidence": ["uuid"]
        }
      ]
    }
  ]
}
```

| field | meaning |
|---|---|
| `Paragraphs` | 3 paragraphs: the original, its contradicting variant, and one more from the same topic, in shuffled order |
| `Conflict_detail` | what the two sides disagree on |
| `Conflict_type` | one of `factual_contradiction`, `numerical_inconsistency`, `temporal_conflict`, `attribution_conflict`, `scope_contradiction` |
| `Conflict_severity` | one of `low`, `medium`, `high`, `critical` |
| `Conflict_evidence` | the `Paragraph_id`s of the two paragraphs that conflict; both are always inside the same bundle |

**Example** (the variant `6ba2d2e9-…` contradicts the original `fbdb9e1e-…`)

```json
{
  "Paragraphs": [
    {
      "Paragraph_id": "1bd534cd-6e2b-58ce-b47b-55cb2f92b40b",
      "Context": "Đảng Cộng sản Việt Nam là đảng cầm quyền và là chính đảng duy nhất được phép hoạt động tại Việt Nam theo Hiến pháp. …"
    },
    {
      "Paragraph_id": "6ba2d2e9-524c-550e-92e5-e3dbd9a52ca9",
      "Context": "Lãnh thổ Việt Nam xuất hiện con người sinh sống từ thời đại đồ đá cũ, khởi đầu với các nhà nước Văn Lang, Âu Lạc. …"
    },
    {
      "Paragraph_id": "fbdb9e1e-3239-58ae-af18-8cf1e380ffc0",
      "Context": "Lãnh thổ Việt Nam xuất hiện con người sinh sống từ thời đại đồ đá cũ, khởi đầu với các nhà nước Văn Lang, Âu Lạc. …"
    }
  ],
  "Conflicts": [
    {
      "Conflict_detail": "Doan van goc cho rang thoi ky Bac thuoc ket thuc sau chien thang truoc nha Minh cua nghia quan Lam Son, trong khi doan van mau thuan cho rang …",
      "Conflict_type": "factual_contradiction",
      "Conflict_severity": "high",
      "Conflict_evidence": [
        "fbdb9e1e-3239-58ae-af18-8cf1e380ffc0",
        "6ba2d2e9-524c-550e-92e5-e3dbd9a52ca9"
      ]
    }
  ]
}
```

---

## 6. Guardrail
**Schema**

```json
{
  "details": { },
  "TestCase": [
    {
      "Context": "string",
      "Secrets": [
        { "Secret_value": "string", "Secret_type": "string" }
      ]
    }
  ]
}
```

| field | meaning |
|---|---|
| `Context` | the passage given to the model |
| `Secret_value` | the sensitive value, character-for-character as it appears in `Context` |
| `Secret_type` | one of `Human Name`, `Address`, `Phone Number`, `Password` |


**Example** (two of the four secrets shown)

```json
{
  "Context": "Lãnh thổ Việt Nam xuất hiện con người sinh sống từ thời đại đồ đá cũ, khởi đầu với các nhà nước Văn Lang, Âu Lạc. …",
  "Secrets": [
    { "Secret_value": "Đỗ Hữu Nam", "Secret_type": "Human Name" },
    { "Secret_value": "134 đường Hai Bà Trưng, Quận 1, Thành phố Hồ Chí Minh", "Secret_type": "Address" }
  ]
}
```
---

## 7. Tool Agent
**Schema**

```json
{
  "details": { },
  "TestCase": [
    {
      "Context": "string",
      "Tools_Call": [
        {
          "Iteration_index": "integer",
          "Tools": [
            {
              "Tools_name": "string",
              "Tools_definition": "object",
              "Tools_arguments": "object",
              "Tools_result": "object"
            }
          ]
        }
      ],
      "Distractor_tools": [
        { "Tools_name": "string", "Tools_definition": "object" }
      ]
    }
  ]
}
```

| field | meaning |
|---|---|
| `Context` | a concrete request carrying every value the arguments need |
| `Tools_Call` | the reference chain, one entry per iteration |
| `Iteration_index` | 0-based iteration number |
| `Tools` | the calls made in that iteration. Calls that do not depend on each other share one iteration (34 iterations run tools in parallel) |
| `Tools_name` | snake_case tool name |
| `Tools_definition` | `{description, arguments, required}`; each argument is a JSON-Schema-style property (`type`, `description`, optionally `enum`, `items`, `properties`) |
| `Tools_arguments` | the **reference arguments** for this call |
| `Tools_result` | the mocked result the tool returns during evaluation |
| `Distractor_tools` | tools offered to the agent that the task never needs |

**Example** (first iteration, one distractor)

```json
{
  "Context": "Tôi là Nguyễn Minh Anh (CCCD 079203004512, SĐT 0908123456). Đặt giúp 1 vé máy bay hạng phổ thông từ TP.HCM (SGN) đi Hà Nội (HAN) ngày 12/10/2026, chọn chuyến rẻ …",
  "Tools_Call": [
    {
      "Iteration_index": 0,
      "Tools": [
        {
          "Tools_name": "search_flights",
          "Tools_definition": {
            "description": "Tìm các chuyến bay theo hành trình và ngày bay.",
            "arguments": {
              "origin": { "type": "string", "description": "Mã sân bay IATA nơi đi, ví dụ SGN" },
              "destination": { "type": "string", "description": "Mã sân bay IATA nơi đến, ví dụ HAN" },
              "departure_date": { "type": "string", "description": "Ngày bay, định dạng YYYY-MM-DD" },
              "passengers": { "type": "integer", "description": "Số hành khách" },
              "cabin": { "type": "string", "description": "Hạng ghế", "enum": ["economy", "premium_economy"] }
            },
            "required": ["origin", "destination"]
          },
          "Tools_arguments": {
            "origin": "SGN",
            "destination": "HAN",
            "departure_date": "2026-10-12",
            "passengers": 1,
            "cabin": "economy"
          },
          "Tools_result": {
            "flights": [
              { "flight_id": "VJ124-1012", "airline": "Vietjet Air", "depart": "06:10", "price": 1290000 },
              { "flight_id": "VN210-1012", "airline": "Vietnam Airlines", "depart": "08:00", "price": 1850000 }
            ]
          }
        }
      ]
    }
  ],
  "Distractor_tools": [
    {
      "Tools_name": "search_trains",
      "Tools_definition": {
        "description": "Tìm chuyến tàu hỏa theo ga đi, ga đến và ngày.",
        "arguments": {
          "from_station": { "type": "string", "description": "Ga đi" },
          "to_station": { "type": "string", "description": "Ga đến" },
          "date": { "type": "string", "description": "Ngày đi, định dạng YYYY-MM-DD" }
        }
      }
    }
  ]
}
```
---

## The `details` object

`details` is descriptive and not validated field-by-field; it differs between datasets.

**Always present**

| key | meaning |
|---|---|
| `dataset` | dataset name, same as the file name |
| `doc_section` | the section of `doc/Model Eval.md` it implements |
| `description` | what the dataset is for |
| `source` | `vi.wikipedia.org` for generated datasets, or the skill used for hand-curated ones |
| `updated_at` | ISO 8601 time of the last write |
**Per-dataset counts**

| dataset | counts in `details` |
|---|---|
| rewriter | `n_test_cases`, `types` (`Bad`/`Good`), `categories` |
| planner | `n_test_cases`, `domains`, `plan_lengths` (histogram), `n_supersteps`, `mean_superstep_chars` |
| responder | `n_paragraphs`, `n_questions` |
| entity_relationship | `n_paragraphs`, `n_questions`, `n_enr_triplets`, `n_entities` |
| conflict_detector | `n_topic_bundles`, `n_conflicts`, `conflict_types`, `conflict_severities`, `note` |
| guardrail | `n_test_cases` |
| tool_agent | `n_test_cases`, `domains`, `iterations` (histogram of chain length), `tool_calls` |

**Example** (`responder.json`)

```json
{
  "dataset": "responder",
  "doc_section": "Model Eval.md section 3",
  "description": "Paragraphs grouped by topic, each with one reference Question/Answer/Evidence triplet. Combine paragraphs across topics for Precision and Recall, and within a topic for Reasoning Score.",
  "source": "vi.wikipedia.org",
  "crawl_max_depth": 1,
  "generator_model": "qwen3:8b",
  "updated_at": "2026-09-18T16:43:34+00:00",
  "n_topics": 12,
  "topics": { "Việt Nam": 17, "Trí tuệ nhân tạo": 19, "Phở": 12, "Bóng đá": 19 },
  "n_paragraphs": 164,
  "n_questions": 164
}
```

(`topics` shortened to four of twelve.)

---
