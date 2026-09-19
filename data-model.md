# AI 心意管家 — 数据模型文档（v1.1）

> 本文档描述 v1 的全部数据结构，含 6 张本地 SQLite 表、1 张 Supabase 云端表、11 个 AsyncStorage Key、18 个节日常量。所有字段名、表名严格与 spec.md / plan.md 一致。

---

## 一、概览

### 1.1 存储分层

```
┌──────────────────────────────────────────────────┐
│              客户端 (React Native)                │
│                                                    │
│  ┌──────────────────────┐  ┌───────────────────┐ │
│  │   SQLite (expo)      │  │  AsyncStorage     │ │
│  │   6 张业务表          │  │  11 个 KV 键值     │ │
│  │   (联系人/日子/缓存/  │  │  (设置/引导标记/   │ │
│  │    历史/错过提醒)     │  │   登录态缓存)      │ │
│  └──────────────────────┘  └───────────────────┘ │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │  TypeScript 常量: 18 个节日常量              │ │
│  └──────────────────────────────────────────────┘ │
└────────────────────┬─────────────────────────────┘
                     │ 登录认证 / 设置同步
                     ▼
┌──────────────────────────────────────────────────┐
│          Supabase (云端 — v1 轻量使用)            │
│                                                    │
│  ┌──────────────────────┐  ┌───────────────────┐ │
│  │  Auth (邮箱+密码)     │  │  profiles (1表)   │ │
│  │  users 表自动管理     │  │  用户设置云端备份  │ │
│  └──────────────────────┘  └───────────────────┘ │
│                                                    │
│  ┌──────────────────────────────────────────────┐ │
│  │  Edge Functions: 3 个 (DeepSeek/百炼)        │ │
│  └──────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────┘
```

### 1.2 表关系总览（ER 图）

```
contacts (联系人)
    │ 1
    │
    │ N
    ▼
 days (日子) ─────────────────────────────┐
    │ 1                                    │ N
    │                                      │
    │ 1                N                   ▼
    ▼          ┌── notification_missed (错过提醒)
 reminder_cache (提醒缓存)
    │
    │ (日子到期后写入历史)
    │ 松散关联 (day_id 可失效)
    ▼
 history (历史记录)
    │ 1
    │
    │ N
    ▼
 history_blessing_versions (祝福历史版本)


Supabase 云端:

 auth.users (Supabase 内置)
    │ 1
    │
    │ 1
    ▼
 profiles (用户设置云端备份)
```

### 1.3 关系一览

| 父表 | 子表 | 关系类型 | 外键 | 说明 |
|---|---|---|---|---|
| contacts | days | 一对多 | days.contact_id → contacts.id | 删除联系人时级联删除其所有 days |
| days | reminder_cache | 一对一 | reminder_cache.day_id → days.id | 按 day_id + reminder_year 唯一 |
| days | history | 松散一对多 | history.day_id → days.id | 松散关联：联系人删除后 day_id 可能失效，history 独立保留 |
| days | notification_missed | 一对多 | notification_missed.day_id → days.id | 删除日子时级联删除错过记录 |
| history | history_blessing_versions | 一对多 | history_blessing_versions.history_id → history.id | 删除历史时级联删除版本记录 |
| auth.users | profiles | 一对一 | profiles.id → auth.users.id | Supabase 内置；用户注册时触发器自动创建 profiles 行 |

---

## 二、本地 SQLite 表（v1 核心存储）

### 2.1 contacts（联系人）

存储用户录入的联系人信息。关系的归一化结果存入 `relationship_normalized`（内部逻辑使用），UI 始终展示 `relationship`（用户原始输入）。

| # | 字段名 | SQLite 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|---|
| 1 | `id` | TEXT | **是** (PK) | — | UUID v4，客户端生成 |
| 2 | `name` | TEXT | **是** | — | 姓名字段（必填），如"张三" |
| 3 | `relationship` | TEXT | **是** | — | 用户原始输入的关系文本（如"妈妈""大学室友老王"）；UI 展示此值 |
| 4 | `relationship_normalized` | TEXT | 否 | NULL | 归一化后的标准关系（10 组映射：母亲/父亲/妻子/丈夫/儿子/女儿/公公/婆婆/岳父/岳母）；匹配不上时为 NULL；UI 不展示 |
| 5 | `traits` | TEXT | 否 | NULL | 特征描述（选填，可空字符串），如"爱养花，今年刚退休" |
| 6 | `reminder_enabled` | INTEGER | 否 | 1 | 整个联系人的提醒开关（功能 6 局部关闭粒度一）：1=开启，0=关闭 |
| 7 | `created_at` | TEXT | 否 | — | ISO 8601 时间戳，如 `"2026-09-15T10:30:00.000Z"` |
| 8 | `updated_at` | TEXT | 否 | — | ISO 8601 时间戳，每次修改时更新 |

#### 索引

| 索引名 | 列 | 说明 |
|---|---|---|
| `idx_contacts_relationship_normalized` | `relationship_normalized` | 按归一化关系查询（身份自动关联匹配时使用） |
| `idx_contacts_created_at` | `created_at` | 按创建时间排序（联系人列表默认排序） |

#### 约束

