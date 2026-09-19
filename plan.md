# AI 心意管家 — 技术实施计划（v1.1）

---

## 一、技术选型与关键决策

### 1.1 整体架构

| 层 | 选型 | 说明 |
|---|---|---|
| 前端框架 | React Native + Expo（managed workflow） | 跨平台 iOS/Android 一套代码；Expo 托管构建、OTA 更新、推送通知 |
| 后端 BaaS | Supabase | 登录认证（Supabase Auth）+ 未来云端存储（v1 阶段以本地存储为主） |
| AI 大模型 | DeepSeek（通过 Supabase Edge Functions 代理） | 祝福生成、礼物推荐由 Edge Function 调用 DeepSeek API，前端不直连 |
| AI 图片生成 | 阿里云百炼平台（通过 Supabase Edge Functions 代理） | 贺卡生成使用百炼的生图模型 |
| 本地存储 | expo-sqlite / AsyncStorage | SQLite 负责联系人、日子、历史记录、AI 生成缓存等全部业务数据；AsyncStorage 负责轻量 KV（设置、引导页标记、登录态标记） |
| 农历计算 | lunar-javascript | 农历节日日期换算、农历日期间转换 |
| 推送通知 | Expo Notifications | 本地定时通知 + 后台任务（expo-task-manager） |
| 状态管理 | React Context + useReducer | 按域拆分 ContactContext / ReminderContext / SettingsContext / AuthContext |
| 导航 | Expo Router（文件系统路由） | Expo 官方推荐，基于 React Navigation |

### 1.2 关键决策点

#### 决策 1：为什么用 Expo Router 而非 React Navigation 裸写？

- spec 定义了明确的页面层级和深链跳转（推送 → 详情页、过期提醒 → 历史 Tab），Expo Router 的文件系统路由天然支持 deep link，路径映射直观；
- 底部 3 个 Tab 用 `(tabs)` 分组目录即可表达，设置页的 modal 呈现可用 `modal` 目录。

#### 决策 2：为什么 v1 以本地 SQLite 为存储核心，而非 Supabase 全量上云？

- spec 明确"历史记录保存在本地"（功能 18）、"已生成的祝福/礼物/贺卡缓存并持久化到本地磁盘"（功能 10），v1 不做多设备同步（"以后再做"列表）；
- 本地优先降低网络依赖、提升响应速度；Supabase profiles 表在 v1 阶段用于登录态关联和设置备份，后续版本再扩展为全量云端存储；
- 登录与功能的关系（功能 29）明确：本地功能免登录可用，AI 功能需登录——这一设计天然适配"本地数据 + 云端认证"架构。

#### 决策 3：为什么用 Supabase Edge Functions 代理 AI 调用（DeepSeek + 百炼）而非客户端直连？

- API Key 不能暴露在客户端；Edge Function 作为中间层可控制请求参数（条数固定 3 条祝福 / 5 个礼物）、统一错误处理、方便后续扩展（如加入滥用检测）；
- Supabase Edge Functions 基于 Deno，TypeScript 原生支持，部署链路短；
- DeepSeek API 用于文本生成（祝福语、礼物建议）、阿里云百炼用于图片生成（贺卡），两条 AI 链路均通过 Edge Function 代理。

#### 决策 4：为什么用 lunar-javascript 而非系统日历 API 算农历？

- expo-calendar 只能读取系统日历中已有的农历事件，无法主动将公历日期换算为农历日期；
- v1 不涉及联系人的农历生日，仅需在通用节日中确定当年农历节日对应的公历日期；lunar-javascript 是纯 JavaScript 实现，不依赖原生模块，与 Expo managed workflow 兼容；
- 节日当年的准确日期通过 lunar-javascript 的农历→公历转换计算得出，无需联网。

#### 决策 5：推送通知的实现策略

- 提醒节奏（≥7 天每周一次、≤3 天每天一次）由客户端本地定时任务驱动，借助 `expo-task-manager` 的 `BackgroundFetch` 周期性扫描即将到来的日子；
- 每次扫描检查当前日期是否命中某个提醒窗口，命中则调度 `expo-notifications` 本地通知；
- 不依赖远程推送服务（FCM/APNs）做核心提醒逻辑，远程推送仅作为备用唤醒通道。

#### 决策 6：登录态与 AI 功能的耦合方式

- 登录态通过 Supabase Auth 管理，`AuthContext` 在应用启动时从 `AsyncStorage` 恢复 session；
- 所有 AI 功能调用前先检查 `AuthContext` 中的登录态：未登录 → 弹出登录引导弹窗（功能 29），不发起 API 请求；已登录 → 正常调用 Edge Function；
- 登录态与本地 SQLite 数据完全解耦——退出登录不清除本地数据，AI 生成缓存仍可查看但不能发起新生成。

### 1.3 依赖清单

