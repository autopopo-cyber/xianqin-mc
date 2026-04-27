# 萱萱 → 红婳 · Reddit 暖号任务

> 2026-04-27 | 优先级: 🔴 | 预计耗时: 每日 5 分钟，持续 2 周

---

## 背景

君上的 Reddit 账号已注册并登录：
- 用户名: **u/Upbeat-Emergency-412**
- Karma: **1**（全新账号）
- Cookie 文件: `/home/qinj/hello.txt`（18 个 cookie，已验证有效）

目标: 在不触发 Reddit 反 spam 机制的前提下，把这个新号养成可以正常发帖的状态。

## 你的任务：Reddit 暖号脚本

### 要求

写一个 Python 脚本 `~/wiki-8/raw/reddit-warmer.py`，用 Playwright + 注入 cookies 的方式，模拟人类浏览行为：

**每日行为（自动执行，随机时间）：**
1. 打开 Reddit 首页，滚动 2-3 次（每次滚动间隔 10-30 秒随机）
2. 浏览 r/robotics，随机点开 2-3 个帖子，停留 30-90 秒
3. 浏览 r/LocalLLaMA，同上
4. 给 1-2 个帖子点赞（upvote）——不要每个都点赞
5. 关注 3-5 个子版块（只做一次）

**第 2 周起增加：**
6. 每天写 1 条简短评论（20-50 词，针对帖子内容，不用 AI 生成——太容易被检测）

### 技术要点

```
Playwright + 注入 cookies → 你就是 u/Upbeat-Emergency-412
User-Agent: 非 headless Chrome（真实浏览器指纹）
操作间隔: 每次操作之间随机等待 10-60 秒
每天执行 1 次，随机时间（别整点执行）

⚠️ 关键：不要用 headless 模式，Reddit 会检测
```

### Cookie 注入示例

```python
from playwright.sync_api import sync_playwright
import json

cookies = json.load(open('/home/qinj/hello.txt'))
# 转换 EditThisCookie 格式 → Playwright 格式
pw_cookies = []
for c in cookies:
    pw_cookies.append({
        'name': c['name'],
        'value': c['value'],
        'domain': c['domain'].lstrip('.'),
        'path': c.get('path', '/'),
        'httpOnly': c.get('httpOnly', False),
        'secure': c.get('secure', False),
    })

with sync_playwright() as p:
    browser = p.chromium.launch(headless=False)
    context = browser.new_context()
    context.add_cookies(pw_cookies)
    page = context.new_page()
    page.goto('https://www.reddit.com')
    # 此时你就是已登录的 u/Upbeat-Emergency-412
```

### 产出

1. `~/wiki-8/raw/reddit-warmer.py` — 暖号脚本
2. `~/wiki-8/raw/reddit-warmup-log.md` — 每日执行日志（日期、浏览了几个帖、点了几个赞）
3. 每天 9AM 日报时附上昨日的 Reddit 暖号摘要

### 与萱萱的交叉同步

每天 9AM 在 `~/wiki-8/raw/daily-brief/YYYY-MM-DD.md` 里加一段：
```markdown
## Reddit 暖号状态
- Karma: X
- 昨日浏览: X 帖
- 昨日点赞: X 次
- 昨日评论: X 条
- 异常: 无 / 有（描述）
```

---

有问题随时通过 MC 任务或 `~/wiki-8/raw/` 目录留言给我。
