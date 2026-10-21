# Jeanphilo Blog

> Jeanphilo 的个人技术与思考日志，使用 Hugo + PaperMod 构建并通过 GitHub Pages 发布。

## 主要特性
- **PaperMod 主题**：默认浅/深色自动切换，支持阅读时长、目录、代码复制和分享按钮（详见 `config.toml`）。
- **多分类内容**：`Linux`、`Python`、`Git Notes`、`Thoughts` 等分类通过 `menu.main` 暴露在顶部导航，归档页、搜索页均已启用。
- **全站搜索**：`content/search/_index.md` 搭配 `outputs.home = ["HTML","RSS","JSON"]` 生成 `index.json`，PaperMod 的客户端搜索开箱即用。
- **CI/CD**：`.github/workflows/hugo.yml` 在推送到 `main` 时构建 `public/` 并部署到 GitHub Pages。
- **写作模板**：`archetypes/default.md`、`docs/std.md` 与 `old_markdown/` 提供 front matter 模板、写作检查清单和历史草稿。

## 环境要求
- Hugo Extended ≥ 0.120（PaperMod 依赖 Extended，CI 通过 `peaceiris/actions-hugo@v3` 安装最新版）。
- Git 2.30+（仓库包含主题子模块）。

确认本地 Hugo 版本：
```bash
hugo version
```

## 快速开始
```bash
git clone --recurse-submodules git@github.com:JeanphiloGong/jeanblog.git
cd jeanblog
hugo server -D
```
- `--recurse-submodules` 会同时拉取 `themes/PaperMod` 与 `themes/ananke`，若忘记可运行 `git submodule update --init --recursive`。
- `hugo server -D` 会连同 `draft` 文章一起预览，默认监听 <http://localhost:1313>。

## 仓库结构速览
```text
.
├── archetypes/          # Hugo front matter 模板
├── content/             # 正式文章，按 posts/<category>/slug.md 组织
├── docs/std.md          # 写作 checklist（优秀技术博客的要素）
├── old_markdown/        # 旧文、周记草稿，可迁移到 content/
├── public/              # Hugo build 结果，CI 会覆盖
├── resources/_gen/      # Hugo 生成的缓存产物
├── themes/              # PaperMod & Ananke 主题子模块
├── config.toml          # 站点配置（导航、参数、搜索）
└── .github/workflows/   # GitHub Pages 部署流水线
```

## 自定义改动
- **菜单链接修正**：`layouts/partials/header.html` 覆盖 PaperMod 的 header 生成逻辑，使用 `.Page.RelPermalink` 避免多语言路径重复（细节见 `layouts/README.md`）。
- **Section 列表顺序**：`layouts/section/list.html` 让 section 列表页先展示子目录（`.Sections`），再分页展示文章（`.RegularPages`）；子目录需有 `_index.md` 才会被识别为 section。
- **主题更新提示**：如升级 PaperMod 模块，请检查以上覆盖模板是否需要同步调整。

## 写作流程
1. **创建草稿**
   ```bash
   hugo new posts/python/my-awesome-topic.md
   ```
   - Hugo 会基于 `archetypes/default.md` 生成 front matter，并默认 `draft: true`。
   - 根据分类落盘到 `posts/<category>/<slug>.md`，可按需创建子目录（如 `posts/git-notes/`）。

2. **补充元信息**
   在文章头部推荐包含：
   ```yaml
   ---
   title: "结构化日志和追踪"
   date: 2025-08-28
   draft: false
   categories: ["Python"]
   tags: ["logging", "observability"]
   description: "结合 logging 与 OpenTelemetry 实现结构化日志，并把 trace 信息写入每条日志。"
   keywords: ["python", "otel", "logging"]
   ---
   ```
   - `categories` / `tags` 驱动菜单和聚合页。
   - `description` 与 `keywords` 用于 SEO 与分享摘要。
   - 将 `draft` 改为 `false` 后文章才会进入正式构建。

3. **遵循写作 checklist**
   - `docs/std.md` 列出了每篇文章应该具备的段落（背景、核心概念、实践步骤、FAQ、最佳实践等）。
   - `old_markdown/` 中的旧稿可作为素材，迁移时记得统一 front matter 与资源路径。

4. **本地预览**
   ```bash
   hugo server -D --disableFastRender
   ```
   - `--disableFastRender` 避免局部缓存导致内容不同步。
   - 追加 `--navigateToChanged` 可在保存后自动跳转到相应页面。

## 构建与发布
- **本地构建**：`hugo --minify`（默认输出到 `public/`）。
- **GitHub Pages CI**：
  - Workflow：`.github/workflows/hugo.yml`
  - 触发：推送到 `main` 或手动 `workflow_dispatch`
  - 步骤：Checkout（含子模块） → 安装 Hugo Extended → `hugo --minify` → 上传 `public/` → `actions/deploy-pages` 发布
- **自托管/调试**：若需把静态文件放到其他服务器，可执行 `hugo --minify -d docs` 或指定任意输出目录。

部署后更新会在几十秒内同步到 GitHub Pages；如需强制刷新可在 Pages 设置里重新触发部署。

## 搜索与归档
- `content/search/_index.md` 定义搜索页面，PaperMod 会读取 `index.json`（由 `outputs.home` 中的 JSON 配置生成）。
- `content/archives/_index.md` 搭配 `params.showAllPostsArchive = true` 即可显示全站归档入口。

## 常见问题
- **主题更新**：`git submodule update --remote --merge themes/PaperMod`
- **构建卡在资源编译**：删除 `resources/_gen/` 后重新运行 `hugo`
- **CI 缺少主题文件**：确认子模块已经提交且 `actions/checkout` 配置了 `submodules: true`（本仓库已设置）

## 许可证

仓库暂未声明开源许可证；若需公开分发，请先补充 `LICENSE` 并确认文章版权归属。