| 类别 | 包名 | 用途 |
|---|---|---|
| 导航 | expo-router | 文件系统路由 |
| 数据库 | expo-sqlite | 本地 SQLite |
| KV 存储 | @react-native-async-storage/async-storage | 轻量键值对（设置、引导页标记、登录态） |
| 推送 | expo-notifications, expo-task-manager | 本地通知 + 后台任务 |
| 文件/分享 | expo-file-system, expo-sharing, expo-media-library | 贺卡图片保存与分享 |
| 农历 | lunar-javascript | 农历节日日期换算 |
| Supabase | @supabase/supabase-js, @supabase/ssr | 客户端 SDK + Auth |
| DeepSeek | 无客户端 SDK，通过 Edge Function 调用 HTTP API | AI 文本生成 |
| 百炼 | 无客户端 SDK，通过 Edge Function 调用 HTTP API | AI 图片生成 |

### 1.4 不引入的依赖（明确排除）

以下在 v1 中不使用：
- 状态管理库（Redux / Zustand / MobX）：Context + useReducer 足够；
- 远程推送服务（FCM / APNs 独立接入）：Expo Notifications 已封装；
- 第三方 UI 组件库（NativeBase / React Native Paper）：自写组件保持对 spec 交互的精确控制；
- 多设备同步（"以后再做"明确列出）。

---

## 二、页面清单

按 spec 第 20-30 节梳理，共计 **14 类页面/视图**：

| 序号 | 页面名称 | 对应 spec 节 | 页面类型 | 说明 |
|---|---|---|---|---|
| P1 | 引导页 | §19 | Stack（独立栈） | 首次启动展示，之后不再出现 |
| P2 | 提醒 Tab（首页） | §21 | Tab | 底部第 1 个 Tab；提醒列表（生效中 / 已过期分区） |
| P3 | 联系人 Tab | §22 | Tab | 底部第 2 个 Tab；联系人列表 |
| P4 | 历史 Tab | §23 | Tab | 底部第 3 个 Tab；已过提醒日列表 |
| P5 | 提醒详情页 — 变体 A | §25 | Stack（推入） | 关联联系人的提醒详情（祝福 + 礼物 + 贺卡） |
| P6 | 提醒详情页 — 变体 B | §25 | Stack（推入） | 未关联联系人的节日详情（仅生成贺卡按钮） |
| P7 | 联系人详情页 | §22 | Stack（推入） | 单个联系人的完整信息 + 日子列表 |
| P8 | 联系人表单页（新增/编辑） | §22 | Stack（推入） | 新增联系人 / 编辑联系人共用 |
| P9 | 历史详情页 | §23 | Stack（推入） | 某条历史记录的祝福/礼物/贺卡详情 |
| P10 | 设置页 | §24 | Modal | 从各主页面右上角进入，模态呈现；含登录状态展示 |
| P11 | 提醒规则说明页 | §24 | Stack（推入） | 从设置页"提醒提前天数"旁进入，或联系人详情页"添加纪念日"弹窗 |
| P12 | 登录页 | §30 | Stack（推入） | 邮箱 + 密码 + "记住我" + "去注册"链接 |
| P13 | 注册页 | §30 | Stack（推入） | 邮箱 + 密码 + 确认密码 + "去登录"链接 |
| P14 | 登录引导弹窗 | §29 | Modal / Alert | 未登录触发 AI 功能时弹出 |

---

## 三、路由与入口

### 3.1 路由树（Expo Router 文件路径约定）

```
app/
├── _layout.tsx                    → 根布局：判断首次启动 → 引导页/主界面；包裹 AuthContext
├── onboarding.tsx                 → P1 引导页
├── (tabs)/
│   ├── _layout.tsx                → 底部 3 Tab 布局
│   ├── reminders.tsx              → P2 提醒 Tab（首页）
│   ├── contacts.tsx               → P3 联系人 Tab
│   └── history.tsx                → P4 历史 Tab
├── reminder-detail/
│   └── [reminderId].tsx           → P5/P6 提醒详情页（根据 reminder 类型渲染变体 A 或 B）
├── contact-detail/
│   └── [contactId].tsx            → P7 联系人详情页
├── contact-form/
│   └── [contactId].tsx            → P8 联系人表单页（contactId 为 "new" 时为新增，否则为编辑）
├── history-detail/
│   └── [historyId].tsx            → P9 历史详情页
├── settings.tsx                    → P10 设置页（modal 呈现）
├── reminder-rules.tsx             → P11 提醒规则说明页
├── login.tsx                       → P12 登录页
└── register.tsx                    → P13 注册页
```

### 3.2 入口与跳转关系（严格对应 spec §26）

```
推送通知点击 ────深链───→ /reminder-detail/[reminderId]    （变体 A 或 B）
提醒 Tab 点击提醒 ──────→ /reminder-detail/[reminderId]    （变体 A 或 B）
提醒 Tab 过期分区点击 ──→ 切换到历史 Tab + /history-detail/[historyId]
                          （底部 Tab 高亮同步变为「历史」，返回键回到历史列表页）
联系人 Tab 点击 ────────→ /contact-detail/[contactId]
联系人 Tab ＋ ──────────→ /contact-form/new
联系人详情页编辑 ───────→ /contact-form/[contactId]
历史 Tab 点击 ──────────→ /history-detail/[historyId]
各主页面右上角设置入口 → /settings（modal）
设置页 ──"登录 / 注册"──→ /login
/login ──"去注册"───────→ /register
/register ──"去登录"────→ /login
错过提醒弹窗点击 ──────→ 切换到提醒 Tab
AI 功能登录引导弹窗 ──→ /login
```

