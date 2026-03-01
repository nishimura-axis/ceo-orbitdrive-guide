---
title: Telegram Bot + 自動振り返りセットアップガイド
type: knowledge
category: development
tags: [telegram, bot, whisper, claude-api, cloud-run, setup, guide]
created: 2026-03-01
updated: 2026-03-01
---

# Telegram Bot + 自動振り返り セットアップガイド

> このガイドはClaude Codeに読み込ませて使う。
> ユーザーが「このガイドに沿ってセットアップを進めて」と送ったら、以下の内容に従って対話形式でセットアップを進める。

---

## 1. Claude Code向けの指示

### ユーザーのレベル

- AIリテラシー: まだ高くない
- システム・プログラミング知識: ほぼない
- Claude Code + Obsidian + GitHub は使える状態（セットアップ済み）
- Telegramアカウントは作成済み（スマホ + PC版）

### 進め方のルール

- **1ステップずつ進める。** 一度に複数のことをやらない
- **各ステップで「何をやっているか」「なぜやるのか」を簡潔に説明する**
- **専門用語を使う場合は必ず補足する**（例: 「デプロイ（=クラウドに配置すること）」）
- **ステップが完了したら確認してから次に進む**（「ここまでOKですか？次に進みますか？」）
- **ユーザーに作業してもらう場面では、具体的に何をすればいいか画面の操作レベルで伝える**
- **コードやコマンドを実行する前に、何をするか説明してから実行する**
- **勝手に判断しない。選択肢がある場合はユーザーに聞く**

### エラー時の対応

- エラーが出たらパニックさせない。「よくあることです」と伝える
- エラーの原因と対処法をわかりやすく説明する
- ユーザーが何もしなくてもClaude Codeで解決できるものは自分で直す
- ユーザーの操作が必要な場合は具体的に指示する

### セットアップの流れ（概要）

以下の順序で進める。各フェーズの冒頭でユーザーに全体の進捗を伝える。

```
Phase 1: 準備（APIキー・アカウント取得）  ← ユーザーがブラウザで手動操作
Phase 2: プロジェクト構築                ← Claude Codeが実行
Phase 3: ローカル動作確認               ← ユーザーがTelegramでテスト
Phase 4: クラウドにデプロイ             ← Claude Codeが実行
Phase 5: 自動振り返り設定               ← Claude Codeが実行
Phase 6: 最終確認                       ← ユーザーがTelegramでテスト
```

---

## 2. 全体像

### ゴール

このセットアップが完了すると、以下の3つができる:

1. **携帯からTelegramに音声/テキストを送る → Obsidian Vaultに自動で書き込まれる**
2. **毎日決まった時間にAIが振り返りを作成し、Telegramに届く**
3. **Claude Codeで自分でBotの挙動をカスタマイズできる**

### アーキテクチャ

```
携帯（Telegram）
  ↓ 音声/テキスト送信
Telegram Bot（Cloud Run で常時稼働）
  ↓ 音声 → Whisper API で文字起こし → ユーザーに確認 → 承認後に保存
  ↓ テキスト → そのまま保存
  ↓ GitHub API 経由で Obsidian Vault に書き込み
  ↓
Cloud Scheduler（毎日21時に実行）
  → Claude API で振り返り生成
  → Telegram に送信 + Vault に保存
```

PCを起動しておく必要はない。24時間クラウドで稼働する。

---

## 3. 技術仕様

### 3-1. プロジェクト構成

- 言語: Python 3.12
- Telegram Bot: aiogram 3
- Web フレームワーク: FastAPI
- 依存関係: requirements.txt
- 環境変数: .env ファイル（.gitignore に含める）

### 3-2. Bot の機能

#### テキストメッセージ

```
ユーザーがテキスト送信
  → Vault に書き込み
  → 「保存しました」と返信
```

#### 音声メッセージ

