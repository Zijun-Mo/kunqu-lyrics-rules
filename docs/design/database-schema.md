# 参考数据库结构与 Agent 查询

[架构设计](architecture.md) · [规则与采录来源](../checklist.md)

首版在同一 SQLite 文件内实现 11 个逻辑库，用表名前缀对应 D 编号。以下是实现契约。各库共用版源、摘录、发布快照和工具响应结构。来源书目与采录边界见规则表第三节。

## 共用结构

- 字段记法采用 `id` 主键、`field→table` 外键、`?` 可空、`#` 整数、`!` 布尔。其余字段为必填 TEXT。排序字段从 1 起。列表用关联表保存。按范围使用复合唯一键。
- `source(id, title, author?, edition, uri?, source_kind)` 保存可区分的版本。`evidence(id, source_id→source, locator, quote?, note?, level)` 保存卷页或条目定位和原注，`level` 取明示、通融、例见、未确认。逐条事实用 `evidence_id→evidence`，需附旁证时按 `〈实体名〉_evidence(record_id→该实体, evidence_id→evidence, relation)` 建关联表，共用支持、反证、旁证三类关系。重要冲突保留多条及各自出处。
- `snapshot(id, published_at, schema_version)` 指向一个只读发布版本。实现可为每次发布导出独立 SQLite 文件。任务固定快照，谱格、规则、程序查询和模型工具使用同一版。当前稿的字位、选定韵部和诊断保存在应用库，按标识与参考数据关联。
- 工具统一返回 `{status, items, evidence, snapshot_id}`。状态取 `found | ambiguous | missing | error`；数据项返回来源记录标识，`evidence` 按 ID 带版本、定位及必要摘录。一次请求绑定体式、韵系和快照，模型提供字、词、句位或检索词等查询参数。

## D01 · 曲牌、宫调与体式

- `d01_mode(id, tradition, name, evidence_id)`。南北曲与宫调名组合唯一。
- `d01_tune(id, mode_id→d01_mode, canonical_name, evidence_id)`、`d01_form(id, tune_id→d01_tune, label, evidence_id)`。体式归入具体宫调下的曲牌，以 `d01_form_kind(form_id→d01_form, kind)` 记录引子、过曲、本序、换头、别体等分类。
- `d01_alias(id, tune_id→d01_tune, name, evidence_id)`。异名索引允许同名多条。
- **取用**：`resolve_tune(name, tradition?, mode?, form?)` 先按正名、异名取候选，再联 D02 列底本与覆盖。选定 form_id 与 score_id 后交两位 Agent 和 K01。

## D02 · 单支谱格

- `d02_score(id, form_id→d01_form, evidence_id)` 一条对应一个体式在一个底本中的谱格。`d02_line(id, score_id→d02_score, seq#, evidence_id)` 按谱唯一排序。
- `d02_slot(id, line_id→d02_line, seq#, kind, rhyme_status, break_after!?, evidence_id)`，kind 取正字、允许衬位或复现标记，rhyme_status 取必押、可押、明不押、未知。一个衬位容纳几个字按 D08 的位点许可查。
- `d02_slot_pattern(id, evidence_id)`、`d02_slot_pattern_term(pattern_id→d02_slot_pattern, slot_id→d02_slot, tone?, yin_yang?, ping_ze?)`。每个模式保存一组共同成立的声序或声类条件。约束相同字位范围的模式形成备选集合；单字位也用此结构。绑定证据等级，例见供参考。
- `d02_pair(id, score_id→d02_score, left_first→d02_slot, left_last→d02_slot, right_first→d02_slot, right_last→d02_slot, evidence_id)` 保存对偶组的两侧范围。
- **取用**：按 score_id 一次读取整支谱格，给写作 Agent 逐句字位、允许模式和韵位。程序与审校 Agent 以相同 slot_id 对齐已写字句及问题。可按 slot_id 展开谱注及 D08 例外。

## D04 · 联套、韵段与合前

