# AR開発環境 構築・運用シナリオ

具体的な例を使って、初期構築から日常運用までの流れを説明します。

---

## 目次

1. [Phase 1: 初期構築（1回のみ）](#phase-1-初期構築1回のみ)
2. [Phase 2: 新規案件開始](#phase-2-新規案件開始)
3. [Phase 3: 日常開発](#phase-3-日常開発)
4. [Phase 4: 案件終了](#phase-4-案件終了)

---

## Phase 1: 初期構築（1回のみ）

### シナリオ

```
あなたはフリーランスエンジニア。
AR開発環境を構築して、どこでも開発できるようにしたい。
```

### Day 1: ハードウェア購入

```
【購入リスト】
□ XREAL One Pro（公式サイト: ¥69,800）
□ Beam Pro（公式サイト: ¥26,800）
□ Sony LinkBuds（Amazon: ¥17,000）
□ ProtoArc XK01 TP（Amazon: ¥6,999）
□ モバイルバッテリー 10000mAh（Amazon: ¥3,000）

合計: 約¥124,000
```

### Day 2-3: アカウント・デバイス設定

#### 👤 人の作業（01-human-prerequisites.md を見ながら）

**1. AWSアカウント設定**
```
1. https://aws.amazon.com でアカウント作成
2. ルートユーザーでログイン → MFA設定
3. IAM → ユーザー作成
   - ユーザー名: admin
   - AdministratorAccess をアタッチ
4. 管理者ユーザーでログインし直し → MFA設定
5. ルートユーザーは封印（パスワードは金庫へ）
```

**2. ローカルPC設定**
```bash
# AWS CLI インストール
brew install awscli

# ログイン（管理者IAMユーザーで）
aws login
# ブラウザで認証

# 確認
aws sts get-caller-identity
# → admin ユーザーが表示されればOK
```

**3. Beam Pro 初期設定**
```
1. 電源ON → Googleアカウントでログイン
2. 言語: 日本語、タイムゾーン: 東京
3. Google Playから以下をインストール:
   - Tailscale
   - Termius
   - Gboard
   - Chrome
4. Gboard設定: 日本語フリック入力を有効化
5. Chrome設定: PC版サイトをデフォルトに
```

**4. デバイス接続**
```
1. XREAL One Pro を Beam Pro に USB-C接続
   → 自動で認識、表示開始
2. LinkBuds を Beam Pro に Bluetooth接続
3. ProtoArc キーボードを Beam Pro に Bluetooth接続
```

**5. 他サービスアカウント作成**
```
- GitHub: https://github.com
- Cloudflare: https://cloudflare.com
- Tailscale: https://tailscale.com
```

### Day 4: コントロールEC2構築

**Claude Code にコントロールEC2セットアップを依頼：**

```
👤 あなた:
02-control-ec2-setup.md を読んで、コントロールEC2をセットアップしてください。

🤖 Claude Code:
[EC2作成中...]
[環境構築中...]
[管理スクリプト作成中...]

==========================================
🎉 コントロールEC2セットアップ完了
==========================================
インスタンス: ar-dev-control
Elastic IP: 52.198.xxx.xxx

【管理コマンド】
- list-projects    : 案件一覧
- start-project XX : 案件XX起動
- stop-project XX  : 案件XX停止
- start-all        : 全案件起動
- stop-all         : 全案件停止
==========================================
```

**Termius 設定（1つだけ）：**
```
Label: 🎮 Control
Hostname: 52.198.xxx.xxx
Username: ubuntu
認証: ar-dev-control-key.pem
```

**これで初期構築完了！**

---

## Phase 2: 新規案件開始

### シナリオ

```
Acme社からECサイトリニューアル案件を受注。
Next.js + TypeScript で開発予定。
リポジトリ: github.com/acme-corp/ec-site-renewal
```

### Step 1: コントロールEC2に接続

```
Termius → 🎮 Control → 接続
tmux attach -t ctrl （または start-ctrl）
```

### Step 2: Claude Code で新規案件セットアップを依頼

**コントロールEC2内で claude を起動し、以下を入力:**

```
03-claude-code-new-project-setup.md を読んで、新規案件のEC2をセットアップしてください。
```

### Step 3: Claude Code との対話

```
🤖 Claude Code:
コントロールEC2内で実行していますか？

👤 あなた:
はい

🤖 Claude Code:
新規案件の情報を教えてください：

1. クライアント名（英小文字、例: acme）: 
2. インスタンスタイプ（デフォルト: t4g.small）: 
3. ストレージサイズ（デフォルト: 30GB）: 
4. リージョン（デフォルト: ap-northeast-1）: 

👤 あなた:
1. acme
2. デフォルト
3. デフォルト
4. デフォルト

🤖 Claude Code:
EC2インスタンスを作成します...
[キーペア作成中...]
[セキュリティグループ作成中...]
[EC2起動中...]
[Elastic IP割り当て中...]
[IAMロール作成中...]

==========================================
EC2インスタンス作成完了
==========================================
クライアント: acme
インスタンス名: ar-dev-acme-server
インスタンスID: i-0a1b2c3d4e5f6g7h8
Elastic IP: 54.250.123.45
Private IP: 10.0.1.100
キーペア: ~/.ssh/ar-dev-acme-key.pem

エイリアス追加完了:
$ acme で案件EC2に接続できます

Termius設定:
  Label: 🟢 EC2-Acme-ECサイト
  Hostname: 54.250.123.45
  Username: ubuntu
  Key: ar-dev-acme-key.pem をインポート
==========================================

次のタスク（EC2環境構築）に進みます。
```

### Step 4: 環境構築（自動）

```
🤖 Claude Code:
[SSH接続中...]
[システム更新中...]
[AWS CLI インストール中...]
[Node.js インストール中...]
[tmux インストール中...]
[Claude Code インストール中...]
[Playwright インストール中...]
[GitHub CLI インストール中...]
[cloudflared インストール中...]

==========================================
EC2環境構築完了
==========================================
```

### Step 5: GitHub認証（人の作業が必要）

```
🤖 Claude Code:
==========================================
[要作業] GitHub認証が必要です
==========================================

tmux windowで案件EC2に接続して、以下のコマンドを実行してください：

  Alt + ] でwindow移動、または新規windowで acme と入力して接続
  gh auth login

対話形式で以下を選択：
  1. GitHub.com
  2. SSH
  3. Yes (SSH鍵生成)
  4. Login with a web browser

完了したら「認証完了」と入力してください。
==========================================

👤 あなた:
（tmux windowで案件EC2に接続して gh auth login を実行）
認証完了

🤖 Claude Code:
[認証確認中...]
GitHub認証OK

Gitのユーザー情報を教えてください：
- 名前（例: Taro Yamada）: 
- メールアドレス: 

👤 あなた:
- 名前: Taro Yamada
- メール: taro@example.com

🤖 Claude Code:
[Git設定完了]
```

### Step 6: Cloudflare認証（人の作業が必要）

```
🤖 Claude Code:
==========================================
[要作業] Cloudflare認証が必要です
==========================================

案件EC2のtmux windowで、以下のコマンドを実行してください：

  cloudflared tunnel login

完了したら「認証完了」と入力してください。
==========================================

👤 あなた:
認証完了
```

### Step 7: 自動停止設定

```
🤖 Claude Code:
自動停止機能を有効にしますか？

- はい: SSH切断後3時間で自動停止
- いいえ: 手動停止のみ
- カスタム: 時間を指定

👤 あなた:
はい

🤖 Claude Code:
[自動停止スクリプト作成中...]
[systemdサービス登録中...]

==========================================
自動停止機能セットアップ完了
==========================================
設定:
  - 閾値: SSH切断後3時間
==========================================
```

### Step 8: tmux window設定

```
🤖 Claude Code:
==========================================
[要作業] tmux windowに案件を追加
==========================================

コントロールEC2のtmux内で：

1. Alt + ] でwindowを移動（まだの場合は新規作成）

2. エイリアスで案件EC2に接続
   $ acme

完了したら「完了」と入力してください。
==========================================

👤 あなた:
完了
```

### Step 9: 動作確認

```
🤖 Claude Code:
[テストプロジェクト作成中...]
[tmuxセッション作成中...]

==========================================
[要確認] 動作確認をお願いします
==========================================

1. コントロールEC2から Alt + ] で案件EC2のwindowに切り替え
2. tmux attach -t "Acme-dev" でセッション接続
3. 右ペインに表示される Cloudflare Tunnel URL をコピー
4. Beam Pro の Chrome でそのURLにアクセス
5. "AR Dev Server Running!" が表示されることを確認

確認できましたか？（はい/いいえ）
==========================================

👤 あなた:
はい

🤖 Claude Code:
==========================================
🎉 新規案件セットアップ完了！
==========================================

【作成されたリソース】
- EC2: ar-dev-acme-server (54.250.123.45)
- セキュリティグループ: ar-dev-acme-sg
- キーペア: ar-dev-acme-key （コントロールEC2の ~/.ssh/ に保存）
- IAMロール: ar-dev-acme-role
- 自動停止: SSH切断後3時間で停止

【接続方法】
- コントロールEC2から: $ acme
- tmux window切り替え: Alt + ] / Alt + [
- tmux: Acme-dev
==========================================
```

### Step 9: 実際のリポジトリをクローン

```
👤 あなた:
Acme社のリポジトリをクローンしてください。
リポジトリ: acme-corp/ec-site-renewal
ブランチ: develop

🤖 Claude Code:
[クローン中...]

gh repo clone acme-corp/ec-site-renewal
cd ec-site-renewal
git checkout develop
pnpm install

==========================================
リポジトリクローン完了
==========================================
プロジェクト: ec-site-renewal
ブランチ: develop
パッケージマネージャー: pnpm
==========================================
```

---

## Phase 3: 日常開発

### シナリオ

```
毎日の開発作業。
今日はログイン機能を実装する。
```

### 朝: 開発開始

**Beam Pro + XREAL を装着してカフェへ**

**1. Termius でコントロールEC2に接続**

```
Termius起動 → 🎮 Control をタップ → 接続
```

**2. tmuxセッションにアタッチ**

```bash
tmux attach -t ctrl
# または
start-ctrl
```

**3. 案件EC2を起動（コントロールEC2内で）**

```bash
# 全案件起動
start-all

# または特定の案件だけ
start-project acme
start-project beta

# 1-2分待機
```

**4. 案件一覧確認**

```bash
list-projects

=== 案件EC2一覧 ===
| Name                 | Client | State   | Private IP   |
| ar-dev-acme-server   | acme   | running | 10.0.1.100   |
| ar-dev-beta-server   | beta   | running | 10.0.1.101   |
```

**5. 案件EC2に接続（tmux window切り替え）**

```bash
# 新規windowで接続（初回）
acme  # エイリアスで接続

# 以降はwindow切り替え
Alt + ]    → 次のwindow
Alt + [    → 前のwindow
```

**6. 案件EC2内でtmuxセッションにアタッチ**

```bash
tmux attach -t Acme-dev
```

**7. サービス起動**

```bash
# バックグラウンドでサービス起動
start-services 3000 ~/project

# Tunnel URL確認
get-urls 3000
```

**8. Tunnel URL をブラウザで開く**

```
get-urls で表示されたApp URLをコピー
Chrome でアクセス
→ XREAL で開発中のサイトが見える！
```

> 💡 **ポイント**: すべてキーボードだけで操作完結。Beam Pro に触る必要なし！

### 午前: 機能実装

**Claude Code に実装を依頼（EC2内で claude コマンド）**

```
👤 あなた:
ログイン機能を実装してください。
要件:
- メールアドレスとパスワードでログイン
- JWT認証
- ログイン状態の永続化

🤖 Claude Code:
[ファイル作成・編集中...]

src/app/login/page.tsx を作成しました
src/lib/auth.ts を作成しました
src/hooks/useAuth.ts を作成しました
...

実装が完了しました。以下のURLで確認できます：
https://random-name-here.trycloudflare.com/login
```

**ブラウザで確認しながら修正**

```
👤 あなた:
ログインボタンの色を青に変更して

🤖 Claude Code:
src/app/login/page.tsx を更新しました
```

### 午後: コミット・PR作成

```
👤 あなた:
変更をコミットしてPRを作成して

🤖 Claude Code:
[ステージング...]
[コミット中...]

git add .
git commit -m "feat: ログイン機能を実装"
git push origin feature/login

PRを作成しますか？

👤 あなた:
はい

🤖 Claude Code:
/project:pr-create "feat: ログイン機能を実装"

PRを作成しました:
https://github.com/acme-corp/ec-site-renewal/pull/42
```

### 夕方: 作業終了

**1. SSH切断**
```
tmux detach（またはCtrl+B, d）でデタッチ
exit（SSH切断）
```

**2. 3時間後に自動停止**
```
EC2は自動停止されるので、手動停止は不要。
（すぐ停止したい場合は Chrome「🔴 EC2停止」）
```

**3. デバイス充電**
```
Beam Pro、LinkBuds、キーボードを充電
```

---

## Phase 4: 案件終了

### シナリオ

```
Acme社のECサイトリニューアル案件が完了。
リソースを削除してコストを削減。
```

### Step 1: 最終確認

```
👤 あなた:
/project:ec2-list

🤖 Claude Code:
案件一覧:
┌──────────────────────┬────────┬─────────────┬─────────┐
│ 名前                  │ Client │ 状態        │ IP      │
├──────────────────────┼────────┼─────────────┼─────────┤
│ ar-dev-acme-server   │ acme   │ stopped     │ 54.250.123.45 │
│ ar-dev-beta-server   │ beta   │ running     │ 13.115.67.89  │
└──────────────────────┴────────┴─────────────┴─────────┘
```

### Step 2: 削除依頼

```
👤 あなた:
04-claude-code-operations.md を参照して、acme案件のリソースを削除してください

🤖 Claude Code:
==========================================
⚠️ 案件削除の確認
==========================================

以下のリソースがすべて削除されます：
  - EC2インスタンス: ar-dev-acme-server
  - Elastic IP
  - セキュリティグループ: ar-dev-acme-sg
  - キーペア: ar-dev-acme-key
  - IAMロール: ar-dev-acme-role

本当に削除しますか？削除する場合はクライアント名「acme」を入力してください:

👤 あなた:
acme

🤖 Claude Code:
=== acme 案件リソース削除開始 ===
✓ Elastic IP解放
✓ インスタンス削除開始: i-0a1b2c3d4e5f6g7h8
✓ インスタンス削除完了
✓ セキュリティグループ削除
✓ キーペア削除
✓ IAMロール削除

==========================================
acme 案件リソース削除完了
==========================================
```

### Step 3: Termius 接続先削除（手動）

```
Beam Pro の Termius で「🟢 EC2-Acme-ECサイト」を削除
```

---

## 複数案件の並行運用

### シナリオ

```
Acme社案件に加えて、Beta社の管理画面開発を受注。
2つの案件を並行して進める。
```

### 新規案件追加

```
👤 あなた:
03-claude-code-new-project-setup.md を読んで、Beta社案件のEC2をセットアップしてください。
クライアント名: beta

🤖 Claude Code:
[セットアップ実行...]

==========================================
EC2インスタンス作成完了
==========================================
クライアント: beta
インスタンス名: ar-dev-beta-server
Elastic IP: 13.115.67.89
==========================================
```

### Termius 接続先

```
🟢 EC2-Acme-ECサイト (54.250.123.45)
🟡 EC2-Beta-管理画面 (13.115.67.89)
```

### 案件切り替え

```
【Acme案件で作業】
1. Termius で 🎮 Control に接続
2. Alt + ] で Acme windowに移動
3. tmux attach -t Acme-dev
4. 作業...

【Beta案件に切り替え】
1. Ctrl+B, d（Acmeをデタッチ）
2. Alt + [ または Alt + ] でBeta windowに移動
3. tmux attach -t Beta-dev
4. 作業...
```

### コスト管理

```
👤 あなた:
/project:ec2-list

🤖 Claude Code:
案件一覧:
┌──────────────────────┬────────┬─────────┬─────────────────┐
│ 名前                  │ Client │ 状態    │ IP              │
├──────────────────────┼────────┼─────────┼─────────────────┤
│ ar-dev-acme-server   │ acme   │ running │ 54.250.123.45   │
│ ar-dev-beta-server   │ beta   │ stopped │ 13.115.67.89    │
└──────────────────────┴────────┴─────────┴─────────────────┘

※ 使用していない案件は自動停止されています
```

---

## タイムライン例（1日の流れ）

```
08:00 ☕ カフェ到着
      Beam Pro + XREAL 装着
      Termius「🎮 Control」に接続
      start-all で全案件EC2起動
      1-2分待機

08:03 💻 Acme案件開始
      Alt + ] → Acme windowに移動
      tmux attach -t Acme-dev
      start-services 3000 ~/project

08:10 💻 開発
      Claude Code で実装
      ブラウザで確認

10:00 🔄 Beta案件に切り替え
      Ctrl+B, d → Acme tmuxデタッチ
      Alt + ] → Beta windowに切り替え
      tmux attach -t Beta-dev
      ※ Acmeは接続維持

12:00 🍱 昼食（tmuxデタッチのみ）
      両方のEC2は起動したまま

13:00 💻 再開
      Alt + ] → Acme windowに戻る
      tmux attach

17:00 📝 PR作成（各案件で）

18:00 🏠 終了
      コントロールEC2: SSH切断
      → 案件EC2: 3時間後（21:00頃）に自動停止

22:00 💤 就寝
      コントロールEC2: 常時起動（$7.6/月）
      案件EC2: 停止中（EBS のみ）
```

---

## 案件切り替えのキー操作まとめ

```
【コントロールEC2 - 案件切り替え】
Alt + ]    → 次のwindow
Alt + [    → 前のwindow

【作業用EC2内 - ブランチ切り替え】
Alt + Shift + ]  → 次のwindow
Alt + Shift + [  → 前のwindow
Ctrl+B, d        → セッションデタッチ
tmux attach      → セッションアタッチ
```

**サービス管理コマンド（作業用EC2内）**
```bash
start-services [ポート] [ディレクトリ]  # サービス起動
stop-services [ポート]                   # サービス停止
get-urls [ポート]                        # Tunnel URL取得
services-status                          # 状態確認
```

---

## トラブルシューティング

### EC2に接続できない

```
👤 あなた:
/project:ec2-list

🤖 Claude Code:
ar-dev-acme-server は stopped です

👤 あなた:
/project:ec2-start acme
```

### Tunnel URLが取得できない

```
👤 あなた:
/project:tunnel-url 3000

🤖 Claude Code:
=== Tunnel URLs (port 3000) ===
App:         https://random-name.trycloudflare.com
code-server: https://another-name.trycloudflare.com
```

### 開発サーバーがエラーで止まった

```
👤 あなた:
/project:dev-restart 3000 ~/project

🤖 Claude Code:
サービスを再起動しました: port 3000
```

### GitHub認証が切れた

```
👤 あなた:
gh auth status

（エラーが出た場合）
gh auth login
```
