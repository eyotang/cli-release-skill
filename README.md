# CLI Release Skill

命令行工具开发与发布 Agent Skill：**npm/npx 首次安装引导 → 原生命令检查 GitHub Release → 用户确认 → 校验并替换 → 跨平台验证与发布**。

Native CLI development and delivery with npm bootstrap installation, consent-based GitHub Release updates, and verified cross-platform releases.

## 安装 / Install

```bash
npx skills add eyotang/cli-release-skill --skill cli-release
```

安装到 Codex（全局）：

```bash
npx skills add eyotang/cli-release-skill --skill cli-release -a codex -g
```

## 使用 / Use

```text
使用 $cli-release，为当前命令行工具实现首次安装和原生自动更新，
每次启动检测 GitHub Release，1.5 秒超时跳过；确认后才更新。
```

```text
Use $cli-release to verify this native CLI's installation and self-update
flow, then publish a new release after the relevant checks pass.
```

## 默认行为 / Defaults

- npm/npx 用于首次安装；后续更新由原生命令完成。
- 每次启动检测 GitHub Release；检测总时限 1.5 秒，超时或失败继续执行。
- 交互终端中仅明确输入 `y` / `yes` 才更新；脚本运行不等待输入。
- 按系统和架构选择产物，校验 SHA-256，替换失败时恢复旧程序。
- 更新成功后提示重新运行原命令；配置和凭据继续保留。
- 私有仓库复用已有 GitHub 授权；不会把凭据打进 npm 包。
- 测试、提交、标签、构建和 Release 交付形成可核验的闭环。

已有发布渠道和用户明确选择优先。该 skill 是实施工作规范，不是预制安装器，也不会仅因被加载就授权发布。Go 实现说明按需加载；其他语言保留相同的行为约定。

## 内容 / Contents

- [SKILL.md](skills/cli-release/SKILL.md)：开发、安装、更新和发布流程。
- [Go implementation notes](skills/cli-release/references/go.md)：版本注入、GitHub API、平台命名、替换和 CI 实施要点。

## skills.sh

根据 [官方 FAQ](https://www.skills.sh/docs/faq)，技能托管在 GitHub，用户通过官方 CLI 安装后由平台自动收录和排名；不需要额外上传，也不保证即时展示。

## License

[MIT](LICENSE).
