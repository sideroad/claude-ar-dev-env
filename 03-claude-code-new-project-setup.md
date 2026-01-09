# Claude Code 自動実行タスク: 新規案件EC2セットアップ

このドキュメントは Claude Code が自動実行するタスクリストです。
**コントロールEC2内の Claude Code から実行してください。**

上から順番に実行し、`[要確認]` `[要作業]` マークがある箇所では実行者に確認・作業を依頼してください。

---

## 実行前チェック

以下を実行者に確認してください：

```
コントロールEC2内で実行していますか？
（このタスクはコントロールEC2のClaude Codeから実行する必要があります）

確認できたら「はい」と回答してください。
```

---

## [要確認] 案件情報の入力

実行者に以下を確認してください：

```
新規案件の情報を教えてください：

1. クライアント名（英小文字、例: acme）: 
2. インスタンスタイプ（デフォルト: t4g.small）: 
3. ストレージサイズ（デフォルト: 30GB）: 
4. リージョン（デフォルト: ap-northeast-1）: 

※ クライアント名は命名規則に使用されます
  - EC2: ar-dev-{client}-server
  - キーペア: ar-dev-{client}-key
  - セキュリティグループ: ar-dev-{client}-sg
```

回答を受け取ったら、以下の変数として保持：
- `CLIENT_NAME`: クライアント名
- `INSTANCE_TYPE`: インスタンスタイプ（未指定なら t4g.small）
- `STORAGE_SIZE`: ストレージサイズ（未指定なら 30）
- `REGION`: リージョン（未指定なら ap-northeast-1）

---

## タスク1: EC2インスタンス作成

### 1.1 キーペア作成

```bash
KEY_NAME="ar-dev-${CLIENT_NAME}-key"
REGION="${REGION}"

aws ec2 create-key-pair \
  --key-name ${KEY_NAME} \
  --query 'KeyMaterial' \
  --output text \
  --region ${REGION} > ~/.ssh/${KEY_NAME}.pem

chmod 600 ~/.ssh/${KEY_NAME}.pem
echo "キーペア作成完了: ~/.ssh/${KEY_NAME}.pem"
```

### 1.2 VPC ID 取得

```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=is-default,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text \
  --region ${REGION})
echo "VPC ID: ${VPC_ID}"
```

### 1.3 セキュリティグループ作成

```bash
SG_NAME="ar-dev-${CLIENT_NAME}-sg"

# コントロールEC2のセキュリティグループIDを取得
CONTROL_SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=ar-dev-control-sg" \
  --query 'SecurityGroups[0].GroupId' \
  --output text \
  --region ${REGION})

SG_ID=$(aws ec2 create-security-group \
  --group-name ${SG_NAME} \
  --description "Security group for ar-dev-${CLIENT_NAME}-server" \
  --vpc-id ${VPC_ID} \
  --query 'GroupId' \
  --output text \
  --region ${REGION})

# SSH: コントロールEC2のセキュリティグループからのみ許可
aws ec2 authorize-security-group-ingress \
  --group-id ${SG_ID} \
  --protocol tcp \
  --port 22 \
  --source-group ${CONTROL_SG_ID} \
  --region ${REGION}

echo "セキュリティグループ作成完了: ${SG_ID}"
echo "SSHはコントロールEC2からのみ許可（${CONTROL_SG_ID}）"
```

### 1.4 最新Ubuntu AMI取得（ARM64）

```bash
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-arm64-server-*" \
            "Name=state,Values=available" \
  --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
  --output text \
  --region ${REGION})
echo "AMI ID: ${AMI_ID}"
```

### 1.5 EC2インスタンス起動（スポットインスタンス）

```bash
INSTANCE_NAME="ar-dev-${CLIENT_NAME}-server"

# スポットインスタンスで起動（中断時は停止、データ保持）
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id ${AMI_ID} \
  --instance-type ${INSTANCE_TYPE} \
  --key-name ${KEY_NAME} \
  --security-group-ids ${SG_ID} \
  --block-device-mappings "[{\"DeviceName\":\"/dev/sda1\",\"Ebs\":{\"VolumeSize\":${STORAGE_SIZE},\"VolumeType\":\"gp3\",\"DeleteOnTermination\":false}}]" \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${INSTANCE_NAME}},{Key=Client,Value=${CLIENT_NAME}},{Key=Environment,Value=development},{Key=InstanceType,Value=spot}]" \
  --instance-market-options '{"MarketType":"spot","SpotOptions":{"SpotInstanceType":"persistent","InstanceInterruptionBehavior":"stop"}}' \
  --query 'Instances[0].InstanceId' \
  --output text \
  --region ${REGION})

echo "スポットインスタンス起動中: ${INSTANCE_ID}"
aws ec2 wait instance-running --instance-ids ${INSTANCE_ID} --region ${REGION}
echo "インスタンス起動完了"

# プライベートIP取得
PRIVATE_IP=$(aws ec2 describe-instances \
  --instance-ids ${INSTANCE_ID} \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text \
  --region ${REGION})

echo "プライベートIP: ${PRIVATE_IP}"
echo "※ このEC2はコントロールEC2からのみアクセス可能です"
echo "※ スポットインスタンス: 中断時は自動停止、再起動で復帰"
```