- `name` 不可为空字符串
- `relationship` 不可为空字符串
- `reminder_enabled` 取值仅 0 或 1

#### 生命周期

- **创建**：用户新增联系人（功能 1），客户端生成 UUID，先写 contacts 再写对应的 birthday day（days 表）
- **更新**：用户编辑联系人信息（功能 3），更新 `updated_at`；`relationship` 变更触发 days 表身份自动关联重算
- **删除**：用户删除联系人（功能 4），**级联删除**所有关联的 days（FK），但相关 history 记录独立保留（通过快照字段）

---

### 2.2 days（日子）

每个联系人可拥有多个日子（一对多，无数量上限）。日子来源分两种：手动设定（source=manual）和身份自动关联（source=auto）。

| # | 字段名 | SQLite 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|---|
| 1 | `id` | TEXT | **是** (PK) | — | UUID v4 |
| 2 | `contact_id` | TEXT | **是** (FK) | — | 所属联系人 ID → contacts.id；未关联联系人的节日（holiday_unlinked）时此字段为空字符串 `""` |
| 3 | `name` | TEXT | **是** | — | 日子名称：生日固定为联系人姓名（自动填入，用户不可改）；纪念日由用户自定义命名（如"结婚纪念日""入职纪念日"）；节日为 fest 名称（如"母亲节"） |
| 4 | `type` | TEXT | **是** | — | 日子类型，枚举：`birthday`（生日）、`anniversary`（纪念日）、`holiday_auto`（身份自动关联的节日）、`holiday_unlinked`（未关联联系人的通用节日） |
| 5 | `date_month` | INTEGER | **是** | — | 月份，范围 1-12 |
| 6 | `date_day` | INTEGER | **是** | — | 日，范围 1-31 |
| 7 | `year` | INTEGER | 否 | NULL | 出生年份或纪念日年份（选填）；节日不填（NULL） |
| 8 | `lunar` | INTEGER | 否 | 0 | 是否为农历日期：1=农历，0=公历；v1 仅通用节日中的农历节日（春节/元宵节/除夕/端午节/七夕节/中秋节/重阳节/清明节）为 1，联系人日子均为 0 |
| 9 | `repeat_yearly` | INTEGER | 否 | 0 | 是否每年重复提醒：生日=1，节日=1，纪念日默认=0（由用户手动开启"每年一次"） |
| 10 | `reminder_enabled` | INTEGER | 否 | 1 | 单日提醒开关（功能 6 局部关闭粒度二）：1=开启，0=关闭 |
| 11 | `expired` | INTEGER | 否 | 0 | 是否已过期：仅当 repeat_yearly=0 且日子已过后置为 1（"仅一次"过期）；repeat_yearly=1 的日子永不过期 |
| 12 | `source` | TEXT | **是** | — | 来源枚举：`manual`（手动设定）、`auto`（身份自动关联，见功能 2） |
| 13 | `holiday_key` | TEXT | 否 | NULL | 节日键名，如 `mothers_day`、`womens_day`；仅 source=auto 或 type=holiday_unlinked 时有值；手动日子为 NULL |
| 14 | `created_at` | TEXT | 否 | — | ISO 8601 时间戳 |
| 15 | `updated_at` | TEXT | 否 | — | ISO 8601 时间戳 |

#### 索引

| 索引名 | 列 | 说明 |
|---|---|---|
| `idx_days_contact_id` | `contact_id` | 按联系人查询其所有日子（联系人详情页核心查询） |
| `idx_days_date_month_day` | `date_month, date_day` | 按日期查询（提醒扫描、节日匹配时使用） |
| `idx_days_type_expired` | `type, expired` | 提醒列表查询：按类型和过期状态筛选 |
| `idx_days_holiday_key` | `holiday_key` | 按节日键名查询（合并规则、身份关联删除检测） |
| `idx_days_source` | `source` | 区分手动/自动日子 |

#### 约束

- `date_month` 范围 1-12
- `date_day` 范围 1-31（不校验月份与日期组合的合法性，由应用层处理 2/29、2/30 等边界）
- `type` 仅允许 `birthday` / `anniversary` / `holiday_auto` / `holiday_unlinked`
- `source` 仅允许 `manual` / `auto`
- 当 `type = holiday_unlinked` 时，`contact_id` 为空字符串 `""`
- 当 `type = holiday_auto` 时，`source` 必须为 `auto`，`holiday_key` 不可为 NULL
- `repeat_yearly`：`birthday` 和 `holiday_*` 类型默认 1，`anniversary` 默认 0

#### 生命周期

- **手动创建**：用户为联系人添加纪念日（功能 2），source=manual
- **自动创建**：关系归一化匹配 holiday.linkedRelationships 后自动写入（功能 2），source=auto，contact_id 指向对应联系人
- **节日创建**：应用启动时扫描当年 18 个节日，为每个未关联联系人的节日创建一条 type=holiday_unlinked 的记录（contact_id=""）
- **关联节日创建**：用户添加联系人后，触发身份关联 → 为匹配的节日创建 type=holiday_auto 记录
- **更新**：关系变更时重算自动关联日子——新增匹配的、删除不再匹配的（弹窗询问）
- **过期**："仅一次"（repeat_yearly=0）的日子到期后次日置 expired=1，reminder_enabled 永久置 0
- **删除**：删除联系人时级联删除；用户手动删除单个日子（含自动关联的日子）

