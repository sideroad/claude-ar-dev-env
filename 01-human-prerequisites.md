# AR開発環境 - 人による事前作業ガイド

このドキュメントは、Claude Codeによる自動セットアップの**前に**完了しておく必要がある作業をまとめています。

---

## チェックリスト

### ハードウェア準備

- [ ] XREAL One Pro 購入・開封
- [ ] Beam Pro 購入・開封・初期設定完了
- [ ] Sony LinkBuds 購入・開封
- [ ] ProtoArc XK01 TP 購入・開封
- [ ] モバイルバッテリー 10000mAh 購入

### アカウント準備

- [ ] AWS アカウント作成済み
- [ ] AWS ルートユーザーMFA設定済み
- [ ] AWS 管理者IAMユーザー作成済み
- [ ] Cloudflare アカウント作成済み
- [ ] GitHub アカウント作成済み
- [ ] Tailscale アカウント作成済み

### ローカルPC（Mac/Windows）

- [ ] AWS CLI インストール済み
- [ ] AWS CLI ログイン完了（管理者IAMユーザーで `aws login`）
- [ ] Tailscale インストール・ログイン完了

### 自宅MacBook Pro（接続先Bを使う場合）

- [ ] SSH有効化（システム設定 → 一般 → 共有 → リモートログイン ON）
- [ ] Amphetamine インストール（スリープ防止）
- [ ] Homebrew インストール済み

---

## 詳細手順

### 0. AWS初期設定（初回のみ）

AWSは3段階の権限構成で運用します：

```
┌─────────────────────────────────────────────────────────┐
│ レベル1: ルートユーザー（初期設定のみ、以降は使用禁止）  │
│   └─→ レベル2: 管理者IAMユーザー（日常のAWS操作）       │
│         └─→ レベル3: EC2用IAMロール（EC2内からの操作）  │
└─────────────────────────────────────────────────────────┘
```

#### 0.1 ルートユーザーの保護

1. AWSコンソール（https://console.aws.amazon.com）にルートユーザーでログイン
2. 右上のアカウント名 → 「セキュリティ認証情報」
3. 「MFAデバイスの割り当て」→ 認証アプリ（Google Authenticator等）を設定
4. **以降ルートユーザーは使用しない**（請求設定変更など緊急時のみ）

#### 0.2 管理者IAMユーザーの作成

1. IAMコンソール → 「ユーザー」 → 「ユーザーを作成」
2. ユーザー名: `admin`（任意）
3. 「AWSマネジメントコンソールへのユーザーアクセスを提供する」にチェック
4. 「IAMユーザーを作成します」を選択
5. パスワード設定（カスタムパスワード推奨）
6. 「次へ」→ 「ポリシーを直接アタッチする」
7. `AdministratorAccess` にチェック → 「次へ」→ 「ユーザーの作成」
8. サインインURL・ユーザー名・パスワードを保存
9. 作成したユーザーでログインし直し、MFAを設定

#### 0.3 確認

```
ルートユーザー: MFA設定済み、封印
管理者IAMユーザー: MFA設定済み、日常使用
```

---

### 1. AWS CLI インストール・認証

#### Mac

```bash
# インストール
brew install awscli

# または公式インストーラー
curl "https://awscli.amazonaws.com/AWSCLIV2.pkg" -o "AWSCLIV2.pkg"
sudo installer -pkg AWSCLIV2.pkg -target /

# ログイン（管理者IAMユーザーで認証）
aws login
# ブラウザが開くので、管理者IAMユーザーでログイン
```

#### 認証確認

```bash
aws sts get-caller-identity
# 以下のような出力が表示されればOK:
# {
#     "UserId": "AIDA...",
#     "Account": "123456789012",
#     "Arn": "arn:aws:iam::123456789012:user/admin"
# }
```

> ⚠️ ルートユーザーではなく、管理者IAMユーザーでログインしていることを確認

#### 1.2 Tailscale インストール（ローカルPC）

コントロールEC2への初期設定時に使用します。

**Mac:**
```bash
# Homebrew でインストール
brew install --cask tailscale

# または公式サイトからダウンロード
# https://tailscale.com/download/mac
```

**Windows:**
```
https://tailscale.com/download/windows からダウンロード
```

インストール後：
1. Tailscale を起動
2. 「Log in」をクリック
3. ブラウザでTailscaleアカウントにログイン

---

### 2. Beam Pro 初期設定

#### 基本設定

1. 電源ON、Googleアカウントでログイン
2. システム更新を実行
3. 言語・タイムゾーン設定

#### 必須アプリインストール

Google Play Storeから以下をインストール：

- [ ] **Tailscale** - VPN接続
- [ ] **Termius** - SSH クライアント
- [ ] **Gboard** - 日本語キーボード
- [ ] **Tasker** - 自動化（有料）
- [ ] **Chrome** - ブラウザ

#### Gboard 設定

1. 設定 → システム → 言語と入力 → 画面キーボード
2. Gboard を有効化
3. Gboard設定 → 言語 → 日本語（12キー）追加
4. フリック入力のみ ON

#### Chrome 設定

1. Chrome → 設定 → サイトの設定 → PC版サイト → ON
2. ホーム画面にブックマークウィジェット配置

---

### 3. XREAL One Pro 設定

1. Beam Pro に USB-C 接続
2. XREAL アプリの指示に従って初期設定
3. 表示モード：「ブレ補正」推奨
4. 輝度：環境に合わせて調整（バッテリー消費に影響）
5. 広視野角を活かして画面分割表示を確認

---

### 4. LinkBuds 設定

1. Sony Headphones Connect アプリをインストール（Beam Pro）
2. Bluetooth ペアリング
3. アダプティブサウンドコントロール設定
4. 片耳モード確認（バッテリー節約用）

---

### 5. ProtoArc XK01 TP 設定

1. Bluetooth ペアリング（Beam Pro と）
2. キーレイアウト確認
3. トラックパッド感度調整

---

### 6. 自宅MacBook Pro 設定（接続先Bを使う場合）

#### SSH有効化

1. システム設定 → 一般 → 共有
2. 「リモートログイン」を ON
3. 許可ユーザーを設定

#### スリープ防止

1. App Store から「Amphetamine」をインストール
2. 起動して「Indefinitely」を選択
3. メニューバーから有効化

---

## Claude Code セットアップ開始前の最終確認

以下がすべて完了していることを確認してから、Claude Code用タスクを実行してください：

```
【AWS】
□ ルートユーザーMFA設定済み（封印）
□ 管理者IAMユーザー作成済み（MFA設定済み）
□ AWS CLI認証済み（管理者IAMユーザーで aws sts get-caller-identity）

【Beam Pro】
□ 初期設定完了
□ 必須アプリインストール完了（Tailscale, Termius, Gboard, Chrome）

【デバイス】
□ XREAL One Pro 接続確認済み
□ LinkBuds ペアリング済み
□ ProtoArc XK01 TP ペアリング済み
```

準備ができたら、`02-control-ec2-setup.md` を Claude Code に渡してコントロールEC2のセットアップを開始してください。
