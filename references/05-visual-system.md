# 05 视觉系统：头像与背景图的生成规范（v2）

> **v2 修订原因**：v1 的 prompt 把"暗场"写成了"整体压黑"，生成结果大面积是**只剩轮廓的黑影**——主体看不清材质、背景和主体糊成一块。这种图在信息流里缩到 48px 就是一块黑方块，等于没有头像。
>
> **根因**：v1 同时用了 `deep black palette` + `dramatic chiaroscuro` + `half swallowed by shadow` + `near-black background` 四个压黑词，模型没有收到任何一个"照亮主体"的指令。
>
> **v2 铁律一句话**：**暗的是背景，不是主体。** 主体必须比背景亮，且亮到能看清材质。

---

## v2.1 出图执行协议（先看这一节）

**图必须由你生成，不能只写 prompt。** 用户要头像或背景图，你就要调用生图工具把文件做出来。

| 步骤 | 动作 | 关键点 |
|---|---|---|
| 1 告知 | 一句话说明"会消耗生图额度，先出 1 张" | 不解释原理，不长篇 |
| 2 拼 prompt | 固定风格锚 + 主体 + 三段光 + 背景 + 构图 + 曝光锁 + 质量 | 风格锚整句照抄，保证矩阵统一 |
| 3 调工具 | 生图工具（ImageGen），头像 `1024x1024` | 头像必须正方形；背景图先要 3:2 横构图 |
| 4 后处理 | 头像重采样到 512×512 PNG；背景图裁切到平台尺寸 | 用本地脚本（PIL），不要交给模型 |
| 5 叠金句 | **每张背景图必须叠一句 ≤12 字金句**（PIL + `msyhbd.ttc`） | 不许交空白底图；金句不与标题、简介重复 |
| 6 自检 | 曝光五条 + 48px 缩略图测试 | 不合格改 prompt 重出 1 次 |
| 7 交付 | 给出图片文件 + 绝对路径 + 尺寸 | 图片要让用户直接看到 |

**硬性禁止**

- **头像出现物品当主角**（兽首、棋子、刀剑单独陈列、空盔甲、面具浮空、静物、logo、文字）——头像是人像，必须看到一个具体的人
- 只输出英文 prompt / 设计说明，让用户自己去找工具跑
- 用文字描述"这张图大概是什么样"代替真实图片
- 一次连出 3 张以上试错（浪费额度，先 1 张）
- 声称已生成但实际没有文件

**保存位置**：当前会话工作目录 `henren-assets/`，命名 `avatar-01.png` / `avatar-02.png` / `banner-xhs-01.png` / `banner-dy-01.png` / `banner-sph-01.png`。

**尺寸不可选时的降级**：若工具只出正方形，背景图按 1024×1024 生成（关键元素全放中央 60%），再裁成 1125×633 一类的横条；裁切后主体仍须完整，不得把头或符号切掉。

---

## 零、曝光底线（最高优先级，先于任何风格词）

生成前先记住这五条。任何一条不满足，无论风格多准都要重做。

| # | 指标 | 标准 | 判断方法 |
|---|---|---|---|
| 1 | 主体亮度 | 主体区域灰度均值 **85–140**（0–255） | 眯眼看：盔甲/衣料/皮肤的纹理是否可辨 |
| 2 | 背景亮度 | **25–60**，且比主体暗 **≥35** | 背景要有层次（渐变/雾），不是死黑 |
| 3 | 必须有高光 | 画面存在至少一处 **≥200** 的亮部（金属反光、眼缝光、轮廓光） | 缩略图里的"亮点"就是视觉抓手 |
| 4 | 主体占比 | 画面宽度的 **45–65%**，头顶留白 ≤12% | 主体太小 = 缩略图里是个点 |
| 5 | 缩略图测试 | 缩到 **48×48** 仍能分辨主体和焦点 | 手机端真实尺寸 |

**三种必须重做的情况**