---

### 2.3 reminder_cache（提醒详情的生成内容缓存）

缓存 AI 生成内容（祝福语/礼物/贺卡）。同一 day_id + reminder_year 组合唯一（年度隔离）。

| # | 字段名 | SQLite 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|---|
| 1 | `id` | TEXT | **是** (PK) | — | UUID v4 |
| 2 | `day_id` | TEXT | **是** (FK) | — | 关联的日子 ID → days.id |
| 3 | `reminder_year` | INTEGER | **是** | — | 提醒所属年份（用于年度 session 隔离，功能 10）；如 2026 |
| 4 | `reminder_date` | TEXT | **是** | — | 最新一次提醒日期（如 `"2026-03-05"`），每次推送进入详情页时更新 |
| 5 | `selected_blessing_index` | INTEGER | 否 | NULL | 用户当前选定的祝福语索引（0-2），NULL 表示尚未选定任何一条（功能 12） |
| 6 | `blessings_json` | TEXT | 否 | NULL | 3 条祝福语 JSON，结构：`["祝福语1", "祝福语2", "祝福语3"]`；最后一次生成的结果 |
| 7 | `gifts_json` | TEXT | 否 | NULL | 5 条礼物建议 JSON，结构：`[{"name":"礼物名","reason":"推荐理由"}, ...]` |
| 8 | `gift_direction` | TEXT | 否 | NULL | 用户输入的礼物方向（如"实用""浪漫""预算500以内"），用于重新生成礼物 |
| 9 | `card_template` | TEXT | 否 | NULL | 贺卡使用的模板标识，枚举：`birthday` / `festival_general` / `festival_spring` / `festival_valentine` / `anniversary_generic_1` / `anniversary_generic_2` / `anniversary_love` |
| 10 | `card_image_path` | TEXT | 否 | NULL | 贺卡导出图片的本地文件路径 |
| 11 | `card_signature` | TEXT | 否 | NULL | 当前落款文本（可临时修改，功能 17）；为 NULL 时按关系反推落款 |
| 12 | `created_at` | TEXT | 否 | — | ISO 8601 时间戳 |
| 13 | `updated_at` | TEXT | 否 | — | ISO 8601 时间戳 |

#### 索引

| 索引名 | 列 | 说明 |
|---|---|---|
| `idx_reminder_cache_day_year` | `day_id, reminder_year` | 唯一查询：某日子某年的缓存 |
| `idx_reminder_cache_day_id` | `day_id` | 按日子查询所有年度缓存 |

#### 约束

- `day_id` + `reminder_year` 组合唯一（UNIQUE INDEX → 实际建为唯一索引）
- `selected_blessing_index` 取值仅 NULL / 0 / 1 / 2
- `blessings_json` 非 NULL 时必须为长度 3 的 JSON 数组

#### 生命周期

- **创建**：用户首次进入提醒详情页（变体 A），触发 AI 生成后写入
- **更新**：用户点"重新生成"祝福语/礼物/贺卡时更新对应字段；`reminder_date` 在每次推送进入详情页时更新
- **年度隔离**：下一年度 reminder_year 不同 → 新建一条记录（功能 10：上一年度缓存不跨年保留）
- **删除**：日子到期写入 history 后不清除，保留至应用卸载或手动清理

---

### 2.4 history（历史记录）

记录已过日子的祝福、礼物、贺卡内容。以快照方式存储联系人信息，与 contacts/days 表解耦（联系人被删除后历史仍可独立展示）。

| # | 字段名 | SQLite 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|---|
| 1 | `id` | TEXT | **是** (PK) | — | UUID v4 |
| 2 | `day_id` | TEXT | **是** (FK→days) | — | 关联的日子 ID → days.id（查询时可能已失效） |
| 3 | `contact_name` | TEXT | **是** | — | 写入时快照的联系人姓名（删除联系人后仍可显示） |
| 4 | `contact_relationship` | TEXT | **是** | — | 写入时快照的关系文本（用户原始输入） |
| 5 | `day_name` | TEXT | **是** | — | 日子名称快照（如"生日""结婚纪念日""母亲节"） |
| 6 | `day_type` | TEXT | **是** | — | 快照类型：`birthday` / `anniversary` / `holiday_auto` / `holiday_unlinked` |
| 7 | `occurred_date` | TEXT | **是** | — | 日子日期（如 `"2026-09-15"`），即该次提醒所对应的日子当天 |
| 8 | `reminder_disabled` | INTEGER | 否 | 0 | 该日子在到期时提醒是否被手动关闭：1=已关闭，0=未关闭；关闭时 blessings_json/gifts_json/card_image_path 均为 NULL |
| 9 | `blessings_json` | TEXT | 否 | NULL | 祝福语（最终版本，用户最后一次重新生成的），提醒关闭时为空 |
| 10 | `gifts_json` | TEXT | 否 | NULL | 礼物建议（最终版本），提醒关闭时为空 |
| 11 | `card_image_path` | TEXT | 否 | NULL | 贺卡图片路径，提醒关闭时为空 |
| 12 | `created_at` | TEXT | 否 | — | ISO 8601 时间戳（写入历史的时间） |

