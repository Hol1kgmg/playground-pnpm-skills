# pnpm agent skills 検証レポート

- 検証日: 2026-11-03
- 環境: pnpm 12.16.1 / macOS / Claude Code
- 参考: <https://pnpm.io/ja/agent-skills>

## 結論

**pnpm agent skills を使うと、agent skill を npm パッケージと同じようにバージョン単位で受け取り、Dependabot などで自動更新する仕組みを作れる。**

- スキルは npm パッケージに同梱されて配布され、`package.json` / `pnpm-lock.yaml` でバージョン管理される
- `pnpm install` だけで、ロックされたバージョンのスキルがエージェント用ディレクトリにリンクされる
- 依存の一つとして扱われるため、Dependabot などの依存更新ツールで新バージョンを検知・PR 化できる

前提として、スキルは **npm tarball に `skills/` を含めて公開されたパッケージ** である必要がある。任意の GitHub リポジトリからスキルを取ってくる汎用インストーラではない。

## 仕組み

```
[配布側]                         [利用側]
@scope/skills@1.2.0              package.json      ← バージョン範囲を宣言
└── skills/<name>/SKILL.md  ──▶  pnpm-lock.yaml    ← バージョンを固定
        (npm publish)            pnpm-workspace.yaml ← permissions で承認
                                       │ pnpm install
                                       ▼
                                 .agents/skills/pnpm-@scope+skills-<name>  (symlink)
                                       ▲
                                 Dependabot が新バージョンを PR 化
```

| npm パッケージ | agent skill（pnpm skills） |
|---|---|
| `package.json` で依存宣言 | 同じ（`devDependencies` に追加） |
| `pnpm-lock.yaml` でバージョン固定 | 同じ |
| `pnpm install` で `node_modules` に展開 | 加えて `.agents/skills` 等にリンク |
| Dependabot / Renovate で更新 PR | 同じ |
| `allowBuilds` でビルドスクリプトを許可 | `permissions.<pkg>.skills` でスキルを許可 |

## 検証結果

| 項目 | 内容 | 検証 |
|---|---|---|
| バージョン単位での導入 | `pnpm add -D @hol1kgmg/skills` → `pnpm approve` でリンク | ✅ |
| 非対話での承認 | `pnpm approve <pkg>` | ✅ |
| 承認の宣言的記録 | `pnpm-workspace.yaml` の `permissions.<pkg>.skills: true` | ✅ |
| エージェントからの認識 | Claude Code がリンクされたスキルを即座に認識 | ✅ |
| リンクの Git 除外 | pnpm が `.agents/skills/.gitignore`（`pnpm-*`）を自動生成 | ✅ |
| バージョン固定 | `package.json` の範囲指定 + `pnpm-lock.yaml`。`-E` で完全固定 | ✅ |
| Dependabot による自動更新 PR | `package-ecosystem: npm` で通常の依存として扱う | ⚠️ 未検証 |

### 導入手順（実施内容）

```bash
pnpm init
pnpm add -D @hol1kgmg/skills
# => Agent skills awaiting approval: @hol1kgmg/skills.
pnpm approve @hol1kgmg/skills
```

結果:

```
.agents/skills/
├── .gitignore                                  # pnpm-*（自動生成）
├── pnpm-@hol1kgmg+skills-dependabot-review -> ../../node_modules/.pnpm/.../skills/dependabot-review
└── pnpm-@hol1kgmg+skills-speckit-explain   -> ../../node_modules/.pnpm/.../skills/speckit-explain
```

`pnpm-workspace.yaml`:

```yaml
minimumReleaseAgeExclude:
  - '@hol1kgmg/skills@1.0.0'   # 公開直後のため自動追加
permissions:
  '@hol1kgmg/skills':
    skills: true
```

### 自動更新の構成例（Dependabot）

```yaml
# .github/dependabot.yml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
```

更新 PR をマージ後、各環境で `pnpm install` を実行すると新バージョンのスキルがリンクされる。

### 補足: 更新の反映を自動化する（direnv）

スキルの更新反映に必要なのは `git pull` 後の `pnpm install` だけだが、毎回手動で実行するのは手間で、忘れると古いバージョンのスキルが使われ続ける。
[direnv](https://direnv.net/) を使うと、lockfile の変更を検知して自動で `pnpm install` を実行できる。

```bash
# .envrc
watch_file pnpm-lock.yaml
pnpm install --frozen-lockfile --silent
```

- `watch_file` により、`pnpm-lock.yaml` が変わると次のプロンプトで `.envrc` が再評価される
- リポジトリに `cd` したときにも実行されるため、常に lockfile 通りのスキルがリンクされた状態になる
- 初回および `.envrc` 変更時に `direnv allow` が必要

```
Dependabot PR マージ → git pull → (direnv が lockfile 変更を検知) → pnpm install → スキル更新
```

> 代替: git hook（lefthook の `post-merge` / `post-checkout` 等）で `pnpm install` を実行しても同様の効果が得られる。`git pull` 以外で lockfile が変わるケースまで追従したい場合は direnv が向く。

## 制約

| 項目 | 内容 |
|---|---|
| 配布形態 | npm tarball に `skills/` が含まれていなければ対象外（GitHub 上のスキルは直接取れない） |
| エコシステムの対応状況 | 主要ライブラリは現時点で未対応（下表） |
| global / `pnpm dlx` / `pnpm deploy` | スキルはリンクされない |
| `pnpm install --prod` | devDependencies のスキルはリンクされない |
| 承認の粒度 | パッケージ単位で以降の全バージョンに適用。更新時の再承認はない |

### 主要パッケージの対応状況

| パッケージ | リポジトリに skills | npm tarball に skills | pnpm skills で取得 |
|---|---|---|---|
| `shadcn@4.23.0` | ✅ ルート `skills/` | ❌ | ❌ |
| `next@16.4.4` / `16.5.0-canary.38` | ✅ ルート `skills/`（Claude Code プラグイン形式） | ❌ | ❌ |
| `drizzle-kit@0.31.11` / `1.0.0-beta.22`（公式ドキュメントの例） | - | ❌ | ❌ |
| `@hol1kgmg/skills@1.0.0` | ✅ | ✅（`files: ["skills/"]`） | ✅ |

## 運用上の注意

- **更新時のレビュー**: 承認が以降の全バージョンに及ぶため、Dependabot PR の差分（`SKILL.md` やスクリプト）が実質的なレビューポイントになる。
- **minimumReleaseAge**: 公開直後のバージョンは導入が保留されうる。`minimumReleaseAgeExclude` に自動追加される。
- **lockfile 互換性**: pnpm 12 の lockfile（`lockfileVersion: '9.0'`、先頭が `---` の YAML）を Dependabot が正しく更新できるかは要検証。
- **シンボリックリンク構成**: 本リポジトリでは `.claude/skills -> ../.agents/skills` のため、リンクは 1 箇所にのみ作成される。

## スキル配布側の要件

```
my-package/
├── package.json        # "files": ["skills/"] を含める
└── skills/
    └── <name>/
        └── SKILL.md
```

npm に公開すれば、利用側は `pnpm add` + `pnpm approve` で導入できる。リンク名は `pnpm-<package>-<skill>`（スコープの `/` は `+`）。

## 未検証事項

- Dependabot による更新 PR の作成と lockfile 更新
- direnv による更新反映の自動化
- git 依存 / tarball 依存でのスキル導入
- `skills.dirs` 設定によるリンク先の変更
- 別環境（CI 等）での再現