- 整张图只有轮廓、没有材质（全黑剪影）
- 主体与背景对比度 < 35，边界糊掉
- 暗角/黑边占画面 >15%，把主体边缘吃掉

### 一个 10 秒自检

把图缩成 120px 宽，放到手机锁屏上看：**能不能一眼说出"这是一个戴盔甲的人 / 一张侧脸 / 一个兜帽人"？** 说不出就是太暗，或者主体根本不是人。

---

## 一、光的三段（解决"暗但没细节"）

暗调不等于不布光。恰恰相反，暗调头像比明亮头像更依赖布光，因为背景不提供任何信息，全靠光把主体"雕"出来。

| 光位 | 作用 | 写法（prompt 用） |
|---|---|---|
| **主光 key** | 照亮主体材质，是唯一的信息来源 | `strong key light from front-left, 35° above` |
| **辅光 fill** | 把暗部提起来，保证不死黑 | `soft fill light at 30% intensity, shadow detail retained` |
| **轮廓光 rim** | 勾边界，把主体从背景里剥出来 | `crisp rim light along the shoulder and helmet edge` |
| **背景光** | 只做氛围，亮度必须低于主体 | `dim crimson glow behind, never brighter than the subject` |

**口诀**：主体亮 → 暗部有细节 → 边缘有光 → 背景最暗。

---

## 二、色彩规范

| 用途 | 色值 | 占画面比例（v2 调整） |
|---|---|---|
| 主体金属/衣料 | `#2A2A2E` → `#6E6E74` 灰阶 | **40–50%**（靠光打亮，不靠调色） |
| 背景暗场 | `#141416`（下限，不许更黑） | **30–40%** |
| 血红 | `#8C1C13` | 5–10%，唯一"活气" |
| 暗金 | `#C9A227` | 10%，金属高光 |
| 骨白 | `#E8E2D6` | 高光与文字 |

**v1 的错误**：主暗场写"占画面 60–70%"，实际执行时模型把主体也一起算了进去。

**禁止**：纯白背景、糖果色渐变、多色霓虹、马卡龙色、ins 风冷调过曝，以及**整体压黑的剪影图**。

---

## 三、头像生成（1:1）

规格：ImageGen `size = "1024x1024"`，交付时压缩为 512×512、PNG、≤500KB。

### 硬规则：头像里必须有一个"人"

这是最高优先级，先于一切风格词。**画面主体只能是一个人的半身像或胸像**，头、颈、肩至少能看到两处，身上要有可辨认的衣料或护甲。

- **允许**：穿盔甲的人、戴兜帽的人、侧脸的人、背影的人、一张脸加一只眼。
- **禁止**：兽首/鹰/狼/狮、棋子、刀剑单独陈列、空盔甲、浮空面具、静物、建筑、logo、纯图形、文字排版、风景。
- **符号只能由这个人佩戴或手持**（王冠戴在头上、刀握在手里），不能自己占满画面。
- 生成后第一眼判断：**能不能看出这是一个人？** 看不出就重出，不论多好看。

### Prompt 组装公式

```
[风格锚] + [人：谁的半身/胸像 + 头颈肩可见] + [材质可见] + [三段光效] + [背景（明确亮度）] + [构图] + [曝光锁定] + [质量]
```

**固定风格锚 v2**（每次都用同一句，保证矩阵画风统一）：

```
Cinematic low-key portrait, hyper-realistic, strong key light sculpting the subject from front-left,
soft fill keeping shadow detail readable, crisp rim light separating subject from background,
film grain, charcoal and crimson palette, subject clearly lit and fully visible,
```

与 v1 的差异（不要改回去）：

| v1 | 问题 | v2 |
|---|---|---|
| `dramatic chiaroscuro lighting` 单独出现 | "明暗对照"被理解成 80% 压黑 | 改成具体的 key/fill/rim 三段光 |
| `deep black and crimson palette` | 直接把整图拉到接近纯黑 | `charcoal and crimson palette`（炭灰，不是纯黑） |
| 无 | 缺少曝光指令 | `subject clearly lit and fully visible` |