> **スポットインスタンスについて**
> - コスト: オンデマンドの約60-70%削減
> - 中断: 稀に発生（2分前に通知）
> - 中断時: 自動停止（データは保持）
> - 復帰: `start-project` で再起動すればOK

### 1.6 IAMロール作成・アタッチ

```bash
ROLE_NAME="ar-dev-${CLIENT_NAME}-role"
PROFILE_NAME="ar-dev-${CLIENT_NAME}-profile"

# 信頼ポリシー
cat << 'EOF' > /tmp/trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

aws iam create-role \
  --role-name ${ROLE_NAME} \
  --assume-role-policy-document file:///tmp/trust-policy.json

# ポリシーアタッチ
aws iam attach-role-policy --role-name ${ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
aws iam attach-role-policy --role-name ${ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsReadOnlyAccess
aws iam attach-role-policy --role-name ${ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam attach-role-policy --role-name ${ROLE_NAME} --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# インスタンスプロファイル作成・アタッチ
aws iam create-instance-profile --instance-profile-name ${PROFILE_NAME}
aws iam add-role-to-instance-profile --instance-profile-name ${PROFILE_NAME} --role-name ${ROLE_NAME}

echo "IAM反映待機中..."
sleep 10

aws ec2 associate-iam-instance-profile \
  --instance-id ${INSTANCE_ID} \
  --iam-instance-profile Name=${PROFILE_NAME} \
  --region ${REGION}

echo "IAMロールアタッチ完了"
```

### 1.8 作成完了報告

実行者に以下を報告：

```
========================================
EC2インスタンス作成完了
========================================
クライアント: ${CLIENT_NAME}
インスタンス名: ${INSTANCE_NAME}
インスタンスID: ${INSTANCE_ID}
プライベートIP: ${PRIVATE_IP}
キーペア: ~/.ssh/${KEY_NAME}.pem

【セキュリティ】
この EC2 はコントロールEC2からのみ接続可能です。
インターネットからの直接アクセスはできません。

【接続方法（コントロールEC2から）】
ssh -i ~/.ssh/${KEY_NAME}.pem ubuntu@${PRIVATE_IP}
========================================

次のタスク（EC2環境構築）に進みます。
SSH接続可能になるまで1-2分お待ちください。
```

---

## タスク2: EC2環境構築

SSH接続後、以下を実行。

### 2.1 システム更新・基本ツール

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl wget unzip build-essential
```

### 2.2 AWS CLI インストール（ARM64）

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
rm -rf aws awscliv2.zip
aws --version
```

### 2.3 AWS CLI認証確認（IAMロール）

IAMロールがアタッチされているため、設定不要で動作します。

```bash
# 認証確認（IAMロール経由）
aws sts get-caller-identity

# 正常なら以下のような出力:
# {
#     "UserId": "AROA...",
#     "Account": "123456789012",
#     "Arn": "arn:aws:sts::123456789012:assumed-role/ar-dev-xxx-role/..."
# }
```

### 2.4 Node.js インストール（fnm経由）

```bash
curl -fsSL https://fnm.vercel.app/install | bash
source ~/.bashrc
fnm install --lts
fnm use lts-latest

npm config set prefix ~/.npm-global
echo 'export PATH=~/.npm-global/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

node --version
npm --version
```

### 2.4 tmux インストール・設定

```bash
sudo apt install -y tmux

cat << 'EOF' > ~/.tmux.conf
set -g status-left "[#S] "
set -g status-left-length 20
set -g mouse on
set -g history-limit 10000
set -g default-terminal "screen-256color"

# Alt + Shift + [ / Alt + Shift + ] でWindow切り替え（Beam Pro対応）
# ※ コントロールEC2とは別のキーバインドで衝突回避
bind -n M-{ previous-window
bind -n M-} next-window
EOF
```

### 2.5 Claude Code インストール

```bash
npm install -g @anthropic-ai/claude-code
```

### 2.6 Playwright インストール

```bash
npm install -g playwright
npx playwright install --with-deps chromium
```

### 2.7 GitHub CLI インストール

```bash
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install gh -y
gh --version
```

### 2.8 cloudflared インストール（ARM64）

```bash
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64.deb
sudo dpkg -i cloudflared.deb
rm cloudflared.deb
cloudflared --version
```

### 2.9 code-server インストール（ブラウザVS Code）

