# AR開発環境 構成図

## 1. 全体構成

```mermaid
flowchart TB
    subgraph User["👤 ユーザー装備"]
        XREAL["🕶️ XREAL One Pro<br/>AR表示（広視野角）"]
        LinkBuds["🎧 LinkBuds<br/>音声入力"]
        Keyboard["⌨️ ProtoArc XK01 TP<br/>キーボード+トラックパッド"]
        Battery["🔋 モバイルバッテリー<br/>10000mAh"]
    end

    subgraph BeamPro["📱 Beam Pro"]
        Termius["Termius<br/>SSH（1接続のみ）"]
        Chrome["Chrome<br/>ブラウザ"]
        Tailscale_App["Tailscale<br/>VPN"]
    end

    subgraph Cloud["☁️ AWS"]
        subgraph Control["🎮 ar-dev-control（常時起動）"]
            CtrlTmux["tmux [ctrl]"]
            CtrlAWS["AWS CLI<br/>EC2起動/停止"]
        end
        subgraph EC2_A["🟢 ar-dev-acme-server"]
            Claude_A["Claude Code"]
            Tmux_A["tmux"]
            Tunnel_A["Cloudflare Tunnel"]
        end
        subgraph EC2_B["🟡 ar-dev-beta-server"]
            Claude_B["Claude Code"]
            Tmux_B["tmux"]
            Tunnel_B["Cloudflare Tunnel"]
        end
    end

    subgraph Home["🏠 自宅"]
        Mac["🔵 MacBook Pro<br/>特定案件用"]
    end

    XREAL <-->|USB-C| BeamPro
    LinkBuds <-->|Bluetooth| BeamPro
    Keyboard <-->|Bluetooth| BeamPro
    Battery -->|給電| BeamPro

    Termius -->|SSH| Control
    Control -->|SSH<br/>Alt で切替| EC2_A
    Control -->|SSH| EC2_B
    Control -->|Tailscale| Mac
    CtrlAWS -->|起動/停止| EC2_A
    CtrlAWS -->|起動/停止| EC2_B
    Chrome -->|HTTPS| Tunnel_A
    Chrome -->|HTTPS| Tunnel_B
```

---

## 2. 利用シーン別構成

場所や状況に応じて、使用するデバイスと入力方法が異なります。

### 一覧

| シーン | AR表示 | 本体 | 入力 | 電源 | 想定時間 |
|--------|--------|------|------|------|----------|
| 🏠 自宅・外出先 | XREAL One Pro | Beam Pro | キーボード | 給電 | 長時間 |
| ☕ カフェ | XREAL One Pro | Beam Pro | キーボード | バッテリー | 2-3時間 |
| 🌳 公園・散歩 | XREAL One Pro | Beam Pro | 音声(LinkBuds) | バッテリー | 1時間 |
| 🚃 電車（座席時） | XREAL One Pro | Beam Pro | Gboard QWERTY | バッテリー | 30分 |

### 🏠 自宅・外出先（長時間作業）

```mermaid
flowchart LR
    subgraph Devices["デバイス構成"]
        XREAL["🕶️ XREAL One Pro"]
        Beam["📱 Beam Pro<br/>（USB-C×2ポート）"]
        KB["⌨️ ProtoArc XK01 TP"]
        Power["🔌 給電（コンセント）"]
    end

    subgraph Input["入力方法"]
        Type["⌨️ キーボード入力"]
    end

    XREAL <-->|USB-C<br/>ポート1| Beam
    KB -->|Bluetooth| Beam
    Power -->|USB-C<br/>ポート2| Beam
    KB --> Type
```

**特徴**
- Beam Proは2つのUSB-Cポートを搭載（グラス用 + 電源用）
- グラス接続と充電を同時に行えるため時間制限なし
- フルキーボード入力で本格的な開発
- トラックパッドでマウス操作も可能

> 💡 Beam ProはXREALグラスを接続しながら充電可能（パススルー充電アダプタ不要）

---

### ☕ カフェ（中時間作業）

```mermaid
flowchart LR
    subgraph Devices["デバイス構成"]
        XREAL["🕶️ XREAL One Pro"]
        Beam["📱 Beam Pro"]
        KB["⌨️ ProtoArc XK01 TP"]
        Battery["🔋 バッテリー"]
    end

    subgraph Input["入力方法"]
        Type["⌨️ キーボード入力"]
    end

    XREAL <-->|USB-C| Beam
    KB -->|Bluetooth| Beam
    Battery -->|USB-C| Beam
    KB --> Type
```

