# Claude Code 自動実行タスク: コントロールEC2セットアップ

すべての案件EC2を管理するコントロールEC2を構築します。
**初回のみ実行**（全案件共通で使用）

---

## 概要

```
Beam Pro
    │
    └── Termius（1接続のみ）
            │
            └── コントロールEC2 (t4g.micro) ──── 常時起動
                    │
                    ├── AWS CLI（案件EC2の起動/停止）
                    ├── SSH鍵（各案件EC2への接続用）
                    │
                    └── tmux [ctrl]
                          ├── window 0: 管理用シェル
                          ├── window 1: Acme EC2 へSSH
                          ├── window 2: Beta EC2 へSSH
                          └── ...
```

---

## 実行前チェック

```
AWS CLI認証済みですか？（aws sts get-caller-identity で確認）
```

---

## タスク1: コントロールEC2作成

### 1.1 変数設定

```bash
INSTANCE_NAME="ar-dev-control"
REGION="ap-northeast-1"
INSTANCE_TYPE="t4g.micro"
KEY_NAME="ar-dev-control-key"
SG_NAME="ar-dev-control-sg"
```

### 1.2 キーペア作成

```bash
aws ec2 create-key-pair \
  --key-name ${KEY_NAME} \
  --query 'KeyMaterial' \
  --output text \
  --region ${REGION} > ~/.ssh/${KEY_NAME}.pem

chmod 600 ~/.ssh/${KEY_NAME}.pem
echo "キーペア作成完了: ~/.ssh/${KEY_NAME}.pem"
```

### 1.3 セキュリティグループ作成

```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text \
  --region ${REGION})

SG_ID=$(aws ec2 create-security-group \
  --group-name ${SG_NAME} \
  --description "Security group for AR dev control server" \
  --vpc-id ${VPC_ID} \
  --query 'GroupId' \
  --output text \
  --region ${REGION})

# SSH許可（初期設定用、Tailscale設定後に削除）
aws ec2 authorize-security-group-ingress \
  --group-id ${SG_ID} \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0 \
  --region ${REGION}

echo "セキュリティグループ作成完了: ${SG_ID}"
echo "※ SSHは初期設定用に一時開放。Tailscale設定後に閉鎖します。"
```

### 1.4 EC2インスタンス作成

```bash
# Ubuntu 24.04 LTS AMI（東京リージョン、ARM64）
AMI_ID=$(aws ec2 describe-images \
  --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-arm64-server-*" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text \
  --region ${REGION})

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id ${AMI_ID} \
  --instance-type ${INSTANCE_TYPE} \
  --key-name ${KEY_NAME} \
  --security-group-ids ${SG_ID} \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":20,"VolumeType":"gp3"}}]' \
  --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${INSTANCE_NAME}},{Key=Role,Value=control},{Key=Environment,Value=development}]" \
  --query 'Instances[0].InstanceId' \
  --output text \
  --region ${REGION})

echo "インスタンス作成中: ${INSTANCE_ID}"
aws ec2 wait instance-running --instance-ids ${INSTANCE_ID} --region ${REGION}
echo "インスタンス起動完了"
```

### 1.5 Elastic IP割り当て

```bash
ALLOCATION_ID=$(aws ec2 allocate-address \
  --domain vpc \
  --query 'AllocationId' \
  --output text \
  --region ${REGION})

aws ec2 associate-address \
  --instance-id ${INSTANCE_ID} \
  --allocation-id ${ALLOCATION_ID} \
  --region ${REGION}

ELASTIC_IP=$(aws ec2 describe-addresses \
  --allocation-ids ${ALLOCATION_ID} \
  --query 'Addresses[0].PublicIp' \
  --output text \
  --region ${REGION})

aws ec2 create-tags \
  --resources ${ALLOCATION_ID} \
  --tags Key=Name,Value=${INSTANCE_NAME}-eip \
  --region ${REGION}

echo "Elastic IP: ${ELASTIC_IP}"
```

### 1.6 IAMロール作成・アタッチ

```bash
ROLE_NAME="ar-dev-control-role"
PROFILE_NAME="ar-dev-control-profile"

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

# EC2操作権限（案件EC2の起動/停止用）
cat << 'EOF' > /tmp/ec2-control-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:StartInstances",
        "ec2:StopInstances"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ec2:ResourceTag/Environment": "development"
        }
      }
    },
    {
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name ${ROLE_NAME} \
  --policy-name ec2-control-policy \
  --policy-document file:///tmp/ec2-control-policy.json

# インスタンスプロファイル
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

### 1.7 作成完了報告

```
==========================================
コントロールEC2作成完了
==========================================
インスタンス名: ar-dev-control
インスタンスID: ${INSTANCE_ID}
Elastic IP: ${ELASTIC_IP}
キーペア: ~/.ssh/ar-dev-control-key.pem

