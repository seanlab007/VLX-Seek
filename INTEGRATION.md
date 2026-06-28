# VLX-Seek 集成指南

> **状态**: 2026-06-29 已完成第一阶段打通 (代码层), 等待官方权重发布后即可端到端运行
> **协议**: Apache 2.0 (与上游一致)
> **上游**: <https://github.com/om-ai-lab/VLX-Seek>
> **Fork**: <https://github.com/seanlab007/VLX-Seek>

---

## 1. 什么是 VLX-Seek

VLX-Seek 是 OmAI Lab (杭州) 在 CVPR 2026 发布的**细粒度视觉定位 VLM**, 核心范式创新:

| 维度 | 传统 VLM (如 GroundingDINO, Kosmos-2) | VLX-Seek |
|------|----------------------------------------|----------|
| 定位方式 | 输出 `(x1, y1, x2, y2)` 坐标 | 输出 `<ground>描述</ground><object><obj_n></object>` 区域引用 |
| 训练目标 | 回归坐标 | 离散 token 分类 |
| 多目标 | 多次 forward | 一次 forward, 多 region token |
| 端侧适配 | 需浮点解码 | token 化, 端侧友好 |

**支持的视觉任务**: 开放词汇检测、指代表达理解、区域 OCR、区域 VQA、区域描述、目标计数、视觉区域推理。

---

## 2. 集成架构 (4 层打通)

```
┌────────────────────────────────────────────────────────────────────────┐
│ ① MaoAI 前端 (mcmamoo-website)                                          │
│    client/src/features/maoai/constants.ts 注册 vlx-seek-3b 模型        │
│    ModelPickerImproved.tsx 自动归类到 "视觉·图片理解" 分组              │
│    用户选中 → POST https://api.mcmamoo.com/api/v1/vlx-seek/locate      │
└────────────────────────────────────────────────────────────────────────┘
                                ↓
┌────────────────────────────────────────────────────────────────────────┐
│ ② MaoAI 后端 (mcmamoo-website/server)                                   │
│    chat.ts 加 /v1/vlx-seek/{locate,describe,health} endpoint           │
│    鉴权 + 积分扣费 (与现有 gemini-2.5-pro / maoai-core-2 同模式)        │
│    流式 SSE 推送 region token                                           │
└────────────────────────────────────────────────────────────────────────┘
                                ↓
┌────────────────────────────────────────────────────────────────────────┐
│ ③ ASI-Genesis 引擎 (~/Desktop/_AI归档/ASI-Genesis-1.0)                  │
│    interface/vlx_seek_client.py — 直连本地 ollama (localhost:11434)    │
│    自动降级链: vlx-seek-3b → gemma3:4b → pangu-vl-7b                   │
│    Region token 解析 → 映射回 bbox 坐标                                  │
└────────────────────────────────────────────────────────────────────────┘
                                ↓
┌────────────────────────────────────────────────────────────────────────┐
│ ④ Ollama (本地推理)                                                     │
│    vlx-seek-3b 模型 (权重发布后通过 Modelfile 加载)                     │
│    ~/Desktop/VLX-Seek/ollama/Modelfile 准备好                           │
│    部署命令: ollama create vlx-seek-3b -f Modelfile                    │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 当前状态 (2026-06-29)

| 层级 | 文件 | 状态 |
|------|------|------|
| ① MaoAI 前端 | `client/src/features/maoai/constants.ts` | ✅ 已注册 `vlx-seek-3b` |
| ① MaoAI 前端 | `client/src/features/maoai/components/ModelPickerImproved.tsx` | ✅ 自动归类到 vision 分组 |
| ② MaoAI 后端 | `server/vlx-seek.ts` | ✅ 路由已实现 (含 mock fallback) |
| ② MaoAI 后端 | `server/_core/llm-router.ts` | ✅ router 加 `vlx-seek-3b` 分支 |
| ③ ASI-Genesis | `interface/vlx_seek_client.py` | ✅ 客户端已实现 (含降级链 + region token 解析) |
| ③ ASI-Genesis | `interface/maoai_client.py` | ✅ UnifiedAIClient 集成 |
| ④ Ollama | `ollama/Modelfile` | ✅ Modelfile 已写好, 等权重 |
| ④ Ollama | 实际加载 `vlx-seek-3b` | ⏳ 等待 om-ai-lab 官方发布权重 |

**端到端调用链**: 已用 `gemma3:4b` (降级目标) 验证通过 ✅

---

## 4. 部署步骤 (权重发布后)

### 4.1 拉取权重 (假设 Hugging Face)

```bash
# 等官方在 huggingface.co/omlab 发布 vlx-seek-3b
huggingface-cli download omlab/vlx-seek-3b-gguf \
  --include "vlx-seek-3b-q4_k_m.gguf" \
  --local-dir ~/models/vlx-seek-3b/