#### 索引

| 索引名 | 列 | 说明 |
|---|---|---|
| `idx_history_day_id` | `day_id` | 按日子查询（判断是否已写入历史） |
| `idx_history_occurred_date` | `occurred_date` | 按日期倒序排列（历史列表页默认排序） |

#### 约束

- `day_type` 仅允许 `birthday` / `anniversary` / `holiday_auto` / `holiday_unlinked`
- `reminder_disabled=1` 时 `blessings_json`、`gifts_json`、`card_image_path` 必须为 NULL

#### 生命周期

- **创建**：日子次日 App 启动时扫描写入（功能 18）；将 reminder_cache 内容快照 + 旧版本写入
- **读取**：历史 Tab 列表页 + 历史详情页
- **删除**：用户可在历史详情页手动删除某条历史记录（不影响提醒列表和联系人）；不级联删除
- **保留**：联系人被删除后历史仍保留；day_id 外键为松散关联——查询时若找不到对应 days 行，仍正常展示

---

### 2.5 history_blessing_versions（祝福历史版本）

存储同一历史记录中用户多次重新生成的全部旧版本祝福语，支持版本切换回看（功能 18）。

| # | 字段名 | SQLite 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|---|
| 1 | `id` | TEXT | **是** (PK) | — | UUID v4 |
| 2 | `history_id` | TEXT | **是** (FK) | — | 关联历史记录 ID → history.id |
| 3 | `version_index` | INTEGER | **是** | — | 版本序号（从 1 起递增） |
| 4 | `blessings_json` | TEXT | **是** | — | 该版本的 3 条祝福语 JSON |
| 5 | `created_at` | TEXT | 否 | — | ISO 8601 时间戳 |

#### 索引

| 索引名 | 列 | 说明 |
|---|---|---|
| `idx_hbv_history_id` | `history_id` | 按历史记录查询所有版本 |
| `idx_hbv_history_version` | `history_id, version_index` | 唯一排序：同一历史下版本号唯一 |

#### 约束

- 同一 `history_id` 内 `version_index` 唯一
- `blessings_json` 必须为长度 3 的 JSON 数组

#### 生命周期

- **创建**：历史写入时，将 reminder_cache 在提醒周期内所有重新生成的旧版本一并写入（最终版本存于 history.blessings_json）
- **读取**：历史详情页版本切换查看
- **删除**：删除历史记录时级联删除

---

### 2.6 notification_missed（错过提醒记录）

记录因推送开关关闭或系统通知权限关闭而错过的提醒，用于恢复后弹窗摘要（功能 9）。

| # | 字段名 | SQLite 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|---|
| 1 | `id` | TEXT | **是** (PK) | — | UUID v4 |
| 2 | `day_id` | TEXT | **是** (FK) | — | 关联的日子 ID → days.id |
| 3 | `day_name` | TEXT | **是** | — | 日子名称 |
| 4 | `contact_name` | TEXT | **是** | — | 联系人姓名 |
| 5 | `scheduled_date` | TEXT | **是** | — | 计划的提醒日期（如 `"2026-09-15"`） |
| 6 | `missed` | INTEGER | 否 | 0 | 是否错过：1=错过（未推送），0=已推送（标记后不再提醒） |
| 7 | `created_at` | TEXT | 否 | — | ISO 8601 时间戳 |

#### 索引

| 索引名 | 列 | 说明 |
|---|---|---|
| `idx_nm_day_id` | `day_id` | 按日子查询 |
| `idx_nm_missed` | `missed` | 按错过状态筛选（弹窗摘要查询） |
| `idx_nm_scheduled_date` | `scheduled_date` | 按计划日期排序 |

#### 约束

- `missed` 取值仅 0 或 1

#### 生命周期

- **创建**：每次提醒应推送但因开关关闭而未推送时写入一条（missed=1）
- **读取**：推送恢复后查询 missed=1 的记录生成摘要弹窗
- **标记**：用户查看摘要后，对应记录置 missed=0 或删除（取决于实现选择）

---

## 三、Supabase 云端表

### 3.1 profiles（用户设置云端备份）

v1 中仅用于已登录用户的设置持久化。实际读写以 AsyncStorage 为主，此表为备份/同步目标。用户注册时由触发器自动创建行。

| # | 字段名 | PostgreSQL 类型 | 必填 | 默认值 | 说明 |
|---|---|---|---|---|---|
| 1 | `id` | uuid | **是** (PK) | — | 与 `auth.users.id` 一致，外键关联 Supabase Auth |
| 2 | `nickname` | text | 否 | NULL | 用户昵称/落款（功能 17） |
| 3 | `advance_days` | smallint | 否 | 7 | 提醒提前天数，范围 1–60（功能 7） |
| 4 | `push_enabled` | boolean | 否 | true | 推送通知总开关（功能 9） |
| 5 | `holiday_general_enabled` | boolean | 否 | true | 通用节日提醒总开关 ①（功能 8） |
| 6 | `holiday_unlinked_enabled` | boolean | 否 | true | 未关联联系人的节日提醒 ②（功能 8） |
| 7 | `holiday_linked_enabled` | boolean | 否 | true | 已关联联系人的节日提醒 ③（功能 8） |
| 8 | `default_platform` | text | 否 | `'taobao'` | 默认电商平台，可选值：`taobao` / `jd` / `pinduoduo` / `vip` / `amazon`（功能 14） |
| 9 | `created_at` | timestamptz | 否 | `now()` | |
| 10 | `updated_at` | timestamptz | 否 | `now()` | |

