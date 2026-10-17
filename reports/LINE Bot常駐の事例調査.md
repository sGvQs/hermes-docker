# LINE Bot常駐は例が薄く、技術的には成立する

**結論から言うと、「LINEグループに受信専用Botを常駐させ、自宅のローカルLLMで判定し、特定の1人にだけ別経路（Telegram）で通知する」構成の完成事例は、今回の調査では見つからなかった。** 部品ごとの実例はある。グループ全発言を受けて保存・要約するOSS（kkdai/LINE-Bot-ChatSummarizer、waipod5142/line_tripreport）、ローカルLLM（Ollama等）で要約して個人に届けるTelegram/Discord向けOSS（ggghbc/telegram-summary-bot、sboy99/cortex）、WhatsAppで全発言をエージェントに流し価値がなければ無返信にするOpenClawの設計である。ただしLINE×ローカルLLM×自宅サーバーの組み合わせ、およびキーワード型・配偶者向けの通知の実例は見つからなかった。技術面では、LINE公式仕様の範囲で実現できる。Botはグループの全発言をwebhookで受け取れ、返信しなければ無料枠（月200通）を消費しない。一方で、Botはグループ内に公式アカウントとして存在する。メンバーからの見え方、周知、取消イベントの扱いを運用で補う必要がある。後藤さんのケースでは、段階0（手動エクスポート）か共有リマインダーで効果を確かめ、必要が確認できてからBot方式に進むのが現実的と考える。これは推論であり、実機検証までは仮説として扱ってほしい。出典の確度は [公式]（LINE/Hermesの公式文書）、[ソース]（リポジトリ）、[3rd]（第三者記事）、[推論]（筆者の推論）で区別する。公式文書は要約取得ツール経由で読んでいるため、言い回しは正確でない可能性がある。検索は日本語に弱く、網羅性は限定的である。

## 最も近い事例は部品ごとにしか存在しない

調査で見つかった事例を、後藤さんの構成との距離が分かるように並べる。

| 事例 | 構成 | LLM | ホスティング | 受信専用か | 記憶保持 | 費用 | 日付 | 確度 |
|---|---|---|---|---|---|---|---|---|
| kkdai/LINE-Bot-ChatSummarizer | LINEグループ発言を保持、`:sum_all`で要約を送信者に非公開で返す | OpenAI | Heroku/Render | 要求時のみ返答（常時無返信ではない） | 既定メモリ、`DATABASE_URL`でPostgreSQL | 記載なし | 2022-12-29の元記事 | [3rd] 実コード・README |
| waipod5142/line_tripreport | LINE→署名検証・冪等webhook→Postgres(Supabase)→グループ別受信箱→CSV | なし（要約機能は2026-08-17に削除） | Vercel+Supabase | ほぼ受信専用。新規グループは`pending`で管理者承認まで破棄 | 保持期間の記載なし | 無料枠構成と読める | 2026年 | [3rd] 実コード |
| ikechi氏（Zenn） | 家族・友人グループ向け。メンションでのみ応答、朝のCron通知 | Gemini 2.5 Flash | Cloudflare Workers+Hono+KV | 発言は保存せず。メンション時のみ応答 | KVに誕生日等のみ | 月$1未満見込み | 2026-03-31 | [3rd] 記事 |
| noir氏（note） | Botなし。「トーク履歴を送信」のテキストをAIに渡して要約 | 汎用AI | なし | 手動 | なし | 無料 | 2026-05-21 | [3rd] 記事 |
| ggghbc/telegram-summary-bot | Telegramグループ履歴をSQLite(WAL)に保存し要約 | OpenAI互換API（Ollama等も可） | 自前（systemd/Docker） | 未確認 | SQLite、最大3,000件を要約 | 記載なし | 時期未確認 | [3rd] README |
| sboy99/cortex | Discordのチャンネル発言を集め日次/週次でDMダイジェスト | Ollama | 自前 | 未確認 | SQLite、古いデータ自動削除 | 記載なし | 時期未確認 | [3rd] 検索要約のみ・未精査 |
| OpenClaw WhatsApp | 個人アカウントにリンクし、`always`モードで全発言をエージェントへ、不要ならNO_REPLY | 任意 | 自前 | 事実上の聴取（無返信を選べる） | 直近約50件を文脈に | 記載なし | 時期未確認 | [3rd]/公式docs |

