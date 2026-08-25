# git-hooks-template

开箱即用的 git hooks 预提交检查模板。基于 **husky v9** + **lint-staged** + **prettier** + **commitlint** + **typecheck**，含 merge 合并跳过保护。

使用此模板创建的项目，每次 `git commit` 会自动：

1. 检测是否在合并中 → 跳过全部检查，避免冲突解决时误伤
2. 对暂存文件运行 prettier 格式化（只改已暂存文件，不改其他）
3. 全量 TypeScript 类型检查（`pnpm typecheck`，可替换为你的命令）
4. 校验 commit message 是否符合 Conventional Commits 格式

## 特性

- **husky v9** — 轻量 git hooks 管理，通过 `core.hooksPath` 注册，不写 `.git/hooks/`
- **lint-staged** — 只对暂存文件跑 prettier，秒级完成
- **commitlint** — 校验 commit message 格式，merge 提交自动跳过
- **merge 跳过** — `MERGE_HEAD` 存在时跳过 pre-commit，避免冲突解决时 lint-staged 误操作
- **pnpm 原生** — 默认 pnpm，如需 npm/yarn 见定制指南
- **退出码传播** — hook 失败 → git 中止提交，不会污染仓库

## 文件结构

```
.
├── .husky/
│   ├── pre-commit          # 提交前：merge 检测 → lint-staged → typecheck
│   └── commit-msg          # 提交前：commitlint 校验 message
├── .lintstagedrc           # lint-staged 配置（默认 prettier 全文件）
├── .prettierrc             # prettier 配置（默认 singleQuote）
├── commitlint.config.cjs   # commitlint 配置（继承 conventional）
├── package.json            # 依赖 + prepare 脚本 + typecheck 占位
└── README.md
```

## 快速开始

```bash
# 1. 使用这个模板创建仓库（GitHub → Use this template）
#    或克隆后重命名

# 2. 安装依赖（自动注册 hooks）
pnpm install

# 3. 在 package.json 中替换 typecheck 脚本为你的检查命令
#    "typecheck": "your-typecheck-command"

# 4. 提交代码体验效果
git add .
git commit -m "feat: 体验 git hooks 自动检查"
```

## 定制指南

### 替换 typecheck

模板的 `package.json` 中 typecheck 为占位命令。请替换为你的项目类型检查命令：

```json
{
  "typecheck": "tsc --noEmit"          # TypeScript
  "typecheck": "cargo check"            # Rust
  "typecheck": "mypy src"               # Python
  "typecheck": "gradle check"           # Gradle
  "typecheck": "echo skip && exit 0"    # 无类型检查，直接跳过
}
```

### 换包管理器（npm / yarn）

如果使用 npm 或 yarn，需要修改三个地方：

1. **package.json** — 删除 `packageManager` 字段
2. **.husky/pre-commit** — 将 `pnpm exec` 改为 `npx`（npm）或 `yarn`（yarn）
3. **.husky/commit-msg** — 同上

示例（npm）：

```sh
npx lint-staged
npm run typecheck
```

### 改 lint 规则

- **lint-staged 配置**（`.lintstagedrc`）：默认 `{ "*": "prettier --ignore-unknown --write" }`，可改为 eslint + prettier
- **prettier 配置**（`.prettierrc`）：按项目需要修改
- **commitlint 规则**（`commitlint.config.cjs`）：默认 `extends: ['@commitlint/config-conventional']`，可自定 rules

### 加 pre-push hook

```bash
echo 'pnpm test' > .husky/pre-push
chmod +x .husky/pre-push
```

### 去掉某环节

直接编辑 `.husky/pre-commit` 删除对应行即可。例如去掉 typecheck：

```sh
pnpm exec lint-staged
# 删掉 pnpm typecheck 这行
```

## 工作原理

1. `pnpm install` 时触发 `prepare = husky`，husky 设置 `core.hooksPath = .husky/_`
2. `git commit` 时，git 通过 `core.hooksPath` 找到 `.husky/_/pre-commit` 执行
3. dispatcher（`.husky/_/pre-commit`）source 同目录下的 `h` 脚本
4. `h` 脚本定位到 `.husky/pre-commit`（用户钩子），以 `sh -e` 执行并透传参数
5. 用户钩子依次执行：merge 检测 → lint-staged → typecheck
6. 任一环节退出码非零 → husky 捕获 → git 中止提交

commit-msg 钩子同理：`git commit` 获取 message 后调用 `.husky/_/commit-msg`，最终执行 `.husky/commit-msg` → `commitlint --edit $1`。

## merge 场景行为

| 场景                       | pre-commit              | commit-msg                             |
| -------------------------- | ----------------------- | -------------------------------------- |
| `git merge feature` 无冲突 | 跳过（MERGE_HEAD 存在） | 放行（commitlint 自动忽略 merge 消息） |
| 冲突解决后 `git commit`    | 跳过（MERGE_HEAD 存在） | 校验 merge 提交的 message              |
| 正常提交                   | 执行                    | 校验                                   |

## 常见问题

### hook 未生效

检查 `core.hooksPath` 是否指向 `.husky/_`：

```bash
git config --get core.hooksPath
# 应输出 .husky/_
```

如果未设置，运行 `pnpm exec husky` 重新注册。

### 如何跳过 hook

```bash
git commit --no-verify -m "紧急修复"
# 跳过所有 pre-commit / commit-msg 检查
```

### commitlint 拒绝了我的消息

commitlint 遵循 Conventional Commits 格式。标准格式：

```
<type>[(scope)][!]: <description>

[body]
```

常见 type：`feat`、`fix`、`chore`、`docs`、`refactor`、`test`、`style`、`perf`。

### lint-staged 修改了文件但未自动暂存

lint-staged 默认会重新暂存 prettier 格式化后的文件。如果未暂存，检查 `.lintstagedrc` 配置是否正确。

## License

MIT