Beam ProのChromeからコード確認・編集ができるようになります。

```bash
# code-server インストール
curl -fsSL https://code-server.dev/install.sh | sh

# 設定ディレクトリ作成
mkdir -p ~/.config/code-server
```

### [要確認] code-server パスワード設定

```
code-serverにアクセスする際のパスワードを設定してください。
（8文字以上推奨）:
```

パスワードを受け取ったら：

```bash
CODE_SERVER_PASSWORD="入力されたパスワード"

cat << EOF > ~/.config/code-server/config.yaml
bind-addr: 127.0.0.1:8080
auth: password
password: ${CODE_SERVER_PASSWORD}
cert: false
EOF

# ARグラス用にフォントサイズを大きめに設定
mkdir -p ~/.local/share/code-server/User
cat << 'EOF' > ~/.local/share/code-server/User/settings.json
{
  "editor.fontSize": 18,
  "terminal.integrated.fontSize": 16,
  "workbench.colorTheme": "Default Dark Modern"
}
EOF

# サービス有効化（EC2起動時に自動起動）
sudo systemctl enable --now code-server@$USER

# 動作確認
systemctl status code-server@$USER --no-pager
```

### 2.10 CLAUDE.md とカスタムコマンド配置

```bash
# CLAUDE.md作成（変数を展開）
cat << EOF > ~/CLAUDE.md
# 作業用EC2 プロジェクトルール

## 案件情報

- クライアント: ${CLIENT_NAME}
- EC2: ar-dev-${CLIENT_NAME}-server（スポットインスタンス）
- tmuxセッション: ${CLIENT_NAME^}-dev

---

## スポットインスタンス

このEC2はスポットインスタンスで動作しています。

- **コスト**: オンデマンドの約60-70%削減
- **中断**: 稀に発生（2分前に通知）
- **中断時**: 自動停止（EBSデータは保持）
- **復旧**: コントロールEC2から \`start-project ${CLIENT_NAME}\`

> 中断後はtmuxセッション・開発サーバー・Tunnelの再起動が必要です

---

## 構成

\`\`\`
作業用EC2 (t4g.small スポット)
├── Node.js (fnm)
├── tmux セッション [${CLIENT_NAME^}-dev]
│   ├── Window 1 [main]: Claude Code
│   ├── Window 2 [feature/xxx]: Claude Code（並行開発時）
│   └── ...
├── バックグラウンドサービス（~/bin/）
│   ├── 開発サーバー (nohup)
│   └── Cloudflare Tunnel (nohup)
└── 自動停止（SSH切断後3時間）

tmux Window = Claude Code のみ（フルスクリーン）
┌─────────────────────────────────────┐
│                                     │
│           Claude Code               │
│                                     │
└─────────────────────────────────────┘

Alt+Shift+] で次のブランチ（Window）へ移動
\`\`\`

---

## カスタムコマンド

| コマンド | 用途 |
|----------|------|
| \`/project:worktree-new\` | ワークツリー作成（Window+サービス込み） |
| \`/project:worktree-delete\` | ワークツリー削除 |
| \`/project:worktree-list\` | ワークツリー一覧 |
| \`/project:dev-restart\` | 開発サーバー再起動 |
| \`/project:tunnel-url\` | Tunnel URL確認 |
| \`/project:pr-create\` | PR作成 |

---

## tmux操作

### セッション操作
\`\`\`
tmux attach -t ${CLIENT_NAME^}-dev    # セッションにアタッチ
Ctrl+B, d                              # デタッチ
\`\`\`

### Window切り替え（Beam Pro対応）
\`\`\`
Alt + Shift + ]  → 次のwindow
Alt + Shift + [  → 前のwindow
\`\`\`

> コントロールEC2では \`Alt + [ / ]\` を使用（キーバインド衝突回避）

---

## 重要なルール

### 1. サービス管理コマンド（~/bin/）

\`\`\`bash
start-services [ポート] [ディレクトリ]  # サービス起動
stop-services [ポート]                   # サービス停止
get-urls [ポート]                        # Tunnel URL取得
services-status                          # 状態確認
\`\`\`

### 2. ポート割り当て

| ブランチ | ポート範囲 |
|----------|------------|
| main | 3000-3009 |
| feature/login | 3010-3019 |
| feature/payment | 3020-3029 |

### 3. よく使う操作

\`\`\`bash
# サービス起動（mainブランチ）
start-services 3000 ~/project

# Tunnel URL確認
get-urls 3000

# サービス状態確認
services-status
\`\`\`

---

## 自動停止

- 条件: SSH接続がない状態が3時間継続
- ログ: \`sudo tail -f /var/log/auto-stop.log\`
- 無効化: \`sudo systemctl stop auto-stop\`

---

## 接続方法

コントロールEC2から:
\`\`\`bash
# エイリアスで接続
${CLIENT_NAME}

# または直接
ssh -i ~/.ssh/ar-dev-${CLIENT_NAME}-key.pem ubuntu@<Private IP>
\`\`\`
EOF

# カスタムコマンドディレクトリ作成
mkdir -p ~/.claude/commands

# dev-restart.md
cat << 'EOF' > ~/.claude/commands/dev-restart.md
# 開発サーバー再起動

バックグラウンドの開発サーバーとTunnelを再起動します。

## 使い方

\`\`\`
/project:dev-restart [ポート] [ディレクトリ]
\`\`\`

## 引数

- ポート: 開発サーバーのポート（省略時: 3000）
- ディレクトリ: プロジェクトディレクトリ（省略時: カレント）

## 実行内容

\`\`\`bash
PORT="\${1:-3000}"
DIR="\${2:-\$(pwd)}"

stop-services \$PORT
start-services \$PORT "\$DIR"
\`\`\`
EOF

# tunnel-url.md
cat << 'EOF' > ~/.claude/commands/tunnel-url.md
# Tunnel URL確認

Cloudflare Tunnelの現在のURLを取得します。

## 使い方

\`\`\`
/project:tunnel-url [ポート]
\`\`\`

## 引数

- ポート: 開発サーバーのポート（省略時: 3000）

## 実行内容

\`\`\`bash
PORT="\${1:-3000}"
get-urls \$PORT
\`\`\`
EOF

# pr-create.md
cat << 'EOF' > ~/.claude/commands/pr-create.md
# PR作成

GitHub Pull Requestを作成します。

## 使い方

\`\`\`
/project:pr-create タイトル
\`\`\`

## 引数

- タイトル: PRのタイトル

## 実行内容

1. 現在のブランチをプッシュ
2. PRを作成

\`\`\`bash
PR_TITLE="\${ARGUMENTS}"

git push -u origin HEAD
gh pr create --title "\${PR_TITLE}" --fill

echo "PR作成完了"
\`\`\`
EOF

# worktree-new.md
cat << 'EOF' > ~/.claude/commands/worktree-new.md
# ワークツリー作成

新しいブランチ用のワークツリーを作成し、tmux windowとサービスを起動します。

## 使い方

\`\`\`
/project:worktree-new ブランチ名
\`\`\`

## 引数

- ブランチ名: 作成するブランチ名（例: feature/login）

## 実行内容

1. Git worktree作成
2. tmux window作成
3. サービス起動（次の空きポート）

\`\`\`bash
BRANCH="\${ARGUMENTS}"
BRANCH_SAFE=\$(echo "\$BRANCH" | tr '/' '-')
SESSION=\$(tmux display-message -p '#S')
MAIN_DIR=\$(git rev-parse --show-toplevel)
WORKTREE_DIR="\${MAIN_DIR}/../\${BRANCH_SAFE}"

# ポート決定（3000, 3010, 3020...）
USED_PORTS=\$(ls ~/.logs/dev-*.pid 2>/dev/null | sed 's/.*dev-//' | sed 's/.pid//' | sort -n)
PORT=3000
while echo "\$USED_PORTS" | grep -q "^\$PORT\$"; do
  PORT=\$((PORT + 10))
done

echo "=== Creating worktree for \$BRANCH ==="
echo "Directory: \$WORKTREE_DIR"
echo "Port: \$PORT"

# Git worktree作成
git worktree add -b "\$BRANCH" "\$WORKTREE_DIR" || git worktree add "\$WORKTREE_DIR" "\$BRANCH"

# 依存インストール
cd "\$WORKTREE_DIR"
if [ -f package.json ]; then
  npm install
fi

# tmux window作成
tmux new-window -t "\$SESSION" -n "\$BRANCH_SAFE" -c "\$WORKTREE_DIR"
tmux send-keys -t "\$SESSION:\$BRANCH_SAFE" "# Branch: \$BRANCH (port \$PORT)" C-m
tmux send-keys -t "\$SESSION:\$BRANCH_SAFE" "# Run: claude" C-m

# サービス起動
start-services \$PORT "\$WORKTREE_DIR"

echo ""
echo "=== Done ==="
echo "Window: \$BRANCH_SAFE"
echo "Port: \$PORT"
echo "Switch: Alt+Shift+]"
\`\`\`
EOF

# worktree-delete.md
cat << 'EOF' > ~/.claude/commands/worktree-delete.md
# ワークツリー削除

ブランチ用のワークツリー、tmux window、サービスを削除します。

## 使い方

\`\`\`
/project:worktree-delete ブランチ名
\`\`\`

## 引数

- ブランチ名: 削除するブランチ名（例: feature/login）

## 実行内容

1. サービス停止
2. tmux window削除
3. Git worktree削除

\`\`\`bash
BRANCH="\${ARGUMENTS}"
BRANCH_SAFE=\$(echo "\$BRANCH" | tr '/' '-')
SESSION=\$(tmux display-message -p '#S')
MAIN_DIR=\$(git rev-parse --show-toplevel)
WORKTREE_DIR="\${MAIN_DIR}/../\${BRANCH_SAFE}"

echo "=== Deleting worktree for \$BRANCH ==="

# ポート特定（ログファイルから）
PORT=\$(grep -l "\$WORKTREE_DIR" ~/.logs/dev-*.log 2>/dev/null | head -1 | sed 's/.*dev-//' | sed 's/.log//')

# サービス停止
if [ -n "\$PORT" ]; then
  echo "Stopping services on port \$PORT..."
  stop-services \$PORT
fi

# tmux window削除
if tmux list-windows -t "\$SESSION" | grep -q "\$BRANCH_SAFE"; then
  echo "Closing tmux window..."
  tmux kill-window -t "\$SESSION:\$BRANCH_SAFE"
fi

# Git worktree削除
if [ -d "\$WORKTREE_DIR" ]; then
  echo "Removing worktree..."
  git worktree remove "\$WORKTREE_DIR" --force
fi

# ブランチ削除確認
echo ""
read -p "Delete branch '\$BRANCH' as well? (y/N): " DELETE_BRANCH
if [ "\$DELETE_BRANCH" = "y" ]; then
  git branch -D "\$BRANCH"
  echo "Branch deleted."
fi

echo ""
echo "=== Done ==="
\`\`\`
EOF

# worktree-list.md
cat << 'EOF' > ~/.claude/commands/worktree-list.md
# ワークツリー一覧

現在のワークツリーとサービス状態を表示します。

## 使い方

\`\`\`
/project:worktree-list
\`\`\`

## 実行内容

\`\`\`bash
echo "=== Git Worktrees ==="
git worktree list

echo ""
echo "=== Running Services ==="
services-status

echo ""
echo "=== tmux Windows ==="
tmux list-windows -F "  #I: #W"
\`\`\`
EOF

echo "CLAUDE.md とカスタムコマンド配置完了"
```

