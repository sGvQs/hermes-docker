# iPhone Shortcuts と LINE グループトークの取得・エクスポート・反応 (iOS 26 / iOS 27 beta, 2026-10 時点)

注: 調査は約10回の検索/取得のみ。iOS 27 の情報は beta 時点のブログ報告が中心で、正式リリース版の公式仕様は未確認。

## 1. パーソナルオートメーションのトリガーと無人実行・頻度

### Takeaway
iPhone の個人用オートメーションには時刻・App・Wi-Fi・充電・NFC・メッセージ等のトリガーがあり、多くは確認なしで自動実行できる。ただし LINE のメッセージ内容を読むトリガーは iOS 26 には存在しない。時刻トリガーは「毎時」を直接は持たない(推論)。

### Cited Findings
- Apple のガイドは、確認なしで自動実行できるトリガーとして Time of Day, Alarm, Sleep, Arrive, Leave, CarPlay, Email, Message, Transaction, Wi-Fi, Bluetooth, Apple Watch Workout, NFC, App, Airplane Mode, Do Not Disturb, Low Power Mode, Battery Level, Charger, Sound Recognition を列挙(取得時点のページ、iOS 版ガイド) — [Apple Shortcuts User Guide](https://support.apple.com/guide/shortcuts/apdfbdbd7123/ios)
- 確認なしで実行するには、ショートカットの Info > Privacy で「Allow Running When Locked」を有効にし、各アクションも自動実行可能である必要がある。有効にすると実行通知も出ない(同ガイドの記述) — [Apple Shortcuts User Guide](https://support.apple.com/guide/shortcuts/apdfbdbd7123/ios)
- トリガーのカテゴリは Event / Travel / Communication / Setting。Focus オン・オフ、Wi-Fi、Bluetooth も Setting トリガー — [Apple Shortcuts User Guide (iOS 26)](https://support.apple.com/en-ca/guide/shortcuts/apd690170742/9.0/ios/26)
- iOS 26 のリリースノート解説で追加されたトリガー群(Time of Day, Alarm, Email, Message, Folder, File, External Drive, Wi-Fi, Bluetooth, Display, Stage Manager, App, Battery Level, Charger, Focus)は macOS 向けの話であり iPhone のリストではない — [Matthew Cassinelli: What's New in Shortcuts 26](https://matthewcassinelli.com/whats-new-in-shortcuts-for-ios-ipados-macos-watchos-visionos-26/)
- Message トリガーは Apple の列挙上は「メッセージ」トリガー。LINE 等サードパーティ対応の記述は見つからなかった(下記参照)。
- パーソナルオートメーションは端末単位で iCloud バックアップはされるが他端末へ同期されない — 同上 Apple guide 検索結果要約

### Inferences
- 時刻トリガーは「毎日/毎週/毎月」の指定が基本で、毎時には対応していない。毎時相当なら複数の時刻オートメーションを作る必要がある(推論。今回 Apple 文書で毎時オプションの有無は確認していない)。
- Back Tap / Action Button / ロック画面ウィジェットは手動起動(1タップ)の入口であり、無人実行ではない(推論・一般知識)。

### Gaps
- Run Immediately / Ask Before Running の公式文言とトリガー別の例外一覧は、取得したページには出てこなかった。時刻トリガーの最短間隔の公式記載も未確認。

## 2. サードパーティ App (LINE) の通知を読むトリガー

### Takeaway
iOS 26 までは存在しない(今回の調査で確認できず)。iOS 27 beta で「アプリの通知」トリガーが追加されたが、beta 1 時点ではフィルタが壊れており、通知本文をショートカット側が読めないという報告がある。LINE 本文の自動取得手段としては現状不確実。

### Cited Findings
- iOS 27 / iPadOS 27 / macOS 27 Developer Beta 1 で、アプリが通知を送ったときにショートカットを起動する Notification オートメーションが追加。タイトル(既定)または本文でのフィルタを意図しているが beta 1 ではフィルタが動かず、フィルタなしでのみ発火。デバイスがロック中でも実行される — [Derek Seaman (2026-06)](https://www.derekseaman.com/2026/06/home-assistant-notifications-that-run-apple-shortcuts-yes-really.html)
- 検証されたのは Home Assistant アプリのみ。「理論上は任意アプリ」は著者の推測であり未検証 — 同上
- MacStories 系の記事は、通知トリガーは特定アプリからの通知で、タイトルまたは本文に特定テキストを含むものに反応すると説明。公開されている変数一覧は記事中に無い — [MacStories (Deflector)](https://www.macstories.net/ios/deflector-turns-ios-27-notification-triggers-into-a-shortcut-launcher/)
- Homey フォーラム(逸話): トリガーは発火するが「ショートカットは通知の内容を読めない」との報告。返信では beta の制限で、Messages を含め他アプリの情報はショートカットに公開されていないとの見解、後のアップデートで変わる可能性 — [Homey Community](https://community.homey.app/t/ios-27-notifications-and-new-shortcuts/156044)
- 上記は矛盾する側面あり: MacStories は Deflector で通知本文をパースしてショートカット名を取り出していると述べる(自己通知のため本文が読める可能性)ので、内容の可読性は beta 内でも報告が割れている。

### Inferences
- LINE は通知本文を「送信者名: 内容」の形で出すが、通知プレビュー設定やグループのミュート設定で通知自体が来ない場合がある(一般知識)。通知ベースは取りこぼしリスクが高く、「今日の全メッセージ」という要件には不向き。
- Deflector の例は自己通知なので、LINE 通知の本文が変数として取れる根拠にはならない。

### Gaps
- iOS 27 正式版でのトリガー出力変数(通知タイトル/本文/アプリ名)の公式仕様、およびサードパーティ通知本文の取得可否は未確認。LINE での実機検証報告は見つからなかった。
- iOS 26 に通知トリガーが無いことは「見つからなかった」ことによる確認で、公式の否定記述ではない。

## 3. LINE 固有のフック(トーク履歴送信、App Intents、URL スキーム、OCR)

### Takeaway
LINE は「トーク履歴を送信」でトークルーム単位のテキストファイルを手動で書き出せる。これを共有シート経由でショートカットに渡す構成が最小手動の現実解。LINE が公式の App Intents/ショートカット専用アクションを提供している証拠は見つからなかった。

### Cited Findings
- トーク履歴の送信: 対象トークを開き「メニュー > 設定 > トーク履歴を送信」(旧版は「その他」)。トークルーム単位でテキスト形式、スタンプ・写真等は項目名のみで、トークへの復元には使えない — [appllio](https://appllio.com/line-talk-history-send-mail), [Lトーーク (App Store)](https://apps.apple.com/jp/app/l%E3%83%88%E3%83%BC%E3%83%BC%E3%82%AF-%E3%83%88%E3%83%BC%E3%82%AF%E5%88%86%E6%9E%90-%E6%8C%AF%E3%82%8A%E8%BF%94%E3%82%8A%E3%82%A2%E3%83%97%E3%83%AA/id1555059527)(いずれも非公式解説。LINE 公式ヘルプ自体は今回取得できていない)
- 出力はメール添付やPC保存で使える旨の記述あり(共有シートが使える根拠。ショートカットが共有シートの受け先になれる点は Shortcuts の一般仕様) — [appllio](https://appllio.com/line-talk-history-send-mail)
- ショートカットアプリ側の LINE アクションとして紹介されているのは、時刻指定での「メッセージ送信予約」やトーク画面を開くホーム画面ショートカット。トーク内容の読み取りアクションは確認できず — [appllio](https://appllio.com/line-message-sending-reservation), [linestep](https://linestep.jp/2023/02/21/line-shortcut/)
- LINE の App Intents 公式情報は検索で見つからなかった。IFTTT 経由の LINE 連携は存在するが、LINE 公式アカウント(友だち追加)経由で、自分のトーク取得用ではない — [IFTTT](https://ifttt.com/connect/ios_shortcuts/line_v2)

### Inferences
- 共有シートで受けるショートカット(「共有シートからファイル/テキストを受け取る」)を作れば、「トーク履歴を送信」から共有先に自分のショートカットを選ぶだけで処理が走る。ただし LINE が共有時に添付ファイルとして渡すかテキストとして渡すかは未検証。
- 画面に表示した状態でスクリーンショットし、「画像からテキストを抽出」(OCR) で処理する経路は技術的に可能だが、1画面分しか取れずグループの1日分には非現実的(推論)。Back Tap でスクリーンショット自体は撮れるが手動スクロールが必要。
- 「今日だけ」の抽出は、全履歴ファイルの日付行をショートカットまたはMac側でフィルタする設計になる(LINE の出力形式は日付見出しを含む旨が一般的だが今回未検証)。

### Gaps
- LINE 公式ヘルプのエクスポート仕様(日付範囲指定の有無、サイズ制限)、LINE URL スキーム(line://)の公式一覧は取得できていない。
- 共有シートでの受け渡し形式、LINE 側が Shortcuts にアクションを公開しているかの実機確認は無い。

## 4. 自動化できる範囲と最小タップのワークフロー

### Takeaway
「選択グループの今日のメッセージを定期的に自動取得」はショートカット単体では実現できない(iOS 26 時点)。現実的なのは「1グループにつき長押し/メニュー → トーク履歴を送信 → ショートカットを選択」の手動数タップ+以降は自動処理。

### Cited Findings
- 上記 §1 §2 §3 の通り、LINE のトーク本文を読むトリガー/アクションは確認できず、履歴送信は手動かつトークルーム単位。
- 検索では「ショートカットで LINE のトーク履歴を自動でテキスト化・共有する方法は確認できなかった」(Japanese 検索結果の要約) — appllio 等検索結果

### Inferences
- 最小手動: (1) LINE 内でトーク履歴を送信 → (2) 共有先で自分のショートカットを選択。以降の整形・送信は自動。グループ N 個なら毎回 N 回の手動操作。
- 時刻オートメーション(Ask Before Running でない設定)で「通知を出して手動操作を促す」リマインドまでは自動化可能。
- iOS 27 正式版で通知本文が取得できる場合のみ、LINE 通知を逐次キャプチャして日次で集約する案が成立しうるが、取りこぼし(ミュート、通知プレビュー無効、既読済み)のリスク。公式の完全ログではない。
- 規約: LINE アプリ操作の自動化(UI スクリプト等)による回避は推奨しない。公式のエクスポート機能と共有シートの範囲に留める。

### Gaps
- iOS 27 での実機検証が無く、通知トリガー案の網羅性は未知。

## 5. 結果の送り先 (Telegram / Mac / iCloud / リマインダー / 配偶者共有)

### Takeaway
送信側はショートカット標準機能で全て可能(一般知識)。今回の調査で個別の公式出典は取得していないため、推論として記す。

### Cited Findings
- 今回、送信方法そのものに関する出典は取得していない(IFTTT/Pushcut が Shortcuts と外部サービスの連携手段として存在することのみ確認) — [IFTTT](https://ifttt.com/connect/ios_shortcuts/line_v2), [Pushcut](https://pushcut.io/help/json)

### Inferences
- 「URL の内容を取得」で Telegram Bot API の sendMessage / sendDocument に POST できる(一般知識)。メッセージ長4096字の制限があるため長文はファイル送信か分割。
- Mac mini への HTTP エンドポイント POST、iCloud Drive へのファイル保存、共有リマインドリスト、メッセージ共有はいずれも標準アクションで可能(一般知識)。SSH 実行アクションも存在(一般知識)。
- 配偶者共有: 共有リマインドや iCloud 共有フォルダは追加アプリ不要。LINE への再送信は共有シート経由で手動。

### Gaps
- 各送信方式の iOS 26/27 での挙動(ロック中の実行可否・ネットワーク制約)は未検証。

## 6. 実例(日英)

### Takeaway
LINE トークを自動取得する実例は見つからなかった。見つかるのは送信予約と Home Assistant 等の iOS 27 通知トリガー実験のみ。

### Cited Findings
- 日本語: LINE の送信予約ショートカットや、トーク履歴のテキスト送信手順の解説は多数。内容を読む自動化の事例は確認できず — [appllio](https://appllio.com/line-message-sending-reservation)
- 英語: iOS 27 beta の通知トリガー(Home Assistant で Focus 切替)、通知内容は読めないとの報告 — [Derek Seaman](https://www.derekseaman.com/2026/06/home-assistant-notifications-that-run-apple-shortcuts-yes-really.html), [Homey Community](https://community.homey.app/t/ios-27-notifications-and-new-shortcuts/156044)

### Inferences
- 実例が無いこと自体が、制約が強いことの傍証にすぎず、不可能の証明ではない。

### Gaps
- Reddit r/shortcuts、Zenn/Qiita/note は検索で直接ヒットせず未確認。