```
ユーザーが音声送信
  → Whisper API で文字起こし
  → 文字起こし結果をユーザーに表示
  → 「この内容で保存しますか？」とインラインボタンで確認
  → 「はい」→ Vault に書き込み → 「保存しました」と返信
  → 「いいえ」→ 「正しい内容をテキストで送ってください」と返信
```

※ 音声認識は誤字脱字が多いため、確認ステップは必須。省略しない。

#### Vault 書き込み形式

- 書き込み先: GitHub API 経由（Contents API で commit）
- ファイルパス: `00_Inbox/YYYY-MM-DD_HHmm_telegram.md`
- フォーマット:

```markdown
---
title: Telegramメモ YYYY-MM-DD HH:mm
type: inbox
created: YYYY-MM-DD
source: telegram
---

（メッセージ内容）
```

### 3-3. 自動振り返り

- トリガー: Cloud Scheduler → Cloud Run エンドポイントを叩く
- 実行時間: 毎日 21:00 JST
- 処理:
  1. その日の Vault 内 `00_Inbox/*_telegram.md` を GitHub API で取得
  2. Claude API（haiku）で振り返りを生成
  3. 生成結果を Telegram に送信
  4. 振り返り内容を Vault に保存（`07_DailyLog/YYYY-MM-DD_review.md`）
- 振り返りが送信されるその日の投稿が0件の場合は「今日の記録はありません」と送信

### 3-4. セキュリティ

- すべてのキーは環境変数で管理（.env）
- .env は .gitignore に含める
- Bot は特定の Telegram ID からのメッセージのみ受け付ける（ユーザーの chat_id でフィルタ）
- GitHub トークンはコードにハードコードしない

### 3-5. デプロイ

- プラットフォーム: Google Cloud Run
- コンテナ: Dockerfile で定義
- Webhook 方式: Cloud Run の URL を Telegram Webhook に設定
- Cloud Scheduler: 振り返り用の cron ジョブ（`0 21 * * *` Asia/Tokyo）

---

## 4. セットアップ手順

### Phase 1: 準備（ユーザーがブラウザで操作）

以下のキー/IDをユーザーに取得してもらう。1つずつ案内する。

#### 1-1. Telegram Bot トークン

- BotFather（Telegram内）で `/newbot` コマンド
- 表示名とユーザー名（`xxx_bot`）を決める
- 発行されたトークンを保存

案内のポイント:
- BotFather の探し方（検索バーで検索、青チェックマーク付き）
- ユーザー名は半角英数 + `bot` で終わる必要がある
- トークンはパスワードと同じ扱い

#### 1-2. OpenAI API キー

- https://platform.openai.com でアカウント作成
- API Keys → Create new secret key
- $5 のクレジットチャージが必要

案内のポイント:
- 「音声を文字に変換するために使います」と説明
- キーは一度しか表示されない
- 料金: 1分あたり約1円

#### 1-3. Anthropic API キー

- https://console.anthropic.com でアカウント作成
- API Keys → Create Key
- $5 のクレジットチャージが必要

案内のポイント:
- 「AIが振り返りを作るために使います」と説明
- 料金: 1回数円

#### 1-4. Google Cloud プロジェクト

- https://console.cloud.google.com でログイン
- 新しいプロジェクトを作成
- プロジェクトIDをメモ

案内のポイント:
- 「Botを24時間動かすためのクラウドです」と説明
- 無料トライアル（$300クレジット）あり
- クレカ登録が必要だが、トライアル中は課金されない

#### 1-5. 取得したキーの確認

全キーが揃ったか確認:
- [ ] Telegram Bot トークン
- [ ] OpenAI API キー
- [ ] Anthropic API キー
- [ ] Google Cloud プロジェクト ID
- [ ] GitHub ユーザー名（既に持っている）
- [ ] GitHub アクセストークン（既に持っている）
- [ ] GitHub Vault リポジトリ名（既に持っている）

### Phase 2: プロジェクト構築（Claude Code が実行）

