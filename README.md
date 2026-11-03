# HB Digital

用 Python 与 Power BI 帮个人和小团队解决实际问题的服务站点。

**在线地址**：https://ghb-onb.github.io/hb-digital/
**管理后台**：https://ghb-onb.github.io/hb-digital/admin.html

---

## 两个入口，别混在一起

这个站把「找人办事」和「给网站提意见」刻意分成了两条路，因为它们的隐私要求完全相反：

| 入口 | 用途 | 是否公开 | 适合 |
|---|---|---|---|
| **邮件**（`2195154945@qq.com`） | 客户咨询、谈需求、报价 | 私密 | 普通客户，不需要任何账号 |
| **GitHub Issue 表单** | 报告本站 Bug、提功能建议 | 公开 | 开发者、技术同行 |

**为什么不把客户咨询也做成表单？** 纯静态站要让匿名访客往 GitHub 写数据，必须把写入凭证放到前端，等于把仓库交给任何打开控制台的人。这个技术边界绕不过去，所以客户走邮件，公开表单只留给技术反馈——里面不收集任何联系方式。

---

## 技术架构：GitHub 当无服务器后端

| 层 | 承担者 | 具体实现 |
|---|---|---|
| 展示层 | GitHub Pages | 无框架静态页面（主页为 Apple 风格极简排版）+ 独立 3D 实验页（Three.js 太空场景、Bloom 辉光） |
| 数据层 | GitHub Issues | 每条技术反馈 = 一条 Issue，自带标签、时间戳、评论、状态机、分页与检索 |
| 鉴权层 | Fine-grained PAT | 仅授权本仓库 Issues 读写，存在标签页的 `sessionStorage`，关页即失效 |
| 计算层 | Issue Forms + 前端聚合 | 服务端校验必填项并自动打标；看板读静态快照，缺失时匿名聚合兜底 |

**总成本：0 元。** Pages 托管、Issues 存储、REST API、Actions 额度全部在 GitHub 免费范围内。

---

## 一条反馈的链路

```
访客点「🐞 技术反馈」
  └─> Issue Forms（.github/ISSUE_TEMPLATE/feedback.yml）
        ├─ 服务端校验必填项（类型 / 具体描述 / 公开确认）
        └─ 自动打上「反馈」「待处理」标签
              └─> 沉淀为 Issue（描述 + 复现步骤 + 环境 + 状态）
                    ├─> 管理台 admin.html 读取 / 搜索 / 标记 / 回复 / 关闭
                    └─> 首页看板展示统计
```

**看板的数据来源有两条，互为兜底：**

1. 优先读 `stats.json`——由 Actions 定时聚合的静态快照，零 API 消耗；
2. 快照不存在、或 `updatedAt` 不是合法日期时，前端匿名读公开 Issues 自行汇总，并在 `sessionStorage` 缓存 10 分钟。

判定快照是否有效**只看 `updatedAt`，不看 `total`**：真实的「0 条反馈」也是有效数据，会原样展示 0。两条路径都拿不到数据时显示 `—`，不会把未知状态伪装成 0。

这样即使 Actions 暂停、延迟或额度受限，看板也不会变成一块死数字；正常运行时又能把 API 配额消耗降到零。

---

## 目录结构

```
.
├── index.html                          # 首页（展示 + 架构说明 + 数据看板）
├── admin.html                          # 反馈管理台（未登录为只读）
├── stats.json                          # 聚合快照（可选，Actions 或手动生成）
├── avatar.jpg                          # 创始人头像
├── .nojekyll                           # 让 Pages 跳过 Jekyll 构建
├── .github/
│   ├── ISSUE_TEMPLATE/feedback.yml     # 技术反馈表单
│   └── workflows/
│       ├── triage.yml                  # 新反馈自动分诊
│       └── stats.yml                   # 自动聚合并更新 stats.json
```

---

## 管理后台使用说明

1. 打开 `admin.html`，点右上角「登录」。
2. 创建一个 **Fine-grained token**：https://github.com/settings/personal-access-tokens/new
   - Repository access：**只勾 `hb-digital`**
   - Permissions：**只给 Issues: Read and write**，其他一个都不用给
   - 有效期：建议 7–30 天