- `d04_suite(id, name, evidence_id)`、`d04_part(id, suite_id→d04_suite, seq#, form_id→d01_form, score_id→d02_score, role, transition, evidence_id)`。每个可采用的套序单独成套；role 含尾牌，transition 记录常格、借宫、犯调。
- `d04_rhyme_span(id, suite_id→d04_suite, seq#, change_status, evidence_id)`、`d04_rhyme_slot(span_id→d04_rhyme_span, part_id→d04_part, slot_id→d02_slot)` 定义韵段及允许的换韵边界。实际选韵存在稿次。`d04_ending(suite_id→d04_suite, tail_status, last_part_id→d04_part?, evidence_id)` 记录必有、可免或未知。
- `d04_reuse(target_part_id→d04_part, target_slot_id→d02_slot, source_part_id→d04_part, source_slot_id→d02_slot, evidence_id)` 逐位保存合前来源。四表的部件和字位均须属于对应套式和谱格。
- **取用**：联套时以 suite_id 把支序、韵段、尾牌及复用关系加入固定上下文。K14/K16 按韵段和支曲复核，K18 按目标字位展开并去重。

## D05 · 曲韵与读法

- `d05_system(id, name, tradition, source_id→source)`、`d05_group(id, system_id→d05_system, label, evidence_id)`。同一韵系内韵部标签唯一。
- `d05_reading(id, glyph, system_id→d05_system, group_id→d05_group?, sense_id→d13_sense?, sense_hint?, historical_tone?, effective_tone?, yin_yang?, evidence_id)` 保存一个完整读法，北曲派声用 effective_tone 与 group_id 同行。韵部所属体系须一致。
- `d05_position(id, reading_id→d05_reading, scope, status, evidence_id)` 保存明载限制，scope 含句中、韵脚或具体位置，由 D08 补充限定。查询分别标示无条与未确认。
- **取用**：索引 `(system_id, glyph)` 支撑查字，`(system_id, group_id, effective_tone)` 支撑反查。`lookup_character` 联 D13 义项，`find_rhyme_words` 按目标字位筛。程序保存每个候选 reading_id 并整体核声类、阴阳与韵部。

## D06 · 历史音系特征

- `d06_profile(id, system_id→d05_system, label, evidence_id)` 标识一套可比较的分类口径。
- `d06_feature(id, reading_id→d05_reading, profile_id→d06_profile, fanqie?, homophone_key?, initial_key?, rime_key?, closure?, coda?, evidence_id)`。同口径的键分别用于同音、双声、叠韵与开闭口。跨体系旁证保留原标签与出处。
- **取用**：`lookup_phonology(reading_ids)` 返回所用口径的特征。K17、K19、K20、K22 对同 profile_id 的候选比较，并应用 D08。审校可查标记的原因和原证。

## D07 · 鱼模细分

- `d07_attestation(id, reading_id→d05_reading, original_label, proposed_class?, evidence_id)` 每书每字例一条。`d07_decision(id, reading_id→d05_reading, policy_id→d08_policy, class?, status, evidence_id)` 保存该口径下的已核、待核或冲突结果。决定与支持、反对字例通过 `d07_decision_attestation(decision_id→d07_decision, attestation_id→d07_attestation, relation)` 关联。
- **取用**：选用李渔口径时按 reading_id 和 policy_id 联查。写作选字和 K14/K16 使用已核分类，审校可展开冲突字例。已核结果须绑定鱼或模。

## D08 · 格律口径及例外

- `d08_policy(id, rule_code, kind, effect, activation, evidence_id)`。kind 为入代、通押、位置限制、衬字上限、曲禁、连声等，effect 为规定、允许、禁止或提示，activation 标识默认启用或须选用的口径。
- `d08_scope(id, policy_id→d08_policy, tradition?, form_id→d01_form?, score_id→d02_score?, slot_id→d02_slot?, span_id→d04_rhyme_span?, system_id→d05_system?, position_kind?)`。同一行各非空条件同时成立，多行表示备选范围。
- 参数按类型分表。`d08_rhyme_pair(policy_id→d08_policy, from_group_id→d05_group, to_group_id→d05_group)`、`d08_tone_substitution(policy_id→d08_policy, source_tone, target_tone, reading_id→d05_reading?)`、`d08_slot_limit(policy_id→d08_policy, limit_kind, max_count#)`、`d08_pattern(policy_id→d08_policy, feature, comparison, threshold#?, exempt_text?)`。每类只接受注册的字段和值。
- **取用**：`resolve_policies(rule_codes, scope)` 按已启用口径和谱位筛取。程序读取有据条件，编排器把本轮适用条目和出处附到相应规则提示。提示文本与按阶段启用条件由规则注册表版本化。

