# iPhone中心の「配偶者に伝えるべき情報の自動フラグ」パターン調査 (as of 2026-10, iOS 26系)

調査上の注意: 検索は計約10回。LINE公式ヘルプの該当ページは直接取得できず(取得したページに通知設定の記載なし)、ベンダー/メディア記事ベースの確認が多い。「見つからなかった」ことを明示する。事実=Documented、体験談=Anecdote、推論=Inference。

## Pattern A: iOS通知ベース (Apple Intelligence通知要約 / Focus / 通知サマリー)

### Takeaway
iOS 26のApple Intelligence通知要約は存在するが、LINE通知に対して要約・キーワード検知・転送ができると確認できた資料はない。通知内容を別アプリに自動転送する公式手段も見つからなかった。

### Cited Findings
- iOS 26ではNews/Entertainment系の通知要約が復活。要約はアプリ単位でオフにでき、ミスの可能性が警告される — [9to5Mac](https://9to5mac.com/2025/10/27/ios-26-brought-back-a-controversial-ai-feature-heres-whats-new/)
- 検索ではLINE通知が要約対象になるかを扱う資料は見つからなかった(LINE固有の記述なし) — 検索結果(上記9to5Mac等)
- LINE通知のプレビューは iOS設定「通知」→「LINE」→「プレビューを表示」で常に/ロック解除時のみ等に制御可能(古い記事) — [appllio](https://appllio.com/ios-hidden-message-notification-iphone-ipad)
- ショートカットの「通知を受信」トリガーでLINE通知内容を処理する方法は検索で確認できなかった — 検索結果(該当なし)

### Inferences
- Documented不明。一般知識として、iOSのショートカットのオートメーションにはアプリ通知の本文を取得するトリガーが無い(要検証)。そうであれば通知起点の自動フラグは不可能で、通知要約を目視するのが限界。
- Focusのフィルタは「誰/どのアプリから通知を許可するか」を絞るもので、本文キーワードでの絞り込みではない(一般知識・要検証)。
- 通知要約はLINEグループの「結婚式の情報」を意味的に拾う保証がなく、要約が出ても「配偶者に伝えるべき」判定はユーザーの目視。

### Gaps
- LINE通知がApple Intelligence要約の対象かの確証、Focusでキーワード絞り込みの有無、通知トリガーの存否はApple公式ドキュメントで未確認。

## Pattern B: LINE標準機能

### Takeaway
LINEに「個人トークの本文キーワードで通知/自動転送する」機能は、今回の調査では見つからなかった。あるのは通知ON/OFF、メンション通知、ノート、Keep、予約送信(ショートカット経由)等の周辺機能。「スマート通知」は公式アカウントからのニュース配信でトーク内容の検知ではない。

### Cited Findings
- 「LINEスマート通知」は公式アカウントを友だち追加し、キーワードをフォローするとニュースが届く機能(2021年記事)。自分のトーク本文を監視するものではない — [All About](https://allabout.co.jp/gm/gc/492668/), [Impress](https://webtan.impress.co.jp/n/2021/11/17/41944)
- グループ通知をオフにしても自分宛メンションは通知される。「自分へのメンション通知」は設定>通知でオフにでき、グループ別ではなく全体設定 — [マイナビニュース](https://news.mynavi.jp/article/20230705-2720447/)
- ノートはグループ内で話題ごとの情報共有ができコメント可能(家計管理の利用例) — [リベシティ記事](https://library.libecity.com/articles/01JQXX3V1H4S932BVD9SV0WE26)
- iPhoneのショートカットでLINEメッセージの送信予約が可能(個人LINE) — [appllio](https://appllio.com/line-message-sending-reservation)
- 個人LINEのトーク履歴を外部へ自動転送する公式手段は検索で確認できなかった — [検索結果](https://lp.yoom.fun/blog-posts/lineworks-telegram-message-auto-notification-efficiency) (Yoom等は公式アカウント/LINE WORKS向けで別物)
- LINE Help Centerの通知カテゴリ(https://help.line.me/line/smartphone/categoryId/200000276)は存在するが本文は取得できず、キーワード通知の記述は確認できなかった。

### Inferences
- 「キーワード通知/自動転送機能は無い」と断定はできないが、メディア記事・ヘルプ構成の範囲では存在の証拠なし。ユーザーが「見つけたら私に教える」にはメンション運用(友人に「@自分」を付けてもらう)か手動が現実的。
- 手動転送(トーク長押し>転送、またはシェア)は配偶者個別トークへ1〜3タップで可能(一般知識)。トーク履歴の「トークを送信(エクスポート)」は手動操作でテキストファイルを共有シートに出せる(一般知識・今回未検証)。

### Gaps
- LINE Help Centerの該当記事本文(通知、トーク履歴送信)の一次確認。

## Pattern C: スクリーンショット/OCR/画面収録

### Takeaway
技術的には「スクショ→ショートカットでテキスト抽出→Use Model→Telegram」は組めるが、スクショ自体が手動トリガーで全自動にならず、画面収録の常時監視はLINEの規約・バッテリー・プライバシーの面で現実的でない。

### Cited Findings
- iOS 26のショートカットに「Use Model」アクションがあり、オンデバイス/Private Cloud Compute/ChatGPTを選べる — [9to5Mac](https://9to5mac.com/2025/06/16/shortcuts-app-has-chatbot-like-apple-intelligence-powers-in-ios-26/), [Matthew Cassinelli](https://matthewcassinelli.com/ios-26-public-beta-shortcuts-actions-apple-intelligence-messages-notes-checklists/)
- 他アプリ内容の取り込みについては記事に記載なし(Text→Use Modelの例のみ) — 9to5Mac(上記)

### Inferences
- 共有シートのショートカット受け口に画像/テキストを渡し、テキスト抽出(Vision OCR系アクション)→Use Model→メッセージ送信、という構成は iOS標準アクションだけで可能と推論(要実機検証)。
- 手動で毎回スクショが必要で「自動検知」ではなく「選別後の転送」の効率化に留まる。

### Gaps
- Telegram送信アクション(Telegram Bot API=URL取得アクション)の具体的手順の一次資料は未取得。ショートカットのバックグラウンド実行制限の公式資料も未確認。

## Pattern D: Mac mini + iPhone Mirroring / LINE for Mac の自動操作

### Takeaway
iPhone Mirroringは画面共有であり、自動化APIは提供されていない(記事に記載なし)。通知は目視でMacに出るが本文を機械的に取る公式手段は確認できない。既存調査(サポートされた方法なし)と整合する。

### Cited Findings
- 要件: Apple silicon/T2 Mac+macOS Sequoia、iOS 18、同一Apple ID(2FA)、近接。ミラーリング中iPhoneはロック必須 — [MacRumors](https://macrumors.com/guide/iphone-mirroring)
- 制限: カメラ/マイク不可、Notification Center/Control Center/ロック画面は表示不可、ウィンドウはリサイズ不可 — [MacRumors](https://macrumors.com/guide/iphone-mirroring), [Android Authority](https://www.androidauthority.com/iphone-mirroring-macos-hands-on-3454379)
- iPhone通知はMacの通知センターにiPhoneアイコン付きで出る — [MacRumors](https://macrumors.com/guide/iphone-mirroring)
- 発売時EUでは利用不可との読者コメントあり(記事は未確認)。Automation/Shortcuts対応の記載なし。

### Inferences
- Accessibility自動操作でLINE画面を読む方式は、規約上の扱いが不明確かつ壊れやすい。LINE利用規約の全文確認が必要(今回の検索では自動化禁止条項を確認できず、公式全文は未取得)。本調査は規約回避手段を提示しない。
- Mac miniは常時稼働でLLM処理には向くが、入力ソースが無い(通知もプレビュー限定)。

### Gaps
- iPhone Mirroringのmacで26(Tahoe)以降の仕様変更、LINE利用規約の自動化関連条項の原文。

## Pattern E: ショートカット + 共有シート + LLM + Telegram (最小タップ設計)

### Takeaway
最も現実的なのは「半自動」: 気になるメッセージをユーザーが選び共有シートでショートカットに渡す。LLMで要点整理して Telegram(自分か配偶者)へ送る。全自動検知は不可。

### Cited Findings
- Use Modelアクションはオンデバイスも選べ、テキストを渡して要約等が可能 — [9to5Mac](https://9to5mac.com/2025/06/16/shortcuts-app-has-chatbot-like-apple-intelligence-powers-in-ios-26/)
- AIで要点整理して1枚のページにまとめLINEで共有し、夫婦の結論が出たという体験談 — [リベシティ記事](https://library.libecity.com/articles/01KY5WB49EE4BQ35ZFG9E0F1FP) (Anecdote)

### Inferences
- 日次フロー(推論): ①通知/トークで気づく ②メッセージ長押し→コピーまたはトーク画面スクショ ③共有シート→ショートカット(2〜3タップ) ④Use Modelで要約+「配偶者に関係あるか」判定 ⑤Telegram送信(自動)。合計約4〜6タップ/件。トークを選択コピー+LINEの転送を使うなら、LLM無しで2〜3タップ(配偶者との個別トークへ転送)で足り、LLMの付加価値は小さい。
- 「全グループを常時監視して漏れなく検知」はこの方式では不可能。見落とし防止の本質は気づく部分で、そこが自動化できない。

### Gaps
- 日本語LINEトークでの on-device モデルの判定精度、ショートカットからのTelegram Bot API呼出しの実例資料は未取得。

## 事例・共有漏れ防止ツール

### Takeaway
「チャットを読まずに共有漏れを減らす」実例は共有リスト/ノート/予定アプリ中心。ショートカット+LLMでLINEを自動監視する実例は見つからなかった。

### Cited Findings
- LINEノートで夫婦/家族の情報共有、共有ノートやスケジュール共有アプリに集約する方法が紹介 — [リベシティ記事](https://library.libecity.com/articles/01JQXX3V1H4S932BVD9SV0WE26), [dスタイル/ichioshi](https://ichioshi.smt.docomo.ne.jp/articles/limited/35456)
- 家事・育児を共同プロジェクト化し予定共有で「言った言わない」を減らす(2026年版比較記事) — 同上
- ショートカットによるLINE自動投稿の「共有漏れ防止」事例は検索で未確認。Famm、Apple共有リマインダー、Notion等の個別評価も本調査では一次資料未取得。

### Inferences
- 「気づいた人が共有リスト(Apple共有リマインダー)に1行入れる」運用は、LINE本文を読まずに済みプライバシー上も軽い。

### Gaps
- Famm等の機能・価格の一次情報(未取得)。

## 比較表: iPhone中心 vs Messaging API Bot

(ToS/プライバシーは一般的見解+推論。Botに関する詳細は別調査参照)

| 項目 | A 通知要約 | B LINE標準 | C OCR/スクショ | D Mac+Mirroring | E 共有シート半自動 | Messaging API Bot |
|---|---|---|---|---|---|---|
| 検知の自動性 | 不可(目視) | メンション依存 | 手動トリガー | 技術的に不安定/未確認 | 手動選別 | 自動(常時) |
| 手間 | 低 | 低(運用依頼) | 中 | 高 | 低〜中(毎回4〜6タップ) | 初期構築のみ |
| 費用 | 0 | 0 | 0 | 既存Mac | 0(Telegram Bot無料) | 無料枠内〜(要確認) |
| 信頼性 | 低 | 中 | 中 | 低 | 中 | 高 |
| グループでの可視性 | なし | なし | なし | なし | なし | Bot参加が見える |
| プライバシー | 端末内 | 端末内 | 端末内/PCC | 端末内 | 端末内 or クラウドLLM | 全メッセージがサーバーへ |
| 規約リスク | 低 | 低 | 低〜中(要確認) | 中〜高(未確認) | 低 | 公式API内なら低(グループ参加の周知が望ましい) |

### Inferences
- 「Botを入れたくない」「無料」「配偶者に伝える」制約では、E(半自動)+B(重要な人にメンション依頼)+共有リマインダー運用が最小コスト。全漏れ防止が必須ならBotだが社会的可視性とグループ同意が必要。