#### 索引

| 索引名 | 列 | 说明 |
|---|---|---|
| `profiles_pkey` | `id` | 主键索引（自动创建） |

#### 约束

- `id` 外键引用 `auth.users(id)` ON DELETE CASCADE
- `advance_days` CHECK 约束：`advance_days >= 1 AND advance_days <= 60`
- `default_platform` CHECK 约束：`default_platform IN ('taobao', 'jd', 'pinduoduo', 'vip', 'amazon')`

#### 触发器

```sql
-- 用户注册时自动创建 profiles 行
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS trigger AS $$
BEGIN
  INSERT INTO public.profiles (id) VALUES (new.id);
  RETURN new;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

#### RLS 策略

```sql
-- 用户只能读写自己的 profiles 行
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can read own profile"
  ON public.profiles FOR SELECT
  USING (auth.uid() = id);

CREATE POLICY "Users can update own profile"
  ON public.profiles FOR UPDATE
  USING (auth.uid() = id);
```

#### v1 同步策略

```
未登录状态:
  AsyncStorage ←→ 本地 UI (唯一读写路径)

登录后:
  AsyncStorage → Supabase profiles (首次同步)
  之后: AsyncStorage + Supabase 双写

退出登录:
  Supabase session 清除
  AsyncStorage 保留 (继续作为本地设置源)
```

---

## 四、AsyncStorage 键值对

以下 11 个 Key 用于轻量设置和状态缓存，不建 SQLite 表。

| # | Key | 值类型 | 默认值 | 说明 |
|---|---|---|---|---|
| 1 | `has_onboarded` | boolean | false | 是否已完成引导页（功能 19），完成后置 true |
| 2 | `last_reminder_scan` | ISO 时间戳 string | null | 上次后台提醒扫描时间 |
| 3 | `missed_notification_summary` | JSON | null | 未读的错过提醒摘要（功能 9），结构：`{"count":3,"latest":"2026-09-15"}` |
| 4 | `supabase_session` | JSON | null | 缓存的 Supabase Auth session（"记住我"持久化登录态时写入） |
| 5 | `user_nickname` | string | null | 本地缓存的用户昵称（功能 17）；null=未设置，需按关系反推落款 |
| 6 | `advance_days` | number | 7 | 提醒提前天数本地缓存，范围 1-60（功能 7） |
| 7 | `push_enabled` | boolean | true | 推送通知总开关本地缓存（功能 9） |
| 8 | `holiday_general_enabled` | boolean | true | 通用节日提醒总开关 ① 本地缓存（功能 8） |
| 9 | `holiday_unlinked_enabled` | boolean | true | 未关联联系人的节日提醒 ② 本地缓存（功能 8） |
| 10 | `holiday_linked_enabled` | boolean | true | 已关联联系人的节日提醒 ③ 本地缓存（功能 8） |
| 11 | `default_platform` | string | `"taobao"` | 默认电商平台本地缓存（功能 14） |

> **读写优先级**：启动时先从 AsyncStorage 读取（即时可用），登录后在后台静默同步 Supabase profiles；写入时双写（AsyncStorage + Supabase）。

---

## 五、TypeScript 常量（非数据库）

### 5.1 节日常量（constants/holidays.ts）

18 个节日以 TypeScript 硬编码维护，不作为数据库表存储。每年启动时根据此常量 + lunar-javascript 计算当年实际日期，写入 days 表。

```typescript
export interface Holiday {
  key: string;                    // 唯一键名，如 'mothers_day', 'womens_day'
  name: string;                   // 节日名，如"母亲节"
  category: 'public' | 'traditional' | 'themed' | 'western';
  dateType: 'solar' | 'relative' | 'lunar';
  month?: number;                 // 公历固定：公历月；农历：农历月
  day?: number;                   // 公历固定：公历日；农历：农历日
  dayOfWeek?: number;             // 0=周日（仅相对日期）
  weekOfMonth?: number;           // 第几个（仅相对日期）
  linkedRelationships: string[];  // 自动关联的标准关系列表
}
```

**18 个节日数据（v1 完整清单）：**

| key | name | category | dateType | 日期规则 | linkedRelationships |
|---|---|---|---|---|---|
| `new_year` | 元旦 | public | solar | 1月1日 | [] |
| `spring_festival` | 春节 | public | lunar | 农历正月初一 | [] |
| `lantern` | 元宵节 | traditional | lunar | 农历正月十五 | [] |
| `womens_day` | 妇女节 | themed | solar | 3月8日 | ["母亲","妻子","婆婆","岳母"] |
| `qingming` | 清明节 | public | lunar | 农历（lunar-javascript 计算） | [] |
| `labor_day` | 劳动节 | public | solar | 5月1日 | [] |
| `mothers_day` | 母亲节 | themed | relative | 5月第2个周日 | ["母亲","婆婆","岳母"] |
| `dragon_boat` | 端午节 | public | lunar | 农历五月初五 | [] |
| `fathers_day` | 父亲节 | themed | relative | 6月第3个周日 | ["父亲","公公","岳父"] |
| `childrens_day` | 儿童节 | themed | solar | 6月1日 | []（不自动关联儿子/女儿） |
| `qixi` | 七夕节 | traditional | lunar | 农历七月初七 | [] |
| `mid_autumn` | 中秋节 | public | lunar | 农历八月十五 | [] |
| `national_day` | 国庆节 | public | solar | 10月1日 | [] |
| `double_ninth` | 重阳节 | traditional | lunar | 农历九月初九 | [] |
| `new_years_eve` | 除夕 | traditional | lunar | 农历腊月三十（或二十九） | [] |
| `christmas` | 圣诞节 | western | solar | 12月25日 | [] |
| `valentine` | 情人节 | western | solar | 2月14日 | [] |

### 5.2 关系同义归一化对照表（constants/relationship-map.ts）

10 组映射，硬编码于 TypeScript。

```typescript
export const RELATIONSHIP_MAP: Record<string, string[]> = {
  '母亲': ['妈妈', '母亲', '妈咪', '老妈', '娘', '娘亲'],
  '父亲': ['爸爸', '父亲', '爸', '老爸', '爹', '爹爹'],
  '妻子': ['老婆', '妻子', '太太', '夫人', '内人'],
  '丈夫': ['老公', '丈夫', '先生', '夫君'],
  '儿子': ['儿子', '小子', '男娃'],
  '女儿': ['女儿', '闺女', '丫头', '女娃'],
  '公公': ['公公', '家公'],
  '婆婆': ['婆婆', '家婆', '婆母'],
  '岳父': ['岳父', '老丈人', '丈人', '岳丈'],
  '岳母': ['岳母', '丈母娘', '岳母娘'],
};
```

### 5.3 落款反推规则（constants/signature-fallback.ts）

关系 → 落款的映射（功能 17），仅 8 种可反推的关系。

```typescript
export const SIGNATURE_FALLBACK: Record<string, string> = {
  '母亲': '你的孩子',
  '父亲': '你的孩子',
  '妻子': '你的丈夫',
  '丈夫': '你的妻子',
  '公公': '你的儿媳',
  '婆婆': '你的儿媳',
  '岳父': '你的女婿',
  '岳母': '你的女婿',
};
// 儿子/女儿及其他关系 → 落款默认留空
```

---

## 六、Edge Functions 请求/响应结构

虽然 Edge Functions 不直接对应数据库表，但其输入输出结构是数据流的重要组成部分，在此一并记录。

### 6.1 generate-blessings

```
POST /functions/v1/generate-blessings
Authorization: Bearer <supabase_access_token>

