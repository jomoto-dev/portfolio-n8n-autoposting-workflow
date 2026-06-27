# n8n AI動画自動投稿ワークフロー

## 概要

n8nを使って、LINEから送信された動画生成プロンプトをもとにAI動画を生成し、YouTubeへ自動投稿するワークフローです。
LINEで受け取ったテキストをGoogle Sheetsに保存し、スケジュール実行で未処理データを1件ずつ取得します。
その後、Replicate APIで動画生成を開始し、生成完了時にReplicate Webhookで動画URLを受け取ります。
生成された動画はGoogle Driveへ保存し、GeminiでYouTube用のタイトル・説明文を生成してからYouTubeへアップロードします。

## 作成背景

AI動画生成ツールやSNS投稿ツールを個別に操作すると、以下のような作業が発生します。

* 動画生成プロンプトを管理する
* 動画生成APIを実行する
* 生成完了を確認する
* 動画ファイルを保存する
* 投稿タイトル・説明文を考える
* YouTubeへアップロードする
* 失敗時に原因を確認する

これらの作業をn8nで自動化し、「LINEにプロンプトを送るだけで、動画生成からYouTube投稿まで進められる仕組み」を構築しました。

また、Google Sheets上で `queued` / `processing` / `done` といったステータスを管理し、二重実行や処理漏れを防ぐことも意識しています。

## 全体像
<img width="1864" height="1194" alt="Image" src="https://github.com/user-attachments/assets/550a838e-c2d4-4217-802f-387f4c0568a1" />

## 主な機能

* LINE Webhookによるプロンプト受信とGoogle Sheetsへのキュー登録
* `queued` / `processing` / `done` によるステータス管理と二重実行防止
* Schedule Trigger + Limitノードによる1件ずつの定期処理
* Replicate APIでの動画生成とWebhookによる非同期完了受信
* Google Driveへの動画保存とGeminiによるYouTubeメタデータ生成
* YouTubeへの自動アップロード

## 開発方針

* ワークフローの設計・構築・テストはすべて自分で行いました
* 技術調査やアイデア整理の補助としてChatGPT・Claudeを活用しました
* n8nの公式ドキュメントや各APIのリファレンスを参照しながら実装を進めました

## 使用技術・サービス

| 技術・サービス | 用途 | 選定理由 |
| --- | --- | --- |
| n8n | ワークフロー全体の自動化 | Webhook、API連携、Googleサービス連携をまとめて扱えるため |
| LINE Messaging API | プロンプト入力 | スマートフォンから手軽に動画生成プロンプトを送れるため |
| Google Sheets | キュー管理・ステータス管理 | `queued` / `processing` / `done` などの状態を一覧で確認しやすいため |
| Replicate | AI動画生成API | HTTP Requestノードから呼び出しやすく、Webhookで生成完了通知を受け取れるため |
| Google Drive | 生成動画の保存 | 生成されたmp4ファイルを後続処理で扱いやすくするため |
| Gemini | YouTube用タイトル・説明文生成 | 動画生成プロンプトをもとに、投稿用テキストを自動生成するため |
| YouTube | 動画投稿先 | 生成した動画を自動投稿する対象として利用 |

### 動画生成API（Replicate）の選定理由

HTTP RequestノードからAPIを呼び出しやすく、Webhookによる非同期完了通知にも対応しているため、n8n上で「動画生成依頼」と「生成完了後の処理」を分離しやすいことから、採用に至りました。

## API・Webhook一覧

| 種類 | 用途 |
| --- | --- |
| LINE Webhook | LINEから送信されたテキストメッセージをn8nで受信 |
| Google Sheets API | プロンプトキューの追加・取得・更新 |
| Replicate API | AI動画生成の開始 |
| Replicate Webhook | 動画生成完了通知の受信 |
| Google Drive API | 生成動画ファイルの保存・取得 |
| Gemini API | YouTube用タイトル・説明文の生成 |
| YouTube Data API | 動画アップロード |

## ワークフローの技術的構成と設計意図

このワークフローでは、LINEでプロンプトを受け取る処理、動画生成を依頼する処理、生成完了後に動画を保存・投稿する処理、ステータスを更新する処理を段階的に分けています。