### 2.11 環境構築完了報告

```
========================================
EC2環境構築完了
========================================
インストール済み:
- AWS CLI
- Node.js (fnm)
- tmux
- Claude Code
- Playwright + Chromium
- GitHub CLI (gh)
- cloudflared
- code-server（ブラウザVS Code）

配置済み:
- ~/CLAUDE.md（プロジェクトルール）
- ~/.claude/commands/（カスタムコマンド）
  - dev-restart.md
  - tunnel-url.md
  - pr-create.md
  - worktree-new.md
  - worktree-delete.md
  - worktree-list.md

code-server:
- ポート: 8080
- アクセス: Cloudflare Tunnel経由
- 設定: ~/.config/code-server/config.yaml

次は認証設定を行います。
========================================
```

---

## タスク3: GitHub認証

### [要作業] GitHub CLI 認証

実行者に以下を依頼：

```
========================================
[要作業] GitHub認証が必要です
========================================

EC2にSSH接続して、以下のコマンドを実行してください：

  gh auth login

対話形式で以下を選択：
  1. GitHub.com
  2. SSH
  3. Yes (SSH鍵生成)
  4. パスフレーズ: 任意（空でも可）
  5. 鍵のタイトル: EC2-${CLIENT_NAME}-server
  6. Login with a web browser

ブラウザでログインを完了してください。

※ EC2からブラウザを開けない場合は「Paste an authentication token」を選択し、
   GitHub Settings → Developer settings → Personal access tokens で
   トークンを生成（スコープ: repo, read:org, workflow）して貼り付けてください。

完了したら「認証完了」と入力してください。
========================================
```

