# CLAUDE.md

本文件为在此仓库工作的 Claude Code (claude.ai/code) 提供指引。

## 此仓库在本机的实际用途

这里是 Anthropic 官方插件 marketplace 的**本地克隆**，仓库主把它当作**个人插件管理工作台**使用，**不是**作为上游贡献者。在此运行的 Claude 实例扮演**插件顾问**，不是 marketplace 维护者。具体行为：

- 用户描述需求时，主动检索 `.claude-plugin/marketplace.json`，给出 1–3 个候选并附安装命令 `/plugin install <name>@claude-plugins-official`。
- 用户选定插件后，给出最小部署路径（安装 → 必要的配置/Key → 验证），不要丢一堆链接让用户自己读。
- 安装/使用中遇到的坑、非显然的发现，写入 memory（命名形如 `issue_<plugin>_<tag>.md`、`finding_<plugin>.md`），方便后续会话引用。
- 对 `plugins/`、`external_plugins/` 下本地副本的改动是为用户自用，不必每次都跑 `validate-marketplace` / sort 检查——那些是为下面"上游 PR 路径"准备的。

## 此仓库客观上是什么

一份 Claude Code 插件的精选目录。仓库本身**不是**应用代码，而是一份清单加上配套工具。唯一权威的产物是 `.claude-plugin/marketplace.json`；其余一切（校验脚本、CI、`plugins/` 与 `external_plugins/` 下内置的插件）都是为了让这份清单保持正确和最新。

向上游贡献的外部 PR 会被 `.github/workflows/close-external-prs.yml` 自动关闭——只有有 `admin`/`write` 权限的协作者能合入。第三方走 `README.md` 中链接的提交表单。（这些约束都不影响本地副本。）

## marketplace.json —— 唯一权威清单

`.claude-plugin/marketplace.json` 列出每一个插件。**插件必须按 `name` 字母序排列（大小写不敏感）**——CI 会强制检查。每条至少需要 `name`、`description`、`source`。`source` 字段在文件中混用三种形态：

- `"./plugins/foo"` —— 字符串型本地路径，用于本仓库内置的插件（通常是 `plugins/` 下 Anthropic 自家的）。
- `{"source": "url", "url": "...", "sha": "..."}` —— 整个上游仓库，可选 SHA 钉版。
- `{"source": "git-subdir", "url": "...", "path": "...", "ref": "...", "sha": "..."}` —— 上游仓库的某子目录，通常钉到 tag/branch + SHA。

`plugins/` 与 `external_plugins/` 目录里存放着部分插件的本地副本，但 **`marketplace.json` 中大多数条目指向的是上游 Git URL**，而不是本仓库里的目录。不要假设清单里列的插件本仓库一定有对应文件夹。

## 常用命令

所有脚本由 [Bun](https://bun.sh)（TypeScript）或 Python 3 运行——根目录没有 `package.json`，也无需在根目录 `npm install`。在仓库根目录执行：

```bash
# 校验 marketplace.json 结构（必填字段、无重名）
bun .github/scripts/validate-marketplace.ts .claude-plugin/marketplace.json

# 检查插件是否按字母序排列；--fix 可原地排序
bun .github/scripts/check-marketplace-sorted.ts
bun .github/scripts/check-marketplace-sorted.ts --fix

# 校验 agents/skills/commands 的 YAML frontmatter。需要 `yaml` 包
# （CI 用 `cd .github/scripts && bun install yaml` 安装）。
bun .github/scripts/validate-frontmatter.ts                       # 扫描当前目录
bun .github/scripts/validate-frontmatter.ts plugins/foo            # 扫描指定目录
bun .github/scripts/validate-frontmatter.ts path/to/SKILL.md ...   # 指定文件

# 发现已过期的 SHA 钉，可选地原地改写 marketplace.json。
# 需要已认证的 GitHub CLI (`gh`)。每周的 bump 工作流就是用它。
python3 .github/scripts/discover_bumps.py --dry-run
python3 .github/scripts/discover_bumps.py --plugin some-plugin --max 5
```

如果你**确实**在为上游准备 PR 而修改 `marketplace.json`，提交前跑 `validate-marketplace.ts` 与 `check-marketplace-sorted.ts`——这正是 CI 上拦 PR 的两道检查。本机自用编辑则不必每次都跑。

## CI 工作流（`.github/workflows/`）

- **`validate-marketplace.yml`** —— 当 PR 修改 `.claude-plugin/marketplace.json` 时，跑上面那两个 `bun` 脚本。
- **`validate-frontmatter.yml`** —— 对改动到的 `agents/*.md`、`skills/*/SKILL.md`、`commands/*.md` 跑 `validate-frontmatter.ts`。fork PR 直接跳过（反正会被自动关闭）。
- **`bump-plugin-shas.yml`** —— 每周一 07:23 UTC cron。用 `discover_bumps.py` 找出上游已经走在钉版之前的插件，把所有更新打包成单个带 `sha-bump` 标签的 PR；若已存在开着的 `sha-bump` PR 则跳过。借助 GitHub App `2812036` 创建 PR（默认的 `GITHUB_TOKEN` 因组织策略无法创建 PR）。
- **`close-external-prs.yml`** —— 在每个新开 PR 上跑；若作者无 write 权限，则用模板评论关闭。

## 插件文件约定（编辑 `plugins/` 或 `external_plugins/` 时）

frontmatter 校验器靠路径区分文件类型：

- `**/agents/*.md` —— agent 定义；frontmatter 必须含 `name` 与 `description`。位于 `skills/<name>/` 下的文件即便路径里再次出现 `agents/`，也**不**视为 agent——那是 skill 自己的内容。
- `**/skills/<name>/SKILL.md` —— skill；必须含 `description`（或老式的 `when_to_use`）。
- `**/commands/*.md` —— 斜杠命令；必须含 `description`。这是老格式；按 `plugins/example-plugin/README.md` 所述，新插件应优先用 `skills/<name>/SKILL.md`。

frontmatter 中含 YAML 特殊字符（`{} [] * & # ! | > % @ \``）的值需要加引号；校验器会在解析前自动加引号，但保险起见落盘也写引号形式。

`plugins/example-plugin/` 是内置插件的标准参考布局（`.claude-plugin/plugin.json` + `commands/` + `skills/`）。