Request:
{
  "relationship": "母亲",        // 归一化后的标准关系
  "traits": "爱养花，刚退休",     // 特征描述，可为空字符串
  "day_type": "birthday"          // birthday / anniversary / holiday_auto / holiday_unlinked
}

Response (200):
{
  "blessings": [
    "祝福语1（≤600字）",
    "祝福语2（≤600字）",
    "祝福语3（≤600字）"
  ]
}

Response (401):
{ "error": "Unauthorized" }
```

### 6.2 generate-gifts

```
POST /functions/v1/generate-gifts
Authorization: Bearer <supabase_access_token>

Request:
{
  "traits": "爱养花，刚退休",
  "day_type": "birthday",
  "relationship": "母亲",
  "direction": "实用"              // 可选，用户输入的礼物方向
}

Response (200):
{
  "gifts": [
    { "name": "园艺工具套装", "reason": "适合爱养花的她" },
    { "name": "旅行背包", "reason": "退休后出行必备" },
    ...
  ]  // 共 5 条
}
```

### 6.3 generate-greeting-card

```
POST /functions/v1/generate-greeting-card
Authorization: Bearer <supabase_access_token>

Request:
{
  "recipient": "妈妈",             // 称呼（优先）或姓名，只传一个
  "day_type": "birthday",          // birthday / anniversary / holiday_auto / holiday_unlinked
  "blessing": "祝您生日快乐...",   // 用户选定的祝福语文本
  "template": "birthday",          // 模板标识
  "signature": "你的孩子"          // 落款（昵称 > 关系反推 > 空字符串）
}

Response (200):
{
  "image_url": "https://...",      // 贺卡图片 URL 或 base64 data URL
  "format": "png"
}
```

---

## 七、为 Supabase 迁移做准备

### 7.1 v1 → 后续版本的迁移路径

| 当前（v1） | 后续版本 | 迁移要点 |
|---|---|---|
| SQLite contacts/days | Supabase contacts/days 表 | 数据结构不变，新增 user_id 字段关联 profiles.id |
| SQLite reminder_cache | 保留本地（性能） | 不改动；AI 生成缓存对延迟敏感，适合本地 |
| SQLite history | Supabase history 表 | 为用户提供多设备历史查看；冲突以设备本地为准 |
| AsyncStorage 设置 | Supabase profiles | 已有 profiles 表，届时改为 Supabase 为主、AsyncStorage 为缓存 |
| TypeScript 节日常量 | 无需迁移 | 纯逻辑数据，不涉及存储 |

### 7.2 云端建表 SQL 模板（预留）

```sql
-- 示例：contacts 未来上云时的建表语句
CREATE TABLE public.contacts (
  id         uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id    uuid NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
  name       text NOT NULL,
  relationship text NOT NULL,
  relationship_normalized text,
  traits     text,
  reminder_enabled boolean DEFAULT true,
  created_at timestamptz DEFAULT now(),
  updated_at timestamptz DEFAULT now()
);