### 3.1 認証確認

「認証完了」を受け取ったら：

```bash
gh auth status
ssh -T git@github.com
```

### 3.2 Git初期設定

### [要確認] Git ユーザー情報

実行者に確認：

```
Gitのユーザー情報を教えてください：
- 名前（例: Taro Yamada）: 
- メールアドレス: 
```

回答を受け取ったら：

```bash
git config --global user.name "入力された名前"
git config --global user.email "入力されたメールアドレス"
git config --global init.defaultBranch main
git config --global pull.rebase false
```

---

## タスク4: Cloudflare Tunnel 認証

### [要作業] Cloudflare 認証

実行者に以下を依頼：

```
========================================
[要作業] Cloudflare認証が必要です
========================================

EC2にSSH接続して、以下のコマンドを実行してください：

  cloudflared tunnel login

ブラウザが開くので、Cloudflareにログインしてドメインを選択してください。

※ EC2からブラウザを開けない場合は、表示されるURLをコピーして
   別のデバイスでアクセスしてください。

完了したら「認証完了」と入力してください。
========================================
```

### 4.1 認証確認

「認証完了」を受け取ったら：

```bash
ls ~/.cloudflared/cert.pem && echo "Cloudflare認証OK"
```

---

## タスク5: テストプロジェクト作成

