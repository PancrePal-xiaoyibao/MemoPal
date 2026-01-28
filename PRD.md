# 📝 第一部分：产品需求文档 (PRD) - v2.0

## 1. 项目定位

*   **项目名称**：小忆宝 (MemoPal / MemoryPal)
寓意：延续小胰宝、小胃宝、小馨宝的"宝"字辈，"忆"代表记录与回忆，谐音"益"（公益）。
定位：这是志愿者档案与贡献的"记忆库"，既存放数据（人口普查），也存放情感（故事瞬间）。
*   **H5活动页面名称**：微光年志：2025守护者手札
*   **双重属性**：
    1.  **情感属性**：收集感人瞬间，生成纪念卡片（对外）。
    2.  **管理属性**：作为年度志愿者“人口普查”工具，更新志愿者花名册（对内）。

## 2. 核心功能流程 (H5)

### 2.1 登录/鉴权
*   **微信环境**：静默获取 OpenID（或开发环境模拟登录）。
*   **关联**：将 OpenID 与 Supabase `profiles` 表关联，确保一人一档。

### 2.2 数据采集页 (核心交互)
页面采用**长滚动 (Long Scroll)** 形式，分为四个清晰的模块。

#### 📌 模块 A：归属感 (Identity)
*   **标题**：“我是小X宝社区的...”
*   **交互**：标签矩阵 (Grid Layout)，单选，选中高亮。
*   **选项配置**：
    1.  🔵 小胰宝 (胰腺癌组)
    2.  🟢 小胃宝 (胃癌组)
    3.  💗 小馨宝 (心理支持组)
    4.  ⚪️ 小肺宝 (肺癌组)
    5.  🟣 小粉宝/小妍宝 (乳腺癌组)
    6.  🟤 罕见病/慢病组
    7.  🛠️ 技术/RAG研发组
    8.  ✨ 其他/通用志愿者
*   **补充输入**：每个选项下方或选中后，出现一个**文字输入框**。
    *   *Placeholder*：“具体在哪个小组？（例如：内容组/数据清洗组）”

#### 📌 模块 B：故事类型 (Category)
*   **标题**：“我想记录一个...”
*   **交互**：带图标的情感化标签，单选。
*   **选项配置**：
    1.  🌟 高光时刻 (确诊/康复好消息)
    2.  🌙 深夜守候 (凌晨回复/安抚)
    3.  💡 硬核科普 (读懂指南/报告)
    4.  🤝 温暖传递 (病友互助)
    5.  🚩 幕后攻坚 (整理数据/熬夜)
*   **补充输入**：选中后出现**文字输入框**。
    *   *Placeholder*：“一句话描述这个瞬间（作为卡片副标题）”

#### 📌 模块 C：多模态内容 (Content)
*   **图片**：上传截图/照片 (Max 3)，支持预览与删除。
*   **语音**：大麦克风图标。
    *   *交互*：按住录音 -> 松开结束 -> 显示波形条与时长 -> (后台异步转文字)。
*   **文字**：多行文本域 (Textarea)。
    *   *提示*：“展开讲讲那天发生了什么...”

#### 📌 模块 D：署名与提交 (Signature)
*   **昵称**：用于生成海报。
*   **微信号/手机号** (可选/隐藏)：用于志愿者管理库更新（仅后台可见）。

### 2.3 结果反馈
*   生成“年度守护者”海报，包含：
    *   背景色根据【模块 A】的选择变化（如小胰宝=蓝，小粉宝=粉）。
    *   展示【模块 B】选中的关键词。
    *   展示上传的图片或文字摘要。

---

# 💻 第二部分：Supabase 数据库设计 (Schema)

请直接在 Supabase SQL Editor 中运行此脚本：

```sql
-- 1. 启用必要的扩展
create extension if not exists "uuid-ossp";

-- 2. 志愿者档案表 (用于管理/普查)
create table public.profiles (
  id uuid references auth.users not null primary key,
  nickname text,
  real_contact text, -- 微信号或手机号，用于管理
  avatar_url text,
  created_at timestamptz default now()
);

-- 3. 故事/贡献表
create table public.stories (
  id uuid default gen_random_uuid() primary key,
  user_id uuid references public.profiles(id) not null,
  
  -- 模块 A: 归属
  group_tag text not null, -- 存 'XiaoYiBao', 'TechGroup' 等 Key
  group_detail text,       -- 用户手填的具体小组信息
  
  -- 模块 B: 类型
  story_category text not null, -- 存 'Highlight', 'LateNight' 等 Key
  category_detail text,         -- 用户手填的一句话描述
  
  -- 模块 C: 内容
  content_text text,
  image_urls text[],       -- 图片数组
  voice_url text,          -- 语音文件地址
  voice_transcription text,-- 语音转文字结果
  
  is_public boolean default true,
  created_at timestamptz default now()
);

-- 4. 存储桶策略 (Storage)
insert into storage.buckets (id, name, public) values ('uploads', 'uploads', true);

-- 5. 开放读取权限给中台 (RLS)
alter table public.stories enable row level security;
create policy "Users can insert own story" on public.stories for insert with check (auth.uid() = user_id);
create policy "Public/Admin can view all" on public.stories for select using (true);
```

