# boardraw-skill

> **语言：** [English](./README.md) | 中文

一个 [Claude Code](https://claude.ai/code) Skill，将自然语言描述转化为完全可编辑的流程图，并直接上传到 [boardraw.com](https://boardraw.com)。

用任意语言描述一个流程，Claude 会设计图表、请你确认，然后通过 `ExcalidrawDoc` 构建器生成文件（所有元素均可拖动、样式修改和重新连线），最后发布到你的 boardraw 工作区并返回可分享链接。

---

## 效果演示

https://darren94me.oss-cn-beijing.aliyuncs.com/boardraw/boardraw-skill.mp4

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

        以上设计是否符合你的需求？确认后我将生成并上传到 boardraw.com。

你：    可以，上传吧。

Claude：✅ 流程图已上传到 boardraw.com！

        🔗 在线查看/编辑：https://www.boardraw.com/workspace/c2f9b3b6-4b5e-4c22-9a28-fb3e2a4c9d11
        📁 文件名：        skill--a83f29cd
        👤 账户：          Olivia Carter (olivia@example.com)

        文件同样可以直接在 Excalidraw 或任意兼容工具中打开编辑。
```

---

## 支持的图表类型

| 类型 | 示例描述 |
|---|---|
| 流程图 | "画一个带邮箱验证的用户注册流程" |
| 思维导图 | "给我们 Q3 产品路线图做个思维导图" |
| 架构图 | "画一个微服务系统：API 网关、鉴权服务、三个后端服务" |
| 组织架构图 | "画工程团队架构：CTO → 3 个 EM → 每人 5 个 IC" |
| 看板/便签墙 | "创建一个看板：待办、进行中、已完成三列" |

---

## 环境要求

| 要求 | 说明 |
|---|---|
| [Claude Code](https://claude.ai/code) | CLI、桌面端或 IDE 插件均可 |
| boardraw.com 账户 | API 访问需要 **Pro 或 Team** 套餐 |
| Python 3.8+ | 必须安装——构建器脚本负责生成 Excalidraw JSON |

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
claude config add skills "$(pwd)/boardraw"
```

或仅在当前项目中注册，编辑 `.claude/settings.json`：

```json
{
  "skills": [
    "/absolute/path/to/boardraw.com-skill/boardraw"
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

获取 API Key：登录 [boardraw.com](https://boardraw.com) → **Settings → API Keys** → 创建新 Key（需要 Pro 或 Team 套餐）。

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
/generate 画一个架构图：React 前端 → API 网关 →
鉴权服务 + 订单服务 + 通知服务 → PostgreSQL + Redis
```

Claude 的执行步骤：
1. 以文字大纲形式**预览**图表设计
2. **等待**你确认（或要求调整）
3. 用 `scripts/excalidraw_builder.py` **构建** `.excalidraw` 文件
4. **校验** JSON，然后**上传**到 boardraw.com
5. **返回**可分享链接——在 boardraw 或 Excalidraw 中完整可编辑

---

## 独立脚本

### 上传已有的 `.excalidraw` 文件

```bash
# 从文件上传
python boardraw/scripts/upload.py whiteboard.excalidraw

# 从标准输入上传
cat whiteboard.excalidraw | python boardraw/scripts/upload.py -
```

### 直接使用构建器

```python
from boardraw.scripts.excalidraw_builder import ExcalidrawDoc, vertical_flow_positions

doc = ExcalidrawDoc()
positions = vertical_flow_positions(3, x0=300, y0=40, w=200, h=80, gap=80)

for (x, y), label in zip(positions, ["开始", "处理", "结束"]):
    shape = doc.rectangle(x, y, 200, 80, bg="blue")
    doc.label(shape, label)

doc.save("output.excalidraw")
```

两个脚本均使用相同的 `BOARDRAW_API_KEY` 读取逻辑（环境变量 → `.env` 文件向上查找），无需任何第三方依赖。

---

## 项目结构

```
boardraw.com-skill/
├── boardraw/
│   ├── SKILL.md                    # Skill 定义文件——Claude 调用时读取
│   ├── scripts/
│   │   ├── excalidraw_builder.py   # ExcalidrawDoc 构建器——生成合法 Excalidraw JSON
│   │   └── upload.py               # 独立上传脚本（仅用标准库）
│   └── references/
│       └── excalidraw-schema.md    # 原始 JSON 字段参考（高级定制用）
├── .env.example                    # API Key 配置模板
├── README.md                       # 英文文档
└── README-zh.md                    # 本文件（中文）
```

---

## 工作原理

```
用户描述
    │
    ▼
[Claude 解析意图 → 文字大纲]
    │
    │ 用户确认（或调整）
    ▼
[excalidraw_builder.py 构建元素]   ← 从不手写 JSON
    │
    ▼
[JSON 校验]
    │
    ▼
[POST /api/keys/auth → boardraw.com]
    │
    ▼
[返回可分享链接]
```

**为什么要用构建器脚本？** Excalidraw 格式要求形状与标签之间严格双向绑定（`boundElements` + `containerId`）、箭头精确吸附（`startBinding` / `endBinding`），以及每个元素独立的随机 `seed` / `versionNonce`。手写这些 JSON 极易出错，构建器自动处理所有细节。

---

## API 参考

**接口：** `POST https://www.boardraw.com/api/keys/auth`

| 字段 | 说明 |
|---|---|
| 认证请求头 | `x-api-key: br_...` |
| 请求体 | `{ "whiteboard": { ...excalidraw 对象... } }` |
| 成功响应 | `201` — 返回 `file.uuid` 和用户信息 |
| 套餐要求 | Pro 或 Team |

**错误码：**

| 状态码 | 含义 |
|---|---|
| 401 | API Key 缺失、无效或已撤销 |
| 403 | 免费套餐——需要升级 |
| 400 | Whiteboard JSON 格式错误 |

---

## 开源协议

MIT — 详见 [LICENSE](./LICENSE)。