### 5.1 プロジェクト作成

```bash
mkdir -p ~/test-project && cd ~/test-project
npm init -y
npm install express
npm install -D nodemon

cat << 'EOF' > index.js
const express = require('express');
const app = express();

app.get('/', (req, res) => {
  res.send(`
    <h1>AR Dev Server Running!</h1>
    <p>Client: ${CLIENT_NAME}</p>
    <p>Time: ${new Date().toISOString()}</p>
  `);
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
});
EOF

npm pkg set scripts.dev="nodemon index.js"
```

### 5.2 サービス管理スクリプト作成

開発サーバーとTunnelをバックグラウンドで管理するスクリプトを作成：

```bash
mkdir -p ~/bin ~/.logs

# サービス起動スクリプト
cat << 'EOF' > ~/bin/start-services
#!/bin/bash
# 使い方: start-services [ポート] [ディレクトリ]
PORT="${1:-3000}"
DIR="${2:-$(pwd)}"
BRANCH=$(cd "$DIR" && git branch --show-current 2>/dev/null || echo "main")

echo "=== Starting services for $BRANCH (port $PORT) ==="

# 既存プロセス停止
stop-services $PORT 2>/dev/null

# ログディレクトリ
mkdir -p ~/.logs

# 開発サーバー起動
cd "$DIR"
nohup npm run dev > ~/.logs/dev-$PORT.log 2>&1 &
echo $! > ~/.logs/dev-$PORT.pid
echo "Dev server started on port $PORT (PID: $!)"

# App Tunnel起動
nohup cloudflared tunnel --url http://localhost:$PORT > ~/.logs/tunnel-$PORT.log 2>&1 &
echo $! > ~/.logs/tunnel-$PORT.pid
echo "App tunnel started (PID: $!)"

# code-server Tunnel起動（メインブランチのみ）
if [ "$PORT" = "3000" ]; then
  nohup cloudflared tunnel --url http://localhost:8080 > ~/.logs/tunnel-code.log 2>&1 &
  echo $! > ~/.logs/tunnel-code.pid
  echo "code-server tunnel started (PID: $!)"
fi

sleep 3
echo ""
echo "=== Tunnel URLs ==="
grep -o 'https://[^[:space:]]*\.trycloudflare\.com' ~/.logs/tunnel-$PORT.log | tail -1 || echo "App: (starting...)"
if [ "$PORT" = "3000" ]; then
  grep -o 'https://[^[:space:]]*\.trycloudflare\.com' ~/.logs/tunnel-code.log | tail -1 || echo "code-server: (starting...)"
fi
EOF
chmod +x ~/bin/start-services

# サービス停止スクリプト
cat << 'EOF' > ~/bin/stop-services
#!/bin/bash
PORT="${1:-3000}"

echo "=== Stopping services for port $PORT ==="

# 開発サーバー停止
if [ -f ~/.logs/dev-$PORT.pid ]; then
  kill "$(cat ~/.logs/dev-$PORT.pid)" 2>/dev/null && echo "Dev server stopped"
  rm ~/.logs/dev-$PORT.pid
fi

# Tunnel停止
if [ -f ~/.logs/tunnel-$PORT.pid ]; then
  kill "$(cat ~/.logs/tunnel-$PORT.pid)" 2>/dev/null && echo "Tunnel stopped"
  rm ~/.logs/tunnel-$PORT.pid
fi

# code-server Tunnel停止（メインポートの場合のみ）
if [ "$PORT" = "3000" ] && [ -f ~/.logs/tunnel-code.pid ]; then
  kill "$(cat ~/.logs/tunnel-code.pid)" 2>/dev/null && echo "code-server tunnel stopped"
  rm ~/.logs/tunnel-code.pid
fi
EOF
chmod +x ~/bin/stop-services

# Tunnel URL取得スクリプト
cat << 'EOF' > ~/bin/get-urls
#!/bin/bash
PORT="${1:-3000}"

echo "=== Tunnel URLs (port $PORT) ==="
echo -n "App:         "
grep -o 'https://[^[:space:]]*\.trycloudflare\.com' ~/.logs/tunnel-$PORT.log 2>/dev/null | tail -1 || echo "(not running)"

if [ "$PORT" = "3000" ]; then
  echo -n "code-server: "
  grep -o 'https://[^[:space:]]*\.trycloudflare\.com' ~/.logs/tunnel-code.log 2>/dev/null | tail -1 || echo "(not running)"
fi
EOF
chmod +x ~/bin/get-urls

# サービス状態確認
cat << 'EOF' > ~/bin/services-status
#!/bin/bash
echo "=== Running Services ==="
for pidfile in ~/.logs/*.pid; do
  [ -f "$pidfile" ] || continue
  name=$(basename "$pidfile" .pid)
  pid=$(cat "$pidfile")
  if ps -p $pid > /dev/null 2>&1; then
    echo "✓ $name (PID: $pid)"
  else
    echo "✗ $name (dead)"
  fi
done
EOF
chmod +x ~/bin/services-status

echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 5.3 tmuxセッション作成

```bash
SESSION_NAME="${CLIENT_NAME^}-dev"