## D09 · 字形与异文

- `d09_glyph_relation(id, original, related, relation_kind, lookup_allowed!, dedup_allowed!, evidence_id)`。关系含繁简、正异体、通假，两个许可分别控制检索扩展与查重归并。
- `d09_variant(id, variant_set, score_id→d02_score?, slot_id→d02_slot?, original, attested, evidence_id)`。同校勘组可含多版证据，保留原字及所在字位。
- **取用**：`lookup_variants(glyph, score_id?, slot_id?)` 联 D05 扩展已允许的查字形式，结果注明原查字和命中形式。K18/K24 仅按允许归并的关系计算，诊断位置仍指原稿。

## D11、D13、D14 · 共用段落检索

- `passage(id, evidence_id→evidence, text, prev_id→passage?, next_id→passage?)` 保存可独立引用的原文段落，同一篇内可取相邻段。`passage_tag(passage_id→passage, kind, label)` 按题材、情境、主题等索引。按字词或片段建立全文索引，先按结构字段筛选，再按相关度截取少量可引用结果。
- **D11 曲例**。`d11_example(id, passage_id→passage, tune_id→d01_tune?, form_id→d01_form?, line_id→d02_line?, author?, work_title, role?)`、`d11_comment(id, example_id→d11_example, text, origin, evidence_id→evidence?)` 区分原谱评与整理推断。`lookup_examples` 优先同体同句位，候选标明匹配层级。供拟稿参考与 H 类审词引用。
- **D13 字义与典源**。`d13_headword(id, headword, locality?, period?)`、`d13_sense(id, headword_id→d13_headword, seq#, gloss, register?, evidence_id→evidence)`；`d13_allusion(id, name, summary?)`、`d13_alias(allusion_id→d13_allusion, phrase)`、`d13_origin(allusion_id→d13_allusion, passage_id→passage, relation)`。典源关系标原典、异说、后用。`lookup_character` 返回 D05 所联义项及匹配的其他义项，`lookup_allusion` 按名称或引语返回原典、位置及必要前后文。后用条目可另联 D11。
- **D14 曲论**。`d14_principle(id, passage_id→passage, topic, claim, scope?, claim_kind)`、`d14_rule(principle_id→d14_principle, rule_code)`、`d14_example(principle_id→d14_principle, example_id→d11_example)`。claim 可记录整理摘要，passage 保留原论；claim_kind 标明作者主张、习见或明定约束。`lookup_principles(rule_codes, topic?)` 按本轮审写规则取相应原论。确定性格律条件通过 D02/D08 执行。

## 两位 Agent 如何取用

| 时机 | 输入或工具结果 |
|---|---|
| 选定曲牌后 | D01 → D02，联套再加 D04。编排器序列化曲牌标识、谱本、逐句字位及套式，形成两位 Agent 共享的固定上下文。 |
| 起草、改词或审校开始时 | 按稿次、任务类型和目标范围选择适用规则提示，加入 D08 的有效条件及与该规则相关的 D14 摘录。审校 Agent 同时读取程序诊断，写作 Agent 改词时读取目标批注。 |
| 落笔或审读时 | 通过上述只读接口按需查字、选韵、找典和找例。程序可按相同记录标识复核，审校 Agent 可自行补查；双方引证均落到 evidence ID。 |

一次改韵脚时，编排器从 D02 给出韵位和声类，从稿次取本段已定韵部，加入 K14/K15/H15 提示。写作 Agent 用 D05 找候选，结合 D13 核义，按口径补查 D07/D08。程序运行整句及相关韵段校验，审校 Agent 按词意和用韵结果复核。若多读或资料缺项影响判断，诊断保留待核。
