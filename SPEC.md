# YouTube コメント読み上げアプリ 仕様書

**バージョン**: 2.2.0  
**最終更新**: 2026-05-19  
**リポジトリ**: https://github.com/ngcnj175/youtube-reader

---

## 1. 概要

YouTubeの動画コメントを取得し、ブラウザのテキスト読み上げ（TTS）機能を使って音声再生するWebアプリケーション。日本語対応のシングルページアプリ。

---

## 2. アーキテクチャ

| 項目 | 内容 |
|------|------|
| 構成 | シングルファイル（`index.html` 1ファイルに HTML/CSS/JS を集約） |
| フレームワーク | なし（バニラ JavaScript ES6+） |
| ビルドツール | なし |
| バックエンド | Cloudflare Workers（サーバーレス） |
| デザイン | ニューモーフィズム（ソフトUI）、モバイルファースト |
| フォント | Zen Maru Gothic（Google Fonts） |

---

## 3. 使用技術・API

### フロントエンド
- **HTML5 / CSS3 / JavaScript（ES6+）** — バニラ実装
- **Web Speech API** — ブラウザネイティブのTTS（`window.speechSynthesis`）

### バックエンド
- **Cloudflare Workers** — CORS回避のプロキシ兼APIゲートウェイ
  - エンドポイント: `https://youtube-comment-reader-worker.ngcnj175.workers.dev`

### 外部サービス
- **YouTube Data API v3** — 動画メタデータ・コメント取得（Workerを経由）
- **翻訳API** — コメントの日本語翻訳（Worker経由のバッチ処理）

---

## 4. 機能一覧

### 4.1 コメント取得
- YouTube URL（`youtube.com`、`youtu.be`、`youtube.com/embed`）からビデオIDを抽出
- コメントを最大1000件取得（1ページ100件 × 最大10ページ）
- 取得中はスピナーと件数カウントを表示
- APIエラー時はデモモードにフォールバック

### 4.2 コメント表示
- アバター画像・投稿者名・投稿日・本文をカード形式で表示
- 連番表示のON/OFF切り替え
- 現在再生中のコメントをハイライト＆自動スクロール

### 4.3 コメント並び替え
| 順序 | YouTube API パラメータ |
|------|----------------------|
| 関連度順（デフォルト） | `order=relevance` |
| 新しい順 | `order=time` |
| 古い順 | `order=time`（取得後に逆順） |

### 4.4 音声再生
- **言語**: `ja-JP`（日本語優先、なければデフォルト音声）
- **ピッチ**: コメントごとにランダム変化（0.7 / 0.9 / 1.1 / 1.3）
- **再生速度**: 0.5x〜2.0x（スライダー調整）
- **音量**: 0〜100%（スライダー調整）
- **連続再生**: ON時はコメント終了後500ms後に次のコメントへ自動進行
- **絵文字除去**: TTSエラー防止のため読み上げ前に絵文字を削除

### 4.5 再生操作
| 操作 | 動作 |
|------|------|
| 再生/停止ボタン | トグル再生 |
| 前へボタン | 前のコメントへ（再生中なら即時切り替え） |
| 前へ長押し | 先頭コメントへジャンプ |
| 次へボタン | 次のコメントへ |
| コメントカードクリック | そのコメントから再生開始 |
| Space キー | 再生/停止トグル |
| ← キー | 前のコメントへ |
| → キー | 次のコメントへ |

### 4.6 フルスクリーン表示
- 現在のコメント・投稿者アバター・動画タイトルを大画面表示
- テキスト量に応じてフォントサイズを自動調整
- フルスクリーン内でも再生操作・音量調整が可能

### 4.7 翻訳機能
- 全コメントを一括で日本語に翻訳
- コメントを500バイト単位のチャンクに分割してAPI送信
- 区切り文字 ` ||| ` で複数コメントを1リクエストにまとめる
- 翻訳後にコメントリストを再描画

---

## 5. データフロー

```
[ユーザーがURLを入力]
        ↓
extractVideoId() — 正規表現でビデオIDを抽出
        ↓
fetchComments()
    ├─ fetchVideoTitle()  → Worker → YouTube API → 動画タイトル表示
    └─ fetchAllComments() → Worker → YouTube API（ページネーション）
                                    ↓
                            コメント配列 comments[] に格納
                            { author, authorProfileImageUrl, text, publishedAt }
        ↓
displayComments() — コメントカードをDOMに描画
        ↓
[ユーザーがコメントをクリック or 再生ボタンを押す]
        ↓
startReadingFromIndex(index)
    └─ readCurrentComment()
        ├─ highlightComment() — アクティブ表示・スクロール
        ├─ removeEmoji()      — 絵文字削除
        ├─ SpeechSynthesisUtterance を生成（lang/rate/volume/pitch設定）
        └─ synth.speak()
                ↓（読み上げ完了）
            continuousPlay=ON → 500ms後に次のコメントへ
            continuousPlay=OFF → 停止
```

---

## 6. APIインターフェース

### GET — 動画メタデータ取得
```
GET {WORKER_URL}?videoId={videoId}&mode=video
```
レスポンス:
```json
{
  "items": [{ "snippet": { "title": "動画タイトル" } }]
}
```