3. 粘贴令牌，页面会校验它属于 `ghb-onb`，通过后进入可写模式。

令牌保存在 **`sessionStorage`**：关闭标签页即清除，不写入磁盘。两点要说清楚：一是令牌会随每个管理请求发给 GitHub API，二是 `sessionStorage` **只缩小令牌在本机的持久暴露窗口，不能抵御页面运行期间的 XSS**，所以真正的防线是「权限最小 + 有效期短 + 只在自己的设备上登录」。

未登录时后台是**只读模式**——公开仓库的 Issues 本身就能匿名读取，前端不需要为"看"这件事做鉴权。

---

## GitHub Actions 自动化

两个工作流已经放在 GitHub 可识别的 `.github/workflows/` 目录：

- **triage.yml**：新 Issue 只要带「反馈」标签**或以 `[反馈]` 开头**，都会进入分诊；先幂等补齐缺失的管理标签，再按关键词补「加急」，最后留一条受理回执。
- **stats.yml**：在反馈新增、关闭、重新打开或标签变化时重新统计；另外每 6 小时定时统计，也支持手动触发。运行后将聚合结果写入 `stats.json`。

**启用状态：已开启并验证通过**（2026-11-02 手动运行「反馈数据统计」成功，`updatedAt` 已更新为真实时间）。仓库级默认 `GITHUB_TOKEN` 权限是只读，但两个工作流都各自声明了所需权限（分诊 `issues: write`、统计 `issues: read` + `contents: write`），所以不需要改仓库默认设置。若日后换仓库重建，仍需到 **Settings → Actions → General** 确认 Actions 允许运行。

首页看板在快照有效时读取 `stats.json`；快照未生成或无有效时间时，才使用匿名 Issues API 兜底。首次成功运行之后，看板即可优先使用静态快照。

---

## 安全设计

- **前端不含长期凭证。** 令牌只存在使用者当前标签页的 `sessionStorage`，关闭标签页即清除，不写入磁盘、不写进源码。
- **令牌确实会离开浏览器。** 每一次管理请求都会带 `Authorization: Bearer <token>` 发往 GitHub API——这是它能生效的前提，谈不上"令牌不出本机"。`sessionStorage` 的作用只是**缩短令牌在本机的存活时间**、降低共用设备被复用的风险；它**并不缩小令牌随请求外发的范围**，也**无法抵御页面运行期间的 XSS**。真正的防线是下面三条：最小权限、短有效期、只在自己的设备上登录。
- **最小权限。** 令牌仅限单仓库 Issues 读写，即使泄露也碰不到代码、Actions 或账号下其他仓库。
- **有效期短。** 建议 7–30 天，到期即废，不依赖"忘了清理"。
- **安全边界在服务端。** 所有写操作都经 GitHub API 鉴权，前端只发请求，从不自己判定"能不能改"。
- **防 XSS。** Issue 正文来自外部输入，渲染前统一做 HTML 转义，再套受限的 Markdown 样式。
- **公开数据不入私密通道。** 技术反馈表单不含联系方式字段，并在表单顶部明示公开可见。

---

## 已知限制

| 限制 | 原因 | 处理方式 |
|---|---|---|
| 客户咨询不能直接在线提交 | 静态站无法安全代匿名访客写 Issue | 主入口用邮件，私密且可发附件 |
| 技术反馈需要 GitHub 账号 | 写入 Issue 必须有 GitHub 身份 | 明确定位为技术通道，客户走邮件 |
| 看板实时聚合会消耗匿名 API 配额 | 未认证请求 60 次/小时 | 优先读 `stats.json`，并做 10 分钟本地缓存 |
| 快照不看新鲜度只看有效性 | 只校验 `updatedAt` 是否为合法日期，不设过期阈值 | Actions 未启用 / 长时间未跑时，看板显示的是最后一次成功快照 |
| Actions 定时任务可能延迟 | GitHub 的 cron 属尽力而为 | 后台不依赖 `stats.json`，直读实时 API |

---

## 本地预览

无需构建，起个静态服务即可（`fetch` 需要 http 协议，`file://` 会被 CORS 拦）：

```bash
python -m http.server 8000
# 打开 http://localhost:8000/
```