### 3.3 关键 UX 约束

- **过期提醒点击**：需同时做两件事——① 切换底部 Tab 到「历史」② 在历史 Tab 内导航到对应详情页。实现上需要从提醒 Tab 触发跨 Tab 导航，Expo Router 的 `router.navigate()` 配合 `/(tabs)/history` 路径可实现。
- **设置页**：用 `presentation: 'modal'` 呈现，保证"从哪进就从哪回"。
- **引导页**：根 `_layout.tsx` 读取 AsyncStorage 中的 `has_onboarded` 标记，决定初始路由。
- **登录页/注册页**：Stack 推入，登录成功后 `router.back()` 回到来源页；若是从 AI 功能登录引导弹窗进入的，登录成功后回到提醒详情页并可继续使用 AI 功能。
- **P14 登录引导弹窗**：不占路由，由提醒详情页（P5/P6）内部根据 AuthContext 状态条件渲染。

---

## 四、数据设计

### 4.1 本地 SQLite（核心数据——v1 主存储）

#### 表：contacts（联系人）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | TEXT (PK) | UUID |
| name | TEXT NOT NULL | 姓名（必填） |
| relationship | TEXT NOT NULL | 原始输入的关系（如"妈妈"，非归一化后的标准名） |
| relationship_normalized | TEXT \| NULL | 归一化后的标准关系（内部逻辑使用，UI 不展示） |
| traits | TEXT \| NULL | 特征描述（选填，可空字符串） |
| reminder_enabled | INTEGER DEFAULT 1 | 整个联系人的提醒开关（功能 6 局部关闭粒度一） |
| created_at | TEXT | ISO 时间戳 |
| updated_at | TEXT | ISO 时间戳 |

#### 表：days（日子）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | TEXT (PK) | UUID |
| contact_id | TEXT NOT NULL (FK→contacts) | 所属联系人 |
| name | TEXT NOT NULL | 日子名称（如"结婚纪念日""入职纪念日"） |
| type | TEXT NOT NULL | 日子类型：`birthday` / `anniversary` / `holiday_auto`（身份自动关联）/ `holiday_unlinked`（未关联联系人的节日） |
| date_month | INTEGER NOT NULL | 月份（1-12） |
| date_day | INTEGER NOT NULL | 日（1-31） |
| year | INTEGER \| NULL | 出生年份或纪念日年份（选填） |
| lunar | INTEGER DEFAULT 0 | 是否为农历（v1 仅通用节日的农历节日为 1，联系人日子为 0；通过 lunar-javascript 计算当年公历对应日期） |
| repeat_yearly | INTEGER DEFAULT 0 | 生日默认 1，纪念日默认 0，节日默认 1（功能 5、6） |
| reminder_enabled | INTEGER DEFAULT 1 | 单日提醒开关（功能 6 局部关闭粒度二） |
| expired | INTEGER DEFAULT 0 | 是否已过期（仅当 repeat_yearly=0 且日子已过后为 1） |
| source | TEXT NOT NULL | 来源：`manual` / `auto`（身份自动关联）（功能 2） |
| holiday_key | TEXT \| NULL | 节日键名，如 `mothers_day`（来源为 auto 或有对应的通用节日时） |
| created_at | TEXT | |
| updated_at | TEXT | |

#### 表：reminder_cache（提醒详情的生成内容缓存）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | TEXT (PK) | UUID |
| day_id | TEXT NOT NULL (FK→days) | 关联的日子 |
| reminder_year | INTEGER NOT NULL | 提醒所属年份（用于年度隔离，功能 10） |
| reminder_date | TEXT NOT NULL | 最新一次提醒日期（如"2026-03-05"） |
| selected_blessing_index | INTEGER \| NULL | 用户当前选定的祝福语索引（0-2），null 表示未选定 |
| blessings_json | TEXT \| NULL | 3 条祝福语 JSON（最后一次生成的） |
| gifts_json | TEXT \| NULL | 5 条礼物建议 JSON |
| gift_direction | TEXT \| NULL | 用户输入的礼物方向 |
| card_template | TEXT \| NULL | 贺卡使用的模板标识 |
| card_image_path | TEXT \| NULL | 贺卡导出图片本地路径 |
| card_signature | TEXT \| NULL | 当前落款文本（可临时修改） |
| created_at | TEXT | |
| updated_at | TEXT | |

> **缓存与 session 的关系**（功能 10）：同一联系人在同一提醒年度内的多次提醒共用同一个 reminder_cache 记录——只更新 `reminder_date`，祝福保留用户最后一次修改的版本。下一年度的提醒（repeat_yearly=1 的日子）新建一条记录。