### 1. LINEメッセージを受け取り、Google Sheetsへ保存する処理

LINEからのWebhookをトリガーとし、受信データを整形してデータストア（Google Sheets）に非同期で登録する構成です。

<img width="1000" alt="Image" src="https://github.com/user-attachments/assets/0540da05-6e67-46b8-9bc1-7a57489642ef" />

* **使用ノード**: `Webhook`, `If`, `Edit Fields`, `Google Sheets (Append Row)`  
※Google Sheetsノードは名前がGet tracksになっていますが、これは名前を変更した為です。
* **設計意図**:
  * **イベントフィルタリング**: LINE Messaging APIから送信される多様なイベント（フォロー、画像送信など）から、Ifノードを用いて `type: message` かつ `message.type: text` のテキストメッセージのみを的確にフィルタリングします。

### 2. Google Sheetsを定期的に確認し、動画生成を依頼する処理

一定間隔（Schedule Trigger）でGoogle Sheetsを監視し、未処理のキューから1件ずつReplicate APIへ動画生成をリクエストする構成です。
<img width="1000" alt="Image" src="https://github.com/user-attachments/assets/432a269e-81e9-433f-aad5-4376e87ba13a" />
<img width="1000" alt="Image" src="https://github.com/user-attachments/assets/ebb14bd2-4bd7-433a-a009-70af11830bd0" />

* **使用ノード**: `Schedule Trigger`, `Google Sheets (Get Row(s), Update row)`, `Filter`, `Limit`, `Edit Fields`, `If`, `HTTP Request`  
※HTTP Requestノードは名前がcreatevideoになっていますが、これは名前を変更した為です。
* **設計意図**:
  * **処理件数の絞り込み**: 外部APIの同時実行数制限や利用料金急増を防止するため、`Limit` ノードを用いて取得データを「先頭の1件のみ」に制限します。
  * **Google Sheetsによる処理状態の管理**: Google Sheetsから未処理の行を1件取得した後、そのステータスを `processing` に更新しています。これにより、「この行はこれから動画生成に進める対象である」と分かるようにし、また、同じ行が次回以降も未処理として扱われることを避けやすくしています。さらに動画生成ノードの直前でIfノードによるステータス判定も行うことで、より確実に`processing`のプロンプトのみが送られるようにしています。
  * **処理時間のかかるAPI連携の工夫**: Replicateによる動画生成は完了まで時間がかかるため、n8n側で処理完了を待ち続けるのではなく、動画生成が完了したタイミングでReplicateからWebhook通知を受け取る構成にしています。これにより、動画生成を依頼する処理と、生成完了後に動画を保存・投稿する処理を分けています。

### 3. 動画生成完了の通知を受け取り、保存・投稿する処理

Replicateからの動画生成完了通知（Webhook）を契機に、動画ファイルのダウンロード、保存、LLMによるメタデータ生成、YouTube投稿までを一貫して実行します。

<img width="1000" alt="Image" src="https://github.com/user-attachments/assets/d84b073d-55fa-436d-aa97-0adf8e8264f9" />


* **使用ノード**: `Webhook`, `HTTP Request`, `Google Drive (Upload, Download)`, `Gemini`, `YouTube`
* **設計意図**:
  * **動画ファイルの受信・保存**: Webhookで受信した動画URLから、`HTTP Request` ノードを使用してmp4データを一時保持し、ローカルにファイルを書き出すことなく `Google Drive` へアップロードします。
  * **プロンプトからタイトル・説明文を生成**: Geminiに動画生成時と同じプロンプトを渡し、YouTube Shortsの文字数制限とSEOに適したタイトル・説明文を定義されたJSON形式で直接生成させます。
  * **動画ファイルとタイトル・説明文を組み合わせてアップロード**: Google Driveに保存した動画ファイルオブジェクトとGeminiが生成したタイトル・説明文テキストを `YouTube` ノードに渡し、動画をアップロードします。

### 4. 処理済みの行をdoneに更新する処理

非同期投稿処理の完了に伴い、データキューのステータスを最終状態（`done`）へ更新するバッチ処理です。

