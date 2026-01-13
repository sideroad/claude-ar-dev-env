# AR開発環境ドキュメント

## 3つの環境

| 環境 | 役割 | Claude Code | CLAUDE.md |
|------|------|-------------|-----------|
| ローカルPC | コントロールEC2の初期構築 | ✅ | ❌ 不要 |
| コントロールEC2 | 案件EC2の管理 | ✅ | ✅ 必要 |
| 作業用EC2 | 開発・調査作業 | ✅ | ✅ 必要 |

---

## 構成概要

```
インターネット
    │
    └── Tailscale VPN
            │
            ├── Beam Pro (Tailscale)
            │     ├── Termius → tmux操作
            │     └── Chrome → アプリ確認 / code-server（VS Code）
            │
            └── コントロールEC2 (t4g.micro/常時起動)
                │   ├── Tailscale IP: 100.x.x.x
                │   ├── SSHポート: 閉鎖（Tailscale経由のみ）
                │   │
                │   └── tmux [ctrl]
                │         ├── window 1: 管理シェル
                │         ├── window 2: Acme EC2 へSSH（VPC内部）
                │         └── window 3: Beta EC2 へSSH（VPC内部）
                │
                └── VPC内部ネットワーク
                      │
                      ├── Acme EC2 (t4g.small)
                      │     ├── SSHポート: コントロールEC2のみ許可
                      │     └── Cloudflare Tunnel → アプリ / code-server
                      │
                      └── Beta EC2 (t4g.small)
                            ├── SSHポート: コントロールEC2のみ許可
                            └── Cloudflare Tunnel → アプリ / code-server
```

---

## セキュリティ

| EC2 | SSHアクセス | Elastic IP |
|-----|-------------|------------|
| コントロールEC2 | Tailscale経由のみ | ❌ 不要（Tailscale IP使用） |
| 作業用EC2 | コントロールEC2からのみ | ❌ なし |

**インターネットから直接SSHできるEC2は存在しません。**

---

## セットアップフロー

```
【Phase 1: 初回のみ（ローカルPCから）】
1. 02-control-ec2-setup.md を Claude Code に渡す
2. コントロールEC2 + Tailscale が設定される
3. 03, 04 のドキュメントをコントロールEC2に転送

【Phase 2: 案件追加（コントロールEC2から）】
1. /project:new-project を実行
2. 03-claude-code-new-project-setup.md が読み込まれる
3. 案件EC2が作成される（VPC内部のみアクセス可能）

【Phase 3: 日常作業（作業用EC2で）】
1. Beam Pro の Tailscale → コントロールEC2に接続
2. tmux window で案件EC2に切り替え
3. 作業用EC2内の Claude Code で開発・調査
```

---

## ファイル配置

### ローカルPC（初回のみ使用）

```
~/ar-dev-setup/
├── 02-control-ec2-setup.md              # これを Claude Code に渡す
├── 03-claude-code-new-project-setup.md  # コントロールEC2に転送
└── 04-claude-code-operations.md         # コントロールEC2に転送
```

### コントロールEC2

```
~/
├── CLAUDE.md                       # コントロール用ルール
├── .claude/commands/
│   ├── new-project.md              # /project:new-project
│   └── delete-project.md           # /project:delete-project
├── docs/
│   ├── 03-claude-code-new-project-setup.md
│   └── 04-claude-code-operations.md
├── bin/                            # シェルスクリプト
│   ├── list-projects
│   ├── start-project
│   ├── stop-project
│   ├── start-all
│   └── stop-all
└── .ssh/                           # 案件用SSH鍵
    └── ar-dev-*-key.pem
```

### 作業用EC2

```
~/
├── CLAUDE.md                       # 作業用ルール
├── .claude/commands/
│   ├── dev-restart.md              # /project:dev-restart
│   ├── tunnel-url.md               # /project:tunnel-url
│   └── pr-create.md                # /project:pr-create
├── .config/code-server/
│   └── config.yaml                 # code-server設定（パスワード等）
└── project/                        # 案件リポジトリ
```

### tmux構成（作業用EC2）

```
Acme-dev セッション（各Windowはフルスクリーン Claude Code）
├── Window 1 [main]           ← Alt+Shift+[ / ]
├── Window 2 [feature/login]  ← で切り替え
└── Window 3 [feature/payment]

バックグラウンド（見えない）:
├── main:           dev:3000 + tunnel
├── feature/login:  dev:3010 + tunnel
└── feature/payment: dev:3020 + tunnel

サービス管理: start-services / stop-services / get-urls
```

---

## tmux操作（Beam Pro対応）

ネストされたtmuxセッションを快適に操作するため、Alt キーベースのショートカットを設定しています。

### コントロールEC2
```
Alt + ]    → 次のwindow（案件切り替え）
Alt + [    → 前のwindow
```

### 作業用EC2
```
Alt + Shift + ]  → 次のwindow
Alt + Shift + [  → 前のwindow
```

> コントロールEC2と作業用EC2で異なるキーを使うことで、ネストした状態でも正しいtmuxを操作できます。

---

## カスタムコマンド一覧

### コントロールEC2

| コマンド | 用途 |
|----------|------|
| `/project:new-project` | 新規案件EC2作成 |
| `/project:delete-project` | 案件EC2削除 |

### 作業用EC2

| コマンド | 用途 |
|----------|------|
| `/project:worktree-new` | ワークツリー作成（Window+サービス込み） |
| `/project:worktree-delete` | ワークツリー削除 |
| `/project:worktree-list` | ワークツリー一覧 |
| `/project:dev-restart` | 開発サーバー再起動 |
| `/project:tunnel-url` | Tunnel URL確認 |
| `/project:pr-create` | PR作成 |

---

## シェルコマンド（コントロールEC2）

```bash
list-projects      # 案件一覧
start-project XX   # 案件XX起動
stop-project XX    # 案件XX停止
start-all          # 全案件起動
stop-all           # 全案件停止
```

---

## ドキュメント一覧

| ファイル | 用途 | 使用場所 |
|----------|------|----------|
| [00-architecture-diagrams.md](./00-architecture-diagrams.md) | 構成図（Mermaid版） | 📖 参照用 |
| [00-architecture-diagrams-d3.html](./00-architecture-diagrams-d3.html) | 構成図（D3.js版・インタラクティブ） | 📖 参照用 |
| [01-human-prerequisites.md](./01-human-prerequisites.md) | 事前作業ガイド | 👤 人（参照） |
| [02-control-ec2-setup.md](./02-control-ec2-setup.md) | コントロールEC2構築 | 🤖 ローカルPC（初回のみ） |
| [03-claude-code-new-project-setup.md](./03-claude-code-new-project-setup.md) | 案件EC2セットアップ | 🤖 コントロールEC2 |
| [04-claude-code-operations.md](./04-claude-code-operations.md) | 運用タスク詳細 | 🤖 コントロールEC2 |
| [05-scenario-guide.md](./05-scenario-guide.md) | シナリオガイド | 📖 参照用 |
| [sprites-dev-comparison.md](./sprites-dev-comparison.md) | Sprites.dev比較調査 | 📖 参照用 |

---

## コスト概算

| 項目 | スペック | 月額 |
|------|----------|------|
| コントロールEC2 | t4g.micro（常時起動） | ~$6 |
| 作業用EC2 × 2 | t4g.small スポット（8時間/日） | ~$2 |
| EBS | 20GB + 30GB × 2 | ~$8 |
| **合計** | | **~$15** |

> 作業用EC2はスポットインスタンス使用（オンデマンド比 80%以上削減）