### 方向 1：黑金骑士（默认，最通用）

```
Cinematic low-key portrait, hyper-realistic,
strong key light sculpting the subject from front-left at 35 degrees, soft fill light at 30% intensity keeping shadow detail readable, crisp rim light along the helmet and shoulder edge, film grain, charcoal and crimson palette, subject clearly lit and fully visible,
a broad-shouldered man wearing matte black plate armor with engraved antique-gold filigree, his neck, shoulders and chest clearly visible inside the armor, the armor worn on his body and not empty, armor surface showing scratches, brushed steel highlights and gold inlay clearly readable,
a jagged black crown fused into his closed helmet, a narrow slit visor emitting a blood-red glow as the brightest point in the frame,
tattered crimson cape hanging from his shoulder, three-quarter bust view of a human figure, torso filling 55% of the frame width, head centered with minimal headroom,
background: dark charcoal red radial gradient, mid-tone smoke, background at least two stops darker than the armor, never pure black,
extreme detail on metal texture, one single human subject, no empty suit of armor, no objects as the subject, no text, no logo, no watermark, sharp focus, professional studio lighting, high quality
```

**关键改动**：`a broad-shouldered man wearing…his neck, shoulders and chest clearly visible inside the armor / not empty`——把"这是一个人穿着它"写死，否则模型很容易只画一套空盔甲摆在画面中央。

### 方向 2：冷光侧脸（做真人 IP 用）

```
Cinematic low-key portrait, hyper-realistic,
key light from the left raking across the face at 40 degrees, fill light lifting the shadow side so skin texture and bone structure remain readable, rim light on the hair and jaw, film grain, charcoal and crimson palette,
a man in his thirties in a black high-collar coat, three-quarter profile, eyes in soft shadow but the cheekbone, jawline, brow and nose bridge clearly lit and defined,
skin pores and fabric weave visible, cape/coat collar catching a thin gold edge,
dark charcoal background with one dim crimson source behind, background not brighter than the face, no pure black areas,
bust shot, subject filling 55% of frame width, minimal, restrained, no logo, no text,
sharp focus, professional studio lighting, high quality
```

**v1 的错误**：`face half swallowed by shadow, eyes unseen, only the jawline and brow ridge caught by cold rim light` —— 三个"看不见"叠在一起，模型只画了个黑影。v2 把"eyes in soft shadow"降到"柔阴影"，其余五官必须"clearly lit and defined"。

### 方向 3：兜帽人物（人性 / 识人 / 关系）

> 原来这一格是"兽首图腾"，属于物品类头像，与"头像必须是个人"的硬规则冲突，已替换为人物版。

```
Cinematic low-key portrait, hyper-realistic,
key light from front-right at 35 degrees carving the jaw and chin, soft fill keeping the hood fabric readable, thin rim light along the hood edge and shoulder, film grain, charcoal and crimson palette,
a man wearing a heavy dark charcoal hood pulled low, the brim casting a soft shadow over his upper face so only the nose tip, mouth and jawline are clearly lit, hood fabric weave and worn seams visible, black high-collar coat underneath, his shoulders and chest filling the lower part of the frame,
background: deep charcoal with one dim crimson glow behind him, darker than his face, no pure black,
bust shot of a single human figure, subject filling 55% of frame width, no objects as the subject, no text, no logo,
sharp focus, professional studio lighting, high quality
```

**为什么换成人物**：兽首、棋子、刀这类图腾在 48px 下确实醒目，但观众看不出"这是谁在说话"，而且和"人"的账号人格脱节。人物像同样能靠遮脸做到去人格化，还多一层"这个人真的存在"的可信度。

### 头像验收标准

