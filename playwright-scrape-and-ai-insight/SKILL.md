---
name: playwright-scrape-and-ai-insight
description: 使用系统浏览器（Edge/Chrome）通过 Playwright 抓取内网门户或需要 SSO 认证的网页内容，并调用内部 AI 模型（OpenAI 兼容接口）生成结构化洞察/摘要。适用于需要从内部网站抓取数据后自动生成报告、摘要或洞察的场景。当用户提及'用系统浏览器抓取'、'内网网页抓取'、'调用内部AI模型生成洞察'、'门户数据抓取+AI分析'等需求时使用。

metadata:
  version: 1.0.0
---

# 使用 Playwright 系统浏览器抓取 + AI 模型生成洞察

本 Skill 提供一种通用方法：使用 Playwright 调用系统已安装的浏览器（Edge/Chrome）抓取需要 SSO 或企业认证的内网网页内容，然后调用内部 AI 模型（OpenAI 兼容接口）对抓取结果进行结构化分析，生成洞察/摘要/简报等内容。

## 适用场景

- 需要抓取公司内网门户（如 W3、OA、知识库）的内容
- 这些内网站点依赖系统浏览器已保存的 SSO 会话认证
- 抓取后需要 AI 模型对内容进行分析、摘要、提炼洞察
- AI 模型部署在内网，使用 OpenAI 兼容接口（无需 API Key）

## 工作流程

1. **使用 Playwright 启动系统浏览器**（优先 Edge，降级到 Chrome，最后兜底默认 Chromium）
2. **导航到目标 URL**，等待 JavaScript 执行完成
3. **提取页面结构化数据**（通过 JS evaluate 或 HTML 解析）
4. **调用内部 AI 模型**：发送 prompt 要求 AI 对数据进行分析/摘要/生成洞察
5. **处理 AI 响应**：清理 markdown 代码块包裹后解析 JSON
6. **输出结果**：保存为 JSON/HTML 等格式

## 关键技术要点

### 1. Playwright 系统浏览器配置

使用 `channel` 参数启动系统已安装的 Edge 或 Chrome：

```python
# 优先 Edge，降级 Chrome，最后兜底
browser = await p.chromium.launch(channel='msedge', headless=True)
```

**降级策略**：
```python
try:
    browser = await p.chromium.launch(channel='msedge', headless=True)
except Exception:
    try:
        browser = await p.chromium.launch(channel='chrome', headless=True)
    except Exception:
        browser = await p.chromium.launch(headless=True)
```

### 2. 页面等待策略

- 使用 `wait_until='domcontentloaded'` 等待 DOM 解析完成
- 额外 `asyncio.sleep(5-8秒)` 等待 JavaScript 动态渲染
- 使用 `page.evaluate()` 执行 JS 提取数据（比解析 HTML 更鲁棒）

### 3. 字符编码统一

对于中文门户网站，常见特殊字符有编码差异：
- `•` (U+2022 BULLET) vs `·` (U+00B7 MIDDLE DOT)
- 在匹配标题或做字典映射时，建立双向映射字典来统一

```python
# 建立两种编码的映射
title_to_idx = {}
title_to_idx_bullet = {}
for idx, section in enumerate(data['sections']):
    title = section['title'].replace(' Top5', '')
    title_to_idx[title] = idx
    title_to_idx_bullet[title.replace('·', '•')] = idx
```

### 4. AI 模型调用

对于内部部署的 OpenAI 兼容接口（无 API Key）：

```python
import urllib.request
import json

payload = {
    "model": "模型名称",
    "messages": [
        {"role": "system", "content": "系统提示"},
        {"role": "user", "content": "用户提示"}
    ],
    "temperature": 0.7,
    "max_tokens": 2000
}

data = json.dumps(payload).encode('utf-8')
headers = {"Content-Type": "application/json"}
req = urllib.request.Request(url, data=data, headers=headers, method='POST')

with urllib.request.urlopen(req, timeout=300) as response:
    result = json.loads(response.read().decode('utf-8'))
    content = result['choices'][0]['message']['content']
```

### 5. AI 响应的 JSON 解析（关键点）

AI 模型返回的 JSON 经常包裹在 markdown 代码块中，需要先清理再解析：

```python
def parse_ai_json_response(response: str) -> dict:
    """清理 AI 响应中的 markdown 代码块标记并解析 JSON"""
    cleaned = response.strip()
    if cleaned.startswith('```json'):
        cleaned = cleaned[7:]
    elif cleaned.startswith('```'):
        cleaned = cleaned[3:]
    if cleaned.endswith('```'):
        cleaned = cleaned[:-3]
    cleaned = cleaned.strip()
    return json.loads(cleaned)
```

### 6. 分批调用策略

当需要 AI 模型生成大量结构化内容时，单次调用易超时或返回过长 JSON 导致解析失败。推荐分批处理：

```python
batch1 = sections[:4]
messages1 = build_prompt_for_sections(batch1)
response1 = client.chat(messages1, max_tokens=2000)

batch2 = sections[4:]
messages2 = build_prompt_for_sections(batch2)
response2 = client.chat(messages2, max_tokens=2000)
```

## 脚本说明

本 Skill 提供以下脚本：

- `scripts/scrape_with_system_browser.py`：使用 Playwright 系统浏览器抓取网页内容的通用脚本。
- `scripts/call_internal_ai.py`：调用内部 AI 模型（OpenAI 兼容接口）的通用脚本。

详情见各脚本的 argparse 配置。

## 参考文件

- `references/character_encoding_notes.md`：字符编码处理注意事项
- `references/ai_json_parsing.md`：AI 响应 JSON 解析详细说明

## 约束与注意事项

1. 本 Skill 依赖系统已安装的 Edge 或 Chrome 浏览器，以及 Playwright Python 包
2. 系统浏览器需要有对应内网站点的认证会话（SSO/Cookie），否则抓取可能失败
3. AI 模型接口必须为 OpenAI 兼容格式，且网络可达
4. 对于大输出量的 AI 调用，务必设置足够长的超时时间（建议 300 秒以上）
5. 所有临时文件用完后应清理
