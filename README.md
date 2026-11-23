# VoiceFlow LINE写真・動画入力統合パッチ

本ファイル群は、ユーザー提供の `main.py`（約11,000行）に写真・動画処理を統合した開発用パッチです。

## GitHubに置くファイル

1. 既存 `main.py` のバックアップを取得。**mainブランチに直接上書きせず** `feature/media` ブランチを作る。
2. このZIP内の `main.py` を開発ブランチの `main.py` に置き換える。
3. `voiceflow_media.py` を `main.py` と同じ場所に新規追加。
4. 既存 `requirements.txt` の末尾へ `requirements_additions.txt` の2行を追記。**既存の依存関係は残す。**
5. Supabase SQL Editorで `supabase_media.sql` を実行。
6. Supabase > Storage で `receipt-images` という **Private（非公開）** バケットを新規作成。公開バケットにしない。
7. RenderのEnvironmentに以下を追加する。

```
MEDIA_CRON_SECRET=<十分長いランダムな秘密文字列>
MEDIA_STORAGE_BUCKET=receipt-images
OPENAI_VISION_MODEL=gpt-4.1-mini
MEDIA_MAX_FRAMES=10
MEDIA_MAX_SECONDS=45
```

`LINE_ACCESS_TOKEN`、`OPENAI_API_KEY`、`SUPABASE_URL`、`SUPABASE_SERVICE_ROLE_KEY` は既存のものをそのまま使用。

8. cron-job.orgで `POST https://<Renderのホスト名>/internal/media/process` を1分ごとに実行。HTTPヘッダーに `X-Media-Cron-Secret: <MEDIA_CRON_SECRETと同じ値>` を設定。
9. Render開発環境で、まず写真1枚→LINE結果→「はい」→マイページAPIの証憑URL生成を確認。その後動画で検証。

## 動作概要

- 既存のLINE `/webhook` で画像・動画メッセージのIDだけを登録し、すぐに返信。
- 定期ジョブがLINEから実データを取得してAI解析。動画は鮮明なフレームを一定間隔で抽出し、同じ領収書らしいフレームはまとめる。
- 登録候補を既存 `pending_sessions` に格納し、LINEへ確認メッセージを送る。
- 既存の「はい」「支払方法の回答」「キャンセル」「登録前修正」を使う。
- 確認後、既存の `records` テーブルに `receipt_path` と `receipt_source` を含めて保存。
- マイページ `/api/mypage/bootstrap` の `records` に、署名付き `receipt_url`（有効600秒）を追加。フロントエンド側でこのURLを画像表示に利用できる。

## 現時点の制限

- **既存フロントエンドのHTML/JSは未提供なので、画像アイコン・サムネイル表示のUI変更は含まれていません。** APIに `receipt_url` を追加しています。
- **画像内に複数の領収書が同時に写っている場合は解析対象外**。動画では原則1枚ずつ順番に映す必要があります。
- 領収書がぼけている場合、日付や金額が取得できない場合、候補を作らず失敗通知します。
- 動画内の検出・重複判定は簡易版で、高精度を保証できません。特に同じ店・同じ日付・同じ金額・同じ内容の別領収書は重複扱いとなる場合があります。検証用の実装です。
- 複数レコードの追加と画像の保存はDBトランザクションで一体化されていません。正式公開前に原子性・復旧処理が必要です。
- エラー発生後に `processing` に残ったジョブの自動回復や、未確定ファイルの定期削除は未実装です。
- 領収書保存期間・電子帳簿保存法の要件を満たすことを保証するものではありません。
- 非公開バケットと本人認証付きAPIが前提です。セキュリティ審査を通さず本番公開しないでください。
- `MAIN.py` と `voiceflow_media.py` は静的構文検査済みですが、LINE・Supabase・OpenAI実環境の結合テストは未実施です。

## 追加テスト例

写真：1枚の鮮明な領収書、低照度、ブレ、複数税率、個人情報入り、再送信、確認途中の連続送信。

動画：1枚3〜5秒映す、同じ領収書の繰り返し、連続10枚、反射、画面回転、上限時間・サイズ超過。

プライバシー：他人のuser_idで証憑を参照できないこと、RLSなしの一般権限でprivate bucketから取り出せないこと。