CREATE INDEX idx_contacts_user_id ON public.contacts(user_id);
ALTER TABLE public.contacts ENABLE ROW LEVEL SECURITY;
```

---

## 八、附录：SQLite 建表语句（完整 DDL）

```sql
-- ============================================================
-- contacts（联系人）
-- ============================================================
CREATE TABLE IF NOT EXISTS contacts (
  id                        TEXT PRIMARY KEY NOT NULL,
  name                      TEXT NOT NULL,
  relationship              TEXT NOT NULL,
  relationship_normalized   TEXT,
  traits                    TEXT,
  reminder_enabled          INTEGER NOT NULL DEFAULT 1,
  created_at                TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at                TEXT NOT NULL DEFAULT (datetime('now')),
  CHECK (length(name) > 0),
  CHECK (length(relationship) > 0),
  CHECK (reminder_enabled IN (0, 1))
);

CREATE INDEX IF NOT EXISTS idx_contacts_relationship_normalized
  ON contacts(relationship_normalized);
CREATE INDEX IF NOT EXISTS idx_contacts_created_at
  ON contacts(created_at);

-- ============================================================
-- days（日子）
-- ============================================================
CREATE TABLE IF NOT EXISTS days (
  id                TEXT PRIMARY KEY NOT NULL,
  contact_id        TEXT NOT NULL,
  name              TEXT NOT NULL,
  type              TEXT NOT NULL,
  date_month        INTEGER NOT NULL,
  date_day          INTEGER NOT NULL,
  year              INTEGER,
  lunar             INTEGER NOT NULL DEFAULT 0,
  repeat_yearly     INTEGER NOT NULL DEFAULT 0,
  reminder_enabled  INTEGER NOT NULL DEFAULT 1,
  expired           INTEGER NOT NULL DEFAULT 0,
  source            TEXT NOT NULL,
  holiday_key       TEXT,
  created_at        TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at        TEXT NOT NULL DEFAULT (datetime('now')),
  FOREIGN KEY (contact_id) REFERENCES contacts(id) ON DELETE CASCADE,
  CHECK (length(name) > 0),
  CHECK (type IN ('birthday', 'anniversary', 'holiday_auto', 'holiday_unlinked')),
  CHECK (date_month >= 1 AND date_month <= 12),
  CHECK (date_day >= 1 AND date_day <= 31),
  CHECK (lunar IN (0, 1)),
  CHECK (repeat_yearly IN (0, 1)),
  CHECK (reminder_enabled IN (0, 1)),
  CHECK (expired IN (0, 1)),
  CHECK (source IN ('manual', 'auto'))
);

CREATE INDEX IF NOT EXISTS idx_days_contact_id ON days(contact_id);
CREATE INDEX IF NOT EXISTS idx_days_date_month_day ON days(date_month, date_day);
CREATE INDEX IF NOT EXISTS idx_days_type_expired ON days(type, expired);
CREATE INDEX IF NOT EXISTS idx_days_holiday_key ON days(holiday_key);
CREATE INDEX IF NOT EXISTS idx_days_source ON days(source);

-- ============================================================
-- reminder_cache（提醒缓存）
-- ============================================================
CREATE TABLE IF NOT EXISTS reminder_cache (
  id                        TEXT PRIMARY KEY NOT NULL,
  day_id                    TEXT NOT NULL,
  reminder_year             INTEGER NOT NULL,
  reminder_date             TEXT NOT NULL,
  selected_blessing_index   INTEGER,
  blessings_json            TEXT,
  gifts_json                TEXT,
  gift_direction            TEXT,
  card_template             TEXT,
  card_image_path           TEXT,
  card_signature            TEXT,
  created_at                TEXT NOT NULL DEFAULT (datetime('now')),
  updated_at                TEXT NOT NULL DEFAULT (datetime('now')),
  FOREIGN KEY (day_id) REFERENCES days(id) ON DELETE CASCADE,
  CHECK (selected_blessing_index IS NULL OR selected_blessing_index IN (0, 1, 2))
);

CREATE UNIQUE INDEX IF NOT EXISTS idx_reminder_cache_day_year
  ON reminder_cache(day_id, reminder_year);
CREATE INDEX IF NOT EXISTS idx_reminder_cache_day_id
  ON reminder_cache(day_id);