---

# 🚀 第三部分：给 AI 编程工具的终极指令 (The God Prompt)

**请将以下内容复制到 Cursor Composer / Windsurf / Claude Code：**

> **Role:** You are a Senior Full-Stack Developer specializing in **Next.js 14**, **Supabase**, and **Mobile UI UX**.
>
> **Task:** Build a "Volunteer Story Collection & Census App" optimized for WeChat Mobile Browser.
>
> **Tech Stack Requirements:**
> 1.  **Framework:** Next.js 14 (App Router) + Tailwind CSS + Lucide React (Icons).
> 2.  **Backend:** Supabase (Auth, Database, Storage).
> 3.  **UI Library:** Shadcn/UI (specifically Card, Form, Button, Textarea, RadioGroup).
> 4.  **Styling:** Mobile-first, "Long Scroll" layout. Soft, emotional color palette.
>
> **Implementation Steps:**
>
> **Step 1: Setup & Configuration**
> *   Initialize the project.
> *   Assume `supabase/schema.sql` is already applied (I have created `profiles` and `stories` tables).
> *   Setup a "Mock Login" button for dev (simulating WeChat Auth) that creates a user in Supabase Auth.
>
> **Step 2: The Core Input Page (Strictly follow this layout)**
> Create a page `app/page.tsx` with a form containing these 4 specific modules:
>
> *   **Module A: Identity (Selection + Input)**
>     *   UI: A grid of selectable cards/chips.
>     *   Options:
>         *   🔵 小胰宝 (Pancreatic Cancer Group)
>         *   🟢 小胃宝 (Gastric Cancer Group)
>         *   💗 小馨宝 (Psych Support)
>         *   ⚪️ 小肺宝 (Lung Cancer Group)
>         *   🟣 小粉宝/小妍宝 (Breast Cancer Group)
>         *   🟤 罕见病/慢病组 (Rare Disease)
>         *   🛠️ 技术/RAG研发组 (Tech/RAG)
>         *   ✨ 其他/通用志愿者 (General)
>     *   *Crucial Logic:* When an option is selected, show a small text input below it with placeholder: "Specifically which team? (e.g., Data Cleaning)". Save this to `group_detail`.
>
> *   **Module B: Story Type (Selection + Input)**
>     *   UI: A row of emotional icons/tags.
>     *   Options:
>         *   🌟 高光时刻 (Highlight)
>         *   🌙 深夜守候 (Late Night)
>         *   💡 硬核科普 (Hardcore Sci)
>         *   🤝 温暖传递 (Warmth)
>         *   🚩 幕后攻坚 (Behind Scenes)
>     *   *Crucial Logic:* When selected, show a text input: "One sentence summary". Save this to `category_detail`.
>
> *   **Module C: Content (Multi-modal)**
>     *   **Text:** A large textarea.
>     *   **Images:** A button to upload images (upload to Supabase bucket 'uploads'). Display thumbnails.
>     *   **Voice:** A "Hold to Record" button. Use HTML5 `MediaRecorder` API. Save the blob to Supabase Storage as `.webm`. Display an audio player after recording.
>
> *   **Module D: Signature**
>     *   Input: Nickname (Required).
>     *   Input: WeChat ID/Phone (Optional, for admin use).
>
> **Step 3: Submission & Feedback**
> *   On submit: Save data to `stories` table.
> *   Show a "Success Card" modal. The card background color should change based on Module A selection (e.g., Blue for XiaoYiBao, Pink for XiaoFenBao).
>
> **Step 4: Admin API**
> *   Create a route `app/api/admin/stories/route.ts` that fetches all stories from Supabase. This will be used by the external admin dashboard.
>
> **Tone & Quality:**
> *   The UI must look polished and friendly (rounded corners, soft shadows).
> *   Code must be strictly typed (TypeScript).
> *   Handle loading states (e.g., while uploading images).
>
> **Action:** Start by generating the project structure and the main Form component. Do not wait for confirmation.

---

### 💡 针对“中台接入”的补充说明

一旦项目生成完毕，您只需将以下信息提供给负责中台开发的志愿者：

1.  **API 端点**: `https://[您的项目域名]/api/admin/stories`
2.  **数据格式**: 接口会返回包含 `group_detail` (具体小组) 和 `real_contact` (联系方式) 的完整 JSON 数据，完美契合您“清点管理志愿者”的需求。