SSH接続コマンド:
ssh -i ~/.ssh/ar-dev-control-key.pem ubuntu@${ELASTIC_IP}

【重要】このEC2は常時起動です
==========================================
```

---

## タスク2: コントロールEC2環境構築

SSH接続後、以下を実行。

### 2.1 システム更新・基本ツール

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y git curl wget unzip jq
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

```bash
aws sts get-caller-identity
# control-role が表示されればOK
```

### 2.4 tmux インストール・設定

```bash
sudo apt install -y tmux

cat << 'EOF' > ~/.tmux.conf
# ステータスバー
set -g status-left "[ctrl] "
set -g status-left-length 20
set -g status-right "#[fg=cyan]%Y-%m-%d %H:%M"

# window名を自動リネームしない
set -g allow-rename off

# マウス有効
set -g mouse on

# 履歴
set -g history-limit 10000

# 256色
set -g default-terminal "screen-256color"

# windowインデックスを1から
set -g base-index 1
setw -g pane-base-index 1

# Alt + [ / Alt + ] でWindow切り替え（Beam Pro対応）
bind -n M-[ previous-window
bind -n M-] next-window
EOF
```

### 2.5 管理用スクリプト作成

```bash
mkdir -p ~/bin

# 案件一覧表示
cat << 'EOF' > ~/bin/list-projects
#!/bin/bash
echo "=== 案件EC2一覧 ==="
aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=development" "Name=tag:Name,Values=ar-dev-*-server" \
  --query 'Reservations[].Instances[].[Tags[?Key==`Name`].Value|[0],Tags[?Key==`Client`].Value|[0],InstanceId,State.Name,PrivateIpAddress]' \
  --output table \
  --region ap-northeast-1
EOF
chmod +x ~/bin/list-projects

# 案件EC2起動
cat << 'EOF' > ~/bin/start-project
#!/bin/bash
CLIENT=$1
if [ -z "$CLIENT" ]; then
  echo "Usage: start-project <client-name>"
  exit 1
fi

INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ar-dev-${CLIENT}-server" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text \
  --region ap-northeast-1)

if [ "$INSTANCE_ID" = "None" ] || [ -z "$INSTANCE_ID" ]; then
  echo "エラー: ar-dev-${CLIENT}-server が見つかりません"
  exit 1
fi

echo "起動中: ar-dev-${CLIENT}-server ($INSTANCE_ID)"
aws ec2 start-instances --instance-ids $INSTANCE_ID --region ap-northeast-1
echo "起動コマンド送信完了（1-2分で接続可能）"
EOF
chmod +x ~/bin/start-project

# 案件EC2停止
cat << 'EOF' > ~/bin/stop-project
#!/bin/bash
CLIENT=$1
if [ -z "$CLIENT" ]; then
  echo "Usage: stop-project <client-name>"
  exit 1
fi

INSTANCE_ID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=ar-dev-${CLIENT}-server" \
  --query 'Reservations[0].Instances[0].InstanceId' \
  --output text \
  --region ap-northeast-1)

if [ "$INSTANCE_ID" = "None" ] || [ -z "$INSTANCE_ID" ]; then
  echo "エラー: ar-dev-${CLIENT}-server が見つかりません"
  exit 1
fi

echo "停止中: ar-dev-${CLIENT}-server ($INSTANCE_ID)"
aws ec2 stop-instances --instance-ids $INSTANCE_ID --region ap-northeast-1
echo "停止コマンド送信完了"
EOF
chmod +x ~/bin/stop-project