- **必须是一个人**：一眼能看出头、颈、肩，身上有衣料或护甲。只看到物品、兽首、空盔甲、棋子，直接报废重出
- **48×48 缩小时主体轮廓与焦点仍可分辨**（第二硬的一条）
- 主体灰度均值 85–140，背景 25–60，对比 ≥35
- 画面里只有**一个**视觉焦点，且这个焦点是画面最亮处
- 没有文字（有字在缩小后必糊）
- 暗部不死黑（保留 5% 细节），但**主体不许是纯黑剪影**
- 与背景图同色系

---

## 四、负向词（补充手段，不是主力）

正向描述锁曝光永远比负向词有效。但下面这些词如果出现在你的 prompt 里，**直接删掉**——它们是压黑元凶：

| 危险词 | 替换成 |
|---|---|
| `deep black palette` / `pure black background` | `charcoal palette` / `deep charcoal background` |
| `dramatic chiaroscuro`（单独用） | 具体的 key + fill + rim 三段描述 |
| `half swallowed by shadow` | `in soft shadow` |
| `only rim light, eyes unseen` | `rim light plus frontal fill, features readable` |
| `near-black background` | `dark charcoal background, two stops below subject` |
| `emerging from darkness` / `hidden in shadow` | `lit against a dark backdrop` |
| `silhouette` / `faceless figure in shadow` | `armored figure, material fully visible` |
| `still life` / `product shot` / `close-up of an object` | `bust portrait of a man, shoulders visible` |
| `animal head emblem` / `chess piece` / `empty suit of armor` | 直接删掉，换成人物描述 |
| `heavy vignette` / `dark overlay` | `subtle edge falloff` |

若要用负向描述，统一放在 prompt 末尾：

```
avoid: silhouette, underexposed, crushed blacks, featureless shadow, barely visible subject, pure black background, heavy vignette, low contrast, still life, product shot, empty armor, animal head, chess piece, floating object, no human figure
```

---

## 五、背景图生成（每张必须带一句金句）

**第 0 条：背景图不许交空白底图。** 每生成一张，都必须叠一句金句上去。纯色/纯渐变的空图在主页上是一块死板，既没有信息也没有态度，等于白占位置。

### 平台尺寸

| 平台 | 生成尺寸 | 后期裁切 |
|---|---|---|
| 小红书 | 1536×1024（或用方图 1024×1024） | 裁为 1125×633 |
| 抖音 | 1536×1024（或用方图 1024×1024） | 裁为 1125×633 |
| 视频号 | 1536×1024（或用方图 1024×1024） | 裁为 1080×608 |
| 通用 3:1 | 1536×1024（或用方图 1024×1024） | 裁为 1500×500 |

> 能选比例就选 **3:2 / 横构图**；只能出方图时按 1024×1024 生成，**所有关键元素放在中央 60% 区域**，再裁成横条。裁切与叠字都由本地脚本完成，不额外消耗额度。

### 画面必须"有东西"，不能是一块纯色

背景图比头像暗，但**下限是 `#141416`**，而且必须有可见的内容层次。三条至少做到两条：

- **氛围**：烟雾、雾气、浮尘、光斑、镜头光晕
- **材质**：石面、金属拉丝、磨砂、纸纹、颗粒
- **结构**：边缘的阴影块、斜切的光带、远处的暗色形状

一块干净的径向渐变只算及格线，不是标准交付。生成后如果眯眼看是一片平色，重出。

### 方向 1：暗场氛围 + 留字位（默认）

```
Cinematic dark atmospheric background, deep charcoal base with a soft crimson radial glow behind the center,
layered atmosphere: drifting smoke and haze, fine dust particles catching the light, subtle film grain, faint brushed-stone texture in the lower half,
a few deep shadow shapes at the edges and one soft diagonal light band crossing the frame,
rich tonal depth between #141416 and #2A2A2E, never flat or pure black,
large clean negative space in the upper third reserved for a headline, subtle edge falloff only,
wide 3:2 composition, no text, no letters, no logo,
high quality, photorealistic, cinematic
```

