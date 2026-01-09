# Claude Code 運用タスク集

このドキュメントは日常運用で使用するタスク集です。
必要なタスクを指定して実行を依頼してください。

---

## 目次

1. [案件一覧確認](#タスク-案件一覧確認)
2. [EC2起動](#タスク-ec2起動)
3. [EC2停止](#タスク-ec2停止)
4. [EC2状態確認](#タスク-ec2状態確認)
5. [スポット中断確認・復旧](#タスク-スポット中断確認復旧)
6. [案件削除](#タスク-案件削除)
7. [自動停止ログ確認](#タスク-自動停止ログ確認)
8. [自動停止設定変更](#タスク-自動停止設定変更)
9. [tmuxセッション確認](#タスク-tmuxセッション確認)
10. [開発サーバー再起動](#タスク-開発サーバー再起動)
11. [開発サーバー停止](#タスク-開発サーバー停止)
12. [Tunnel再起動](#タスク-tunnel再起動)
13. [Tunnel URL確認](#タスク-tunnel-url確認)
14. [code-server再起動](#タスク-code-server再起動)
15. [code-server URL確認](#タスク-code-server-url確認)
16. [code-serverパスワード変更](#タスク-code-serverパスワード変更)
17. [障害調査（ログ確認）](#タスク-障害調査ログ確認)
18. [障害調査（メトリクス確認）](#タスク-障害調査メトリクス確認)
19. [リポジトリクローン](#タスク-リポジトリクローン)
20. [PR作成](#タスク-pr作成)
21. [PR確認・マージ](#タスク-pr確認マージ)
22. [CIデバッグ](#タスク-ciデバッグ)
23. [並行開発（Git Worktree）](#並行開発git-worktree)

---

## タスク: 案件一覧確認

### 実行コマンド

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=development" \
  --query 'Reservations[].Instances[].[Tags[?Key==`Name`].Value|[0],Tags[?Key==`Client`].Value|[0],InstanceId,State.Name,PublicIpAddress]' \
  --output table \
  --region ap-northeast-1
```

### 出力例

```
-----------------------------------------------------------------
|                       DescribeInstances                       |
+------------------------+-------+--------------+---------+-----+
|  ar-dev-acme-server    | acme  | i-0123456789 | running | 1.2.3.4 |
|  ar-dev-beta-server    | beta  | i-9876543210 | stopped | None    |
+------------------------+-------+--------------+---------+---------+
```

---

## タスク: EC2起動

### [要確認] クライアント名

```
起動するEC2のクライアント名を教えてください（例: acme）:
```

### 実行コマンド

```bash
CLIENT_NAME="入力されたクライアント名"
REGION="ap-northeast-1"

INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ar-dev-${CLIENT_NAME}-server" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text \
  --region ${REGION})

echo "起動中: ar-dev-${CLIENT_NAME}-server (${INSTANCE_ID})"
aws ec2 start-instances --instance-ids ${INSTANCE_ID} --region ${REGION}

echo "起動完了待機中..."
aws ec2 wait instance-running --instance-ids ${INSTANCE_ID} --region ${REGION}

ELASTIC_IP=$(aws ec2 describe-instances \
  --instance-ids ${INSTANCE_ID} \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text \
  --region ${REGION})

echo ""
echo "=========================================="
echo "EC2起動完了"
echo "=========================================="
echo "インスタンス: ar-dev-${CLIENT_NAME}-server"
echo "IP: ${ELASTIC_IP}"
echo "SSH: ssh -i ~/.ssh/ar-dev-${CLIENT_NAME}-key.pem ubuntu@${ELASTIC_IP}"
echo "=========================================="
```

---

## タスク: EC2停止

### [要確認] クライアント名

```
停止するEC2のクライアント名を教えてください（例: acme）:
```

### 実行コマンド

```bash
CLIENT_NAME="入力されたクライアント名"
REGION="ap-northeast-1"

INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ar-dev-${CLIENT_NAME}-server" "Name=instance-state-name,Values=running" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text \
  --region ${REGION})

if [ "${INSTANCE_ID}" = "None" ]; then
  echo "ar-dev-${CLIENT_NAME}-server は起動していません"
else
  echo "停止中: ar-dev-${CLIENT_NAME}-server (${INSTANCE_ID})"
  aws ec2 stop-instances --instance-ids ${INSTANCE_ID} --region ${REGION}
  echo "停止完了"
fi
```

---

## タスク: EC2状態確認

### [要確認] クライアント名

```
確認するEC2のクライアント名を教えてください（例: acme）:
※ 全案件を確認する場合は「all」と入力
```

### 実行コマンド（特定クライアント）

```bash
CLIENT_NAME="入力されたクライアント名"
REGION="ap-northeast-1"

aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ar-dev-${CLIENT_NAME}-server" \
  --query 'Reservations[0].Instances[0].[State.Name,PublicIpAddress,InstanceType,LaunchTime]' \
  --output text \
  --region ${REGION}
```

### 実行コマンド（全案件）

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=development" \
  --query 'Reservations[].Instances[].[Tags[?Key==`Name`].Value|[0],State.Name,PublicIpAddress]' \
  --output table \
  --region ap-northeast-1
```

---

## タスク: スポット中断確認・復旧

作業用EC2はスポットインスタンスで動作しています。
稀にAWSの都合で中断（自動停止）されることがあります。

### 中断されているか確認

```bash
# 作業用EC2の状態確認
aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=development" \
  --query 'Reservations[].Instances[].[Tags[?Key==`Name`].Value|[0],State.Name,StateReason.Code]' \
  --output table \
  --region ap-northeast-1
```

**StateReason.Code が `Server.SpotInstanceTermination` の場合、スポット中断です。**

### 復旧方法

スポット中断で停止した場合、再起動するだけで復旧できます：

```bash
# クライアント名を指定して再起動
CLIENT_NAME="クライアント名"

INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ar-dev-${CLIENT_NAME}-server" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text \
  --region ap-northeast-1)

aws ec2 start-instances --instance-ids ${INSTANCE_ID} --region ap-northeast-1
echo "再起動開始: ${INSTANCE_ID}"
```

または管理スクリプトで：

```bash
start-project クライアント名
```

> **注意**
> - 中断時、EBSボリューム（データ）は保持されます
> - tmuxセッションは消えるので再作成が必要です
> - 開発サーバーとTunnelも再起動が必要です

---

## タスク: 案件削除

### [要確認] 削除確認

```
========================================
⚠️ 案件削除の確認
========================================

削除するクライアント名を教えてください（例: acme）:

※ 以下のリソースがすべて削除されます：
  - EC2インスタンス: ar-dev-{client}-server
  - Elastic IP
  - セキュリティグループ: ar-dev-{client}-sg
  - キーペア: ar-dev-{client}-key
  - IAMロール: ar-dev-{client}-role

本当に削除しますか？削除する場合はクライアント名を入力してください:
```

### 実行コマンド

```bash
CLIENT_NAME="入力されたクライアント名"
REGION="ap-northeast-1"

echo "=== ${CLIENT_NAME} 案件リソース削除開始 ==="

# インスタンスID取得
INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ar-dev-${CLIENT_NAME}-server" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text \
  --region ${REGION})

# Elastic IP解放
ALLOCATION_ID=$(aws ec2 describe-addresses \
  --filters "Name=tag:Name,Values=ar-dev-${CLIENT_NAME}-server-eip" \
  --query 'Addresses[0].AllocationId' \
  --output text \
  --region ${REGION})

if [ "${ALLOCATION_ID}" != "None" ]; then
  aws ec2 release-address --allocation-id ${ALLOCATION_ID} --region ${REGION}
  echo "✓ Elastic IP解放"
fi

# インスタンス削除
if [ "${INSTANCE_ID}" != "None" ]; then
  aws ec2 terminate-instances --instance-ids ${INSTANCE_ID} --region ${REGION}
  echo "✓ インスタンス削除開始: ${INSTANCE_ID}"
  aws ec2 wait instance-terminated --instance-ids ${INSTANCE_ID} --region ${REGION}
  echo "✓ インスタンス削除完了"
fi

# セキュリティグループ削除
SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=ar-dev-${CLIENT_NAME}-sg" \
  --query 'SecurityGroups[0].GroupId' \
  --output text \
  --region ${REGION})

if [ "${SG_ID}" != "None" ]; then
  aws ec2 delete-security-group --group-id ${SG_ID} --region ${REGION}
  echo "✓ セキュリティグループ削除"
fi

# キーペア削除
aws ec2 delete-key-pair --key-name ar-dev-${CLIENT_NAME}-key --region ${REGION} 2>/dev/null
rm -f ~/.ssh/ar-dev-${CLIENT_NAME}-key.pem
echo "✓ キーペア削除"

# IAMロール・インスタンスプロファイル削除
ROLE_NAME="ar-dev-${CLIENT_NAME}-role"
PROFILE_NAME="ar-dev-${CLIENT_NAME}-profile"

# インスタンスプロファイルからロールを削除
aws iam remove-role-from-instance-profile \
  --instance-profile-name ${PROFILE_NAME} \
  --role-name ${ROLE_NAME} 2>/dev/null

# インスタンスプロファイル削除
aws iam delete-instance-profile \
  --instance-profile-name ${PROFILE_NAME} 2>/dev/null

# ポリシーデタッチ
for POLICY in AmazonEC2FullAccess CloudWatchLogsReadOnlyAccess AmazonSSMManagedInstanceCore AmazonS3ReadOnlyAccess; do
  aws iam detach-role-policy \
    --role-name ${ROLE_NAME} \
    --policy-arn arn:aws:iam::aws:policy/${POLICY} 2>/dev/null
done

# ロール削除
aws iam delete-role --role-name ${ROLE_NAME} 2>/dev/null
echo "✓ IAMロール削除"

echo ""
echo "=========================================="
echo "${CLIENT_NAME} 案件リソース削除完了"
echo "=========================================="
```

---

## タスク: 自動停止ログ確認

EC2内で自動停止サービスの状態とログを確認します。

### 実行コマンド（EC2内で実行）

```bash
# サービス状態
sudo systemctl status auto-stop

# 直近のログ
sudo tail -50 /var/log/auto-stop.log

# リアルタイムログ監視
sudo tail -f /var/log/auto-stop.log
```

### 出力例

```
2024-01-15 10:30:00 Active: SSH connections (1)
2024-01-15 10:31:00 Active: SSH connections (1)
2024-01-15 10:32:00 Idle state started
2024-01-15 10:33:00 Idle: 1/30 min
...
2024-01-15 11:02:00 Idle threshold reached (30 min)
2024-01-15 11:02:00 === Initiating auto-stop ===
```

---

## タスク: 自動停止設定変更

### [要確認] 変更内容

```
自動停止設定をどうしますか？

1. 閾値を変更（現在: SSH切断後3時間）→ 新しい時間（時間単位）を指定
2. 一時的に無効化
3. 再度有効化
4. 完全に無効化（サービス削除）
```

### 1. 閾値変更（EC2内で実行）

```bash
NEW_HOURS=5  # 入力値

# サービス設定更新
sudo sed -i "s/auto-stop.sh [0-9]*/auto-stop.sh ${NEW_HOURS}/" /etc/systemd/system/auto-stop.service
sudo systemctl daemon-reload
sudo systemctl restart auto-stop

echo "閾値を SSH切断後${NEW_HOURS}時間 に変更しました"
```

### 2. 一時的に無効化

```bash
sudo systemctl stop auto-stop
echo "自動停止を一時停止しました（EC2再起動で復活）"
```

### 3. 再度有効化

```bash
sudo systemctl start auto-stop
echo "自動停止を再開しました"
```

### 4. 完全に無効化

```bash
sudo systemctl stop auto-stop
sudo systemctl disable auto-stop
sudo rm /etc/systemd/system/auto-stop.service
sudo rm /usr/local/bin/auto-stop.sh
sudo systemctl daemon-reload

echo "自動停止機能を完全に削除しました"
```

---

## タスク: 開発サーバー再起動

開発サーバーとTunnelはバックグラウンドで動作しています。
`~/bin/` のスクリプトで管理します。

### [要確認] 再起動対象

```
再起動する内容を教えてください：

1. ポート番号（例: 3000, 3010）:
2. プロジェクトディレクトリ（例: ~/project）:
```

### 実行コマンド（EC2内で実行）

```bash
PORT="3000"              # 入力値
DIR="~/project"          # 入力値

# サービス再起動
stop-services $PORT
start-services $PORT "$DIR"
```

### よくある再起動パターン

```bash
# mainブランチ再起動
stop-services 3000
start-services 3000 ~/project

# feature/loginブランチ再起動
stop-services 3010
start-services 3010 ~/project/../feature-login

# Tunnel URLの確認
get-urls 3000
```

### サービス管理コマンド

| コマンド | 用途 |
|----------|------|
| `start-services [ポート] [ディレクトリ]` | 開発サーバー + Tunnel起動 |
| `stop-services [ポート]` | 開発サーバー + Tunnel停止 |
| `get-urls [ポート]` | Tunnel URL取得 |
| `services-status` | 全サービス状態確認 |

---

## タスク: tmuxセッション確認

### 実行コマンド（EC2内で実行）

```bash
tmux list-sessions
```

### セッション接続

```bash
tmux attach -t セッション名
```

---

## タスク: 開発サーバー停止

### 実行コマンド（EC2内で実行）

```bash
PORT="3000"  # 入力値

stop-services $PORT

echo "開発サーバーを停止しました"
```

---

## タスク: Tunnel URL確認

### 実行コマンド（EC2内で実行）

```bash
# 特定ポートのTunnel URL取得
get-urls 3000

# 全サービス状態確認
services-status
```

### 出力例

```
=== Tunnel URLs (port 3000) ===
App:         https://random-name.trycloudflare.com
code-server: https://another-name.trycloudflare.com
```

---

## タスク: code-server再起動

code-serverサービスを再起動します。

### 実行コマンド（EC2内で実行）

```bash
# サービス再起動
sudo systemctl restart code-server@$USER

# 状態確認
sudo systemctl status code-server@$USER --no-pager

echo "code-server再起動完了"
```

### Tunnel再起動（必要な場合）

code-server用Tunnelはmainブランチのサービス起動時に自動起動されます。

```bash
# mainブランチのサービスを再起動
stop-services 3000
start-services 3000 ~/project

# URL確認
get-urls 3000
```

---

## タスク: code-server URL確認

### 実行コマンド（EC2内で実行）

```bash
# code-server URLを取得（mainブランチのみ）
get-urls 3000
```

### 出力例

```
=== Tunnel URLs (port 3000) ===
App:         https://random-name.trycloudflare.com
code-server: https://another-name.trycloudflare.com

※ Beam Pro Chrome でcode-server URLにアクセス
※ パスワードは ~/.config/code-server/config.yaml に記載
```

---

## タスク: code-serverパスワード変更

### [要確認] 新しいパスワード

```
新しいパスワードを入力してください（8文字以上推奨）:
```

### 実行コマンド（EC2内で実行）

```bash
NEW_PASSWORD="入力されたパスワード"

# 設定ファイル更新
sed -i "s/^password:.*/password: ${NEW_PASSWORD}/" ~/.config/code-server/config.yaml

# サービス再起動
sudo systemctl restart code-server@$USER

echo "=========================================="
echo "パスワード変更完了"
echo "=========================================="
echo "次回アクセス時から新しいパスワードが必要です"
```

---

## タスク: 障害調査（ログ確認）

### [要確認] 調査対象

```
障害調査の対象を教えてください：

1. 環境（staging / production）:
2. ロググループ名（例: /aws/ec2/app-name）:
3. 検索キーワード（例: ERROR, OutOfMemory）:
4. 期間（例: 1時間, 24時間）:
```

### 実行コマンド

```bash
LOG_GROUP="/aws/ec2/app-name"  # 入力値
FILTER="ERROR"                   # 入力値
HOURS=1                          # 入力値から変換
REGION="ap-northeast-1"

START_TIME=$(date -d "${HOURS} hours ago" +%s)000

aws logs filter-log-events \
  --log-group-name ${LOG_GROUP} \
  --filter-pattern "${FILTER}" \
  --start-time ${START_TIME} \
  --region ${REGION} \
  --query 'events[].[timestamp,message]' \
  --output text
```

---

## タスク: 障害調査（メトリクス確認）

### [要確認] 調査対象

```
メトリクス確認の対象を教えてください：

1. インスタンスID（例: i-0123456789abcdef0）:
2. 期間（例: 1時間, 24時間）:
```

### 実行コマンド

```bash
INSTANCE_ID="入力値"
HOURS=1
REGION="ap-northeast-1"

START_TIME=$(date -u -d "${HOURS} hours ago" +%Y-%m-%dT%H:%M:%SZ)
END_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)

echo "=== CPU使用率 ==="
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=${INSTANCE_ID} \
  --start-time ${START_TIME} \
  --end-time ${END_TIME} \
  --period 300 \
  --statistics Average \
  --region ${REGION} \
  --query 'Datapoints[*].[Timestamp,Average]' \
  --output text | sort

echo ""
echo "=== ネットワーク入力 ==="
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name NetworkIn \
  --dimensions Name=InstanceId,Value=${INSTANCE_ID} \
  --start-time ${START_TIME} \
  --end-time ${END_TIME} \
  --period 300 \
  --statistics Sum \
  --region ${REGION} \
  --query 'Datapoints[*].[Timestamp,Sum]' \
  --output text | sort
```

---

## タスク: リポジトリクローン

### [要確認] リポジトリ情報

```
クローンするリポジトリの情報を教えてください：

1. リポジトリ（例: owner/repo-name）:
2. ブランチ（例: develop, main）:
```

### 実行コマンド（EC2内で実行）

```bash
REPO="owner/repo-name"    # 入力値
BRANCH="develop"          # 入力値

cd ~
gh repo clone ${REPO}
cd $(basename ${REPO})
git checkout ${BRANCH}

# 依存関係インストール（package.jsonがある場合）
if [ -f "package.json" ]; then
  npm install
  echo "依存関係インストール完了"
fi

# 環境変数ファイル確認
if [ -f ".env.example" ]; then
  echo ""
  echo "⚠️ .env.example が存在します"
  echo "必要に応じて .env.local を作成してください"
fi
```

---

## タスク: PR作成

### [要確認] PR情報

```
PRの情報を教えてください：

1. タイトル:
2. ベースブランチ（例: main, develop）:
3. 説明（任意）:
4. ドラフト？（yes/no）:
```

### 実行コマンド（EC2内で実行）

```bash
TITLE="入力値"
BASE="main"           # 入力値
BODY="入力値"
DRAFT=""              # --draft または 空

gh pr create \
  --title "${TITLE}" \
  --base ${BASE} \
  --body "${BODY}" \
  ${DRAFT}
```

---

## タスク: PR確認・マージ

### [要確認] PR番号

```
確認/マージするPR番号を教えてください:
```

### 実行コマンド（EC2内で実行）

```bash
PR_NUMBER=123  # 入力値

echo "=== PR情報 ==="
gh pr view ${PR_NUMBER}

echo ""
echo "=== CIステータス ==="
gh pr checks ${PR_NUMBER}

echo ""
echo "=== 差分統計 ==="
gh pr diff ${PR_NUMBER} --stat
```

### [要確認] マージ確認

```
PRをマージしますか？
- squash: コミットをまとめてマージ
- merge: 通常マージ
- no: マージしない
```

### マージ実行

```bash
# squashの場合
gh pr merge ${PR_NUMBER} --squash --delete-branch

# mergeの場合
gh pr merge ${PR_NUMBER} --merge --delete-branch
```

---

## タスク: CIデバッグ

### 実行コマンド（EC2内で実行）

```bash
echo "=== 最新のワークフロー実行 ==="
gh run list --limit 5

echo ""
echo "失敗した実行のログを確認するには:"
echo "gh run view <RUN_ID> --log-failed"
```

### [要確認] 詳細確認

```
詳細を確認するRUN_IDを教えてください（不要ならスキップ）:
```

### 詳細ログ確認

```bash
RUN_ID=入力値
gh run view ${RUN_ID} --log-failed
```

---

## 並行開発（Git Worktree）

同一案件で複数ブランチを並行して開発する場合のガイドです。

### カスタムコマンド（推奨）

| コマンド | 用途 |
|----------|------|
| `/project:worktree-new feature/login` | ワークツリー作成 + Window + サービス起動 |
| `/project:worktree-delete feature/login` | ワークツリー削除 + Window + サービス停止 |
| `/project:worktree-list` | ワークツリー一覧 |

これらのコマンドを使うと、以下がすべて自動化されます。

### 概要

Git Worktreeを使うと、同一リポジトリの複数ブランチを**別ディレクトリに同時展開**できます。
各ディレクトリで別々のClaude Codeセッションを起動し、並行開発が可能です。

```
従来: stash → ブランチ切替 → stash pop（コンテキスト切替コスト高）
Worktree: ディレクトリ移動だけ（各ブランチが独立して存在）
```

### tmux構成

```
Acme-dev セッション
├── Window 1 [main]           ← Claude Code (フルスクリーン)
├── Window 2 [feature-login]  ← Claude Code (フルスクリーン)
└── Window 3 [feature-payment]← Claude Code (フルスクリーン)

切り替え: Alt + Shift + ] / Alt + Shift + [
```

---

### ディレクトリ構成

```
~/project/
├── main/                      # ベースブランチ（常にクリーン）
└── worktrees/
    ├── feature/
    │   ├── login/             # feature/login ブランチ
    │   └── payment/           # feature/payment ブランチ
    ├── bugfix/
    │   └── cart-error/        # bugfix/cart-error
    └── review/
        └── pr-123/            # PRレビュー用
```

---

### ポート設計

1プロジェクトで複数ポートを使用する場合（API + フロントエンド等）を考慮した設計：

```
【ポート割り当てルール】

ブランチ         フロント   API      その他
─────────────────────────────────────────
main             3000      4000     5000
feature/login    3010      4010     5010
feature/payment  3020      4020     5020
bugfix/cart      3030      4030     5030

※ 各ブランチに10ポートのレンジを割り当て
```

#### 環境変数での管理

各worktreeの `.env.local` でポートを指定：

```bash
# ~/project/worktrees/feature/login/.env.local
FRONTEND_PORT=3010
API_PORT=4010
```

---

### ポート管理

#### 使用中ポートの確認

```bash
# 使用中のポート一覧
sudo lsof -i -P -n | grep LISTEN

# 特定ポートの使用状況
sudo lsof -i :3000

# 3000番台のポート使用状況
sudo lsof -i -P -n | grep LISTEN | grep -E ':30[0-9]{2}'
```

#### ポート使用状況ファイル（推奨）

ポート割り当てを明示的に管理：

```bash
# ~/project/.ports に使用状況を記録
cat ~/project/.ports
```

```
# ポート使用状況
# 形式: ポート範囲 | ブランチ | 用途 | 状態

3000-3009 | main           | フロント/API | active
3010-3019 | feature/login  | フロント/API | active
3020-3029 | feature/payment| フロント/API | available（削除済み）
3030-3039 | -              | 未割当       | available
```

#### 空きポート範囲の取得

```bash
# 次に使えるポート範囲を確認
NEXT_PORT_BASE=$(cat ~/project/.ports 2>/dev/null | grep "available" | head -1 | awk -F'-' '{print $1}' | awk '{print $1}')

if [ -z "$NEXT_PORT_BASE" ]; then
  # .portsファイルがない場合、使用中ポートから推測
  LAST_USED=$(sudo lsof -i -P -n | grep LISTEN | grep -oE ':30[0-9]{2}' | sort -t: -k2 -n | tail -1 | tr -d ':')
  NEXT_PORT_BASE=$((${LAST_USED:-2990} / 10 * 10 + 10))
fi

echo "次のポート範囲: ${NEXT_PORT_BASE}-$((NEXT_PORT_BASE + 9))"
```

#### ポートの解放（プロセス終了）

```bash
# 特定ポートを使用しているプロセスを終了
PORT=3010
PID=$(sudo lsof -t -i :${PORT})
if [ -n "$PID" ]; then
  kill $PID
  echo "ポート ${PORT} を解放しました（PID: $PID）"
else
  echo "ポート ${PORT} は使用されていません"
fi
```

#### Worktree削除時のクリーンアップ

```bash
# Worktree削除時に.portsファイルも更新
BRANCH_NAME="feature/login"
sed -i "s/| ${BRANCH_NAME} |/| - |/g; s/| active$/| available/g" ~/project/.ports
```

---

### タスク: Worktree作成

#### [要確認] Worktree情報

```
並行開発用のWorktreeを作成します：

1. ベースディレクトリ（例: ~/project/main）:
2. ブランチタイプ（feature/bugfix/review）:
3. ブランチ名（例: login, payment, pr-123）:
4. ベースブランチ（例: main, develop）:
```

#### 実行コマンド（EC2内で実行）

```bash
BASE_DIR="~/project/main"           # 入力値
TYPE="feature"                       # 入力値
NAME="login"                         # 入力値
BASE_BRANCH="main"                   # 入力値

BRANCH_NAME="${TYPE}/${NAME}"
WORKTREE_PATH="~/project/worktrees/${TYPE}/${NAME}"

# ディレクトリ作成
mkdir -p ~/project/worktrees/${TYPE}

# リモートから最新取得
cd ${BASE_DIR}
git fetch origin

# Worktree作成
git worktree add ${WORKTREE_PATH} -b ${BRANCH_NAME} origin/${BASE_BRANCH}

echo "=========================================="
echo "Worktree作成完了"
echo "=========================================="
echo "パス: ${WORKTREE_PATH}"
echo "ブランチ: ${BRANCH_NAME}"
echo ""
echo "移動: cd ${WORKTREE_PATH}"
echo "=========================================="
```

---

### タスク: Worktree一覧確認

```bash
git worktree list
```

出力例：
```
/home/ubuntu/project/main              abc1234 [main]
/home/ubuntu/project/worktrees/feature/login   def5678 [feature/login]
/home/ubuntu/project/worktrees/feature/payment ghi9012 [feature/payment]
```

---

### タスク: Worktree削除

#### [要確認] 削除対象

```
削除するWorktreeのパスを教えてください:
（git worktree list で確認できます）
```

#### 実行コマンド

```bash
WORKTREE_PATH="入力値"

# Worktree削除
git worktree remove ${WORKTREE_PATH}

# ブランチも削除する場合
BRANCH_NAME=$(git worktree list | grep ${WORKTREE_PATH} | awk '{print $3}' | tr -d '[]')
git branch -D ${BRANCH_NAME}

echo "Worktree削除完了: ${WORKTREE_PATH}"
```

---

### タスク: 並行開発用tmuxセッション作成

複数ブランチを同時に開発サーバー起動する場合のtmux構成。

#### [要確認] セッション情報

```
並行開発用のtmuxセッションを作成します：

1. セッション名（例: Acme-parallel）:
2. Worktreeパス一覧（カンマ区切り）:
   例: ~/project/main,~/project/worktrees/feature/login
3. 各Worktreeのポート設定（カンマ区切り）:
   例: 3000:4000,3010:4010（フロント:API形式）
```

#### 実行コマンド

```bash
SESSION_NAME="Acme-parallel"

# セッション作成
tmux new-session -d -s "${SESSION_NAME}"

# Window 1: main (port 3000, 4000)
tmux rename-window -t "${SESSION_NAME}:1" "main"
tmux send-keys -t "${SESSION_NAME}:1" "cd ~/project/main" C-m

# Window 2: feature/login (port 3010, 4010)
tmux new-window -t "${SESSION_NAME}" -n "feature-login"
tmux send-keys -t "${SESSION_NAME}:2" "cd ~/project/worktrees/feature/login" C-m

# Window 3: feature/payment (port 3020, 4020)
tmux new-window -t "${SESSION_NAME}" -n "feature-payment"
tmux send-keys -t "${SESSION_NAME}:3" "cd ~/project/worktrees/feature/payment" C-m

echo "=========================================="
echo "並行開発用tmuxセッション作成完了"
echo "=========================================="
echo "セッション: ${SESSION_NAME}"
echo ""
echo "Window切り替え:"
echo "  Alt+Shift+] → 次のwindow"
echo "  Alt+Shift+[ → 前のwindow"
echo "=========================================="
```

---

### ベストプラクティス

| ポイント | 説明 |
|---------|------|
| mainは常にクリーン | Worktreeのベースとして使用、直接編集しない |
| Worktreeは一時的 | タスク完了後は削除してクリーンに保つ |
| 同一ブランチは1箇所のみ | Gitの制約（コンフリクト防止） |
| ポートは事前に設計 | プロジェクト構成に応じてレンジを決める |
| 定期的に `git fetch` | リモートと同期してからWorktree作成 |

---

### 並行AI開発ワークフロー

複数のClaude Codeセッションを並行実行し、同じ機能を異なるアプローチで実装：

```
1. 複数Worktreeを作成
   └── feature/login-v1, feature/login-v2

2. 各Worktreeで別々のClaude Codeセッションを起動
   └── それぞれに同じ要件を指示

3. 結果を比較
   └── コード品質、パフォーマンス、可読性

4. 最良の実装を選択してマージ

5. 不要なWorktreeを削除
```

> **LLMの非決定性を活用**: 同じ指示でも異なる実装が得られる → 最良を選べる

---

### 参考リンク

- [Parallel AI Coding with Git Worktrees](https://docs.agentinterviews.com/blog/parallel-ai-coding-with-gitworktrees/)
- [Git Worktree Best Practices](https://gist.github.com/induratized/49cdedace4a200fa8ae32db9ba3e9a44)