### GET — コメント取得（ページネーション）
```
GET {WORKER_URL}?videoId={videoId}&order={relevance|time}&pageToken={token}&maxResults=100
```
レスポンス:
```json
{
  "items": [
    {
      "snippet": {
        "topLevelComment": {
          "snippet": {
            "authorDisplayName": "投稿者名",
            "authorProfileImageUrl": "https://...",
            "textDisplay": "コメント本文（HTMLタグ含む）",
            "publishedAt": "2024-01-01T00:00:00Z"
          }
        }
      }
    }
  ],
  "nextPageToken": "トークン文字列（次ページがある場合）"
}
```

### POST — コメント翻訳
```
POST {WORKER_URL}
Content-Type: application/json

{ "mode": "translate", "text": "コメント1 ||| コメント2 ||| ..." }
```
レスポンス:
```json
{ "translated_text": "翻訳1 ||| 翻訳2 ||| ..." }
```

---

## 7. クラス設計（`YouTubeCommentReader`）

### 主なプロパティ
| プロパティ | 型 | 説明 |
|-----------|-----|------|
| `comments` | Array | 取得したコメントの配列 |
| `currentIndex` | number | 現在再生中のインデックス |
| `isReading` | boolean | 再生中フラグ |
| `synth` | SpeechSynthesis | TTSエンジン |
| `currentUtterance` | SpeechSynthesisUtterance | 現在の発話オブジェクト |
| `videoTitle` | string | キャッシュした動画タイトル |
| `sortOrder` | string | `relevance` / `time` / `oldest` |
| `continuousPlay` | boolean | 連続再生フラグ（デフォルト: true） |
| `showNumbering` | boolean | 連番表示フラグ（デフォルト: true） |
| `speechRate` | number | 再生速度（0.5〜2.0、デフォルト: 1.0） |
| `speechVolume` | number | 音量（0〜1、デフォルト: 1.0） |
| `isMuted` | boolean | ミュート状態 |
| `MAX_COMMENTS` | number | 最大取得件数（1000） |
| `WORKER_URL` | string | バックエンドエンドポイント |

### 主なメソッド
| メソッド | 役割 |
|---------|------|
| `loadVoices()` | 日本語TTSボイスを初期化 |
| `initEventListeners()` | 全UI要素のイベントハンドラ登録 |
| `extractVideoId(url)` | URLからビデオIDを正規表現で抽出 |
| `fetchComments()` | コメント取得フロー全体の制御 |
| `fetchAllComments(videoId)` | ページネーションループ |
| `fetchPage(videoId, order, pageToken)` | 1ページ分のAPIコール |
| `displayComments()` | コメントリストをDOMに描画 |
| `highlightComment(index)` | 再生中コメントをハイライト・スクロール |
| `readCurrentComment()` | 現在コメントの読み上げ実行 |
| `togglePlay()` | 再生/停止トグル |
| `startReadingFromIndex(index)` | 指定インデックスから再生開始 |
| `stopReading()` | 音声合成を停止 |
| `prevComment()` / `nextComment()` | コメントナビゲーション |
| `goToFirst()` | 先頭コメントへジャンプ |
| `translateAllComments()` | 全コメントを一括翻訳 |
| `enterFullscreen()` / `exitFullscreen()` | フルスクリーン切り替え |
| `updateFullscreenComment()` | フルスクリーン表示を更新（フォント自動調整） |
| `removeEmoji(text)` | 絵文字を除去してTTSエラーを防止 |
| `runDemoMode()` | サンプルコメントでのデモ起動 |

---

## 8. UIレイアウト

```
┌──────────────────────────────────┐
│ [固定ヘッダー]                    │
│  URL入力欄 ／ 動画タイトル（流れ字）│
├──────────────────────────────────┤
│ [コメントリスト]                  │
│  コメントカード × n              │
│  （アバター・投稿者・日付・本文）  │
├──────────────────────────────────┤
│ [固定フッター：プレイヤーパネル]  │
│  ⚙設定  ⏮前  ▶再生  ⏭次  ⛶全画面│
│  ─────────────────────────────── │
│  [設定パネル（折りたたみ）]       │
│    並び替え / 連続再生 / 連番     │
│    再生速度 / 音量 / 翻訳ボタン   │
└──────────────────────────────────┘
```

---

## 9. 制限事項・仕様上の注意

| 項目 | 内容 |
|------|------|
| 最大取得件数 | 1000件（10ページ × 100件/ページ） |
| データ永続化 | なし（セッション中のみ。localStorage未使用） |
| 対応ブラウザ | Web Speech API対応の近代的なブラウザ |
| 言語 | 日本語UI、TTS優先言語: `ja-JP` |
| コメントソート「古い順」 | APIは `order=time` で取得後、クライアント側で逆順に並び替え |
| 翻訳チャンクサイズ | 1チャンク最大500バイト |
| 連続再生ディレイ | コメント終了から次コメント開始まで500ms |
| ピッチ変化 | コメントごとにランダム（0.7 / 0.9 / 1.1 / 1.3の4段階） |