* **使用ノード**: `Schedule Trigger`, `Google Sheets (Get row(s), Update row)`, `Filter`, `Limit`
* **設計意図**:
  * **ステータスと動画生成開始日時の更新**: 処理開始時刻（`started_at`）やプロンプト内容など、既存の他の重要データを破損させないよう、`id` をキーにして `status` および完了日時のみをピンポイントで更新します。

<img width="1000" alt="Image" src="https://github.com/user-attachments/assets/747b2819-8e85-48c4-b05e-f5b3172e83ce" />

## Google Sheetsの主な列

| 列名 | 用途 |
| --- | --- |
| id | 各行を一意に識別するID |
| prompt | 動画生成プロンプト |
| status | `queued` / `processing` / `done` |
| source | 入力元 |
| line_user_id | LINEユーザーID |
| line_message_id | LINEメッセージID |
| created_at | 登録日時 |
| started_at | 処理開始日時 |
| Sent_to_createvideo_at | 動画生成APIへ送信した日時を記録する列 |

## Gemini出力形式

Geminiには、YouTube Shorts向けのタイトルと説明文をJSON形式で返すように指示しています。

```json
{
  "title_youtube": "タイトル",
  "description_youtube": "説明文"
}
```

生成されたJSONをパースし、YouTube動画のタイトルと説明文として使用します。

## セットアップ方法

### 1. n8nを用意する

n8n Cloud、またはセルフホスト環境を用意します。
本ワークフローではWebhookとSchedule Triggerを使用するため、外部サービスからアクセス可能なn8n環境が必要です。

### 2. Google Sheetsを作成する

Google Sheetsに、キュー管理用のシートを作成します。
シート名の例は以下です。

```text
prompt_queue
```

1行目に「Google Sheetsの主な列」で記載した列を用意します。

### 3. 各サービスの認証情報をn8nに登録する

n8n上で以下の認証情報を設定します。

* Google Sheets OAuth
* Google Drive OAuth
* YouTube OAuth
* Gemini API
* Replicate API Token
* LINE Messaging API関連情報

公開用JSONでは、これらの認証情報はマスクされています。
インポート後に、自分の環境に合わせて再設定してください。

### 4. ワークフローJSONをn8nへインポートする

n8nのワークフロー画面から、以下のJSONをインポートします。

```text
workflows/[Portfolio]AI動画自動投稿ツール.json
```

インポート後、各ノードの認証情報、Webhook URL、Google Sheets ID、Google Drive ID、Google DriveフォルダID、Replicate API URLなどを自分の環境に合わせて設定します。

### 5. Webhook URLを外部サービスに設定する

以下のWebhook URLを各サービス側に登録します。

| Webhook | 登録先 |
| --- | --- |
| LINE受付用Webhook | LINE Developers |
| Replicate完了通知Webhook | Replicate APIリクエスト内の `webhook` |

n8nでは、本番運用時にTest URLではなくProduction URLを使用します。

## 使い方

### 1. LINEからプロンプトを送信する

LINEに動画生成用のテキストを送信します。

```text
(例)
海辺を白い犬が走っている。
```

### 2. Google Sheetsにキューとして登録される

LINE Webhookで受信したテキストは、Google Sheetsに以下のような形で保存されます。

```text
status = queued
source = LINE
prompt = LINEで送信したテキスト
```

### 3. Schedule Triggerで処理される

Schedule Triggerが定期的にGoogle Sheetsを確認し、`status = queued` の行を取得します。
その後、`status`の行 を `processing` に更新し、`started_at`に現在時刻を入力します。
別のSchedule Triggerが `processing` の行を確認し、Replicate APIで動画生成を開始します。  

### 使用上の注意
Google Sheetsのqueuedを監視してReplicateを呼び出すトリガーと、動画生成を開始するトリガーの開始時間は被らないように設定してください。statusの読み込みがうまくいかない可能性があります。  
また、`started_at`は動画生成を開始した時間と完全には一致しません。詳しくは「現在の設計上の制約」「現時点で出来ないこと」の項も参照してください。

### 4. Replicate Webhookで動画生成完了を受信する