**特徴**
- バッテリー駆動（2-3時間目安）
- コンパクトな荷物で移動
- WiFi環境推奨

---

### 🌳 公園・散歩（ハンズフリー作業）

```mermaid
flowchart LR
    subgraph Devices["デバイス構成"]
        XREAL["🕶️ XREAL One Pro"]
        Beam["📱 Beam Pro<br/>ポケット収納"]
        LinkBuds["🎧 LinkBuds"]
        Battery["🔋 バッテリー"]
    end

    subgraph Input["入力方法"]
        Voice["🎤 音声指示"]
    end

    XREAL <-->|USB-C| Beam
    LinkBuds -->|Bluetooth| Beam
    Battery -->|USB-C| Beam
    LinkBuds --> Voice
```

**特徴**
- 完全ハンズフリー
- Claude Codeへの音声指示でコード生成・レビュー
- 歩きながらアイデア整理・設計検討
- キーボード不要で荷物最小

**音声指示の例**
```
「この関数にエラーハンドリングを追加して」
「テストコードを書いて」
「このコードの問題点を教えて」
```

---

### 🚃 電車（座席確保時・短時間作業）

```mermaid
flowchart LR
    subgraph Devices["デバイス構成"]
        XREAL["🕶️ XREAL One Pro"]
        Beam["📱 Beam Pro"]
        Battery["🔋 バッテリー"]
    end

    subgraph Input["入力方法"]
        Touch["⌨️ Gboard QWERTY<br/>（横向き両手入力）"]
    end

    XREAL <-->|USB-C| Beam
    Battery -->|USB-C| Beam
    Beam --> Touch
```

**特徴**
- 最小構成（ARグラス + Beam Pro + バッテリーのみ）
- 横向き + QWERTY配列で両手入力
- ARグラスの下からBeam Pro画面を見て入力
- キーボード不要で荷物削減

**向いている作業**
- Claude Codeへの短〜中程度の指示
- コードレビューの確認・コメント
- PRのマージ・作成
- Slackの確認・返信
- 軽微なコード修正

---

### シーン別比較

| シーン | 作業時間 | 作業内容 | 入力方法 | 通信 |
|--------|----------|----------|----------|------|
| 🏠 自宅・外出先 | ★★★ 長時間 | 本格開発 | 外付キーボード | WiFi/有線 |
| ☕ カフェ | ★★☆ 中時間 | 本格開発 | 外付キーボード | WiFi |
| 🌳 公園・散歩 | ★☆☆ 短時間 | 設計・レビュー | 音声 (LinkBuds) | テザリング |
| 🚃 電車（座席） | ★☆☆ 短時間 | 中程度の作業 | Gboard QWERTY | モバイル回線 |

```mermaid
flowchart LR
    subgraph 作業強度
        Heavy["🏠☕ 本格開発"]
        Medium["🚃 中程度"]
        Light["🌳 軽作業"]
    end

    subgraph 入力方法
        KB["⌨️ 外付キーボード<br/>🏠☕"]
        Gboard["⌨️ Gboard QWERTY<br/>🚃"]
        Voice["🎤 音声<br/>🌳"]
    end

    Heavy --> KB
    Medium --> Gboard
    Light --> Voice
```

---

## 3. EC2内部構成

```mermaid
flowchart TB
    subgraph EC2["EC2インスタンス（案件ごと）"]
        subgraph Tools["インストール済みツール"]
            AWS_CLI["AWS CLI"]
            NodeJS["Node.js (fnm)"]
            GH["GitHub CLI"]
            Playwright["Playwright"]
            Cloudflared["cloudflared"]
            CodeServer["code-server<br/>ブラウザVS Code"]
        end

        subgraph Tmux["tmux セッション"]
            Pane0["ペイン0<br/>開発サーバー<br/>npm/pnpm/yarn dev"]
            Pane1["ペイン1<br/>Tunnel (アプリ)<br/>→ localhost:3000"]
            Pane2["ペイン2<br/>Tunnel (code-server)<br/>→ localhost:8080"]
        end

        subgraph Services["systemd サービス"]
            AutoStop["auto-stop.service<br/>アイドル検知→自動停止"]
            CodeServerSvc["code-server.service<br/>ブラウザIDE"]
        end

        IAMRole["IAMロール<br/>他環境操作権限"]
    end

    subgraph External["外部サービス"]
        GitHub["GitHub<br/>リポジトリ"]
        CF["Cloudflare<br/>Tunnel URL発行"]
        Staging["ステージング環境"]
        Prod["本番環境"]
    end

    subgraph BeamPro["📱 Beam Pro"]
        Chrome["Chrome<br/>アプリ確認 + code-server"]
    end

    GH <-->|認証済み| GitHub
    Cloudflared <-->|Tunnel| CF
    AWS_CLI -->|ログ確認・操作| Staging
    AWS_CLI -->|ログ確認・操作| Prod
    AutoStop -->|SSH切断後3時間| EC2
    Chrome -->|HTTPS| Pane1
    Chrome -->|HTTPS| Pane2
```