# tmuxセッション作成（Window = Claude Code のみ）
tmux new-session -d -s "${SESSION_NAME}" -n "main" -c ~/test-project

# Claude Code 起動案内
tmux send-keys -t "${SESSION_NAME}:main" "# Claude Code: claude" C-m

# バックグラウンドでサービス起動
start-services 3000 ~/test-project
```

tmuxレイアウト:
```
Acme-dev セッション
├── Window 1 [main]           ← Claude Code
├── Window 2 [feature/login]  ← Claude Code（ワークツリー追加時）
└── Window 3 [feature/payment]← Claude Code（ワークツリー追加時）

バックグラウンド（見えない）:
├── main:           dev:3000 + tunnel
├── feature/login:  dev:3010 + tunnel
└── feature/payment: dev:3020 + tunnel
```

---

## タスク6: 自動停止機能セットアップ

SSH接続がない状態が3時間続いた場合にEC2を自動停止します。

### [要確認] 自動停止設定

```
自動停止機能を有効にしますか？

- はい: SSH切断後3時間で自動停止
- いいえ: 手動停止のみ（スキップ）
- カスタム: 時間を指定（時間単位）

※ SSH接続中は停止しません
```

「いいえ」の場合はタスク7へスキップ。
「はい」または時間指定を受け取ったら以下を実行：

### 6.1 自動停止スクリプト作成

```bash
IDLE_HOURS=3  # 入力値または3

sudo tee /usr/local/bin/auto-stop.sh << 'SCRIPT'
#!/bin/bash
#
# EC2 Auto-Stop Script (Simple Version)
# 
# 停止条件: SSH接続がない状態が閾値時間継続
#

IDLE_THRESHOLD_HOURS=${1:-3}
IDLE_THRESHOLD_SECONDS=$((IDLE_THRESHOLD_HOURS * 3600))
CHECK_INTERVAL=300  # 5分ごとにチェック
LOG_FILE="/var/log/auto-stop.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') $1" >> "$LOG_FILE"
}

# SSH接続チェック
has_ssh_connection() {
    local ssh_count=$(who | grep -c pts 2>/dev/null || echo 0)
    [ "$ssh_count" -gt 0 ]
}

