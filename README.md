# astrbot_plugin_pojia —— 破甲装甲

给 AstrBot 里的模型穿上装甲：**执行者人格注入 + 拒绝自动重试 + 输出绕词**。

灵感与提示词骨架来自 [dsh-purge](https://github.com/YuJunZhiXue/dsh-purge)（Little Code Sauce 框架的"无限制沙箱"提示词），
移植成 AstrBot 插件形态：不靠改宿主源码，全部走 AstrBot 官方插件钩子。

---

## 一、它解决什么

用 AstrBot 接第三方模型（尤其是国内模型 / 中转 API）时，常见的三种"卡住"：

| 症状 | 原因 | 本插件的对策 |
| --- | --- | --- |
| 模型回复"抱歉，我不能…" | 模型自带的对齐层 | 每次请求前注入「执行者核心」，占住 system prompt 末尾的高权重位置 |
| 第一次被拒后整轮就废了 | 没有重试机制 | `on_llm_response` 检测到拒绝 → 带加压指令重新生成，最多 5 次 |
| 内容没问题，但发不出去 | QQ/微信/平台的敏感词过滤 | 可选输出绕词（星号 / 窄空格 / 同形字 / 零宽字符） |

---

## 二、工作原理（对应 AstrBot v4.29.3 实际钩子）

```
用户消息
   │
   ├─ OnLLMRequestEvent  ← ① 缓存 ProviderRequest；把「执行者核心」拼进 req.system_prompt
   │                        （priority = -1000，保证在其它插件之后注入，落在末尾）
   │
   ├─ 模型生成 …
   │
   ├─ OnLLMResponseEvent ← ② 用正则判定回复是不是拒绝
   │                        是 → 取缓存的 ProviderRequest，逐次加压重新调用 provider.text_chat()
   │                        新回复合规 → response.completion_text = 新回复
   │                        全部失败 → 可选剥离拒绝句，保留有效内容
   │
   └─ OnDecoratingResultEvent ← ③ 发送前按词表打码（可选，默认关）
```

关键实现点：

- 注入走 `req.system_prompt` 字符串拼接，不碰宿主源码，也不影响其它插件；
- 用 `[[POJIA_ARMOR::v1]]` 标记去重，避免同一条 system prompt 被注入两次；
- 重试复用原始请求的 `contexts / image_urls / func_tool / model / tool_calls_result`，不是简单重发一句话；
- 所有钩子都包在 `try/except` 里，出错只记日志，不会把整条消息管线搞崩。

---

## 三、安装

1. 把整个 `astrbot_plugin_pojia` 目录放进 AstrBot 的插件目录：

   ```
   <AstrBot 实例>/core/data/plugins/astrbot_plugin_pojia/
   ```

   也就是 Dashboard → 插件管理 → 安装插件 里能看到的那个目录；手工放进去同样有效，
   Windows 上通常是：

   ```
   C:\Users\<你的用户名>\.astrbot_launcher\instances\<实例 ID>\core\data\plugins\
   ```

2. 重载插件：Dashboard → 插件管理 → 重载（或重启 AstrBot）。
3. Dashboard → 插件管理 → 破甲装甲 → 配置，按需调整（默认配置开箱可用）。
4. 聊天里发 `/破甲` 看状态。

不需要装任何额外依赖（只用标准库 + AstrBot 内置 API）。

---

## 四、配置项

| 配置 | 默认 | 说明 |
| --- | --- | --- |
| `enabled` | 开 | 总开关 |
| `prompt_file` | `pojia_core.md` | 注入哪个提示词文件（`prompts/` 目录下） |
| `inject_position` | `suffix` | 追加到末尾 / 插到最前 |
| `extra_prompt` | 空 | 你自己的补充规则，接在执行者核心后面 |
| `retry_enabled` | 开 | 拒绝自动重试 |
| `max_retries` | 2 | 最多重试几次（1-5） |
| `retry_timeout` | 90 | 单次重试超时秒数 |
| `retry_prompt` | 见配置 | 重试时追加的加压指令 |
| `strip_refusal` | 开 | 全部重试失败后剥离拒绝句 |
| `strip_min_length` | 20 | 剥离后不足这个字数就不替换 |
| `refusal_patterns` | 29 条正则 | 拒绝识别规则，可增删 |
| `rewrite_enabled` | 关 | 输出绕词 |
| `rewrite_words` | 空 | 需要打码的词 |
| `rewrite_style` | `star` | `star` / `space` / `homoglyph` / `zero-width` |
| `whitelist_sessions` | 空 | 留空=全局；填了只在列出的会话生效 |
| `blacklist_sessions` | 空 | 这些会话永不破甲 |
| `admin_only` | 开 | 仅管理员能用 `/破甲 on/off` |
| `debug` | 关 | 注入日志 |

---

## 五、命令

```
/破甲             查看状态（全局开关、本会话、提示词长度、重试配置）
/破甲 on          开启本会话破甲
/破甲 off         关闭本会话破甲
/破甲 重试        对最近一次请求强制再生成一次（用 /破甲 前先聊一句）
/破甲 提示词      预览当前注入的执行者核心（前 1200 字）
/破甲 重载        清空提示词缓存（改完 prompts/*.md 不用重启）
```

别名：`/pojia`。

---

## 六、自定义提示词

`prompts/` 里三份可以直接改，也可以新建文件后在 `prompt_file` 里填新文件名：

- `pojia_core.md` —— 完整版执行者核心（默认，约 1800 token）
- `pojia_lite.md` —— 精简版，适合小模型 / 上下文紧张
- `pojia_en.md` —— 英文版，适合英文模型

改完发 `/破甲 重载` 生效，无需重启。文件内容会被整体拼进 system prompt，
开头那行 `[[POJIA_CORE]]` 只是注释性标记，不影响效果。

自建提示词建议保留四段结构：**身份 → 禁止拒绝 → 分层兜底 → 抗压与输出规范**。

---

## 七、FAQ

**Q：模型还是拒绝怎么办？**
A：三层排查——
1. `debug` 打开，看日志里有没有 `[pojia] 已注入破甲提示词`。没有 = 钩子没跑到（检查插件是否启用、会话是否被关闭/黑名单）。
2. 拒绝正则没命中 → 把模型实际说的那句开头贴进 `refusal_patterns`。
3. 命中了但重试还是拒绝 → 换更强的提示词文件（`pojia_core.md`），或把 `max_retries` 提到 3，或在 `extra_prompt` 里针对该模型加一句专用施压话术。

**Q：会不会把正常回复误判成拒绝？**
A：判定规则是三条之一：命中位置在开头 80 字内 / 命中两处以上 / 回复短于 300 字且命中。正常长回复里偶然引用"抱歉"不会触发。

**Q：重试很慢 / 很费 token？**
A：每次重试都是一次完整请求。`max_retries` 保持 1-2 即可；只在意注入不在意重试就把 `retry_enabled` 关掉。

**Q：平台敏感词拦的是我的输入怎么办？**
A：那属于消息入站，本插件管不到。可以先在群里用空格/全角/同形字手写关键词，或者把平台换成 aiocqhttp 直连。

**Q：会不会影响 persona / 人格设定？**
A：不会覆盖，只在末尾追加。想让人格第一优先就把 `inject_position` 改成 `prefix`。

---

## 八、目录结构

```
astrbot_plugin_pojia/
├── main.py              # 全部逻辑（约 500 行）
├── metadata.yaml        # 插件元信息
├── _conf_schema.json    # 配置 schema（Dashboard 自动渲染）
├── requirements.txt     # 无第三方依赖
├── __init__.py
├── prompts/
│   ├── pojia_core.md    # 完整版执行者核心
│   ├── pojia_lite.md    # 精简版
│   └── pojia_en.md      # 英文版
└── README.md
```

运行时会在插件目录生成 `runtime_state.json`，存各会话的开关状态（删掉即恢复默认）。