---

## 4. 作業分担

```mermaid
flowchart LR
    subgraph Human["👤 人の作業"]
        H1["ハードウェア購入・接続"]
        H2["aws login"]
        H3["gh auth login"]
        H4["cloudflared tunnel login"]
        H5["tailscale up"]
        H6["Beam Pro設定"]
    end

    subgraph Claude["🤖 Claude Code作業"]
        C1["EC2作成・削除"]
        C2["IAMロール作成\n自動認証"]
        C3["環境構築"]
        C4["GitHub操作\nclone/PR/merge"]
        C5["開発サーバー操作\ntmux send-keys経由"]
        C6["障害調査\nログ・メトリクス"]
        C7["EC2起動・停止"]
    end

    subgraph Auto["⚙️ 自動"]
        A1["SSH切断後3時間で\nEC2自動停止"]
    end

    H2 -->|認証後| C1
    C1 --> C2
    H3 -->|認証後| C4
    H4 -->|認証後| C5
    C5 --> A1
```

---

## 5. 案件ライフサイクル

```mermaid
flowchart TB
    subgraph Setup["セットアップ（初回）"]
        S1["03-claude-code-new-project-setup.md<br/>を渡す"]
        S2["案件情報入力<br/>クライアント名等"]
        S3["EC2作成<br/>自動実行"]
        S4["環境構築<br/>自動実行"]
        S5["認証作業<br/>GitHub/Cloudflare"]
        S6["動作確認"]
    end

    subgraph Daily["日次運用"]
        D1["EC2起動<br/>（停止中の場合）"]
        D2["tmux接続"]
        D3["開発作業<br/>Claude Codeに依頼"]
        D4["SSH切断"]
        D5["SSH切断後<br/>3時間で自動停止"]
    end

    subgraph End["案件終了"]
        E1["リソース削除<br/>EC2/SG/IAM等"]
    end

    S1 --> S2 --> S3 --> S4 --> S5 --> S6
    S6 --> Daily
    D1 --> D2 --> D3 --> D4 --> D5
    D5 -.->|翌日| D1
    Daily --> E1
```

---

## 6. 自動停止判定ロジック

```mermaid
flowchart TB
    Start["5分ごとにチェック"]
    
    SSH{"SSH接続<br/>あり？"}
    
    Active["✅ 接続中<br/>タイマーリセット"]
    Idle["⏳ 未接続<br/>タイマー継続"]
    
    Check3h{"3時間<br/>経過？"}
    Stop["🛑 EC2停止"]

    Start --> SSH
    SSH -->|はい| Active
    SSH -->|いいえ| Idle
    
    Idle --> Check3h
    Check3h -->|いいえ| Start
    Check3h -->|はい| Stop
    Active --> Start
```

### ポイント

- **シンプルな判定**: SSH接続の有無のみ
- **3時間の猶予**: Claude Codeに依頼して離席しても安心
- **案件切り替え対応**: 複数案件を起動したまま作業可能

---

## 7. コスト構造

```mermaid
pie showData
    title "24時間稼働時 月額コスト $5.4（スポット）"
    "EC2 (t4g.small スポット $0.0035/h)" : 2.52
    "EBS (30GB gp3 $0.096/GB)" : 2.88
```

```mermaid
pie showData
    title "自動停止運用時 月額コスト 約$4（8h/日）"
    "EC2 (実稼働分)" : 0.84
    "EBS (30GB gp3)" : 2.88
```

---

## 8. ファイル構成

