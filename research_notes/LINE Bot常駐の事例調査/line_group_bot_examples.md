# LINEグループ常駐Bot（受信・要約・転送）の実例調査

調査日: 2026-10-12。検索エンジンは日本語に弱く、約14回の検索/取得で網羅性は限定的。記載のないものは「見つからなかった」ことを意味する。

## 1. 実在する事例（OSS・記事）と構成

### Takeaway
LINEグループの全発言を受けて保存・要約するOSSは少数見つかった（Go製の要約Bot、Next.js+Supabaseのログ保存アプリ）。個人ブログ系は「メンション時だけ返答する会話Bot」が主流で、受動的な全文ログ+要約+他サービス転送を一括で扱う日本語記事は見つからなかった。

### Cited Findings
- kkdai/LINE-Bot-ChatSummarizer: Go製、グループの発言を保持し `:sum_all` で要約を「送信者に非公開で」返す。Heroku/Render用設定あり、保存は既定でメモリ（`DATABASE_URL` 設定でPostgreSQL）、再起動で消える旨をREADMEが明記。LLMはOpenAI。Apache-2.0、67 stars。READMEにプライバシー/同意への言及なし — [GitHub](https://github.com/kkdai/LINE-Bot-ChatSummarizer)
- 上記の元記事（著者はLINE台湾のEvan Lin、2022-12-29付）: ChatGPT呼び出しは遅いので先に返信してから待つ設計を推奨、davinci001は品質が悪い、プロンプトで言語指定が必要、LIFFからのgroupID取得は2022-12-27以降不可でwebhookのみ — [DEV Community](https://dev.to/evanlin/line-bot-group-chat-summary-generator-3h8k)
- waipod5142/line_tripreport: LINEグループ→署名検証済み・冪等なwebhook→Postgres(Supabase)+非公開ストレージ→グループ別受信箱→CSVエクスポート。Vercel上で稼働。新規グループは `pending` で、管理者が承認するまでメッセージは破棄（生イベントのみ保持）。「追加されただけではチャットを取り込まない」設計。要約機能（AIパイプライン）は2026-08-17に削除。保持期間の記載なし — [GitHub](https://github.com/waipod5142/line_tripreport)
- UnifiedMessageRelay (PyPI): LINE/Telegram/Discord/QQ間でメッセージを中継するPythonフレームワーク（Slackは非対応と記載）。保守状況は未確認 — [PyPI](https://pypi.org/project/unified-message-relay/)
- ikechi氏（Zenn, 2026-03-31）: 家族・友人グループ向け。Cloudflare Workers+Hono（署名検証）、メンション時のみGemini 2.5 Flash呼び出し、誕生日等はCloudflare KV、Cron Triggersで朝通知。無料枠で月$1未満の見込み。発言の保存はせず、Bot導入の周知・同意には触れていない。「正確さだけがBOTの価値ではない」 — [Zenn](https://zenn.dev/penpeen/articles/a7b7b8e2487950)
- note（ノワール, 2026-05-21）: Botではなく、LINEの標準機能「トーク履歴を送信」で書き出したテキストをAIに渡して要約/議事録代わりにする手動方式。プライバシー言及なし — [note](https://note.com/noir_0910/n/n0a448a6a38c5?hl=en)
- GAS+スプレッドシートでのLINE受信メッセージ保存は複数のZenn/Classmethod記事が存在（例: Webhook設定でGASをWebアプリ化、「応答メッセージ」をOFF）。ただし多くは1対1向けで、グループ全発言の保存を扱うかは本文未確認 — [Zenn](https://zenn.dev/hizumen/articles/afd29f92b62762), [Classmethod](https://dev.classmethod.jp/articles/gas-line/)
- AWS Lambda+DynamoDBで文脈を保存するChatGPT LINEボット（1対1中心） — [Classmethod](https://dev.classmethod.jp/articles/chatgpt-api-line-bot-aws-serverless/)

### Inferences
- 「受信専用で要約を特定の人にだけ届ける」構成は、kkdai版の「要約を送信者に非公開で返す」と line_tripreport の「承認制+Web閲覧」が最も近い。
- 全発言を取るならメンション制ではなくwebhookで全イベントを受け、reply/pushを使わない設計になる（技術的には成立する、下記2参照）。

### Gaps
- LINE→Slack/Discord/Notion転送の具体的な日本語ブログ・OSS事例は見つからなかった（検索結果に出ず）。
- 町内会・PTA・サークル向け要約Botの個別事例は見つからなかった。
- Reddit/Hacker News/Stack Overflowは未調査。

## 2. 技術的扱い（webhook・署名・表示名・無返信・コスト）

### Takeaway
Botはグループ内の発言をwebhookで受信できる。表示名取得は本人の同意が必要。無料枠で問題になるのはpush送信であり、reply/受信は数えられない。

### Cited Findings
- グループ/複数人トークでもBotはwebhookで1対1と同様にメッセージイベントを受け取り、`source`に`groupId`/`roomId`が入る。コンソールで「グループトーク・複数人トークへの参加を許可」を有効にする必要があり既定は無効。1つのトークに入れる公式アカウントは1つだけ — [LINE Developers](https://developers.line.biz/en/docs/messaging-api/group-chats/)
- プロフィール取得はユーザーの同意が必要。同意していないユーザーは「グループメンバーのユーザーID取得」結果から除外され、メンバー参加イベントも飛ばない。退出者のプロフィールは取得不可 — [LINE Developers](https://developers.line.biz/en/docs/messaging-api/user-consent/)
- 無料のコミュニケーションプランは月200通まで。課金対象はpush/multicast/broadcast/narrowcastで、reply（応答メッセージ）は通数に含まれない。グループ宛て送信の数え方は確認できず — [ligla](https://ligla.jp/blog/line-official/messagenumber/)（二次情報、2026年時点との記載）
- 署名検証（`x-line-signature`）はLINE公式が推奨 — [LINE Developers](https://developers.line.biz/en/docs/partner-docs/development-guidelines/index.html.md)。実際にHono/Workers、Vercelの事例で実装されている（上記）。
- ホスティング無料枠の実例: Cloudflare Workers+KV（上記ikechi氏）、Vercel+Supabase（line_tripreport）、Heroku/Render（kkdai）、GAS（多数）。
- Cloudflare R2を使ったLINE Bot構成を試すZennスクラップあり — [Zenn scrap](https://zenn.dev/ta1kt0me/scraps/29139e47114489)（内容は検索要約のみ、未精査）

### Inferences
- 受信専用（reply tokenを使わない）運用は仕様上問題なく、通数も消費しないが、要約を通知するpushは月200通枠内に収める必要がある。通知先が個人1:1ならpushは人数分カウント。
- 過去ログの取得APIは見つからなかった（公式に「無い」と明記した資料は未確認）ため、導入以前の発言は拾えない前提で設計すべき。

### Gaps
- 保存メッセージの保持期間・画像等のコンテンツ取得期限の公式記載は取得できなかった。
- 規約上「会話を保存するBot」を制限する記述は見つからなかった。LINEの制限報告も見つからなかった。

## 3. 社会的・倫理的側面

### Takeaway
調べた範囲で、グループ内への周知・同意・反発を報告した著者はほぼいない。唯一、設計で配慮した例は承認制（allowlist）のline_tripreport。

### Cited Findings
- kkdai、ikechi氏、noir氏の記事はいずれも同意・周知・プライバシーに言及しない（上記各記事）。
- line_tripreportは、グループを管理者承認するまで発言を保存しない設計で、取り込みを明示的な操作にしている — [GitHub](https://github.com/waipod5142/line_tripreport)
- 2020年11月、LINEのCLOVA専用アカウントの仕様が悪用され大量のグループ招待が発生した事案がある（Bot側が勝手に入る経路への警戒材料。PDF本文は取得できずWebSearchの要約に基づく） — [LINEヤフー](https://www.lycorp.co.jp/ja/privacy-security/archive/L/linecorp_com_ja_security_article_349.PDF)
- 同意なくメンバーのプロフィールが取れない仕様自体が、プラットフォーム側のプライバシー配慮（上記2）。

### Inferences
- Botはグループのメンバー一覧に表示されるため、存在自体は隠せない。事前に告知する運用が現実的。

### Gaps
- 「Botが入っていることを周知」した体験談、メンバーの反対の報告、LINEによるログ取得Botの制限報告は見つからなかった（未発見であり、存在しないとは言えない）。

## 4. ローカルLLM・自宅サーバー

### Takeaway
LINE向けでOllama/llama.cppを使う事例は見つからなかった。近い設計はTelegram/Discord向けにある。

### Cited Findings
- ggghbc/telegram-summary-bot: C++20、SQLiteにWALで履歴を保存、OpenAI互換APIなら何でも可（Ollama、vLLM、Gemini等を例示）、最大3,000件を要約、systemd/Docker対応。ローカル端点ならデータは自前環境に留まると読める（明記ではない） — [GitHub](https://github.com/ggghbc/telegram-summary-bot)
- sboy99/cortex: Discordのチャンネル発言を集めOllamaで要約し日次/週次でDM、SQLite保存、古いデータを自動削除（検索要約のみ・未精査） — [GitHub](https://github.com/sboy99/cortex)

### Inferences
- LINEはwebhookに公開URLが必須のため、自宅サーバーならTunnel（Cloudflare Tunnel等）が要る。この構成のLINE実例は未発見。

### Gaps
- LINE+Ollama/llama.cpp、LINE+自宅サーバーの具体例は未発見。

## 5. 「家族が知るべき情報が出たら通知」に近い事例

### Takeaway
該当事例は見つからなかった。近い部品はあるが、組み合わせた実例は未発見。

### Cited Findings
- 近い部品: 要約を送信者に非公開で返す（kkdai）、承認制ログ+受信箱（line_tripreport）、Cron+Gemini+pushで定期通知（ikechi氏・Qiita/note等の朝通知Bot）。
- 商用: LINE公式の「AIチャットボット(β)」は1対1のQ&A自動応答で、グループ会話の要約機能ではない（有料のチャットProオプション月3,000円税別、2025-11-13発表） — [LINEヤフー](https://www.lycbiz.com/jp/news/line-official-account/20251113/)
- 商用のグループ要約Botや開示内容（Product Hunt等）は検索では見つからなかった。

### Gaps
- キーワードアラート型・配偶者向け通知の実例、商用グループ要約サービスの開示事項は未発見。
