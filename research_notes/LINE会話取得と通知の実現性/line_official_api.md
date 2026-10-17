# LINE公式開発者プラットフォームによる通常グループトークのメッセージ取得 (2026年時点)

注: 「通常のLINEグループ」を「ユーザー本人のアカウントとして」読むための公式APIは確認できなかった。公式に可能なのは「LINE公式アカウント(Messaging APIのボット)をグループに招待し、webhookで受信する」方式のみ。

## 1. Messaging APIのボットを通常グループ/複数人トークに追加できるか・受信イベント・履歴API

### Takeaway
ボット(LINE公式アカウント)は通常のグループ/複数人トークに招待可能で、メッセージはwebhookのメッセージイベントとしてリアルタイムに届く。過去メッセージ取得APIは無く、受信時に保存しない限り後から取得できない。ボットはメンバーとして参加するため完全な「不可視」運用は公式には想定されない(推測を含む)。

### Cited Findings
- ユーザーはLINE公式アカウントをグループトーク・複数人トークに招待できる。コンソールのMessaging API設定「グループトーク・複数人トークへの参加を許可する」を有効化が必要(デフォルト無効)。1つのグループ/ルームに参加できる公式アカウントは同時に1つのみ — [グループトーク・複数人トーク](https://developers.line.biz/ja/docs/messaging-api/group-chats/)
- グループでメッセージが送られるとボットサーバーにメッセージイベントが届き、`source`に`groupId`(または`roomId`)と`userId`が含まれる — [同上](https://developers.line.biz/ja/docs/messaging-api/group-chats/)
- グループで受け取れるイベント: message, edit, unsend, join, leave, memberJoined, memberLeft, postback。イベント表でメッセージイベントはグループ対象(✅)で「ユーザーがメッセージを送信した時」と記載。メンションのみに限定する記述は無い(メンション時は`mention.mentionees`に`isSelf: true`が入るのみ) — [メッセージ(Webhook)受信](https://developers.line.biz/ja/docs/messaging-api/receiving-messages/)
- 履歴API無し。「Webhookを受信した後で、あらためてテキストを取得するためのAPIはありません。」引用メッセージ内容も再取得不可 — [同上](https://developers.line.biz/ja/docs/messaging-api/receiving-messages/)
- 画像・動画・音声・ファイルは「ユーザーが送ったコンテンツは一定期間後、自動的に削除されます」(期間の数値は公式ページに明記なし)。取得はwebhookのmessage IDで「コンテンツ取得」エンドポイント — [受信](https://developers.line.biz/ja/docs/messaging-api/receiving-messages/), [リファレンス](https://developers.line.biz/en/reference/messaging-api/)
- webhook要件: HTTPSのPOST。署名検証必須(`x-line-signature`)、200番台を返す、非同期処理推奨。長期間失敗するとLINE側がwebhook送信を停止する可能性。再送はデフォルト無効(設定で有効化可、重複は`webhookEventId`、`deliveryContext.isRedelivery`で判別、順序は`timestamp`) — [受信](https://developers.line.biz/ja/docs/messaging-api/receiving-messages/)
- ボットが送るメッセージはグループ全員に表示される。グループでマルチキャスト不可 — [グループトーク](https://developers.line.biz/ja/docs/messaging-api/group-chats/)
- 複数人トークはLINE 10.17.0でグループに統合(新規は常にグループ) — [同上](https://developers.line.biz/ja/docs/messaging-api/group-chats/)

### Inferences
- 全メッセージ受信は公式文書に「メンション不要」と明文化されていない。イベント表の記載上は全メッセージイベントが届くと読めるが、実機検証が必要(Messaging API設定の「応答設定」でグループ内の挙動を確認する価値あり)。
- 「1時間に1回、その日のメッセージを読む」はwebhookで常時受信しDBに蓄積→毎時バッチで判定、という構成にするしかない(webhook稼働前の発言は取得不能)。
- ボットはグループのメンバー一覧に表示される(LINEの仕様上アカウントとして参加)と推測。公式ページに可視性の明記は確認できず。不可視化の手段は公式に無い。
- 「人Xがいないグループは対象外」といったルールはメンバーID取得(次節)またはwebhook上のuserId蓄積で実装可能。

### Gaps
- 全メッセージ受信(メンション無し)の明文化・コンテンツ保持期間の具体日数は公式ページで確認できず。
- ボットが参加時にグループ内で通知/表示される具体的UIは未確認。

## 2. グループメンバー取得API(アカウント種別・特定人物の在否)

### Takeaway
メンバー数・メンバーのuserID一覧・メンバープロフィール・グループ概要のAPIがある。ただしメンバーID一覧は認証済み(または プレミアム)アカウント限定。未認証アカウントでは「発言した人のuserId」をwebhookから集める方法しかない。

### Cited Findings
- リファレンス上のグループAPI: Get group chat summary(`#get-group-summary`)、Get number of users in a group chat(`#get-members-group-count`)、Get group chat member user IDs(`#get-group-member-user-ids`)、Get group chat member profile(`#get-group-member-profile`)、Leave group chat — [Reference](https://developers.line.biz/en/reference/messaging-api/)
- エンドポイント: `GET /v2/bot/group/{groupId}/members/ids`(第三者記事。100件/リクエスト、継続トークン) — [Thai dev guide via search](https://medium.com/linedevth/dafb17e5864a)(二次情報)
- 「This feature is available only for verified or premium accounts.」(メンバーID取得エンドポイント) — [Get user IDs](https://developers.line.biz/en/docs/messaging-api/getting-user-ids/)
- webhookにはユーザーのメッセージ送信・参加時にuserIdが含まれるため、APIなしでも取得可 — [同上](https://developers.line.biz/en/docs/messaging-api/getting-user-ids/)
- ユーザーがプロフィール情報取得に同意していない場合(PC版LINEのみ利用など)、webhookのsource user、メンション、メンバーID系エンドポイントの結果からプロフィール情報が除外され、メンバーシップイベントも送られない。退出/ボット削除済みのユーザーのプロフィールは取得不可 — [ユーザーの同意](https://developers.line.biz/ja/docs/messaging-api/user-consent/)
- LINE公式アカウントヘルプ: 認証済みはグループ全メンバーの内部IDを確認可、未認証は発言したユーザーのみ(検索スニペット経由の二次情報) — [LINE help](https://help2.line.me/official_account/web/categoryId/20006335/about:blank)
- プレミアムID新規購入終了/新規アカウントの認証が現在日本・台湾・タイのみ、との記述(Alibaba Cloud) — [二次情報](https://help.aliyun.com/en/chatapp/user-guide/line-channel-faq)(未検証)

### Inferences
- 「特定人物がグループにいるか」を判定するには、その人のuserId(ボットごとに異なるID)を把握する必要がある。ボット単位のuserIdなので、本人がボットと友だちになるか当該グループで発言してwebhookからIDを紐付ける運用が必要。
- 認証済みアカウント(審査要・法人/個人事業主向けの傾向)を取得しない個人開発では、メンバー全員の列挙は不可。

### Gaps
- 認証済みアカウント取得の個人利用可否・2026年時点の審査条件は一次情報で未確認。
- 当該エンドポイントのレスポンス詳細は大型リファレンスの該当節を取得できず未確認。

## 3. メッセージイベントで送信者・グループID・時刻が分かるか

### Takeaway
分かる。`source.type=group`、`groupId`、`userId`、イベントの`timestamp`(ミリ秒)で「今日のメッセージ」やグループ別集計が実装可能。userIdはユーザーが同意していない場合など欠ける場合がある。

### Cited Findings
- グループのsourceに`groupId`と`userId`を含む — [グループトーク](https://developers.line.biz/ja/docs/messaging-api/group-chats/)
- 順序確認用に`timestamp`を使う旨、重複検知に`webhookEventId` — [受信](https://developers.line.biz/ja/docs/messaging-api/receiving-messages/)
- 未同意ユーザーはプロフィール情報が含まれない — [同意](https://developers.line.biz/ja/docs/messaging-api/user-consent/)

### Inferences
- 「今日」はサーバー側で`timestamp`をJST変換して判定。送信者名は`Get group chat member profile`で取得(表示名)。
- 「Xがいない」ルールは、userId集合に対するメンバーAPIまたは発言履歴ベースで実装。

### Gaps
- webhookのmessageイベント全フィールド表(`timestamp`の型、`replyToken`等)は今回直接引用できていない。

## 4. 公式アカウント vs 個人アカウント(LINE公式アカウント)の区別

### Takeaway
Messaging APIは常に「LINE公式アカウント」(ボット)として動作する。個人のLINEアカウントでトークを読むための公式APIは存在しない(今回の調査で見つからず)。

### Cited Findings
- Messaging APIは公式アカウントがユーザーとやり取りする仕組みで、グループでメッセージ送信やメンバー情報取得が可能 — [概要](https://developers.line.biz/ja/docs/messaging-api/overview/)
- 公式アカウントの認証済み/未認証/プレミアムの区別が機能に影響(メンバーID取得) — [Get user IDs](https://developers.line.biz/en/docs/messaging-api/getting-user-ids/)
- 公式ドキュメント上、ユーザー本人のトーク履歴を第三者アプリが読むAPIの記載は検索範囲で見つからず。LINE Loginは認証・プロフィール取得用途(本調査では詳細未取得)。

### Inferences
- 「公式アカウントとのトークか判別」は、webhookの`source.type`(user/group/room)で、ボット宛1:1かグループかを区別できる。個人間トークはボットが参加していない限り存在自体が見えない。

### Gaps
- LINE Login/ミニアプリがチャット内容を読めるかの一次情報は未取得(権限スコープに該当なしと推測)。

## 5. LINE Notify終了・OpenChat・新API・MCP

### Takeaway
LINE Notifyは2025-03-31終了(代替はMessaging API)。OpenChat向けの公式ボットAPIは見つからず。公式LINE Bot MCPサーバーはあるが送信/管理系ツールのみで、メッセージ読取ツールは無い。

### Cited Findings
- LINE Notifyは2025年3月31日にサービス終了、Messaging APIへの移行を推奨 — [roboin.io](https://roboin.io/article/2024/10/07/line-terminates-line-notify-service)(二次)、[Impress Watch](https://internet.watch.impress.co.jp/docs/yajiuma/1629950.html)
- 公式MCP `line/line-bot-mcp-server`(Apache-2.0)のツール13個: push_text_message, push_flex_message, broadcast_text_message, broadcast_flex_message, get_profile, get_message_quota, rich menu系(get_rich_menu_list, delete_rich_menu, set_rich_menu_default, cancel_rich_menu_default, create_rich_menu), get_follower_ids, get_group_summary。受信/読取ツールなし。要: Node.js v22以降 or Docker、Channel Access Token、任意のDESTINATION_USER_ID — [GitHub(WebFetch経由要約)](https://github.com/line/line-bot-mcp-server)
- OpenChat専用のAPI/ボット仕様は検索で確認できず — 検索結果(Messaging API文書のみ)
- LINEヤフーが2026-07-02に「Agent i in chat」を発表(2026年内提供予定)。トーク内でAIエージェントを呼び、会話文脈を踏まえ回答。トーク履歴の分析に同意したユーザーのみ対象、履歴はAI学習に不使用とのこと。開発者向けAPIの言及なし — [Impress Netshop](https://netshop.impress.co.jp/n/2026/07/03/16352)

### Inferences
- LINEアプリ側の公式AI機能は消費者向けで、開発者が自分のLLMでグループ会話を読むための公式経路ではない。

### Gaps
- LINE Developers公式changelogの2025-2026新API一覧は確認できず。OpenChat APIの有無は断定不可(「見つからなかった」のみ)。

## 6. レート制限・料金・要件・規約

### Takeaway
受信(webhook)自体は無料枠に影響しない。送信は応答メッセージ(reply)が通数にカウントされず、push等はカウント。公開HTTPS webhookと署名検証が必須。規約上の個別AI条項は確認できず、LINEユーザーデータポリシー等の遵守が必要。

### Cited Findings
- 料金(日本例): コミュニケーション 0円/200通、ライト 5,000円/5,000通、スタンダード 15,000円/30,000通(追加3円/通まで)。応答メッセージはカウント対象外、push/multicast/broadcast/narrowcastは対象。掲載日不明のため2026年版か未確認 — [料金](https://developers.line.biz/ja/docs/messaging-api/pricing/)
- レート制限: その他エンドポイント 2,000 req/s、マルチキャスト・メンバーシップ系 200 req/s、超過で429(トークンバケット) — [Reference](https://developers.line.biz/en/reference/messaging-api/)
- 規約群: LINEユーザーデータポリシー(ユーザー情報の取り扱い)、LINE開発者契約、LINE公式アカウントAPI利用規約、LINE公式アカウント利用規約、同ガイドライン等 — [規約とポリシー](https://developers.line.biz/ja/terms-and-policies/)
- 公式ドキュメントのコンテンツ保持: ユーザー送信コンテンツは一定期間後自動削除 — [概要](https://developers.line.biz/ja/docs/messaging-api/overview/)

### Inferences
- 判定結果をユーザー本人にだけ通知するならpush(1:1)が必要で、通数枠(無料200通/月)を消費。1時間1回×通知のみ価値あり時なら無料枠でも収まる可能性。ただし本人が友だち追加している必要あり。
- グループ内の他メンバーの発言をLLM(外部事業者)に送る処理は、ユーザーデータポリシー・プライバシー上の配慮(参加者への告知・同意)が必要と推測。具体条文は未確認。
- ボットをグループに入れること自体がメンバーに見える点、かつ他メンバーの同意なく読み取る点は、運用・規約リスク。

### Gaps
- LINEユーザーデータポリシー/公式アカウントAPI利用規約の本文(LLM送信、保存期間、第三者提供の条項)は本文未取得。
- 2026年時点の料金改定有無は未確認。