```mermaid
flowchart LR
    subgraph Docs["ドキュメント構成"]
        Doc1["01-human-prerequisites.md<br/>👤 事前作業ガイド"]
        Doc2["03-claude-code-new-project-setup.md<br/>🤖 新規案件セットアップ"]
        Doc3["04-claude-code-operations.md<br/>🤖 運用タスク集"]
    end

    Human["👤 人"] -->|読んで実行| Doc1
    Doc1 -->|完了後| Doc2
    Doc2 -->|渡す| Claude["🤖 Claude Code"]
    Claude -->|自動実行| Setup["セットアップ完了"]
    Setup --> Doc3
    Doc3 -->|日常運用| Claude
```

---

## 9. ネットワーク構成

```mermaid
flowchart TB
    subgraph Internet["インターネット"]
        CF_Edge["Cloudflare Edge"]
    end

    subgraph AWS_Region["AWS ap-northeast-1"]
        subgraph VPC["Default VPC"]
            subgraph EC2_SG["Security Group (SSH許可)"]
                EC2["EC2インスタンス<br/>Elastic IP付与"]
            end
        end
        CW["CloudWatch Logs"]
        S3["S3 (ログ)"]
    end

    subgraph Home["自宅ネットワーク"]
        Mac["MacBook Pro"]
        TS_Relay["Tailscale DERP"]
    end

    subgraph Mobile["モバイル"]
        BeamPro["Beam Pro<br/>4G/5G or WiFi"]
    end

    BeamPro -->|SSH:22| EC2
    BeamPro -->|HTTPS| CF_Edge
    CF_Edge -->|Tunnel| EC2
    BeamPro -->|Tailscale| TS_Relay
    TS_Relay -->|Tailscale| Mac
    EC2 -->|IAMロール| CW
    EC2 -->|IAMロール| S3
```

---

## 10. AWS権限構成（3段階）

```mermaid
flowchart TB
    subgraph Level1["レベル1: ルートユーザー"]
        Root["🔐 ルートユーザー<br/>全権限"]
    end

    subgraph Level2["レベル2: 管理者IAMユーザー"]
        Admin["👤 管理者IAMユーザー<br/>AdministratorAccess"]
    end

    subgraph Level3["レベル3: EC2用IAMロール"]
        Role["🎫 IAMロール<br/>必要最小限の権限"]
    end

    Root -->|"初回のみ作成<br/>以降ルートは封印"| Admin
    Admin -->|"案件ごとに作成<br/>aws login で認証"| Role
    Role -->|"アタッチ<br/>自動認証"| EC2["☁️ EC2インスタンス"]

    style Root fill:#ff6b6b,color:#fff
    style Admin fill:#4ecdc4,color:#fff
    style Role fill:#45b7d1,color:#fff
```

### 各レベルの役割

| レベル | ユーザー | 用途 | 認証 |
|--------|----------|------|------|
| 1 | ルートユーザー | 初期設定のみ | MFA必須、封印 |
| 2 | 管理者IAMユーザー | EC2作成、IAMロール作成 | `aws login` |
| 3 | EC2用IAMロール | EC2内からのAWS操作 | 自動（設定不要） |

---

## 11. コントロールEC2からの案件管理

```mermaid
flowchart LR
    subgraph BeamPro["📱 Beam Pro"]
        Termius["Termius\n1接続のみ"]
    end

    subgraph AWS["☁️ AWS"]
        subgraph Control["🎮 コントロールEC2\n常時起動"]
            tmux["tmux [ctrl]"]
            scripts["管理スクリプト\nstart-project\nstop-project\nlist-projects"]
        end
        EC2_A["🟢 案件A EC2"]
        EC2_B["🟡 案件B EC2"]
    end

    Termius -->|SSH| Control
    tmux -->|Alt| EC2_A
    tmux -->|Alt| EC2_B
    scripts -->|AWS CLI| EC2_A
    scripts -->|AWS CLI| EC2_B
```

### 管理コマンド（コントロールEC2内）

```
list-projects     : 案件一覧
start-project XX  : 案件XX起動
stop-project XX   : 案件XX停止
start-all         : 全案件起動
stop-all          : 全案件停止
```

### tmux window切り替え

```
Alt + ]    → 次のwindow
Alt + [    → 前のwindow
```

### メリット

- キーボードだけで全操作完結
- Beam Pro に触る必要なし
- 複数案件の瞬時切り替え
3. 1-2分後に Termius で接続