**关键改动**：把 v1 的 `minimal, no objects` 换成具体的三层内容（烟雾+浮尘 / 石面纹理 / 阴影块与光带）。只写"简约"模型就真的给你一块渐变，写清材质和结构才会出画面。

### 方向 2：符号放大 + 短句压角（有辨识度后使用）

```
Cinematic dark background, a giant black knight helmet silhouette on the right side,
its edge caught by a strong sharp rim light so the shape reads clearly against the background,
faint red glow from the visor, heavy atmosphere, smoke and dust in the air, film grain, wet stone texture at the bottom,
left two-thirds left as dark negative space with layered haze,
wide 3:2 composition, no text, high quality
```

**注意**：这里的"silhouette"是允许的，因为主体只用形状说话、并且靠强轮廓光把形状"画"出来。但边缘光必须明确写，否则同样会糊成一块。

### 叠字：每张都要有一句

生成后由你**直接用本地脚本（PIL）叠加文字**，不要让模型写中文。字体优先 `C:\Windows\Fonts\msyhbd.ttc`（微软雅黑 Bold），其次 `msyh.ttc`、`simhei.ttf`。

**金句来源**：`03-hook-quote-bank.md` 的金句库，或下面的标语库。规则：

- 一句话，≤ 12 字，不加标点（句号可留），不加引号
- **不许与本期视频标题、简介原句重复**——三个位置同一句话，等于浪费三次曝光
- 与头像同色系：骨白 `#E8E2D6` 或暗金 `#C9A227`
- 一句话只表达一个判断，不写两段式（"不是A，而是B"最多留前半句）

**交付**：默认交付**带字版**；用户明确要无字底图时再单独给一张，并在回复里说明分别是什么用途。

### 背景图文字规范

| 项 | 规范 |
|---|---|
| 字数 | ≤ 12 字，一行 |
| 位置 | 画面中央偏上 1/3（避开底部头像与数据区） |
| 字号 | 图高的 6–8% |
| 颜色 | 骨白 `#E8E2D6` 或暗金 `#C9A227` |
| 描边/阴影 | 无描边，加 20% 黑色投影保证压暗底上可读 |
| 标点 | 只用句号，不用感叹号、问号、emoji |

**标语库**（与简介同源但不重复）

- 温顺的人，永远在排队。
- 你的善良，需要一道门槛。
- 别人怎么对你，是你教的。
- 先有牙齿，再谈善良。
- 不解释，是最有力的回应。
- 你不用变好，你要变得不好惹。

---

## 六、封面（笔记/视频首图）

虽然不属于主页四件套，但它是点击率的第一变量。

| 项 | 规范 |
|---|---|
| 比例 | 3:4（小红书）/ 9:16（抖音竖版） |
| 底色 | 与头像同暗场，但不低于 `#141416` |
| 主标 | 4–8 字，占画面 1/4 高度，粗黑 |
| 副标 | 10–14 字，主标下方，字号为主标 40% |
| 元素 | 最多一个符号（刀/棋/剪影），放右下或左下，需有轮廓光 |
| 禁止 | 花字特效、贴纸、多个箭头、荧光笔、整图压黑 |

**封面自查**：在手机信息流里缩成 120px 宽，主标是否还能读清？读不清就重做。

---

## 七、批量一致性

矩阵账号（同一人运营多个号）必须统一：

1. 同一句风格锚（prompt 前缀完全一致）
2. 同一色板（`#2A2A2E` 主体 / `#141416` 背景 / `#8C1C13` 血红 / `#C9A227` 暗金）
3. 同一三段光（都是 front-left 主光 + 30% 辅光 + 边缘光，不混用）
4. 同一构图（都是 3/4 半身或都是居中对称，不混用）
5. 同一曝光（主体灰度均值都落在 85–140 区间）

不一致的视觉，会让观众以为是两个不同的人，矩阵互相不导流。