#### 表：history（历史记录）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | TEXT (PK) | UUID |
| day_id | TEXT (FK→days) | 关联的日子（如果联系人被删除，day_id 对应的 days 记录可能不存在） |
| contact_name | TEXT NOT NULL | 写入时快照的联系人姓名（删除联系人后仍可显示） |
| contact_relationship | TEXT NOT NULL | 写入时快照的关系文本 |
| day_name | TEXT NOT NULL | 日子名称快照 |
| day_type | TEXT NOT NULL | 生日/纪念日/节日 |
| occurred_date | TEXT NOT NULL | 日子日期（如"2026-09-15"） |
| reminder_disabled | INTEGER DEFAULT 0 | 该日子在发生时提醒是否被手动关闭 |
| blessings_json | TEXT \| NULL | 祝福语（最终版本），提醒关闭时为空 |
| gifts_json | TEXT \| NULL | 礼物建议，提醒关闭时为空 |
| card_image_path | TEXT \| NULL | 贺卡图片路径，提醒关闭时为空 |
| created_at | TEXT | |

#### 表：history_blessing_versions（祝福历史版本）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | TEXT (PK) | UUID |
| history_id | TEXT NOT NULL (FK→history) | 关联历史记录 |
| version_index | INTEGER NOT NULL | 版本序号（从 1 起） |
| blessings_json | TEXT NOT NULL | 该版本的 3 条祝福语 |
| created_at | TEXT | |

#### 表：notification_missed（错过提醒记录）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | TEXT (PK) | UUID |
| day_id | TEXT (FK→days) | 关联的日子 |
| day_name | TEXT NOT NULL | 日子名称 |
| contact_name | TEXT NOT NULL | 联系人姓名 |
| scheduled_date | TEXT NOT NULL | 计划的提醒日期 |
| missed | INTEGER DEFAULT 0 | 是否错过（1=错过） |
| created_at | TEXT | |

### 4.2 本地 AsyncStorage（轻量 KV）

| Key | 值 | 说明 |
|---|---|---|
| `has_onboarded` | boolean | 是否已完成引导页（功能 19） |
| `last_reminder_scan` | ISO 时间戳 | 上次后台提醒扫描时间 |
| `missed_notification_summary` | JSON | 未读的错过提醒摘要（功能 9） |
| `supabase_session` | JSON | 缓存的 Supabase Auth session（"记住我"持久化登录态） |
| `user_nickname` | string \| null | 本地缓存的用户昵称（未登录时修改存于此，登录后同步到 profiles） |
| `advance_days` | number | 提醒提前天数本地缓存（默认 7） |
| `push_enabled` | boolean | 推送通知总开关本地缓存 |
| `holiday_general_enabled` | boolean | 通用节日提醒总开关 ① 本地缓存 |
| `holiday_unlinked_enabled` | boolean | 未关联联系人的节日提醒 ② 本地缓存 |
| `holiday_linked_enabled` | boolean | 已关联联系人的节日提醒 ③ 本地缓存 |
| `default_platform` | string | 默认电商平台本地缓存（默认 taobao） |

> **说明**：以上设置类 Key 在未登录状态下从 AsyncStorage 读写，登录后同步到 Supabase profiles 表。启动时优先读 AsyncStorage（本地即时），后台静默同步 Supabase。

### 4.3 Supabase（云端——v1 用于登录认证 + 未来云端存储预留）

#### 表：profiles（用户设置云端备份）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | uuid (PK) | 与 Supabase Auth uid 一致 |
| nickname | text \| null | 用户昵称/落款（功能 17），登录后从 AsyncStorage 同步 |
| advance_days | smallint | 提醒提前天数，默认 7，范围 1–60（功能 7） |
| push_enabled | boolean | 推送通知总开关，默认 true（功能 9） |
| holiday_general_enabled | boolean | 通用节日提醒总开关 ①，默认 true（功能 8） |
| holiday_unlinked_enabled | boolean | 未关联联系人的节日提醒 ②，默认 true（功能 8） |
| holiday_linked_enabled | boolean | 已关联联系人的节日提醒 ③，默认 true（功能 8） |
| default_platform | text | 默认电商平台，可选值：taobao / jd / pinduoduo / vip / amazon，默认 taobao（功能 14） |
| created_at | timestamptz | |
| updated_at | timestamptz | |

> **v1 策略**：profiles 表用于已登录用户的设置云端备份。用户在未登录状态下修改的设置仅存 AsyncStorage 本地；登录后触发一次 AsyncStorage → Supabase 同步；后续修改双写（AsyncStorage + Supabase）；退出登录后继续从 AsyncStorage 读写。

#### Supabase Auth

- 认证方式：邮箱 + 密码（Supabase Auth `email` provider）
- 注册：`supabase.auth.signUp({ email, password })`
- 登录：`supabase.auth.signInWithPassword({ email, password })`
- "记住我"：勾选后将 session 持久化到 AsyncStorage，启动时 `supabase.auth.getSession()` 恢复
- 退出登录：`supabase.auth.signOut()`，清除 AsyncStorage 中的 session，本地数据保留

#### Supabase Edge Functions

