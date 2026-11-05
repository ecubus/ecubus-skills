# EcuBus Skills

EcuBus-Pro 相关的 Agent Skills 集合，遵循 [Agent Skills](https://agentskills.io/) 格式，可用 [skills CLI](https://github.com/vercel-labs/skills) 安装到 Cursor / Claude Code 等代理。

## 安装

### 从 GitHub（推荐）

仓库推送到 GitHub 后：

```bash
# 安装全部 skills
npx skills add ecubus/ecubus-skills

# 只安装某一个
npx skills add ecubus/ecubus-skills --skill ecubus-cli-test

# 先查看有哪些 skill
npx skills add ecubus/ecubus-skills --list
```

若组织/仓库名不同，把 `ecubus/ecubus-skills` 换成实际 `owner/repo`。

### 从本地路径

开发或未推送远程时：

```bash
npx skills add D:/code/ecubus-skills
# 或
npx skills add D:/code/ecubus-skills --skill ecubus-cli-test
```

### 常用选项

| 选项 | 说明 |
|------|------|
| `-g, --global` | 装到用户目录，而不是当前项目 |
| `-a cursor` | 只装到 Cursor |
| `-y` | 跳过确认 |
| `--all` | 全部 skills 装到全部已检测到的代理 |

```bash
npx skills add ecubus/ecubus-skills --skill ecubus-cli-test -a cursor -g -y
```

## Skills 列表

### ecubus-cli-test

编写 EcuBus-Pro CLI/HIL 测试脚本（`describe`/`test`/`assert`、`Util`、`DiagRequest`、`ecb_cli test/build`）。

**适用场景：** 写/改 `.ecb` 工程里的测试节点脚本、CAN/LIN/UDS 请求响应测试、`ecb_cli test`。

### ecubus-panel

设计/编辑/调试 EcuBus-Pro Panel（自由布局面板）：控件类型与属性、组/标签页容器、DBC/LDF 信号与变量绑定、HTML 控件 `window.panel` API、按钮动作、记住值/初始值、旧版面板迁移。

**适用场景：** 搭建或修复 Panel、直接编辑 `.ecb` 中的 `data.panels`、把控件接到信号/变量、编写 HTML 控件脚本、参考 `panel_showcase` / `panel_html` 示例。

## 仓库结构

```
ecubus-skills/
├── README.md
├── skills.sh.json          # skills.sh 分组元数据（可选）
└── skills/
    ├── ecubus-cli-test/
    │   ├── SKILL.md        # 主说明（必填）
    │   └── patterns.md     # 常用测试模式
    └── ecubus-panel/
        ├── SKILL.md        # Panel 架构、绑定、运行时、HTML、迁移
        ├── reference.md    # 控件字段/默认值/绑定兼容表
        └── patterns.md     # JSON 配方、HTML 脚本、节点脚本联动、排错
```

后续新增 skill：在 `skills/<skill-name>/` 下创建带 YAML frontmatter 的 `SKILL.md`，并更新本 README 与 `skills.sh.json`。

## License

MIT