-- ============================================================
-- history（历史记录）
-- ============================================================
CREATE TABLE IF NOT EXISTS history (
  id                      TEXT PRIMARY KEY NOT NULL,
  day_id                  TEXT NOT NULL,
  contact_name            TEXT NOT NULL,
  contact_relationship    TEXT NOT NULL,
  day_name                TEXT NOT NULL,
  day_type                TEXT NOT NULL,
  occurred_date           TEXT NOT NULL,
  reminder_disabled       INTEGER NOT NULL DEFAULT 0,
  blessings_json          TEXT,
  gifts_json              TEXT,
  card_image_path         TEXT,
  created_at              TEXT NOT NULL DEFAULT (datetime('now')),
  CHECK (length(contact_name) > 0),
  CHECK (day_type IN ('birthday', 'anniversary', 'holiday_auto', 'holiday_unlinked')),
  CHECK (reminder_disabled IN (0, 1))
);

CREATE INDEX IF NOT EXISTS idx_history_day_id ON history(day_id);
CREATE INDEX IF NOT EXISTS idx_history_occurred_date ON history(occurred_date);

-- ============================================================
-- history_blessing_versions（祝福历史版本）
-- ============================================================
CREATE TABLE IF NOT EXISTS history_blessing_versions (
  id              TEXT PRIMARY KEY NOT NULL,
  history_id      TEXT NOT NULL,
  version_index   INTEGER NOT NULL,
  blessings_json  TEXT NOT NULL,
  created_at      TEXT NOT NULL DEFAULT (datetime('now')),
  FOREIGN KEY (history_id) REFERENCES history(id) ON DELETE CASCADE
);

CREATE INDEX IF NOT EXISTS idx_hbv_history_id
  ON history_blessing_versions(history_id);
CREATE UNIQUE INDEX IF NOT EXISTS idx_hbv_history_version
  ON history_blessing_versions(history_id, version_index);

-- ============================================================
-- notification_missed（错过提醒记录）
-- ============================================================
CREATE TABLE IF NOT EXISTS notification_missed (
  id              TEXT PRIMARY KEY NOT NULL,
  day_id          TEXT NOT NULL,
  day_name        TEXT NOT NULL,
  contact_name    TEXT NOT NULL,
  scheduled_date  TEXT NOT NULL,
  missed          INTEGER NOT NULL DEFAULT 0,
  created_at      TEXT NOT NULL DEFAULT (datetime('now')),
  FOREIGN KEY (day_id) REFERENCES days(id) ON DELETE CASCADE,
  CHECK (missed IN (0, 1))
);

CREATE INDEX IF NOT EXISTS idx_nm_day_id ON notification_missed(day_id);
CREATE INDEX IF NOT EXISTS idx_nm_missed ON notification_missed(missed);
CREATE INDEX IF NOT EXISTS idx_nm_scheduled_date ON notification_missed(scheduled_date);
```

---

## 九、字段名 / 表名 / 枚举值与 spec 对照

| 数据模型中的值 | spec.md 出处 |
|---|---|
| `contacts.name` = 姓名 | 功能 1 "必填项：姓名" |
| `contacts.relationship` = 关系（自由文本） | 功能 1 "\"关系\"为自由文本，由用户自行输入" |
| `contacts.traits` = 特征描述 | 功能 1 "选填项：特征描述" |
| `days.name` = 纪念日用户可自定义命名 | 功能 1 "每个纪念日用户可自定义命名" |
| `days.type` = birthday / anniversary / holiday_auto / holiday_unlinked | 功能 2 "手动设定 + 身份自动关联"；功能 8 "未关联联系人的节日" |
| `days.source` = manual / auto | 功能 2 "手动设定" / "身份自动关联" |
| `days.repeat_yearly` = 生日每年一次；纪念日默认仅生效一次 | 功能 6 "周期规则：生日每年一次；纪念日默认仅生效一次" |
| `days.expired` + `days.repeat_yearly=0` → "已过期"分区 | 功能 6 "仅一次""到期后提醒开关永久关闭" |
| `reminder_cache.selected_blessing_index` = 0-2 / null | 功能 12 "首次进入详情页时，3 条祝福语均为未选中状态" |
| `reminder_cache.blessings_json` = 3 条 | 功能 11 "默认每次生成 3 条" |
| `reminder_cache.gifts_json` = 5 条 | 功能 13 "默认每次推荐 5 个" |
| `reminder_cache.reminder_year` 年度隔离 | 功能 10 "下一年度的提醒为新开 session" |
| `history.occurred_date` = 日子本身 | 功能 18 "写入的是**日子本身**（如生日那天）" |
| `history.day_type` 不标注"仅一次"/"每年一次" | 功能 18 "不标注该提醒是\"仅一次\"还是\"每年一次\"" |
| `history.reminder_disabled=1` → 空内容 + 默认文案 | 功能 18 "被手动关闭提醒的日子写入历史后祝福/礼物/贺卡内容为空" |
| `history_blessing_versions` = 版本回看 | 功能 18 "所有旧版本一并写入子版本列表，可通过版本切换回看" |
| `profiles.advance_days` 范围 1-60 | 功能 7 "可调范围为 1–60 天" |
| `profiles.default_platform` = taobao / jd / pinduoduo / vip / amazon | 功能 14 "淘宝、京东、拼多多、唯品会、亚马逊" |
| `holiday_key` = mothers_day / fathers_day / ... | 功能 8 "母亲节、父亲节、妇女节、儿童节…" |