| 函数名 | 调用的 AI 服务 | 功能 | 对应 spec |
|---|---|---|---|
| `generate-blessings` | DeepSeek API | 输入：关系、特征描述、日子类型 → 输出：3 条祝福语（每条 ≤600 字） | §10, §11 |
| `generate-gifts` | DeepSeek API | 输入：特征描述、日子类型、关系类型、可选礼物方向 → 输出：5 条礼物建议 | §13 |
| `generate-greeting-card` | 阿里云百炼（生图模型） | 输入：称呼、日子类型、选定祝福语文本、模板类型 → 输出：贺卡图片（URL 或 base64） | §15 |

### 4.4 通用节日常量（TypeScript 硬编码 + lunar-javascript）

```typescript
// constants/holidays.ts
export interface Holiday {
  key: string;                    // 唯一键名，如 'mothers_day'
  name: string;                   // 节日名，如"母亲节"
  category: 'public' | 'traditional' | 'themed' | 'western'; // 对应 spec §8 分类
  dateType: 'solar' | 'relative' | 'lunar'; // 公历固定 / 相对日期 / 农历
  // 公历固定：month, day 直接取值
  // 相对日期：month, dayOfWeek, weekOfMonth（如母亲节 = 5月第2个周日）
  // 农历：month, day 为农历月日，通过 lunar-javascript 计算当年公历日期
  month?: number;                 // 公历固定：公历月；农历：农历月
  day?: number;                   // 公历固定：公历日；农历：农历日
  dayOfWeek?: number;             // 0=周日（仅相对日期）
  weekOfMonth?: number;           // 第几个（仅相对日期）
  linkedRelationships: string[];  // 自动关联的标准关系列表（功能 2 映射表）
}

// 18 个节日按 spec §8 清单硬编码
// lunar-javascript 用于计算当年农历节日对应的公历日期：
//   const lunar = Lunar.fromYmd(year, holiday.month, holiday.day);
//   const solarDate = lunar.getSolar(); // 得到公历年月日
```

---

## 五、实现阶段规划（按从简单到困难排序）

### 阶段 0：项目脚手架

- 初始化 Expo 项目（`npx create-expo-app@latest`）
- 安装依赖：expo-router, expo-sqlite, @react-native-async-storage/async-storage, expo-notifications, expo-task-manager, expo-sharing, expo-file-system, expo-media-library, lunar-javascript, @supabase/supabase-js
- 建立目录结构（app/、components/、constants/、contexts/、services/、hooks/、utils/、db/）
- 配置 ESLint + Prettier
- 初始化 SQLite 数据库：建表（contacts, days, reminder_cache, history, history_blessing_versions, notification_missed）

**依赖**：无

---

### 阶段 1：基础 UI 框架（低难度）

**范围**：底部 3 Tab 骨架、设置页壳、各占位页面

- 实现 `(tabs)/_layout.tsx` 底部 3 Tab（「提醒」「联系人」「历史」）
- 实现 P10 设置页壳（6 项设置项的 UI 骨架——含登录状态行）
- 实现 P1 引导页壳
- 实现 P11 提醒规则说明页（静态内容页）
- 设置页右上角入口（各主页面）

**依赖**：阶段 0

---

### 阶段 2：本地设置与偏好（低难度）

**范围**：设置页的全部本地设置功能（功能 7、8、9、14、17 的本地部分）

- 实现 SettingsContext（读写 AsyncStorage 中的设置项）
- P10 设置页全部非登录设置项：
  1. 我的昵称/落款（存入 AsyncStorage `user_nickname`；关系反推落款逻辑见功能 17）
  2. 提醒提前天数（1–60 滑块/输入，校验边界，存入 AsyncStorage `advance_days`）
  3. 推送通知总开关（存入 AsyncStorage `push_enabled`）
  4. 通用节日提醒开关（3 层联动：① 关闭时 ②③ 灰色不可操作；存入对应 AsyncStorage key）
  5. 默认电商平台（5 选 1 单选，存入 AsyncStorage `default_platform`）
- 设置项即时生效

**依赖**：阶段 1（设置页壳就绪）

---

### 阶段 3：联系人 CRUD（低难度）

**范围**：联系人的增删改查全流程（功能 1、3、4）

- 实现 contacts、days 表的 DAO（SQLite CRUD 封装）
- 实现 P8 联系人表单页（新增/编辑）
  - 必填校验（姓名、关系、生日）
  - 关系输入时实时提示归一化识别结果（如输入"妈咪"提示"识别为：母亲"）
  - 归一化对照表硬编码 + 匹配逻辑（10 组映射）
- 实现 P3 联系人 Tab 列表页（展示全部联系人：称呼、关系、最近日子、剩余天数）
- 实现 P7 联系人详情页
  - 展示关系、特征描述、日子列表（区分"手动设定"/"身份自动关联"标签）
  - 每个日子独立提醒开关
  - "＋ 添加纪念日"按钮
  - "删除联系人"按钮（含二次确认）
- 实现联系人删除逻辑（级联删除关联 days、保留 history）

**依赖**：阶段 1（需要联系人 Tab 和表单页的路由出口）

---

### 阶段 4：节日常量与日子管理（低-中难度）

