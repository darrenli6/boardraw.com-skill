# boardraw-skill

> **语言：** [English](./README.md) | 中文

一个 [Claude Code](https://claude.ai/code) Skill，将自然语言描述转化为流程图，并直接上传到 [boardraw.com](https://boardraw.com)。

用任意语言描述一个流程，Claude 会设计图表、请你确认，然后发布到你的 boardraw 工作区并返回可分享的链接。

---

## 效果演示

```
你：    画一个 CI/CD 流水线：代码推送后触发测试，
        测试通过后部署到 staging，人工审批后部署到生产环境，
        成功或失败都通知 Slack。

Claude：📋 流程图预览
        标题：CI/CD 流水线

          [代码推送] → [运行测试] → {测试通过?}
                                      ├── 是 → [部署 Staging] → {审批通过?}
                                      │                           ├── 是 → [部署生产] → [通知 Slack ✅]
                                      │                           └── 否 → [通知 Slack ❌]
                                      └── 否 → [通知 Slack ❌]

        以上设计是否符合你的需求？确认后我将上传到 boardraw.com。

你：    可以，上传吧。

Claude：✅ 上传成功！
        🔗 https://www.boardraw.com/board/c2f9b3b6-4b5e-4c22-9a28-fb3e2a4c9d11
```

---

## 环境要求

| 要求 | 说明 |
|---|---|
| [Claude Code](https://claude.ai/code) | CLI、桌面端或 IDE 插件均可 |
| boardraw.com 账户 | API 访问需要 **Pro 或 Team** 套餐 |
| Python 3.8+ | 仅在使用独立上传脚本时需要 |

---

## 安装

### 1. 克隆仓库

```bash
git clone https://github.com/your-org/boardraw.com-skill.git
cd boardraw.com-skill
```

### 2. 将 Skill 注册到 Claude Code

注册为全局 Skill，在所有项目中可用：

```bash
claude config add skills "$(pwd)/generate"
```

或仅在当前项目中注册，编辑 `.claude/settings.json`：

```json
{
  "skills": [
    "/absolute/path/to/boardraw.com-skill/generate"
  ]
}
```

### 3. 配置 API Key

复制示例文件并填入你的 Key：

```bash
cp .env.example .env
```

编辑 `.env`：

```env
BOARDRAW_API_KEY=br_your_api_key_here
```

获取 API Key：登录 [boardraw.com](https://boardraw.com) → **Settings → API Keys** → 创建新 Key。

> Skill 会依次从 Shell 环境变量和目录树向上查找的 `.env` 文件中读取 `BOARDRAW_API_KEY`。

---

## 使用方法

安装完成后，直接在 Claude Code 中描述需求即可：

```
画一个移动 App 用户引导流程。
```

```
创建一个订单处理系统的流程图：订单进来后检查库存，
处理付款，然后发货。库存不足时先触发补货流程。
```

```
/generate 画一个微服务架构图，包含 API 网关、鉴权服务和三个后端服务。
```

Claude 的执行步骤：
1. 以文字大纲形式**预览**图表设计
2. **等待**你确认（或要求调整）
3. **生成** Excalidraw JSON
4. **上传**到 boardraw.com 并返回可分享链接

---

## 独立上传脚本

如果你已有 Excalidraw JSON 文件，可以不经过 Skill 直接上传：

```bash
# 从文件上传
python generate/scripts/upload.py whiteboard.json

# 从标准输入上传
cat whiteboard.json | python generate/scripts/upload.py -
```

脚本使用同样的 `BOARDRAW_API_KEY` 读取逻辑（环境变量 → `.env` 文件），无需任何第三方依赖。

---

## 项目结构

```
boardraw.com-skill/
├── generate/
│   ├── SKILL.md          # Skill 定义文件 — Claude 调用时读取
│   └── scripts/
│       └── upload.py     # 独立 Python 上传脚本（仅用标准库）
├── .env.example          # API Key 配置模板
├── README.md             # 英文文档
└── README-zh.md          # 本文件（中文）
```

---

## 工作原理

```
用户描述
    │
    ▼
[Claude 解析意图]
    │
    ▼
[文字大纲展示给用户] ──── 用户调整 ────┐
    │ 确认                              │
    ▼                                  │
[生成 Excalidraw JSON] ◄───────────────┘
    │
    ▼
[POST /api/keys/auth → boardraw.com]
    │
    ▼
[返回可分享链接]
```

生成的 JSON 遵循 [Excalidraw](https://excalidraw.com) 格式，也可以直接在 Excalidraw 或任意兼容工具中打开。

---

## API 参考

**接口：** `POST https://www.boardraw.com/api/keys/auth`

| 字段 | 说明 |
|---|---|
| 认证请求头 | `api_key: br_...` |
| 请求体 | `{ "whiteboard": { ...excalidraw 对象... } }` |
| 成功响应 | `201` — 返回文件 UUID 和 boardraw.com 访问链接 |
| 套餐要求 | Pro 或 Team |

---

## 开源协议

MIT — 详见 [LICENSE](./LICENSE)。
