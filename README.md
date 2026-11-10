# Feature Knowledge Base

人工准备 `urls.json` 和结构型 `profile.json`，通过通用流水线生成官方 Feature KB，供 DialectRanger 前端浏览。仓库含独立运行代码、15 库输入、六库现有数据和 GitHub Actions；不依赖父目录的 FeatureCrawler 项目。

## 目录

```text
.github/workflows/                  SQLite 更新与网页部署
script/FeatureKnowledgeBase/
    run_feature_crawler.py          build / publish / export-web / serve
    crawler_cli.py                  原始 Unified 引擎入口
    ci_update.py                    自动更新：隔离构建、完整性校验、再发布
    agents/                        Generic Pipeline
    inputs/<db>/urls.json           人工提供的官方文档集合
    inputs/<db>/profile.json        结构执行配置
    tests/                         离线测试
data/FeatureKnowledgeBase/
    catalog.json                   网页唯一数据注册表
    <db>/final.json                正式 Feature
    <db>/review.json               独立待审记录
    <db>/summary.json              统计
web/                               静态前端，按数据库切换
```

现有 Final：MySQL 483、PostgreSQL 781、SQLite 225、DuckDB 716、MonetDB 294、Derby 110，共 2609；这些是现有结果数量，不是官方覆盖率认证。未注册的历史目录不会进入网页。

## 上传到新仓库

**将本目录的内容作为新仓库根目录，不要再套一层 `src/`。** 使用新建的空 GitHub 仓库，默认分支为 `main`，不要预先初始化远程 README。

```powershell
Set-Location 'D:\PyCharm 2025.1.2\code\FeatureCrawler\src'
git init -b main
git add .
git status --short
git commit -m "Initial Feature KB pipeline and website"
git remote add origin https://github.com/YOUR_ACCOUNT/YOUR_NEW_REPO.git
git push -u origin main
```

`.gitignore` 排除 `.runtime`、缓存、虚拟环境和 `.env`。不要用网页上传整个含缓存的文件夹；用以上 Git 命令即可。无需上传或配置模型 Key。

## 启动 SQLite 自动抓取

1. GitHub **Settings → Actions → General → Workflow permissions**：启用 **Read and write permissions** 和 **Allow GitHub Actions to create and approve pull requests**，保存。组织策略可能需要管理员允许。
2. **Settings → Pages → Build and deployment → Source → GitHub Actions**。
3. **Actions → Update SQLite Feature KB → Run workflow**，第一次手动触发。
4. 完整抓取和校验成功后，机器人创建或更新一个 SQLite 数据 PR。检查后合并，网页部署自动执行。这里只要求合并数据 PR，不要求手动运行爬虫。
5. 此后每周日北京时间 **02:17** 自动再次抓取官方文档，沿用同一更新 PR；不会直接覆盖主分支。

首次上传已包含网页数据。开启 Pages 后可手动运行 **Deploy Feature KB Website**，无需等待新一轮爬取。

网页通常为 `https://YOUR_ACCOUNT.github.io/YOUR_NEW_REPO/`，以部署工作流输出的地址为准。定时任务只在默认分支执行，可能延迟；公开仓库长期无活动时 GitHub 可能停用定时任务，需要重新启用。

### 更新保护

自动流程每次在线读取 SQLite 的现有输入，使用全新运行目录，不读取旧抓取缓存。Semantic 固定为 `deterministic`，本地处理仍完整执行；在线模型客户端在启动前被禁用，无模型 API Key 需求。

只有输入页面全部抓取成功、未截断、外部模型调用数明确为 0、lineage/投影校验通过、Final 非空且无重复 ID 时，才导出数据。相对已发布数量下降超过 10% 时，保留主分支旧数据并报告失败，等待检查。这个保护不等于官方 Recall/Precision 认证。Review 与 Final 始终分开。

失败详情在工作流日志和 `sqlite-update-diagnostics` Artifact 中。未提供付费模型，不代表 GitHub Actions 的运行额度无限；使用仓库对应的 Actions/Pages 额度。

## 本地运行

建议 Python 3.11；原引擎也曾在 Python 3.8 验证。

```powershell
python -m pip install -r script/FeatureKnowledgeBase/requirements.txt
$env:FEATURE_CRAWLER_HTTP_TRUST_ENV = '0'
python script/FeatureKnowledgeBase/run_feature_crawler.py build --dbms sqlite --semantic-provider deterministic
python script/FeatureKnowledgeBase/run_feature_crawler.py serve
```

浏览 `http://127.0.0.1:8771/`。普通 build 运行完成后直接更新 `data` 和网页；需要自动更新保护时，改用 `python script/FeatureKnowledgeBase/ci_update.py --dbms sqlite`。

代码与配置以当前输入为准，官方字段不存在时保留空值。不会为了字段完整而编造内容。

## 验证

```powershell
python -m pip install pytest
Set-Location script/FeatureKnowledgeBase
python -m pytest tests/test_dialectranger_integration.py tests/test_ci_update.py tests/test_profile_identity_type_precedence.py -q
```

新增更新保护/适配及身份测试共 20 项通过；独立引擎入口、输入、发布和离线端到端测试另有 31 项通过，自动工作流会运行这 51 项。这里只报告选定测试，未声称所有历史测试全部通过。这些测试禁止真实网络。已有 SQLite 缓存端到端验证为 225 Final、91 Review、网页投影 0 loss；GitHub 在线工作流需在上传后实际执行，不能将本地缓存验证当作 GitHub 在线成功。

GitHub 官方说明：[工作流定时触发](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)、[自定义 Pages 部署](https://docs.github.com/en/pages/getting-started-with-github-pages/using-custom-workflows-with-github-pages)。