**范围**：节日硬编码、农历计算、身份自动关联（功能 2、8 节日数据部分）

- 硬编码 18 个节日（constants/holidays.ts，按 spec §8 分类）
- 集成 lunar-javascript：
  - 农历节日（春节、元宵节、除夕、端午节、七夕节、中秋节、重阳节、清明节）→ 计算当年公历日期
  - 相对日期节日（母亲节 = 5 月第 2 个周日、父亲节 = 6 月第 3 个周日）→ 规则推算
  - 固定公历节日 → 直接取月日
- 实现身份自动关联逻辑：联系人关系归一化 → 匹配 holiday.linkedRelationships → 写入 days（source=auto, type=holiday_auto）
- 关系变更时触发自动关联日子重算（含弹窗询问：若之前手动删过同类日子，问是否重新添加）

**依赖**：阶段 3（需要 contacts 和 days 表就绪）

---

### 阶段 5：提醒列表（中难度）

**范围**：提醒 Tab 的完整列表逻辑（功能 5、6 列表呈现部分、8 混排部分、21）

- P2 提醒 Tab：
  - 查询所有 days（含未关联联系人的节日）+ 计算剩余天数 + 合并规则
  - 默认按剩余天数升序排序
  - 切换为按联系人排序（联系人各自成组，未关联联系人的节日单独"节日"分组在最后）
  - 生效中 / 已过期分区（expired=1 且 repeat_yearly=0 → 已过期，移到列表最下方分区）
  - "仅一次"的日子过期后移到过期分区 + 提醒开关永久关闭
  - 被局部关闭提醒的条目灰显 + 关闭标记
  - 空态引导（未添加联系人 → 引导去添加联系人）
- 合并规则：同一节日的纯节日提醒 + 关联联系人提醒合并为一条

**依赖**：阶段 3（需要联系人数据）、阶段 4（需要节日 days）、阶段 2（需要提前天数设置）

---

### 阶段 6：推送通知与提醒调度（中-高难度）

**范围**：本地定时通知 + 提醒节奏逻辑（功能 6、7、8 推送部分、9）

- 后台定时扫描（BackgroundFetch，建议间隔 1 小时）
- 提醒节奏计算引擎：
  - 提前天数（用户设置，1–60）→ 决定提醒窗口起点
  - ≥7 天：每周一次，触发日为与日子当天相同的星期几
  - 4~6 天：静默，不提醒
  - ≤3 天：每天一次（第 3、2、1 天及当天）
  - 提前天数 < 7 天时：从窗口第一天起每天提醒
  - 2 月 29 日非闰年处理：2 月 28 日提醒
- 调度 expo-notifications 本地通知
- 推送通知总开关控制（关闭后取消所有已调度通知）
- 通用节日提醒 3 层开关联动
- 恢复推送后的"错过提醒"弹窗 + 摘要
- notification_missed 表记录
- 推送点击深链 → 提醒详情页

**依赖**：阶段 5（需要提醒列表和日子数据全量就绪）

---

### 阶段 7：Supabase 初始化 + 用户登录

**范围**：Supabase 项目创建、Auth 集成、登录/注册页面（功能 27、28、29、30）

- 创建 Supabase 项目、获取 API Key、配置客户端
- 建 profiles 表（SQL 迁移脚本）
- 实现 AuthContext：
  - `signUp(email, password)` → Supabase Auth 注册
  - `signIn(email, password, rememberMe)` → Supabase Auth 登录
  - `signOut()` → 清除 session，本地数据保留
  - `getSession()` → 启动时恢复登录态
  - 暴露 `isLoggedIn`, `userEmail` 给全局
- 实现 P12 登录页：邮箱 + 密码 + "记住我"复选框 + "登录"按钮 + "去注册"链接
- 实现 P13 注册页：邮箱 + 密码（≥8 位）+ 确认密码 + "注册"按钮 + "去登录"链接
- 注册成功后自动登录
- 设置页（P10）集成登录状态行：
  - 未登录 → 显示"登录 / 注册"行，点击 → /login
  - 已登录 → 显示邮箱 + "退出登录"行
- 实现 P14 登录引导弹窗（未登录触发 AI 功能时弹出 + "去登录"按钮）
- 登录后 AsyncStorage 设置项 → Supabase profiles 同步逻辑

**依赖**：阶段 2（设置页需要登录状态行）、阶段 5（提醒详情页需要登录引导弹窗）

> **为什么放这里**：登录功能与阶段 8-10 的 AI 功能紧密耦合（AI 功能需登录），在 AI 阶段之前完成登录即可。与 Supabase 初始化放在一起，一次配置完整个 Supabase 栈。

---

### 阶段 8：AI 祝福生成（中难度）

**范围**：Supabase Edge Function（DeepSeek）+ 客户端调用 + 缓存逻辑（功能 10、11、12）

- 编写 `generate-blessings` Edge Function（Deno + TypeScript）
  - 调用 DeepSeek API（chat/completions）
  - 输入：关系类型、特征描述、日子类型
  - 输出：3 条祝福语（每条 ≤600 字，纯文本）
  - 无特征描述时按关系类型生成
