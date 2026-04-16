# API リファレンス

Whisper Turbo Transcription の REST API ドキュメントです。  
デフォルトではサーバーは `http://localhost:5000` で起動します。

---

## 目次

- [POST /transcribe](#post-transcribe) — ファイル全体を送信して文字起こし
- [POST /transcribe_async](#post-transcribe_async) — `/transcribe` のエイリアス（ポーリング前提）
- [POST /transcribe_chunk](#post-transcribe_chunk) — チャンクアップロード（同期）
- [POST /transcribe_chunk_async](#post-transcribe_chunk_async) — チャンクアップロード（非同期）
- [POST /transcribe_finalize](#post-transcribe_finalize) — チャンクを結合して文字起こし（同期）
- [POST /transcribe_finalize_async](#post-transcribe_finalize_async) — チャンクを結合して文字起こし（非同期）
- [GET /status/:task_id](#get-statustask_id) — 非同期タスクの状態を取得
- [GET /download/:transcription_id](#get-downloadtranscription_id) — 文字起こし結果をダウンロード

---

## 共通パラメータ

多くのエンドポイントで共通して使用されるフォームフィールドです。

| パラメータ | 型 | デフォルト | 説明 |
|---|---|---|---|
| `device` | string | `cuda:0`（GPU 利用可能時）/ `cpu` | 推論に使用するデバイス（例: `cpu`, `cuda:0`, `cuda:1`） |
| `language` | string | `auto` | 言語コード。`auto` で自動検出。対応例: `ja`, `en`, `zh`, `ko`, `fr`, `de`, `es` |
| `translate` | string | `false` | `true` にすると英語へ翻訳 |

---

## POST /transcribe

ファイルを一括で送信し、文字起こしを行います。  
`polling=true` を指定すると非同期モードになり、`task_id` を返します。

### リクエスト

**Content-Type:** `multipart/form-data`

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `file` | file | ✅ | 音声または動画ファイル（`.mp4`, `.mkv`, `.avi`, `.mov`, その他音声形式） |
| `device` | string | | 推論デバイス |
| `language` | string | | 言語コード |
| `translate` | string | | 英語翻訳フラグ |
| `polling` | string | | `true` で非同期（ポーリング）モード |

### レスポンス

#### 同期モード（`polling` 未指定 or `false`）

**200 OK**

```json
{
  "transcription": "文字起こしされたテキスト",
  "id": "transcription-uuid"
}
```

#### 非同期モード（`polling=true`）

**202 Accepted**

```json
{
  "task_id": "task-uuid"
}
```

`task_id` を使って [`GET /status/:task_id`](#get-statustask_id) でポーリングしてください。

### エラー

| コード | 説明 |
|---|---|
| 400 | ファイルが添付されていない、またはファイルが選択されていない |
| 500 | 文字起こし処理中のエラー |

---

## POST /transcribe_async

`/transcribe` のエイリアスです。内部的には `/transcribe` を呼び出します。  
ポーリングモード（`polling=true`）での利用を想定しています。

リクエスト・レスポンスは [`POST /transcribe`](#post-transcribe) と同じです。

---

## POST /transcribe_chunk

大きなファイルをチャンク分割してアップロードする際に使用します（同期版）。  
すべてのチャンクをアップロードした後、[`POST /transcribe_finalize`](#post-transcribe_finalize) を呼び出して結合・文字起こしを行います。

### リクエスト

**Content-Type:** `multipart/form-data`

| フィールド | 型 | 必須 | 説明 |
|---|---|---|---|
| `file` | file | ✅ | ファイルのチャンク（分割されたバイナリ） |
| `fileId` | string | ✅ | クライアントが生成した UUID。同一ファイルのチャンクを識別する |
| `chunkIndex` | int | ✅ | チャンクのインデックス（0始まり） |
| `totalChunks` | int | ✅ | 総チャンク数 |
| `device` | string | | 推論デバイス |
| `language` | string | | 言語コード |
| `translate` | string | | 英語翻訳フラグ |

### レスポンス

**200 OK**

```json
{
  "message": "チャンク 1/5 を受信しました"
}
```

### エラー

| コード | 説明 |
|---|---|
| 400 | ファイルまたは `fileId` が提供されていない |

---

## POST /transcribe_chunk_async

チャンクアップロードの非同期版です。  
すべてのチャンクをアップロードした後、[`POST /transcribe_finalize_async`](#post-transcribe_finalize_async) を呼び出します。

リクエスト・レスポンスは [`POST /transcribe_chunk`](#post-transcribe_chunk) と同じです。

---

## POST /transcribe_finalize

チャンクアップロード完了後にチャンクを結合し、同期的に文字起こしを行います。

### リクエスト

**Content-Type:** `application/json`

```json
{
  "fileId": "アップロード時に使用した fileId"
}
```

> **注意:** `device`, `language`, `translate` はフォームフィールドとして送信します（`Content-Type` はリクエストボディ部分が JSON ですが、これらはクエリ等で補完されます）。

### レスポンス

**200 OK**

```json
{
  "transcription": "文字起こしされたテキスト",
  "id": "transcription-uuid"
}
```

### エラー

| コード | 説明 |
|---|---|
| 400 | `fileId` が未指定、チャンクが見つからない |
| 500 | FFmpeg 変換エラー、文字起こしエラー |

---

## POST /transcribe_finalize_async

チャンクアップロード完了後にチャンクを結合し、非同期的に文字起こしを開始します。

### リクエスト

[`POST /transcribe_finalize`](#post-transcribe_finalize) と同じです。

### レスポンス

**202 Accepted**

```json
{
  "task_id": "task-uuid"
}
```

`task_id` を使って [`GET /status/:task_id`](#get-statustask_id) でポーリングしてください。

### エラー

[`POST /transcribe_finalize`](#post-transcribe_finalize) と同じです。

---

## GET /status/:task_id

非同期タスクの進行状況を取得します。ポーリングで繰り返し呼び出してください。

### パスパラメータ

| パラメータ | 説明 |
|---|---|
| `task_id` | 非同期エンドポイントから返された `task_id` |

### レスポンス

#### 処理中

**200 OK**

```json
{
  "status": "processing",
  "filename": "example.mp4"
}
```

#### 完了

**200 OK**

```json
{
  "status": "completed",
  "transcription": "文字起こしされたテキスト",
  "id": "transcription-uuid",
  "filename": "example.mp4"
}
```

#### エラー

**200 OK**

```json
{
  "status": "error",
  "error": "エラーメッセージ",
  "filename": "example.mp4"
}
```

#### タスクが見つからない

**404 Not Found**

```json
{
  "error": "タスクが見つかりません"
}
```

---

## GET /download/:transcription_id

文字起こし結果をテキストファイルとしてダウンロードします。

### パスパラメータ

| パラメータ | 説明 |
|---|---|
| `transcription_id` | 文字起こし完了時に返された `id` |

### レスポンス

**200 OK** — テキストファイル（`.txt`）がダウンロードされます。

### エラー

| コード | 説明 |
|---|---|
| 404 | 指定された `transcription_id` のファイルが見つからない |

---

## フロー例

### 1. 同期・一括アップロード

```
POST /transcribe  (file + device + language + translate)
  ↓
200 { transcription, id }
  ↓
GET /download/:id  （任意）
```

### 2. 非同期・一括アップロード（ポーリング）

```
POST /transcribe  (file + device + language + translate + polling=true)
  ↓
202 { task_id }
  ↓
GET /status/:task_id  ← 繰り返しポーリング（推奨間隔: 5秒）
  ↓
200 { status: "completed", transcription, id, filename }
  ↓
GET /download/:id  （任意）
```

### 3. チャンクアップロード（同期）

```
POST /transcribe_chunk  × N回（チャンクごと）
  ↓
POST /transcribe_finalize  (fileId)
  ↓
200 { transcription, id }
```

### 4. チャンクアップロード（非同期・ポーリング）

```
POST /transcribe_chunk_async  × N回（チャンクごと）
  ↓
POST /transcribe_finalize_async  (fileId)
  ↓
202 { task_id }
  ↓
GET /status/:task_id  ← 繰り返しポーリング
  ↓
200 { status: "completed", transcription, id, filename }
```