動画生成が完了すると、Replicateからn8nのWebhookへ通知が送られます。
Webhookで受信した `output` から動画URLを取り出し、HTTP Requestノードでmp4ファイルをダウンロードします。　　

実際に生成した動画
<video src="https://github.com/user-attachments/assets/7080b758-a11d-4598-bfd6-33f8fcf82def" width="80%" controls></video>

### 5. Google Driveに保存する

ダウンロードした動画ファイルをGoogle Driveに保存します。
保存時のファイル名は、`video_タイムスタンプ.mp4` のような形式で生成します。

### 6. GeminiでYouTube投稿文を生成する

Geminiに動画生成プロンプトを渡し、YouTube Shorts向けのタイトルと説明文をJSON形式で生成します。

### 7. YouTubeへ投稿する

Google Driveから動画ファイルを再取得し、YouTubeノードで動画をアップロードします。

### 8. Google Sheetsをdoneに更新する

別のSchedule Triggerが `processing` の行を確認し、対象行の `status` を `done` に更新します。
このdone更新では `started_at` は変更せず、`Sent_to_createvideo_at` を更新します。

### 使用上の注意
`Sent_to_createvideo_at`は生成された動画を受け取った時間と完全には一致しません。詳しくは「現在の設計上の制約」「現時点で出来ないこと」の項も参照してください。

## 現在の設計上の制約

このワークフローでは、LINE受付、動画生成依頼、動画生成完了後の処理、done更新を段階的に分けています。  
一連の処理を1つの流れにまとめようとした場合、動画生成ノードが再実行され、同じプロンプトから動画が複数回生成される問題があり、現在の設計になりました。  
この設計により重複実行のリスクは下げられますが、その一方で、動画生成開始時刻や完了時刻をリアルタイムに正確に記録する仕組みはまだ十分ではありません。  
代替策として、三つのスケジュールトリガーの開始時間をなるべく近づけることで、リアルタイムに近い時間を記録することはできます。ただし、一つのワークフロー内で複数のノードが同時に動いた場合、不具合が生じる場合もあるため、それぞれの処理にかかる時間に合わせて調整を行なってください。（特に、動画生成やタイトル生成、投稿処理等を別のAPIで行う場合は注意が必要です。）

## 動作確認方法

動作確認はn8n上での手動実行と、Google Sheetsのステータス確認で行います。

| 確認項目 | 確認内容 |
| --- | --- |
| LINE受付テスト | LINEからテキストを送り、Google Sheetsに `queued` の行が追加されること |
| キュー処理テスト | `status = queued` の行が `processing` に変わり、createvideoノードが実行されること |
| Replicate Webhookテスト | 生成完了後にWebhookが実行され、`video_url` が取得できること |
| Google Drive保存テスト | mp4ファイルがGoogle Driveに保存されること |
| YouTube投稿テスト | Geminiのタイトル・説明文を使ってYouTubeへ動画が投稿されること |
| done更新テスト | `status = processing` の行が `done` に変わり、`started_at` が上書きされないこと |
| エラー処理確認 | n8nの `errorWorkflow` に設定したワークフローIDが自分の環境向けになっていること |

## エラーハンドリング

n8nにはデフォルトで実行履歴の確認やエラー通知の仕組みが備わっています。
ここでは、それに加えて本ワークフロー独自に行っているエラーハンドリングを紹介します。

* **ステータス管理による再実行防止** — `queued` → `processing` → `done` と段階的に更新し、同一プロンプトの重複実行や投稿済み内容の再処理を防止しています。

* **Filterノードによる処理対象の限定** — `status = queued` や `status = processing` の行だけを抽出し、誤ったステータスのプロンプトが後続処理に渡らないようにしています。

* **ワークフロー分離によるリスク分散** — LINE受付・動画生成依頼・投稿処理・done更新を分離し、一部でエラーが発生しても他の処理が継続できる構成にしています。

* **`retry on fail` / `execute once` によるノード制御** — 一時的な通信失敗が起こりやすいノードでは再試行を有効化し、重複実行の影響が大きい処理では `execute once` で多重実行を防止しています。

## 公開用JSONについて