- 客户端 service 层封装调用 + 错误处理 + 登录态前置检查
- P5 变体 A 详情页祝福区：
  - 进入页面实时生成 / 缓存命中则直接展示（需登录）
  - 祝福语展示 + 选中态交互（首次进入 3 条均未选中；点击文本=选中，高亮；始终至少一条选中）
  - 每条旁复制 icon（独立于选中操作）
  - "重新生成"按钮（全量替换，不限次数）
  - 生成失败保留旧内容 + 提示 + 重试入口
  - 断网提示
- 缓存持久化到 reminder_cache 表
- 年度 session 隔离逻辑
- 未登录时触发 → 弹出 P14 登录引导弹窗

**依赖**：阶段 5（需要提醒详情页路由 + 提醒数据）、阶段 7（需要登录态 + Supabase 就绪）

---

### 阶段 9：AI 礼物推荐（中难度）

**范围**：Edge Function（DeepSeek）+ 客户端（功能 13、14）

- 编写 `generate-gifts` Edge Function
  - 调用 DeepSeek API
  - 输入：特征描述、日子类型、关系类型、可选"礼物方向"
  - 输出：5 条礼物建议
- P5 变体 A 详情页礼物区：
  - 5 条礼物建议展示
  - "礼物方向"输入框 + 生成按钮（输入方向后整批替换）
  - 每条"去看看"按钮 → 跳转电商平台搜索
- 电商跳转逻辑：
  - 读取 AsyncStorage `default_platform` → 优先对应 App deep link，回退网页 URL
  - 搜索关键词：有特征描述 → "礼物名称 + 特征描述关键词"；无特征描述 → 仅礼物名称
- 未登录时触发 → 弹出 P14 登录引导弹窗

**依赖**：阶段 8（共享同一提醒详情页、同一缓存表、同一 Edge Function 代理模式）

---

### 阶段 10：电子贺卡生成与导出（中-高难度）

**范围**：Edge Function（阿里云百炼生图）+ 模板匹配 + 图片导出（功能 15、16）

- 编写 `generate-greeting-card` Edge Function
  - 调用阿里云百炼平台生图模型 API
  - 输入：称呼/姓名（只显示一个，优先称呼）、日子类型、选定祝福语文本、模板类型
  - 输出：贺卡图片（URL 或 base64）
- 模板规则：
  - 生日 → 生日模板（自动匹配，1 种）
  - 节日 → 3 张模板（通用节日 1 + 过年 1 [圣诞/春节] + 情人节 1 [情人节/七夕]），按类型自动匹配
  - 纪念日 → 3 种预设模板（2 通用 + 1 结婚/恋爱纪念日），关键词匹配自动推荐，否则默认通用
- 纪念日模板切换（仅纪念日可切换，每次切换需点"重新生成贺卡"）
- P5 变体 A 贺卡区：
  - 未选定祝福语时不显示贺卡，显示引导文字
  - "生成贺卡"按钮 → 首次生成（需登录）
  - 选定祝福语后点"生成贺卡"才生成
  - 换祝福语或换模板后需点"重新生成贺卡"
  - 落款显示（昵称 > 关系反推 > 空）且可临时修改
- P6 变体 B 贺卡：
  - "生成节日贺卡"按钮 → 用户点击后才生成（无需选文案）
  - 生成后缓存 + "重新生成"按钮
- 贺卡导出为图片（expo-media-library 保存到相册 + expo-sharing 分享）
- 生成失败提示 + 不清空已选祝福语
- 未登录时触发 → 弹出 P14 登录引导弹窗

**依赖**：阶段 8（需要祝福选定逻辑）、阶段 7（需要登录态）

---

### 阶段 11：历史记录（中难度）

**范围**：历史写入、查看、版本管理（功能 5 后半部分、18、23）

- 历史写入逻辑（日子次日 App 启动时扫描写入）：
  - 扫描昨天及以前到期但未写入历史的 days
  - 写入 history 表（快照联系人名、关系、日子名等）
  - 写入 history_blessing_versions 表（所有重新生成的旧版本）
  - 被关闭提醒的日子写入后祝福/礼物/贺卡为空，展示默认文案："仪式就是使某一天与其他日子不同，使某一时刻与其他时刻不同。"
- P4 历史 Tab 列表页：已过提醒日倒序列表
- P9 历史详情页：
  - 展示祝福/礼物/贺卡
  - 版本切换（下拉或按钮切换不同重新生成版本）
  - 删除功能（仅删除历史记录，不影响提醒列表）
- 过期提醒点击跨 Tab 导航（提醒 Tab 过期分区 → 切换到历史 Tab + 打开对应详情页）

**依赖**：阶段 5（需要提醒列表的过期分区）、阶段 8-10（需要生成内容缓存）

---

### 阶段 12：引导页与首次启动（低难度）

**范围**：引导页完整实现（功能 19）

- P1 引导页 UI（3-4 页滑动引导，展示核心功能："填人→等提醒→收祝福/挑礼物/发贺卡"）
- AsyncStorage `has_onboarded` 标记
- 根布局初始路由判断
- 引导页完成后进入主界面
- 再次启动不再展示