# 全案件起動
cat << 'EOF' > ~/bin/start-all
#!/bin/bash
echo "=== 全案件EC2起動 ==="
INSTANCE_IDS=$(aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=development" "Name=tag:Name,Values=ar-dev-*-server" "Name=instance-state-name,Values=stopped" \
  --query 'Reservations[].Instances[].InstanceId' \
  --output text \
  --region ap-northeast-1)

if [ -z "$INSTANCE_IDS" ]; then
  echo "停止中の案件EC2はありません"
  exit 0
fi

echo "起動対象: $INSTANCE_IDS"
aws ec2 start-instances --instance-ids $INSTANCE_IDS --region ap-northeast-1
echo "起動コマンド送信完了（1-2分で接続可能）"
EOF
chmod +x ~/bin/start-all

# 全案件停止
cat << 'EOF' > ~/bin/stop-all
#!/bin/bash
echo "=== 全案件EC2停止 ==="
INSTANCE_IDS=$(aws ec2 describe-instances \
  --filters "Name=tag:Environment,Values=development" "Name=tag:Name,Values=ar-dev-*-server" "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].InstanceId' \
  --output text \
  --region ap-northeast-1)

if [ -z "$INSTANCE_IDS" ]; then
  echo "起動中の案件EC2はありません"
  exit 0
fi

echo "停止対象: $INSTANCE_IDS"
aws ec2 stop-instances --instance-ids $INSTANCE_IDS --region ap-northeast-1
echo "停止コマンド送信完了"
EOF
chmod +x ~/bin/stop-all

# PATHに追加
echo 'export PATH="$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 2.6 案件接続用エイリアス設定ファイル

```bash
cat << 'EOF' > ~/.project_aliases
# 案件接続エイリアス
# 新しい案件を追加したらここにエイリアスを追加

# 例:
# alias acme='ssh -i ~/.ssh/ar-dev-acme-key.pem ubuntu@<Private IP>'
# alias beta='ssh -i ~/.ssh/ar-dev-beta-key.pem ubuntu@<Private IP>'
EOF

echo 'source ~/.project_aliases' >> ~/.bashrc
```

### 2.7 tmuxセッション作成スクリプト

```bash
cat << 'EOF' > ~/bin/start-ctrl
#!/bin/bash
# コントロール用tmuxセッション作成

SESSION="ctrl"

# 既存セッションがあればアタッチ
tmux has-session -t $SESSION 2>/dev/null
if [ $? -eq 0 ]; then
  echo "既存セッションにアタッチします"
  tmux attach -t $SESSION
  exit 0
fi

# 新規セッション作成
tmux new-session -d -s $SESSION -n "main"

# window 0: メイン（管理用）
tmux send-keys -t $SESSION:1 "echo '=== コントロールEC2 ==='; list-projects" C-m

echo "tmuxセッション '$SESSION' を作成しました"
echo "アタッチします..."
tmux attach -t $SESSION
EOF
chmod +x ~/bin/start-ctrl
```

---

## タスク3: Node.js インストール（fnm）

```bash
# fnm インストール
curl -fsSL https://fnm.vercel.app/install | bash

# シェル設定
echo 'FNM_PATH="$HOME/.local/share/fnm"' >> ~/.bashrc
echo 'if [ -d "$FNM_PATH" ]; then' >> ~/.bashrc
echo '  export PATH="$FNM_PATH:$PATH"' >> ~/.bashrc
echo '  eval "$(fnm env)"' >> ~/.bashrc
echo 'fi' >> ~/.bashrc

source ~/.bashrc

# Node.js LTS インストール
fnm install --lts
fnm use lts-latest
fnm default lts-latest

node -v
npm -v
```

---

## タスク4: Claude Code インストール

```bash
npm install -g @anthropic-ai/claude-code
claude --version
```

### [要作業] Claude Code認証

```
==========================================
[要作業] Claude Code認証が必要です
==========================================

以下のコマンドを実行してください：

  claude

初回起動時に認証が求められます。
ブラウザでAnthropicアカウントにログインして認証してください。

完了したら「認証完了」と入力してください。
==========================================
```

### 4.1 CLAUDE.md とカスタムコマンド配置

```bash
# CLAUDE.md作成
cat << 'EOF' > ~/CLAUDE.md
# コントロールEC2 プロジェクトルール

## 役割

このEC2はAR開発環境の管理サーバーです。

- 案件EC2の作成・起動・停止・削除
- 案件EC2へのSSH接続（tmux window経由）
- 常時起動（自動停止なし）

---

## カスタムコマンド

| コマンド | 用途 |
|----------|------|
| `/project:new-project` | 新規案件EC2作成 |
| `/project:delete-project` | 案件EC2削除 |

---

## シェルコマンド（~/bin/）

```bash
list-projects      # 案件一覧
start-project XX   # 案件XX起動
stop-project XX    # 案件XX停止
start-all          # 全案件起動
stop-all           # 全案件停止
```

---

## tmux操作

```
Alt + ]    → 次のwindow
Alt + [    → 前のwindow
```

---

## ドキュメント

| ファイル | 用途 |
|----------|------|
| ~/docs/03-claude-code-new-project-setup.md | 案件EC2作成 |
| ~/docs/04-claude-code-operations.md | 運用タスク |

---

## 重要なルール

1. 案件EC2の作成は /project:new-project を使用
2. SSH鍵は ~/.ssh/ に自動保存
3. エイリアスは ~/.project_aliases に自動追加
EOF

# カスタムコマンドディレクトリ作成
mkdir -p ~/.claude/commands

# new-project.md
cat << 'EOF' > ~/.claude/commands/new-project.md
# 新規案件EC2作成

新しい案件用のEC2インスタンスを作成します。

## 使い方

```
/project:new-project
```

## 実行内容

~/docs/03-claude-code-new-project-setup.md を読み込んで実行してください。

このドキュメントには以下のタスクが含まれています：
- EC2インスタンス作成
- 環境構築（Node.js, Claude Code, tmux等）
- GitHub/Cloudflare認証
- 自動停止設定
EOF

# delete-project.md
cat << 'EOF' > ~/.claude/commands/delete-project.md
# 案件EC2削除

案件用のEC2インスタンスと関連リソースを削除します。

## 使い方

```
/project:delete-project クライアント名
```

## 引数

- $ARGUMENTS: クライアント名（例: acme）

## 実行前確認

以下のリソースが削除されます：
- EC2インスタンス: ar-dev-{client}-server
- セキュリティグループ: ar-dev-{client}-sg
- キーペア: ar-dev-{client}-key
- IAMロール: ar-dev-{client}-role
- SSH鍵: ~/.ssh/ar-dev-{client}-key.pem
- エイリアス

続行する場合は 04-claude-code-operations.md の「案件削除」セクションを参照してください。
EOF

# docsディレクトリ作成
mkdir -p ~/docs

echo "CLAUDE.md とカスタムコマンド配置完了"
echo "※ ~/docs/ に 03-claude-code-new-project-setup.md と 04-claude-code-operations.md を配置してください"
```

---

## タスク5: GitHub CLI インストール

```bash
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg
sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null
sudo apt update
sudo apt install gh -y
```

---

## タスク6: GitHub認証

### [要作業] GitHub認証

```
==========================================
[要作業] GitHub認証が必要です
==========================================

以下のコマンドを実行してください：

  gh auth login

対話形式で以下を選択：
  1. GitHub.com
  2. SSH
  3. Yes (SSH鍵生成)
  4. Login with a web browser

完了したら「認証完了」と入力してください。
==========================================
```

認証完了後：

```bash
gh auth status
```

---

## タスク7: Tailscale インストール

### 7.1 Tailscale インストール

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

### 7.2 Tailscale 起動・認証

```bash
sudo tailscale up
```

表示されるURLをブラウザで開き、Tailscaleアカウントで認証してください。

### [要作業] Tailscale認証

```
==========================================
[要作業] Tailscale認証が必要です
==========================================

1. 上記コマンド実行後に表示されるURLをコピー
2. ブラウザでそのURLを開く
3. Tailscaleアカウントでログイン（なければ作成）
4. デバイスを承認

完了したら「認証完了」と入力してください。
==========================================
```

### 7.3 Tailscale IP確認

```bash
TAILSCALE_IP=$(tailscale ip -4)
echo "Tailscale IP: ${TAILSCALE_IP}"
```

**このIPをメモしてください。Termius接続先に使用します。**

---

## タスク8: セキュリティ強化（SSHルール削除）

Tailscale経由で接続できることを確認後、SSHポートを閉鎖します。

### 8.1 Tailscale経由での接続確認

**新しいターミナル（ローカルPC）から確認：**

```bash
# ローカルPCにもTailscaleをインストール済みの場合
ssh -i ~/.ssh/ar-dev-control-key.pem ubuntu@${TAILSCALE_IP}
```

### [要確認] Tailscale接続確認

```
==========================================
[要確認] Tailscale経由で接続できましたか？
==========================================

新しいターミナルからTailscale IP経由でSSH接続を試してください。

接続できた場合のみ「はい」と回答してください。
接続できない場合は「いいえ」と回答し、トラブルシューティングします。
==========================================
```

### 8.2 SSHルール削除

「はい」の場合、セキュリティグループからSSHルールを削除：

```bash
# セキュリティグループIDを取得
SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=ar-dev-control-sg" \
  --query 'SecurityGroups[0].GroupId' \
  --output text \
  --region ${REGION})

# SSH (0.0.0.0/0) ルールを削除
aws ec2 revoke-security-group-ingress \
  --group-id ${SG_ID} \
  --protocol tcp \
  --port 22 \
  --cidr 0.0.0.0/0 \
  --region ${REGION}

echo "SSHルール削除完了。以降はTailscale経由でのみ接続可能です。"
```

---

## タスク9: ドキュメント転送

### [要作業] ドキュメントをコントロールEC2に転送

ローカルPCで以下を実行（Tailscale IP経由）：

```bash
TAILSCALE_IP="コントロールEC2のTailscale IP"

# ドキュメントを転送
scp -i ~/.ssh/ar-dev-control-key.pem \
    03-claude-code-new-project-setup.md \
    04-claude-code-operations.md \
    ubuntu@${TAILSCALE_IP}:~/docs/

echo "ドキュメント転送完了"
```

転送完了したら「転送完了」と入力してください。

---

## タスク10: Termius設定依頼

### [要作業] Termius設定

```
==========================================
[要作業] Termius設定
==========================================

【事前準備】
Beam Pro に Tailscale アプリをインストールし、
同じアカウントでログインしてください。

【Termius設定】
  Label: 🎮 Control
  Hostname: ${TAILSCALE_IP}  ← Tailscale IP（100.x.x.x）
  Username: ubuntu
  認証: キーペア（ar-dev-control-key.pem をインポート）

※ 今後はこの1つの接続先のみ使用します
※ Elastic IPではなくTailscale IPを使用

設定完了したら「設定完了」と入力してください。
==========================================
```

---

## タスク11: 動作確認

### [要確認] 動作確認

```
==========================================
[要確認] 動作確認
==========================================

1. Beam Pro の Tailscale がコントロールEC2と同じネットワークにいることを確認
2. Termius から コントロールEC2 に接続（Tailscale IP経由）
3. start-ctrl コマンドでtmuxセッション開始
4. list-projects で案件一覧が表示されることを確認
5. claude コマンドでClaude Codeが起動することを確認
6. /project:new-project コマンドが認識されることを確認

確認できましたか？（はい/いいえ）
==========================================
```

---

## セットアップ完了

```
==========================================
🎉 コントロールEC2セットアップ完了
==========================================

【作成されたリソース】
- EC2: ar-dev-control - t4g.micro
- セキュリティグループ: ar-dev-control-sg（SSHポート閉鎖済み）
- キーペア: ar-dev-control-key
- IAMロール: ar-dev-control-role（EC2起動/停止権限）

【インストール済み】
- AWS CLI
- Node.js (fnm)
- Claude Code ← 案件EC2の作成に使用
- GitHub CLI
- tmux
- Tailscale ← セキュアな接続

【セキュリティ】
- SSHポート: 閉鎖（Tailscale経由でのみ接続可能）
- Tailscale IP: ${TAILSCALE_IP}

【接続情報】
- Termius: 🎮 Control → ${TAILSCALE_IP}
- tmuxセッション: ctrl

【管理コマンド】
- list-projects    : 案件一覧
- start-project XX : 案件XX起動
- stop-project XX  : 案件XX停止
- start-all        : 全案件起動
- stop-all         : 全案件停止

【次のステップ】
コントロールEC2内で claude を起動し、
/project:new-project で案件EC2を作成

==========================================
```

---

## 新規案件追加の流れ

```
【コントロールEC2内で実行】

1. Claude Code を起動
   $ claude

2. 案件EC2作成を依頼
   > 03-claude-code-new-project-setup.md を読んで、
   > 新規案件のEC2をセットアップしてください。
   > クライアント名: acme

3. Claude Code が案件EC2を作成
   - EC2作成
   - 環境構築
   - SSH鍵作成（~/.ssh/に保存される）
   - エイリアス追加（~/.project_aliasesに追加される）

4. tmux windowに追加
   新規windowを作成して acme と入力（エイリアスで接続）
   Alt + ] / Alt + [ でwindow切り替え

※ ローカルPCでの作業は不要！
```

---

## 補足：SSH鍵の場所

案件EC2を作成すると、SSH鍵は以下に保存されます：

```
コントロールEC2:
~/.ssh/ar-dev-acme-key.pem
~/.ssh/ar-dev-beta-key.pem
...
```

ローカルPCにもバックアップを取りたい場合：

```bash
# ローカルPCで実行
scp -i ~/.ssh/ar-dev-control-key.pem \
    ubuntu@${CONTROL_IP}:~/.ssh/ar-dev-*-key.pem \
    ~/.ssh/
```