`workflows/[Portfolio]AI動画自動投稿ツール.json` は、GitHub公開用に一部の値をマスクしたサンプルです。

以下の情報は実際の値ではなく、プレースホルダーに置き換えています。

* APIキー
* OAuth認証情報
* Webhook URL
* Google Sheets ID
* Google SheetsのシートID
* Google Drive ID
* Google Drive Folder ID
* LINE関連ID
* n8nのWorkflow ID / Instance ID

利用する場合は、自分のn8n環境にインポートした後、各ノードの認証情報やURLを設定し直してください。

## このプロジェクトで学んだこと

- n8nを使って、Webhook・スケジュール実行・外部API等を組み合わせた一連の自動化ワークフローを構築する方法を学びました。  
- LINE Webhookで受け取ったメッセージを条件分岐し、テキストメッセージのみを処理対象にする方法を学びました。  
- Google Sheetsを簡易的なキューとして利用し、queued / processing / done のステータスで処理状態を管理する方法を学びました。  
- 外部APIを実行する前にステータスを processing に更新することで、同じデータが重複して処理されるリスクを下げる設計を学びました。  
- 動画生成APIのような時間のかかる非同期処理では、処理完了を待ち続けるのではなく、Webhookで完了通知を受け取る構成が有効であることを学びました。  
- 外部の動画生成サービスをn8nから呼び出し、入力したプロンプトをもとに動画を作成する流れを学びました。　　  
- 生成された動画のURLを受け取り、その動画ファイルをダウンロードしてGoogle Driveに保存する方法を学びました。    
- LLM APIを活用して、YouTubeショート動画に最適なタイトルと説明文を自動生成させる方法を学びました。　  

## 現時点でできないこと

* 動画生成やタイトル生成、投稿等、特に重要な処理が失敗した際の代替処理
* 動画生成開始・生成完了・YouTube投稿完了などの正確な時刻をリアルタイムにGoogle Sheetsへ記録すること
* 受信ログ、YouTube投稿IDやGoogle DriveファイルURL等一部処理履歴の追跡
* Webhookが正しいサービスから送られてきたものかを確認する仕組み等、セキュリティ対策

## 今後の改善案

* 動画生成開始時刻・生成完了時刻・YouTube投稿完了時刻を、各Webhookや投稿ノードの直後でGoogle Sheetsへ記録する仕組み
* Replicateのprediction IDやYouTube投稿IDをGoogle Sheetsに保存し、各処理結果を同じ行に紐づける仕組み
* done更新をSchedule Triggerではなく、YouTube投稿成功後に直接実行する構成への見直し
* 投稿完了Webhookの受信ログ保存
* YouTube投稿ID・投稿URLの保存
* Replicate生成失敗時の代替動画生成
* Gemini失敗時の代替処理
* YouTube投稿失敗時、他のAPIを利用した投稿
* 処理失敗時の`failed` ステータスの記録、及びその行を再実行する仕組み
* Google Sheetsではなくデータベースでのキュー管理
* 投稿先をInstagram / TikTokなどに拡張
* n8nのサブワークフロー化による責務分離

## 注意事項

このワークフローは、学習・ポートフォリオ用途を想定しています。

実運用する場合は、以下に注意してください。

* APIキーやOAuth認証情報をGitHubに公開しない
* n8nのProduction Webhook URLを公開しない
* Google Sheets IDやGoogle Drive Folder IDを公開しない
* YouTube投稿の公開範囲を確認する
* Replicateなど外部APIの料金・利用制限を確認する
* 同じ行を複数回処理しないようにする
* createvideoやYouTube投稿ノードを不用意に再実行しない
* n8nの `errorWorkflow` は自分の環境のワークフローIDへ差し替える
* WebhookはTest URLではなくProduction URLを使用する
* READMEやJSONに個人のメールアドレスやアカウントIDを残さない

公開用JSONでは、認証情報・Webhook ID・APIキー・各種サービスIDなどをダミー値に置き換えています。

## ライセンス

このプロジェクトはポートフォリオ用途で公開しています。
利用・改変の条件については、必要に応じて今後ライセンスを設定します。