出典は順に [kkdai](https://github.com/kkdai/LINE-Bot-ChatSummarizer)、[元記事](https://dev.to/evanlin/line-bot-group-chat-summary-generator-3h8k)、[line_tripreport](https://github.com/waipod5142/line_tripreport)、[ikechi氏](https://zenn.dev/penpeen/articles/a7b7b8e2487950)、[noir氏](https://note.com/noir_0910/n/n0a448a6a38c5?hl=en)、[telegram-summary-bot](https://github.com/ggghbc/telegram-summary-bot)、[cortex](https://github.com/sboy99/cortex)、[OpenClaw docs](https://docs.openclaw.ai/channels/group-messages)、[第三者ガイド](https://lobster.shahine.com/guides/whatsapp-channel/) である。

**最も近いのは、LINEでは line_tripreport、ローカルLLMでは telegram-summary-bot と cortex である。** 前者は署名検証・冪等性・承認制という、受信専用Botに必要な骨格を備える。後者はOpenAI互換のローカル端点でデータを自前環境に留める発想を示す。一致しない点は三つある。第一に、いずれも「特定の1人に共有すべきか」という宛先別の判定をしていない。第二に、LINEとローカルLLMの組み合わせ、およびHermesのlisten-only運用の例はない。第三に、誤判定率や通知疲れの実測報告もない。

各事例から学べることは次のとおりである。kkdai版の元記事は、ChatGPT呼び出しが遅いため先に応答を返してから処理する設計を勧める。これは「webhookは即200、判定は非同期」という方針と一致する。LIFFからのgroupID取得は2022-12-27以降できずwebhookのみ、という制約も記されている ([DEV Community](https://dev.to/evanlin/line-bot-group-chat-summary-generator-3h8k))。line_tripreportは「Botが追加されただけでは取り込まない」承認制で、同意の観点で参考になる。ikechi氏の構成は無料枠で運用でき、外部要約を使うなら月1ドル未満という見積もりを示すが、発言は保存しておらず、後藤さんの要件（全発言の判定）とは異なる。

**見つからなかったもの。** LINE→Slack/Discord/Notionへの転送の日本語事例、町内会・PTA・サークル向け要約Botの個別事例、LINE+Ollama/llama.cppの事例、LINE+自宅サーバーの事例、キーワードアラート型・配偶者向け通知の事例、商用グループ要約サービスの開示事項、Hermesのlisten-only運用事例、Reddit/HNの議論、誤検知率の実測は、いずれも見つからなかった。これは存在しない証拠ではなく、検索の薄さを示すにすぎない。なお、LINE公式の「AIチャットボット(β)」は1対1の自動応答でグループ要約ではない ([LINEヤフー](https://www.lycbiz.com/jp/news/line-official-account/20251113/))。

## 受信専用Botの構成は薄い受信スクリプトで組む

### LINE側の設定と費用

個人でもMessaging APIチャネルを無料のコミュニケーションプランで作れる。Japanの無料プランは月200通までで、数えられるのはpush/multicast/broadcast/narrowcastであり、reply（応答メッセージ）は数えない。数え方は受信者ごとで、200人のグループへのpush 1回は200通になる ([公式](https://developers.line.biz/en/docs/messaging-api/pricing/index.html.md))。ただし価格ページはwebhook受信そのものの課金に触れておらず、「無料」と明記されているわけではない。受信専用Botは何も送らないため、**無料枠で足りる**と読める [推論]。通知はHermes経由のTelegramで行うので、LINEのpushは使わない。

設定項目は次のとおりである。まず「あいさつメッセージ」と「応答メッセージ」を無効にする（作成時は既定で有効）([公式](https://developers.line.biz/en/docs/messaging-api/building-bot/index.html.md))。次に、Developers Consoleのチャネルで「Allow bot to join group chats」（日本語UIの正確な表記は確認できていない）を有効にする。既定は無効である ([公式](https://developers.line.biz/en/docs/messaging-api/group-chats/index.html.md))。1つのトークに入れる公式アカウントは1つだけなので、すでに別の公式アカウントがいるグループには入れない。応答メッセージの設定がグループにも効くかは文書で確認できず、実機で確かめる [推論]。無返信の担保は、設定の無効化に加えて、コードでreply/pushを一切呼ばないことである。

### webhookの到達方法の比較

webhook URLはHTTPSで、広く信頼されたCAの証明書が必要であり、自己署名は不可である ([公式](https://developers.line.biz/en/docs/messaging-api/building-bot/index.html.md))。選択肢は次のとおりである。

| 方式 | 利点 | 欠点 | 確度 |
|---|---|---|---|
| Cloudflare名前付きTunnel→Mac mini | 固定URL。Hermes docも固定トンネルを推奨 | ドメイン取得が要る。Mac停止中はLINEが非2xxを受け、再送はベストエフォートで取りこぼしうる | [公式]+[推論] |
| Cloudflare quick tunnel | アカウント・ドメイン不要 | 起動ごとにURLが変わり、SLAなし（検証用） ([Cloudflare](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/trycloudflare/)) | [公式] |
| サーバーレス中継（Workers+KV/D1、Vercel、GAS）+Macがポーリング | Macへの着信経路が不要。Mac停止中も蓄積できる | 無料枠の現行条件は未確認 | [推論] |

Tailscale Funnel、ngrok無料枠は調査していない。**取りこぼしを避けたいなら、サーバーレス中継が最も堅い**と考える [推論]。ikechi氏（Workers）やline_tripreport（Vercel）が実際に同種の構成を無料枠で動かしている点が傍証になる。

### 署名検証・重複排除・時刻・送信者名・取消

署名は`x-line-signature`ヘッダに入り、チャネルシークレットでリクエストの生ボディをHMAC-SHA256しBase64化した値と比較する。処理前に検証する ([公式](https://developers.line.biz/en/reference/messaging-api/index.html.md))。再serializeしたJSONではなく生バイトを使い、定数時間比較を使う [推論]。サーバーは空の`events`配列を含めて200を返し、処理は非同期にする。再送（Webhook redelivery）は既定で無効、有効化しても回数・間隔は非公開で保証なしである ([公式](https://developers.line.biz/en/docs/messaging-api/receiving-messages/))。重複排除は`webhookEventId`（ULID）で行い、`deliveryContext.isRedelivery`で再送を見分ける。再送時もIDは同じで、`timestamp`は元の時刻を保つ。

`timestamp`はUNIXミリ秒で、JSTへの変換は自前で行う。グループ発言の`userId`はiOS/Android版LINEのユーザーのメッセージイベントにのみ入る。表示名は`GET /v2/bot/group/{groupId}/member/{userId}`で取得でき、友だち追加やブロックの有無は問わない。**ただしユーザーの同意が必要で、未同意者はプロフィール取得から外れる** ([公式](https://developers.line.biz/en/docs/messaging-api/user-consent/))。LINEのヘルプによれば、未認証アカウントは「グループで発言した人」しか内部IDを見られない ([LINE OA help](https://help2.line.me/official_account/web/categoryId/20006335/3/pc?lang=en))。後藤さんの場合、「妻がいないグループ」は固定リストなので、メンバー一覧APIは不要という前提が効く。送信者名の取得は、発言した人であればまず問題にならない見込みである [推論]。

テキストはwebhookでしか取れず再取得APIはない。画像等は「一定期間後に削除」され、期間は非公開である。取消は`unsend`イベント（`unsend.messageId`）で届き、公式は「細心の注意をもって」取り扱い、保存済みの該当メッセージを以後使えないようにするよう求める ([公式](https://developers.line.biz/en/docs/messaging-api/receiving-messages/))。したがってログの各行に`messageId`を持たせ、`unsend`で削除・墨消しするのが筋である [推論]。過去ログの取得APIは見つからず、Bot導入前の発言は取れない前提で設計する。

### Hermesとの関係

Hermes同梱のLINEプラグイン（既定ポート8646、`/line/webhook`）には受信専用モードがない。`LINE_ALLOWED_GROUPS`に入れたグループの全発言にエージェントが応答する設計である ([ソース](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/plugins/platforms/line/adapter.py)、[公式](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/user-guide/messaging/line.md))。Hermes docでも、無返信用の設定は確認できなかった。このためプラグインは使わず、外部の薄い受信スクリプトが署名検証・重複排除・保存を行い、判定は別段で行う設計とする。Hermesの汎用webhook（8644、`/webhooks/<route>`）はGitHub、GitLab、Standard Webhooks、Generic V2の署名方式のみで、LINEの`X-Line-Signature`は扱わない ([ソース](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/website/docs/user-guide/webhooks.md))。したがって受信スクリプトがLINE署名を検証した後、Hermes側の署名方式で再署名して転送するか、保存したJSONLをcronで読ませる方式になる [推論]。`deliver: log`のルートがLLMを起動するかは確認できていない。前報告書の「前処理で絞り、残りが0件ならLLMを呼ばない」設計（`wakeAgent: false`）がそのまま使える。

## Botは隠れた存在ではなく、周知と同意が運用の中心になる

### 見え方で確認できたこと・できなかったこと

確認できたのは、Botがユーザーの招待操作でグループに入ること、Join/Leave/MemberJoin/MemberLeaveの各イベントが存在すること、Bot側から退出できることである ([公式](https://developers.line.biz/en/docs/messaging-api/receiving-messages/))。確認できなかったのは、メンバー一覧への表示形式、参加時の通知文、メンバーによる削除の可否、ログ取得中を示す表示の有無である。**公式文書にはこれらの記載がなく、実機でしか確かめられない。** 「ログ取得を知らせる表示はプラットフォームにない」という推論は、文書に記載がないことに基づく [推論]。招待は人が行うので、他のメンバーが無断でBotを入れられる経路は2020年のCLOVA関連の事案のような例外を除いて想定しにくい ([LINEヤフー](https://www.lycorp.co.jp/ja/privacy-security/archive/L/linecorp_com_ja_security_article_349.PDF)、PDF本文は未取得で検索要約に基づく)。

### 周知・同意と日本での作法

調査で見つけた事例（kkdai、ikechi氏、noir氏）はいずれも同意・周知に言及せず、メンバーの反応や反対を報告した記事は見つからなかった。**日本での作法を示す資料は見つからなかった**。周辺情報として、LINEへの無断追加（詐欺文脈）が不安の対象として扱われている ([All About](https://allabout.co.jp/gm/gc/497696/))。これに基づく以下は推論であり根拠資料はない。全発言を読むBotを黙って入れると、同種の不信を招きうる。事前に「何のために、どの発言を、どこまで保存し、誰に見せるか」を一言で伝えるのが無難である。最小案は、グループ名のメモやノートに「家族への共有漏れ防止のため、ログ判定Botを入れる。保存は当日分のみ」と書くことである。line_tripreportの承認制は、取り込みを明示操作にする点で参考になる。

### 法的注意（断定しない）

PPCのQ&Aでは、個人情報データベース等を事業の用に供していれば、規模や営利性を問わず個人情報取扱事業者に当たりうる ([PPC](https://www.ppc.go.jp/personalinfo/faq/APPI_QA/))。夫婦・友人間の私的利用は事業性がなく対象外となる可能性が高いと考えるが、個人的利用の適用除外部分は未読であり、これは推論である。通信の秘密（電気通信事業法）は通信事業者を名宛人とするため、参加者が受信内容を記録する行為とは位相が異なると一般には考えられるが、一次情報は未確認である [推論]。LINE利用規約には、BOT等による不正な操作(15(5))、第三者の個人情報の不正な収集・蓄積(15(8))を禁じる条項がある ([公式規約](https://terms.line.me/line_terms/?lang=ja))。公式のMessaging APIの利用がこれに当たるとは通常考えにくいが、保存と共有の範囲は絞るべきである [推論]。公式文書はグループ内ログの保持・同意について記載していない ([公式](https://developers.line.biz/ja/docs/messaging-api/group-chats/))。クラウドLLMへ送る場合は提供元の保持・学習ポリシーも確認が要る [推論]。ローカルLLMに限れば第三者提供は避けられる [推論]。この節は法的助言ではない。

## 後藤さんの課題には段階0と共有リマインダーが先に来る

後藤さんの課題は、「自分だけが知っている情報」が友人グループで話題になったとき、妻への共有が漏れることである。見つけたい対象は、日程・招待・頼まれごと等、限られた種類の発言と推測される。この前提で、各手段を比較する。

| 手段 | 費用 | 自動度 | 他メンバーへの見え方 | 規約リスク | 主な弱点 |
|---|---|---|---|---|---|
| 共有リマインダー／手動転送テンプレ（長押し→共有リマインダー/Keep） | 0円 | 低（自分が気づいた時のみ） | 見えない | なし | 「そもそも気づかない」場合は防げない [推論] |
| 段階0: 手動エクスポート＋ローカルLLM（PC版「トークを保存」、txt、写真等は含まれない ([3rd](https://minto.tech/line-chat-history-export-pc/))） | 0円 | 中（取得が手動） | 見えない | 低 | 毎日操作が続くか。メンバー一覧は含まれないが、固定リストで足りる |
| Bot常駐（自前受信+中継+ローカルLLM） | ほぼ0円（ドメイン代の可能性） | 高 | 見える（実機確認要） | 低〜中（周知が前提） | 周知・同意の手間、取消対応、運用保守 |
| 非公式ブリッジ・通知ミラー・画面読み取り | 0円〜 | 中〜高 | 見えない | グレー（前報告書参照） | 本レポートでは推奨しない |

推奨順位は次のとおりである（推論）。第一に、共有リマインダーと手動転送のテンプレ運用を、今日から無料で始める。これだけで「気づけた発言」の漏れは減らせる。第二に、段階0で、ローカルLLMが「妻に共有すべき」を実際の1〜2週間分でどこまで当てるか、通知がうるさすぎないかを測る。判断品質が使えるかを、リスクなしで先に知れる。第三に、それでも「そもそも自分が気づけない」発言を拾いたい場合に限り、Bot方式をテスト用グループで検証し、実運用は周知後に導入する。Botの技術的な難易度より、「取得の自動化が判断の質に見合うか」が分岐点になる。

## 小さな検証手順と確認したい質問

検証はテスト用グループ（後藤さんと別アカウント1つ）で行う。

| 手順 | 確かめること |
|---|---|
| 1. 公式アカウントを作成し、あいさつ・応答メッセージを無効化、グループ参加を許可 | 設定の日本語ラベル、応答メッセージ設定がグループに効くか |
| 2. 受信スクリプト（署名検証、生イベントをJSONLに追記、返信しない）を、quick tunnelまたはサーバーレス中継で立てる | メンション無しの全発言が届くか、`timestamp`・`userId`の中身 |
| 3. Botをグループに招待し、別アカウントのスマホで表示を見る | メンバー一覧での見え方、参加通知の文面、メンバーによる削除の可否 |
| 4. 発言の取消・編集を行う | `unsend`/編集イベントと、JSONLからの削除処理 |
| 5. 同意していないアカウントで発言させ、プロフィールAPIを呼ぶ | 表示名が取れるか（取れない場合の代替表示） |
| 6. Mac mini停止中に数通送る | 再送の有無、取りこぼし（中継方式の要否） |
| 7. 段階0のデータ1日分に正解ラベルを付け、ローカルLLMと比較 | 判定精度、出力形式の遵守、通知の頻度 |

後藤さんには次を確認したい。まず、対象グループは何個で、「妻がいないグループ」は何個か。次に、どのような発言を「共有すべき」と考えるか（具体例を3〜5件）。見逃しと誤通知のどちらが痛いか。グループのメンバーにBotの存在を説明できるか、断られた場合に手動方式で十分か。PC版LINEは使えるか（段階0に必要）。Mac miniは常時稼働か（中継方式の選択に影響）。独自ドメインを持っているか、持たない場合に費用をかけられるか。

## 結論

今回の調査で最も意外だったのは、技術的な壁が思ったより低く、むしろ「例がない」こと自体が主な情報だという点である。受信専用のBotは仕様上成立し、費用もほぼ0円で済むが、日本のLINEグループに全発言を読むBotを入れた体験談や作法は見つからなかった。つまり、後藤さんが得られる事前知識は技術に偏っており、メンバーの反応は自分の環境で確かめるしかない。

したがって、決定の順序は技術→運用の順ではなく、「効果→運用の許容→技術」の順がよい。共有リマインダーで漏れの原因が「気づかなかった」のか「忘れた」のかを見分け、段階0で判定の質を測れば、Bot方式の必要性は低コストで判断できる。ここでの推奨はすべて推論であり、実機検証と、後藤さん自身の優先度によって変わりうる。
