# playground-pnpm-skills

[pnpm agent skills](https://pnpm.io/ja/agent-skills) の動作検証用リポジトリ。

## Prerequisites

This project uses [mise](https://mise.jdx.dev/) for tool management.

If you haven't installed `mise` yet:

```bash
# Install mise (macOS)
brew install mise
# OR via curl
curl https://mise.run | sh

# Shell integration (for zsh)
echo 'eval "$(mise activate zsh)"' >> ~/.zshrc
source ~/.zshrc
```

## Setup

```bash
mise run setup
```

This installs the tools in `mise.toml` and registers the `lefthook` pre-commit hook (gitleaks secret scan).

## pnpm agent skills の検証

### 仕組み

パッケージは `skills/<name>/SKILL.md` の形でスキルを同梱できる。pnpm は **承認済み** の依存パッケージのスキルだけを、プロジェクトのエージェント用ディレクトリにリンクする（スキルにはエージェントが実行するスクリプトが含まれうるため）。

```
my-package/
└── skills/
    └── <name>/
        └── SKILL.md
```

### 手順

```bash
pnpm install    # 依存をインストールし、承認待ちのスキルを表示
pnpm approve    # 承認待ちスキルをレビューし、判断を記録
```

承認は `pnpm-workspace.yaml` に記録される。承認はパッケージ単位で、以降のバージョンにも適用される。

```yaml
permissions:
  <package-name>:
    skills: true
```

### リンク先

- `skills.dirs` 未設定時は、ワークスペースルートの既存 `.<name>/skills` にリンクされる
  - このリポジトリでは `.claude/skills` と `.agents/skills`
- リンク名は `pnpm-<package>-<skill>`（スコープ付きは `/` → `+`。例: `@acme/kit` の `auth` → `pnpm-@acme+kit-auth`）
- `skills.dirs` に空リストを設定すると agent skills は無効

### 注意

- `pnpm install --prod` では devDependencies のスキルはリンクされない
- グローバルインストール、`pnpm dlx`、`pnpm deploy` ではリンクされない
