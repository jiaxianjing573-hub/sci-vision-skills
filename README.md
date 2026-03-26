# 科研视频 Pipeline 迭代需求与解决路径文档

本文档整合了用户提出的迭代需求、视觉风格规范，以及 `complementarity_examples.md`、`seedream_vocabulary.md` 和 `camera_and_pacing_guide.md` 等参考文档中的优秀策略，为 `sci-vision-video-agent` 提供完整的迭代需求点与解决路径。

## 1. 旁白文案与字幕模块

### 1.1 情感共鸣与比喻化解释
**需求点**：
- 旁白文案需要引起观众情感共鸣。
- 专业名词需要通俗化、比喻化的解释。
- 开头和结尾通过各种方式加入情感联结。

**解决路径**：
- 修改 `step_a_script_generation.py` 的 `system_prompt`，在生成口播稿时强制要求使用比喻解释专业名词，并在首尾段设置情感 Hook 与升华。
- 修改 `step_b_review_iteration.py` 的审校 Prompt，增加对生硬专业名词和首尾情感平淡的校验与重写规则。

### 1.2 字幕切分与标点清洗
**需求点**：
- 旁白文案自动切分为短句，每句最多不超过 20 字。
- 字幕中不要有任何标点符号。

**解决路径**：
- 修改 `step_f_ffmpeg_compose.py` 中的 `SubtitleGenerator.MAX_CUE_CHARS` 常量，从 27 改为 20。
- 确认现有代码中的正则清洗逻辑 `re.sub(r"[，。；：、？！,.!?;:]", "", text)` 已覆盖所有常见标点。

### 1.3 文字量与论文篇幅对齐
**需求点**：
- 文案内容每个板块的文字量需与论文相对应，允许微调。

**解决路径**：
- 修改 `step_b_review_iteration.py` 的 `_round_2_finalize_script`，放宽原有的 65-75 字硬限制，改为 40-100 字浮动，并移除本地回退逻辑中的强制压缩/扩展。

## 2. 视觉提示词与画面控制模块

### 2.1 旁白-画面互补策略（Complementarity Strategy）
**需求点**：
- 图片提示词生成前需理解旁白信息。
- 画面需与旁白强关联，并补充旁白中没有的信息。

**解决路径**：
- **新增 Skill**：创建 `sci-vision-complementarity-strategy` Skill，引入 5 种互补策略（Scale Anchoring, Process Concretization, Contextual Embedding, Data Spatialization, Emotional Grounding）。
- 修改 `step_a_script_generation.py`，要求模型在生成 `image_intent` 时，必须显式声明使用的互补策略（如 `strategy: scale_anchoring`），并据此生成补充信息的 `image_prompt`。

### 2.2 全片景别与场景比例控制
**需求点**：
- 全片图表不超过 5%。
- 包含人的场景不超过 30%。
- 景别比例：近景不超过 20%，全景不超过 20%，中景约 30%。

**解决路径**：
- 修改 `step_b_review_iteration.py` 的 `_round_1_accuracy_check`，在 Prompt 中增加全局视觉比例强制约束，要求模型在重写 `image_prompt` 时进行全局统筹。

### 2.3 Seedream 专属词汇与负向约束优化
**需求点**：
- 提升生图质量，适配 Seedream 模型的特性。

**解决路径**：
- **更新 Skill**：将 `seedream_vocabulary.md` 中的学科专属词汇（Science-Specific Vocabulary）整合进 `sci-vision-style-guide` Skill 的对应学科分类中。
- 修改 `step_a_script_generation.py`，将科研视频专属的负向词汇（如 `no text labels, no cartoon style, no human figures`）作为全局约束追加到 `image_prompt` 生成规则中。

## 3. 镜头语言与特殊图表模块

### 3.1 叙事节奏与镜头运动映射
**需求点**：
- 解决镜头运动同质化问题，使镜头语言与叙事节奏相匹配。

**解决路径**：
- **新增 Skill**：创建 `sci-vision-camera-pacing` Skill，引入节奏-镜头映射表（Slow/Moderate/Fast/Suspenseful）及 Seedream 参数化镜头指令。
- 修改 `step_a_script_generation.py`，要求模型在生成 `img2video_prompt` 时，根据段落的叙事位置和情感目标，从映射表中选择对应的镜头参数，并遵循"视听同步规则"（Visual-Audio Sync）。

### 3.2 机制图处理（3D渲染 + 首尾帧衔接）
**需求点**：
- 机制图需生成 2-4 张 3D 渲染图，并通过首尾帧衔接。

**解决路径**：
- 修改 `step_a_script_generation.py` 的 `img2video_prompt` 规则，遇到机制图时强制指定 "3D rendered style"，并指示生成多视角状态。
- 修改 `step_f_ffmpeg_compose.py` 的 `_build_segment_visual_cmd`，引入 FFmpeg `xfade` 滤镜实现交叉溶解（crossfade）衔接。

### 3.3 前后变化图表处理（对比图 + 首尾帧衔接）
**需求点**：
- 前后变化图表需渲染成 2 张图，通过首尾帧衔接，严禁分屏对比。

**解决路径**：
- 修改 `step_a_script_generation.py` 的 `img2video_prompt` 规则，遇到对比图时指示生成变化前后的 2 张图，并明确禁止分屏对比。
- 依赖 `step_f_ffmpeg_compose.py` 中的 `xfade` 滤镜实现衔接。

### 3.4 静态图表处理（缓慢放大）
**需求点**：
- 静态图表需保持主体稳定，缓慢放大。

**解决路径**：
- 修改 `step_a_script_generation.py` 的 `_expand_img2video_prompt`，强化静态图表的镜头约束文案，确保仅做缓慢的镜头拉近。