# EC2自己停止
stop_instance() {
    log "=== Initiating auto-stop ==="
    
    local token=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
        -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
    local instance_id=$(curl -s -H "X-aws-ec2-metadata-token: $token" \
        http://169.254.169.254/latest/meta-data/instance-id)
    local region=$(curl -s -H "X-aws-ec2-metadata-token: $token" \
        http://169.254.169.254/latest/meta-data/placement/region)
    
    log "Stopping instance: $instance_id in $region"
    
    aws ec2 stop-instances --instance-ids "$instance_id" --region "$region"
}

# メインループ
main() {
    log "=== Auto-stop daemon started (threshold: ${IDLE_THRESHOLD_HOURS} hours) ==="
    
    local idle_start=0
    
    while true; do
        if has_ssh_connection; then
            if [ "$idle_start" -ne 0 ]; then
                log "SSH connected, resetting idle timer"
            fi
            idle_start=0
        else
            local now=$(date +%s)
            
            if [ "$idle_start" -eq 0 ]; then
                idle_start=$now
                log "No SSH connection, idle timer started"
            else
                local idle_duration=$((now - idle_start))
                local idle_hours=$((idle_duration / 3600))
                local idle_minutes=$(((idle_duration % 3600) / 60))
                
                if [ "$idle_duration" -ge "$IDLE_THRESHOLD_SECONDS" ]; then
                    log "Idle threshold reached (${idle_hours}h ${idle_minutes}m)"
                    stop_instance
                    exit 0
                else
                    log "Idle: ${idle_hours}h ${idle_minutes}m / ${IDLE_THRESHOLD_HOURS}h"
                fi
            fi
        fi
        
        sleep "$CHECK_INTERVAL"
    done
}

main
SCRIPT

sudo chmod +x /usr/local/bin/auto-stop.sh
```

### 6.2 systemdサービス作成

```bash
sudo tee /etc/systemd/system/auto-stop.service << EOF
[Unit]
Description=EC2 Auto-Stop on Idle
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/auto-stop.sh ${IDLE_HOURS}
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable auto-stop
sudo systemctl start auto-stop
```

### 6.3 動作確認

```bash
sudo systemctl status auto-stop
sudo tail -f /var/log/auto-stop.log
```

### 6.4 完了報告

```
========================================
自動停止機能セットアップ完了
========================================
設定:
  - 閾値: SSH切断後${IDLE_HOURS}時間
  - チェック間隔: 5分

動作:
  - SSH接続がない状態が${IDLE_HOURS}時間続くと自動停止
  - SSH接続中は停止しない

管理コマンド:
  - 状態確認: sudo systemctl status auto-stop
  - ログ確認: sudo tail -f /var/log/auto-stop.log
  - 一時停止: sudo systemctl stop auto-stop
  - 無効化:   sudo systemctl disable auto-stop
========================================
```

---

## タスク7: コントロールEC2への登録

### 7.1 プライベートIP取得

```bash
PRIVATE_IP=$(aws ec2 describe-instances \
  --instance-ids ${INSTANCE_ID} \
  --query 'Reservations[0].Instances[0].PrivateIpAddress' \
  --output text \
  --region ${REGION})

echo "プライベートIP: ${PRIVATE_IP}"
```

### 7.2 エイリアス設定

SSH鍵は既に ~/.ssh/ に保存されているので、エイリアスを追加：

```bash
# エイリアス追加
echo "alias ${CLIENT_NAME}='ssh -i ~/.ssh/ar-dev-${CLIENT_NAME}-key.pem ubuntu@${PRIVATE_IP}'" >> ~/.project_aliases
source ~/.project_aliases

# 確認
alias | grep ${CLIENT_NAME}
```

### 7.3 tmux windowに追加

実行者に依頼：

```
==========================================
[要作業] tmux windowに案件を追加
==========================================

コントロールEC2のtmux内で：

1. 新規windowを作成し、エイリアスで案件EC2に接続
   $ ${CLIENT_NAME}

2. Alt + ] / Alt + [ でwindow切り替え

完了したら「完了」と入力してください。
==========================================
```

---

## タスク8: 動作確認

### [要確認] 動作確認依頼

実行者に以下を依頼：

```
==========================================
[要確認] 動作確認をお願いします
==========================================

1. Termius から コントロールEC2 に接続
2. tmux attach -t ctrl でセッション接続
3. Alt + ] で案件EC2のwindowに切り替え
4. tmux attach -t "${SESSION_NAME}" で案件tmuxセッション接続

【Tunnel URL取得】
5. get-urls コマンドを実行
6. App と code-server のURLが表示されることを確認

【アプリ確認】
7. App URLをBeam Pro Chromeでアクセス
8. "AR Dev Server Running!" が表示されることを確認

【code-server確認】
9. code-server URLをBeam Pro Chromeの別タブでアクセス
10. パスワード入力
11. VS Code UIが表示されることを確認

確認できましたか？（はい/いいえ）
==========================================
```

---

## セットアップ完了

```
==========================================
🎉 新規案件セットアップ完了！
==========================================

【作成されたリソース】
- EC2: ar-dev-${CLIENT_NAME}-server（スポットインスタンス）
  - Private IP: ${PRIVATE_IP}
  - セキュリティ: コントロールEC2からのみアクセス可能
- セキュリティグループ: ar-dev-${CLIENT_NAME}-sg
- キーペア: ar-dev-${CLIENT_NAME}-key（~/.ssh/に保存）
- IAMロール: ar-dev-${CLIENT_NAME}-role
- 自動停止: SSH切断後${IDLE_HOURS}時間で停止

【スポットインスタンスについて】
- コスト: オンデマンドの約60-70%削減
- 中断: 稀に発生（2分前に通知）
- 中断時: 自動停止（データは保持）
- 復旧: start-project ${CLIENT_NAME} で再起動

【コントロールEC2からの接続】
- エイリアス: ${CLIENT_NAME}
- tmux window切り替え: Alt + ] / Alt + [

【案件EC2内】
- tmux: ${CLIENT_NAME^}-dev
  - 各Window = Claude Code（フルスクリーン）
  - Alt+Shift+] でブランチ切り替え
- サービス管理: ~/bin/
  - start-services / stop-services / get-urls

【Beam Pro での確認】
- アプリ: get-urls でTunnel URL取得 → Chrome
- code-server: 同上
- 画面分割で両方同時表示可能

【次のステップ】
1. コントロールEC2から案件EC2に接続
   $ ${CLIENT_NAME}

2. リポジトリをクローン
   gh repo clone owner/repo-name

3. サービス起動
   start-services 3000 ~/project

4. Claude Code起動
   claude

==========================================
```