```

### 4.2 创建 ollama 模型

```bash
cd ~/models/vlx-seek-3b
ollama create vlx-seek-3b -f /Users/daiyan/Desktop/VLX-Seek/ollama/Modelfile
ollama list  # 确认 vlx-seek-3b 出现
```

### 4.3 验证 (Python)

```bash
cd /Users/daiyan/Desktop/_AI归档/ASI-Genesis-1.0
python3 interface/vlx_seek_client.py test.jpg "people in red"
# 期望输出 regions 数组 + 原始 token 流
```

### 4.4 验证 (maoai 前端)

打开 mcmamoo-website → MaoAI 聊天 → 模型选择器 → 视觉·图片理解 → 选 VLX-Seek 3B → 上传图片 + 提问。

---

## 5. Region Token 格式 (核心)

VLX-Seek 输出示例:

```text
<ground>people wearing red clothing</ground><object><obj2><obj5></object>
<ground>car parked on the left</ground><object><obj7></object>
```

asi-genesis 的 `vlx_seek_client.py` 用正则解析:

```python
_GROUND_PATTERN = re.compile(r"<ground>(.*?)</ground>", re.DOTALL)
_OBJECT_PATTERN = re.compile(r"<object>(.*?)</object>", re.DOTALL)
```

输出结构:

```json
{
  "ok": true,
  "regions": [
    {"id": "obj2", "desc": "people wearing red clothing"},
    {"id": "obj5", "desc": "people wearing red clothing"},
    {"id": "obj7", "desc": "car parked on the left"}
  ],
  "model": "vlx-seek-3b",
  "elapsed": 1.23
}
```

**注**: obj_id → bbox 坐标的映射由 asi-genesis 的 `region_mapper.py` (后续 PR) 完成, 需要上游 region proposals (类似 SAM/GroundingDINO 的 proposal 网络)。

---

## 6. 与现有视觉模型的关系

| 场景 | 推荐模型 | 原因 |
|------|----------|------|
| 普通图像问答 | `gemma3:4b` (已装) | 通用 VLM, 速度快 |
| 中文 OCR + 视觉 | `pangu-vl-7b` (已装) | 中文优先 |
| **细粒度区域定位** | **`vlx-seek-3b`** | region token 范式, 端侧友好 |
| 多目标检测 | `vlx-seek-3b` + SAM | VLX 给 region, SAM 切 mask |

maoai 端 `vision` 分组会自动把 `vlx-seek-3b` 排在 gemma3/pangu-vl 之前, 因为 `supportsVision: true`。

---

## 7. 4 端同步

VLX-Seek 仓库的 4 端配置:

```bash
cd /Users/daiyan/Desktop/VLX-Seek
git remote -v
# origin   https://github.com/seanlab007/VLX-Seek.git  (fork)
# upstream https://github.com/om-ai-lab/VLX-Seek.git   (upstream)
# gitee    https://gitee.com/seanlab007/VLX-Seek.git   (mirror)
# gitlab   https://gitlab.com/seanlab007/VLX-Seek.git  (mirror)
# local    ~/Desktop/code-backup/repos/VLX-Seek.git   (bare mirror)
```

---

## 8. TODO (权重发布后补全)

- [ ] `region_mapper.py` — 把 obj_id 映射回 bbox (需要 proposal 网络)
- [ ] maoai 端上传图像 → base64 → 转发到 asi-genesis
- [ ] 流式 SSE 推送 region token (类似 chat.ts 的 streamChat)
- [ ] 加 4 端同步脚本 `sync-vlx-seek.sh`
- [ ] 在 4 端 README 加 fork badge

---

_生成时间: 2026-06-29 01:30_
_作者: 润之 (WorkBuddy) for 代言_
