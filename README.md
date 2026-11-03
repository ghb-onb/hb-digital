# HB Digital

贵浩滨的个人技术作品集站点，同时是我三个 Python 项目的落地页。

**在线地址**：https://ghb-onb.github.io/hb-digital/
**3D 实验页**：https://ghb-onb.github.io/hb-digital/lab.html
**管理后台**：https://ghb-onb.github.io/hb-digital/admin.html （保留在源码中，首页不再放置入口）

求职方向：Python 开发 / 数据分析 / 测试开发 实习。

---

## 这个站在做什么

它只有一个任务：让 HR 和技术面试官在几分钟内看清我是谁、会什么、做过什么。

所以页面结构是简历式的，不是官网式的：

| 区块 | 内容 | 为什么这样排 |
|---|---|---|
| 首屏 | 姓名、方向、学校、求职说明、简历下载 | 三秒内知道是谁、在找什么岗位 |
| 技术项目 | InsightAgent / CodeFix-Agent / TestPilot / HB Digital | 每个项目统一按「问题 → 我做了什么 → 核心实现 → 结果」写 |
| 技术能力 | 三组能力，每组挂证据 | 不列名词，只列能被点开验证的东西 |
| 经历 | 教育、企业项目、独立接单、个人探索 | 补上「做过事」的证明 |
| 联系 | 邮件、QQ、GitHub、简历 | 让招聘方用最顺手的方式找到我 |

**刻意去掉的东西**：服务内容介绍、创始人卡片、客户案例包装、首页数据看板。原因很直接——它们会让人误以为这是一家外包公司在招揽生意，而我在找工作。数据看板展示的是「这个网站有多少条反馈」，和我的开发能力没有关系，留在首页只会分散注意力。

---

## 项目数据（都能在仓库里跑出来）

| 项目 | 测试用例 | CI | 说明 |
|---|---|---|---|
| InsightAgent | 44 | 通过 | 规则解析 + 只读 SQL 执行，四道安全约束 |
| CodeFix-Agent | 11 | 通过 | AST 检测 4 类问题，修复后子进程跑 pytest 验证 |
| TestPilot | 23 | 通过 | 六层测试流水线，执行层可替换 |
| 合计 | **78** | — | 页面上的数字与仓库一致 |

TestPilot 的真实浏览器执行与 LLM 用例生成属于后续计划，**当前版本未接入**，页面与简历都如实标注了这一点。

---

## 三个技术点

### 1. 没有服务器，也要有完整的提交流程

站内的技术反馈链路完全跑在 GitHub 上：

```
访客提交 → Issue Forms（服务端校验必填项 + 自动打标）
         → Issue 落库（描述 / 复现步骤 / 环境 / 状态）
         → Actions 定时聚合，生成 stats.json 快照
         → admin.html 读取、搜索、标记、回复、关闭
```

- **鉴权**：Fine-grained PAT，只授权本仓库 Issues 读写，存在管理台的 `sessionStorage`，关页即失效。
- **写入凭证不落前端**：访客只能提交 Issue，令牌只存在于管理台。这是纯静态站绕不过去的边界——把写入凭证放前端等于把仓库交给任何打开控制台的人。
- **成本**：Pages 托管、Issues 存储、REST API、Actions 额度都在免费范围内，0 元。

首页已经不再展示数据看板，但这条链路和管理台仍然可用，作为「用 GitHub 当后端」的实现记录。

### 2. 3D 实验页

`lab.html` 是独立的视觉实验：Three.js 粒子网络、流星、Bloom 辉光，配 ACES 色调映射与阅读遮罩。

它和主页刻意分开：主页用 Apple 风格的克制排版（白底、大字号、留白、胶囊按钮），证明我能做**能读的界面**；实验页证明我能写 **WebGL 与后处理**。这两件事面试官是分开打分的。

### 3. 加 QQ 弹窗

没有后端，号码直接写在页面上，弹窗提供「复制 QQ 号」和「在 QQ 中打开」：

- 桌面：`tencent://AddContact/?fromId=45&...`
- 移动：`mqqwpa://im/chat?chat_type=wpa&...`

Scheme 唤起受浏览器与客户端版本影响，Safari 和微信内置浏览器多会拦截，所以「复制号码」是主路径，唤起只是加速。页面文案按这个前提写，不承诺一定弹窗。

---

## 目录

```
index.html            主页（作品集）
lab.html              3D 实验页
admin.html            反馈管理台（PAT 存 sessionStorage）
resume.pdf            简历，由 _resume_print.html 打印导出
_resume_print.html    简历的 A4 打印源文件
stats.json            Actions 生成的反馈统计快照
avatar.jpg            证件照，目前只在简历里用，页面未引用
.github/              Issue Forms 与 Actions 工作流
```

`resume.pdf` 的源文件是 `_resume_print.html`（A4 打印样式）。改完内容后重新导出：

```bash
msedge --headless --no-pdf-header-footer --print-to-pdf=resume.pdf _resume_print.html
```

---

## 本地预览

```bash
python -m http.server 8000
# 打开 http://localhost:8000
```

直接双击 `index.html` 也能看，但 `resume.pdf` 与管理台的相对路径行为以本地服务器为准。

---

## 已知限制

| 限制 | 说明 |
|---|---|
| 没有后端进程 | 反馈只能走 GitHub，访客需有 GitHub 账号 |
| 管理台令牌在前端 | `sessionStorage` 只能降低暴露面，不能防 XSS |
| QQ 号明文 | 会被爬虫抓取，可能收到垃圾好友申请 |
| 3D 页性能 | 移动端粒子数已下调，低端机仍可能掉帧 |