1. `~/telegram-bot` ディレクトリを作成
2. Git 初期化
3. .env ファイルに全キーを保存
4. .gitignore に .env を含める
5. requirements.txt を作成（aiogram, fastapi, uvicorn, openai, anthropic, httpx）
6. Bot コードを実装（技術仕様 3-2 に従う）
7. FastAPI エンドポイント設定（Webhook受信 + 振り返りトリガー）
8. Dockerfile 作成
9. README.md 作成

ユーザーへの説明:
- 「これからプロジェクトのファイルを作成します。少し時間がかかります。」
- 実行中は適宜進捗を伝える

### Phase 3: ローカル動作確認（ユーザーが Telegram でテスト）

1. ローカルで Bot を起動（ngrok or polling モードで）
2. ユーザーに以下をテストしてもらう:
   - テキスト送信 → 「保存しました」が返るか
   - 音声送信 → 文字起こし結果が表示 → 確認 → 保存されるか
   - Obsidian で 00_Inbox にファイルが作成されているか
3. 問題があればその場で修正

### Phase 4: Cloud Run にデプロイ（Claude Code が実行）

1. gcloud CLI がインストールされているか確認、なければインストール案内
2. gcloud auth login（ブラウザ認証が必要 → ユーザーに案内）
3. gcloud config set project （プロジェクトID）
4. 必要な API を有効化（Cloud Run, Cloud Build, Artifact Registry）
5. Cloud Run にデプロイ
6. デプロイされた URL を Telegram Webhook に設定
7. 環境変数を Cloud Run に設定

ユーザーへの説明:
- 「クラウドにBotを配置します。ブラウザでGoogleアカウントにログインする画面が出ます。」

### Phase 5: 自動振り返り設定（Claude Code が実行）

1. Cloud Scheduler ジョブを作成（毎日21時 JST）
2. 振り返り用エンドポイントを Cloud Scheduler のターゲットに設定
3. 手動実行で振り返りをテスト

ユーザーへの説明:
- 「毎日21時に自動で振り返りが届くように設定します。」

### Phase 6: 最終確認（ユーザーが Telegram でテスト）

1. Telegram からテキスト送信 → Vault に保存されるか
2. Telegram から音声送信 → 文字起こし → 確認 → 保存されるか
3. 振り返りを手動実行 → Telegram に届くか

すべてOKなら:
- 「セットアップ完了です！」と伝える
- カスタマイズの方法を簡単に案内する

---

## 5. カスタマイズ例

セットアップ完了後、ユーザーが「こうしたい」と言ったときの参考:

- **振り返りの時間を変えたい** → Cloud Scheduler の cron 式を変更
- **振り返りの内容を変えたい** → 振り返り生成プロンプトを編集
- **保存先フォルダを変えたい** → Bot のファイルパス生成ロジックを変更
- **特定のキーワードでフォルダ分けしたい** → メッセージパーサーにルールを追加
- **Bot の返信メッセージを変えたい** → 返信テンプレートを編集

---

## 6. ランニングコスト

| サービス | 料金目安 | 備考 |
|---------|---------|------|
| Google Cloud Run | 無料枠内 | 月200万リクエストまで無料 |
| Cloud Scheduler | 無料枠内 | 月3ジョブまで無料 |
| OpenAI Whisper | 約1円/分 | 音声文字起こし |
| Anthropic Claude | 数円/回 | 振り返り生成 |
| **合計** | **月500〜2,000円程度** | 使い方次第 |

---

## 関連メモ

| 種別 | メモ | 関連ポイント |
|------|------|-------------|
| 前提知識 | [[40_Knowledge/Development/Second_Brain_Setup_Guide\|第二の脳構築ガイド]] | このガイドの前提 |
| 前提知識 | [[40_Knowledge/Development/ClaudeCode_Setup_Guide_for_Sharing\|Claude Codeセットアップガイド]] | このガイドの前提 |
| 関連PJ | [[30_Projects/西村AI化構想/CEOオービット/CEO_OrbitDrive\|CEO OrbitDrive]] | このガイドを使うPJ |