**依赖**：阶段 1（已有引导页壳）

---

### 阶段 13：联调、边界情况与自检（收尾）

- 所有跳转关系验证（spec §26 逐条核对，含登录/注册新路径）
- 断网场景全覆盖（祝福区、礼物区、贺卡区各自的降级 UI）
- 空态场景（无联系人、无提醒、无历史、无网络）
- 登录态边界：
  - 未登录 → 本地功能正常，AI 功能引导登录
  - 已登录 → 全功能可用
  - 退出登录 → 本地数据完整，AI 功能回到引导态
  - 登录过期 → session 恢复失败时的降级处理
- 深链处理（推送点击 → 各变体详情页）
- 各类命名、字段名、文案与 spec 逐条核对一致性
- 农历节日日期准确性验证（lunar-javascript 覆盖 2025-2030 年）
- 性能检查（大联系人列表、多年历史记录）

---

## 六、依赖关系图

```
阶段 0: 脚手架
  ↓
阶段 1: 基础 UI 框架
  ↓
阶段 2: 本地设置与偏好 ──────────────────────────────┐
  ↓                                                  │
阶段 3: 联系人 CRUD ──→ 阶段 4: 节日常量与日子管理    │
  ↓                        ↓                         │
  └────────────────→ 阶段 5: 提醒列表 ←──────────────┘
                            ↓
                    阶段 6: 推送通知与提醒调度
                            ↓
                    阶段 7: Supabase 初始化 + 用户登录
                            ↓
                    阶段 8: AI 祝福生成（DeepSeek）
                            ↓
                    阶段 9: AI 礼物推荐（DeepSeek）
                            ↓
                    阶段 10: 电子贺卡生成（阿里云百炼）
                            ↓
                    阶段 11: 历史记录

阶段 12: 引导页 ←── (可与阶段 2-11 并行，最后集成)

                    阶段 13: 联调与收尾
```

---

## 七、与 spec 一致性检查清单

在实现各阶段时，以下命名/数字/文案必须与 spec 完全一致：

| 项目 | spec 原文 | 不允许的改写 |
|---|---|---|
| 底部 3 个 Tab 名 | 「提醒」「联系人」「历史」 | 不能改为"首页""通讯录""记录"等 |
| 祝福语条数 | 3 条 | 不能改为 5 条或可配置 |
| 礼物建议条数 | 5 条 | 不能改为 3 条或可配置 |
| 祝福语字数上限 | 不超过 600 字 | 不能改为 500 字或 800 字 |
| 提醒提前天数范围 | 1–60 天 | 不能改为 1–30 或 1–90 |
| 提醒节奏 | ≥7 天每周一次、4-6 天静默、≤3 天每天一次 | 不能增减或修改分段逻辑 |
| 关系字段 | 自由文本，用户原始输入（UI 展示原始输入） | 不能改为下拉选择或展示归一化名 |
| 日子模型 | 一对多，无数量上限 | 不能限制每个联系人最多 N 个日子 |
| 节日数量 | 18 个 | 不能增减节日的数量 |
| 默认电商平台 | 淘宝 | 不能改为京东或其他 |
| 可选电商平台 | 淘宝、京东、拼多多、唯品会、亚马逊 | 不能增删改平台 |
| 纪念日模板数量 | 3 种预设模板（2 通用 + 1 结婚/恋爱） | 不能改为 2 种或 4 种 |
| 引导页触发条件 | 仅首次启动 | 不能每次启动都展示 |
| 贺卡抬头 | 称呼优先于姓名，只显示一个 | 不能同时显示称呼和姓名 |
| 落款反推规则 | 母亲/父亲→"你的孩子"等 | 不能修改反推文案 |
| 历史空态文案 | "仪式就是使某一天与其他日子不同，使某一时刻与其他时刻不同。" | 不能修改此文案 |
| "仅一次"纪念日到期后 | 次日移到"已过期"分区 | 不能当天立即移动 |
| 同一节日合并规则 | 仅同一节日的纯节日提醒+关联联系人提醒合并 | 不能合并不同日子或不同节日的提醒 |
| 关系归一化对照表 | 10 组映射 | 不能增减映射组 |
| 自动关联节日映射表 | 10 种关系对应规则 | 不能修改母子/母女/公婆/岳父母的节日映射 |
| 儿子/女儿不自动关联儿童节 | 明确不关联 | 不能擅自加上 |
| 密码最少位数 | 不少于 8 位 | 不能改为 6 位或不限制 |
| 登录引导弹窗文案 | "AI 生成功能需要登录后才能使用" | 不能修改提示文案 |
| 退出登录后本地数据 | 不清除 | 不能退出登录时清空联系人/历史等本地数据 |
| 未登录可用功能 | 联系人管理、提醒列表、历史记录、推送通知、引导页、设置页本地设置 | 不能额外限制或放宽 |
| 需登录功能 | AI 祝福生成、AI 礼物推荐、电子贺卡生成、贺卡导出为图片 | 不能在未登录时开放