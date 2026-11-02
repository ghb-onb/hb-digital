# HB Digital

用 Python 与 Power BI 帮个人和小团队解决实际问题的服务站点。

**在线地址**：https://ghb-onb.github.io/hb-digital/
**管理后台**：https://ghb-onb.github.io/hb-digital/admin.html

---

## 这不只是一个静态宣传页

常见的"个人服务站"止步于展示：内容写死在 HTML 里，访客看完没有任何可提交的入口，站长也无从知道有人来过。

这个仓库把 **GitHub 当作一整套无服务器后端**，用零成本的方式补齐了「提交 → 存储 → 处理 → 统计」的完整闭环：

| 层 | 承担者 | 具体实现 |
|---|---|---|
| 展示层 | GitHub Pages | 无框架静态页面 + Three.js 实时渲染太空背景 |
| 数据层 | GitHub Issues | 每条咨询 = 一条 Issue，自带标签、时间戳、评论、状态机、分页与全文检索 |
| 鉴权层 | Fine-grained PAT | 只授权本仓库 Issues 读写，令牌留存本机浏览器，前端零密钥 |
| 计算层 | GitHub Actions | 新咨询自动分诊打标 + 留受理回执；每 6 小时聚合统计写回仓库 |
| 管理端 | admin.html | 自建控制台，列表 / 筛选 / 搜索 / 状态流转 / 回复并关闭 |

**总成本：0 元。** Pages 托管、Actions 额度（公开仓库免费）、Issues 存储、API 调用，全部落在 GitHub 免费范围内，不需要服务器、不需要数据库、不需要第三方账号。

---

## 一条咨询的完整链路

```
访客点击「提交咨询需求」
  └─> GitHub Issue Forms（.github/ISSUE_TEMPLATE/consult.yml）
        └─> 自动带上「咨询」「待处理」标签
              └─> Actions 触发（.github/workflows/triage.yml）
                    ├─ 幂等创建管理标签
                    ├─ 按正文关键词判定「加急」
                    └─ 留一条受理回执，告知 1 个工作日内回复
                          └─> 数据沉淀为 Issue（结构化正文 + 元数据）
                                ├─> 定时聚合（stats.yml，每 6 小时）
                                │     └─> 生成 stats.json → 首页数据看板实时展示
                                └─> 管理台 admin.html 读取 / 标记 / 回复 / 关闭
```

关键取舍：**首页看板读的是 `stats.json` 静态文件，不是实时 API**。因为未认证的 GitHub API 每小时只有 60 次配额，让每个访客都打一次接口，站点一火就会被限流；改成 Actions 定时聚合、前端只读静态 JSON，就把配额消耗和访问量彻底解耦了。

---

## 目录结构

```
.
├── index.html                          # 首页（展示 + 架构说明 + 数据看板）
├── admin.html                          # 咨询管理台（需令牌，未登录为只读）
├── stats.json                          # Actions 自动生成的聚合数据
├── avatar.jpg                          # 创始人头像
├── .nojekyll                           # 让 Pages 跳过 Jekyll 构建
└── .github/
    ├── ISSUE_TEMPLATE/consult.yml      # 结构化咨询表单（当"前端表单"用）
    └── workflows/
        ├── triage.yml                  # 自动分诊：打标 + 回执
        └── stats.yml                   # 定时统计：聚合 → 提交 stats.json
```

---

## 管理后台使用说明

1. 打开 `admin.html`。
2. 点右上角「登录」，按提示创建一个 **Fine-grained token**：
   - 地址：https://github.com/settings/personal-access-tokens/new
   - Repository access 只勾 `hb-digital`
   - Permissions → **Issues: Read and write**（其他权限一个都不用给）
3. 粘贴令牌，页面会校验它属于 `ghb-onb`，通过即进入可写模式。

未登录时后台是**只读模式**——因为公开仓库的 Issues 本身就能匿名读取，前端也就不需要为"看"这件事做鉴权。

---

## 安全设计

- **前端不含任何密钥。** 令牌只存在使用者本地 `localStorage`，退出即清除，不会发往任何第三方服务器。
- **最小权限。** 令牌仅限单个仓库的 Issues 读写，即使泄露也无法触碰代码、Actions 或账号其他仓库。
- **安全边界在服务端。** 所有写操作都经 GitHub API 鉴权，前端只负责发请求，从不自己判断"能不能改"。
- **防 XSS。** Issue 正文来自外部输入，渲染前统一做 HTML 转义，再套用受限的 Markdown 样式。
- **隐私提示前置。** 表单页明确告知内容会公开可见，要求只留邮箱或微信号。

---

## 已知限制

| 限制 | 原因 | 缓解方式 |
|---|---|---|
| 提交咨询需要 GitHub 账号 | 写 Issue 必须有 GitHub 身份 | 首页同时提供邮件入口作为兜底 |
| 咨询内容公开可见 | Issues 是公开的 | 表单顶部警示，只收集联系方式 |
| Actions 定时任务可能延迟 | GitHub 的 cron 属尽力而为 | 后台不依赖 stats.json，直接读实时 API |

---

## 本地预览

无需构建，直接开静态服务即可（`fetch('stats.json')` 需要 http 协议，`file://` 会被 CORS 拦）：

```bash
python -m http.server 8000
# 打开 http://localhost:8000/
```
