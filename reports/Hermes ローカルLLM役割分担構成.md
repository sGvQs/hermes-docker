# Kanban と外部スクリプトで回す実装・レビュー逐次パイプライン

**結論から言うと、Hermes 単体には「実装モデル → レビューモデル」を逐次実行する専用機能はなく、モデルを役割ごとに変えられるのは Kanban（タスクごとにプロファイルと `--model` を指定）だけである。** `delegate_task` は子エージェント全員に単一の `delegation.model` を当てる設計で、タスクごとのモデル指定は公式ドキュメント上存在せず、無視されるとの issue も複数ある。推奨は段階構成で、(0) 同一モデル＋テスト実行をゲートにした最小構成、(1) プロファイル 2 つ（implementer / reviewer）と Kanban による別モデル構成、(2) 往復回数と VERDICT 判定をコードで強制する外部スクリプト、の順に拡張する。ドキュメントは「環境の事実と共通契約は AGENTS.md、役割の人格は各プロファイルの SOUL.md（短く）、手順と審査基準はスキル、案件ごとの受け入れ条件はタスク依頼文」に分ける。Ollama 側では `/v1` が `num_ctx` を受け付けないため、`OLLAMA_CONTEXT_LENGTH` か Modelfile エイリアスでコンテキスト長を固定するのが必須で、Qwen3.5 のツール呼び出し不具合（ollama#14493）は未解決のため、実装・レビュー双方でスモークテストを先に通す必要がある。以下、推奨は調査結果からの推論であり、実機検証までは仮説として扱う。

出典の確度は次の記号で示す。[公式] 公式ドキュメント、[issue] GitHub issue（利用者報告）、[論文] 論文・プレプリント、[3rd] サードパーティ記事・リポジトリ、[推論] 調査結果からの筆者推論。なお、公式ドキュメントは要約取得ツール経由で読んでおり、ソースコード（`tools/delegate*` 等）は直接確認していない。

## 実現手段は Kanban が唯一モデルをロール別に固定できる

Hermes のマルチモデル手段を並べると、役割ごとにモデルを変えられる粒度が決定的に異なる。`delegation:` ブロックは `model` / `provider` / `base_url` / `api_key` を持つが、解決順は `base_url` > `provider` > 親から継承で、`model` は常に適用される。**タスクごとのモデルパラメータは無く、一つの `delegation.model` が全子エージェントに適用される**。ドキュメント自身がタスク単位の上書きには Kanban を案内している ([公式 Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation))。一方で別の調査では、同じドキュメントに「サブエージェントごとの `model` パラメータ、省略時は親」という記述があるとの報告もあり、ドキュメント要約の間で食い違いが残る。さらに delegate_task がタスク別モデルを無視する・常に親を継承するという報告が #46880、#49332、#12440、#17685、#10995 にある（個別には未取得）([issue](https://github.com/NousResearch/hermes-agent/issues/17685))。これらを総合すると、delegation でレビュワーのモデルを動的に切り替える運用は当てにできない。

Kanban では `hermes kanban create "..." --assignee coder --model <m> --provider <p>` や `hermes kanban set-model <id> <model> --provider <p>` でタスク単位にモデルを固定でき、指定がなければ担当プロファイルの `model.default` が使われる。ボードは `~/.hermes/kanban.db` に永続化され、親子リンクで `todo` から `ready` へ昇格し、ゲートウェイ内のディスパッチャ（`kanban.dispatch_in_gateway: true`、既定 60 秒間隔）が `hermes -p <assignee>` を別 OS プロセスとして起動する ([公式 Kanban](https://hermes-agent.nousresearch.com/docs/user-guide/features/kanban))。ユーザーの Docker ゲートウェイ構成とそのまま噛み合う点が大きい。ただし、レビュワーワーカーが実装タスクの成果物をどう受け取るか（コメントや handoff summary の意味論）は、ドキュメントが取得時に途中で切れていたため確認できていない。

| 手段 | 長所 | 短所 | 根拠の確度 |
|---|---|---|---|
| delegate_task（`delegation.model`） | 追加構成なし、子は新規コンテキスト、`output_schema` で形式検証、結果は同期で親に戻る | 全子が同一モデル・同一ツールセット、ロール別モデル不可、タスク別 model 無視の報告、Ollama 宛の既知バグ #44207 | [公式] ＋ [issue]、ソース未確認 |
| プロファイル単体 | モデル・SOUL.md・メモリ・スキルがロール別に隔離、`-p` で切替 | 順序付けは別手段が必要、1 ゲートウェイ = 1 プロファイル = 1 bot トークン、同一プロファイルを 2 プロセスで使うのは禁止 | [公式 Profiles](https://hermes-agent.nousresearch.com/docs/user-guide/profiles) |
| Kanban | タスク単位の model 固定、親子依存、永続、失敗上限（`failure_limit` 既定 2）、ゲートウェイ内で動く | 成果物の受け渡し意味論と往復上限の制御は未検証、仕組みが新しく実例が見つからない | [公式] のみ、実例なし |
| cron | 時刻起動、ジョブ別に `--model/--provider` 固定、`--workdir` で AGENTS.md を読む、`deliver` で Telegram へ | 逐次依存の表現が弱い、ジョブ間の結果受け渡しは自前 | [公式 Cron](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) |
| 補助モデル（auxiliary） | `auxiliary.review` は `/review` 用のモデル固定ができる、圧縮・承認など副次処理を小モデルへ | 主パイプラインの役割分担ではない | [公式 Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration) |
| MoA | 並列に複数意見を集約 | 役割の逐次処理ではない、公式未検証の第三者情報のみ | [3rd](https://www.tonyreviewsthings.com/hermes-agent-mixture-of-agents-20/) |
| 外部スクリプト＋API サーバー | 往復上限・VERDICT 解析・ログをコードで強制、レビュワーの文脈を確実に新規化、プロファイルごとにポート分離可 | 自作・保守が必要、API サーバーは端末を含む全ツールが使える | [公式 API server](https://hermes-agent.nousresearch.com/docs/user-guide/features/api-server) ＋ 設計は [推論] |

フォールバックにも注意が要る。サブエージェントはピン留めされていればフォールバックしない仕様で、`fallback_providers` はターン単位で、切替時にプロンプトキャッシュがリセットされる ([公式 Fallback](https://hermes-agent.nousresearch.com/docs/user-guide/features/fallback-providers))。ローカル 3 回失敗でクラウドへ落ちる設定は、「すべてローカル」の前提では無効にしておくのが筋である（推論）。

## 推奨アーキテクチャは三段階で、最初は同一モデルから始める

以下は推論であり、実機で検証して初めて確定する。

**段階 0（最小構成）。** モデル分離をしない。現在の `qwen3.5:35b-a3b` 一台のまま、メインエージェントが実装し、`delegate_task` で新規コンテキストのレビュワーを呼ぶ。子はタスク文と `context` しか見ないので、実装者の自己評価が混入しない。肝はレビュー基準を「テスト・lint・差分の実行結果」に紐づけることで、自己批評だけの内在的な自己修正は改善せず悪化し得るという結果がある ([論文 Huang et al.](https://arxiv.org/pdf/2310.01798))。同一モデルは自己生成物を好む傾向もあり（[論文 Panickssery et al.](https://arxiv.org/pdf/2404.13076)）、小さな事実確認や形式チェックに限れば許容できるが、リスクの高い変更には足りないという実務的見解がある ([3rd](https://codex.danielvaughan.com/2026/03/28/cross-model-adversarial-review/))。この段階の目的は、ドキュメント構成と VERDICT 形式を、スワップのコストなしに固めることにある。

**段階 1（別モデル・Kanban）。** プロファイルを `implementer` と `reviewer` に分け、それぞれの `config.yaml` の `model.default` に別の Ollama モデルを置く。依頼ごとに実装タスクを作り、レビュータスクを親子リンクで後続させる。公式が示す「オーケストレータは強いモデル、ワーカーは安いモデル」の型に沿うが、本件は全員ローカルなので、実装側は現行の 35b-a3b、レビュー側は別系統のモデルを置くのが [推論] 上の最善手である。クロスファミリーの審査は実務家の推奨であり ([3rd](https://towardsdatascience.com/the-llm-judge-that-kept-agreeing-with-itself))、自己選好は生成が弱い評価者で有害になりやすい ([論文 arXiv 2504.03846](https://arxiv.org/pdf/2504.03846v3))。ただしローカル小型モデル同士のクロスレビューを測った研究は見つからなかった。

**段階 2（外部スクリプト）。** 往復上限、VERDICT のコード判定、失敗時のエスカレーションを Hermes の外（ホスト上のシェルや Python）に出す。Anthropic はチェーン間のプログラム的ゲートと反復上限を推奨しており ([公式 Anthropic](https://www.anthropic.com/engineering/building-effective-agents))、小型モデルに「自分で 2 回までやり直せ」と書くよりコードで切るほうが堅い（推論）。呼び出し方は、API サーバー（127.0.0.1:8642、OpenAI 互換、プロファイルごとに別ポート可）か、`docker exec` による CLI のいずれかだが、後者のワンショット実行コマンドは調査で確認できていない。

## ドキュメントは「事実・契約・役割・手順・案件」を別ファイルに分ける

読み込み仕様の基礎（SOUL.md は `HERMES_HOME` のみ、プロジェクト文脈は AGENTS.md ほか最初の一つだけ、AGENTS.md は cwd から読まれる）は既存レポートに整理済みで、本稿では繰り返さない。ただし要点だけ確認しておく。SOUL.md はプロファイルごとの同一スロットで、プロジェクト文脈は `.hermes.md` → `AGENTS.override.md` → `AGENTS.md` → `CLAUDE.md` → `.cursorrules` の最初の一つだけが読まれ、子エージェントのシステムプロンプトにも埋め込まれる ([公式 Context files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files)、[公式 Delegation](https://hermes-agent.nousresearch.com/docs/user-guide/features/delegation))。親の SOUL.md が子に継承されるかは明記がない。AGENTS.md は全ロールに注入されるため、ロール固有の審査基準を書くとトークンの無駄になり役割が混濁する（推論）。

| 内容 | 置き場所 | 理由 |
|---|---|---|
| 役割プロンプト（人格・厳守事項、300〜500 トークン程度） | 各プロファイルの `SOUL.md` | プロファイルごとに隔離される。短く保つ |
| 環境の事実（パス、ポート、テスト実行コマンド、`/opt/data` と `~/.hermes` の対応、Ollama の到達先） | 作業 cwd の `AGENTS.md` | 全ロール・子に自動注入される共通事実 |
| 引き継ぎ形式（実装者の出力様式） | `AGENTS.md` の短い契約節＋スキルの参照ファイル | 両ロールが同じ形式を知る必要がある |
| レビュー基準・VERDICT 形式・エスカレーション規則 | レビュー用スキル（`SKILL.md` ＋ `references/`） | 必要なロールだけがオンデマンドで読む。本文は 24k 文字未満 |
| 案件ごとの受け入れ条件 | タスク依頼文（Kanban タスク本文、delegate の `context`、スクリプトの入力） | 案件で変わる。実装・レビュー双方に同一文を渡す |
| 往復上限・失敗時の停止 | 外部スクリプト（または Kanban の `failure_limit`） | モデルに守らせるより確実 |

スキルは目録が常に約 3k トークンをシステムプロンプトに載せる ([公式 Skills](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills))。ローカルの 64k では許容範囲だが、使わないスキルとツールセットは切っておく。レビュワーには実装者の会話履歴を渡さず、タスク・受け入れ条件・成果物（差分）・テスト出力だけを渡す（推論、Claude Code や OpenAI Agents SDK の入力フィルタ思想と整合）。

出力形式は、厳格な JSON よりも短いラベル付きプレーンテキストと末尾の VERDICT 一行にする。形式制約は小型モデルの推論精度を落とすという報告がある ([論文 Let Me Speak Freely?](https://arxiv.org/html/2408.02442v1)、[論文 Constraint Tax](https://arxiv.org/pdf/2605.26128)、いずれもプレプリント／要約確認のみ)。証拠を書かせてから判定させると自己選好が下がる ([論文 arXiv 2504.03846](https://arxiv.org/pdf/2504.03846v3))。

実装者の引き継ぎ（成果物はファイルに置き、本文には載せない）:

```
## SUMMARY      何をしたか 3 行以内
## CHANGES      変更ファイル一覧（パス）
## HOW_TO_VERIFY 検証コマンドと期待結果
## OPEN_ISSUES  未解決・自信のない点（なければ NONE）
```

レビュー用スキルの核（`~/.hermes/profiles/reviewer/skills/review-pipeline/SKILL.md`）:

```
---
name: review-pipeline
description: 実装の独立レビューと VERDICT 出力
---
## When to Use
実装者の引き継ぎ（SUMMARY/CHANGES/...）を受け取ったとき
## Procedure
1. 受け入れ条件を C1..Cn に分解する
2. HOW_TO_VERIFY のコマンドを自分で実行し、出力を引用する
3. 各条件を `Cn: PASS|FAIL - <根拠>` で書く（根拠なしは FAIL）
4. 最後の行に `VERDICT: PASS` か `VERDICT: FAIL`、FAIL なら `FIXES:` を番号付きで最大 5 件
## Pitfalls
実装者の自己申告を根拠にしない。迷ったら FAIL。
```

`reviewer` の SOUL.md は「あなたは懐疑的な独立レビュワー。承認はテスト実行結果が示したときだけ。コードは書き換えない」程度の数行に留める。コード側は、出力の最後の `VERDICT:` 行を正規表現で取り、欠落・曖昧は FAIL として扱う（フェイルクローズ）。往復は 2 回までとし、同一の FAIL リストが繰り返されたら `STATUS: ESCALATE` で人間（Telegram）に返す。これらの閾値は推論であり、ローカルモデルでの実測根拠はない。

## Ollama はコンテキスト長をサーバー側で固定し、スワップは明示的に制御する

最大の落とし穴は、OpenAI 互換 `/v1` が `num_ctx` を受け付けない点である。公式は「OpenAI API にはコンテキストサイズを設定する方法がない」とし、Modelfile の `PARAMETER num_ctx` で派生モデルを作る回避策を示す ([公式 Ollama OpenAI compat](https://docs.ollama.com/api/openai-compatibility))。Hermes 側は GGUF からコンテキスト長を読むが Ollama が 4096 で動かし続け、出力が `finish_reason="length"` や文字化けになる問題が #43900 で報告され、2026 年 8 月時点でも未解決で、ネイティブ `/api/chat` アダプタの PR #73285 は保留中だった ([issue #43900](https://github.com/NousResearch/hermes-agent/issues/43900))。実用策は、ホストの Ollama に `OLLAMA_CONTEXT_LENGTH` を設定するか、ロール別の Modelfile エイリアス（例 `impl-64k`、`review-32k`）を作ってそれを Hermes に指定し、`ollama ps` で実際のコンテキストを確認することである。ユーザーの画面表示「64k」は Hermes 側の設定値であり、Ollama が実際に確保した値とは限らない（推論）。

スワップの制御変数は 3 つ。`OLLAMA_MAX_LOADED_MODELS` は「メモリに収まる限り」同時ロード数を決め、収まらなければ要求はキューで待機する。`OLLAMA_KEEP_ALIVE` の既定は 5 分で、リクエスト単位の `keep_alive` が優先され、0 で即時アンロード、負値で常駐する。`OLLAMA_NUM_PARALLEL` はコンテキスト分のメモリを並列数倍にする ([公式 Ollama FAQ](https://raw.githubusercontent.com/ollama/ollama/main/docs/faq.mdx))。`/v1` は `keep_alive` をサポート項目に載せていないため、Hermes 経由でリクエスト単位の指定はできず、ホストの環境変数で決めるしかない（`/v1` に `keep_alive` が通るかは未確認）。macOS では `launchctl setenv` で設定してサービスを再起動する必要があり、再起動で消えるうえ、再起動後は最初の要求までロードされない ([3rd](https://dev.to/jovan_chan_9500711396d4e6/ollama-keeps-reloading-the-model-fix-vram-unloading-cold-starts-and-model-swapping-2026-25b7))。

推奨設定は、モデルが 2 つ同時に載るメモリ余裕があるかで分かれる。Mac mini の搭載メモリはこちらでは把握できていない。35b-a3b（Q4 で約 24GB、[3rd](https://codenote.net/ja/posts/hermes-agent-quick-setup-macos-ollama-qwen3-6/)）に加えて別モデルとコンテキスト分の KV キャッシュが収まるなら `MAX_LOADED_MODELS=2`・`KEEP_ALIVE` 長め（Hermes ガイドは 24h 例）で常駐させ、スワップ自体を消す。収まらないなら `MAX_LOADED_MODELS=1`・`NUM_PARALLEL=1` で逐次スワップとし、タスクのまとまりごと（実装を全部、次にレビューを全部）に処理してスワップ回数を減らす。MoE は有効パラメータが小さくても全重みが常駐するため、ロード時間とメモリは総サイズで決まる（一般論）。20〜40GB クラスの Apple Silicon でのロード時間の実測は見つからなかった。Mac mini の内蔵 SSD なら数秒から数十秒という見積もりは推論にすぎず、実測が要る。スワップ中は Hermes 側のタイムアウトに注意し、`HERMES_API_TIMEOUT`（既定 1800 秒）、`child_timeout_seconds`、プロバイダー別 `request_timeout_seconds` を余裕を持たせる ([公式 Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration))。

llama-swap は必須ではない。`model` フィールドで upstream を起動・入替し、TTL、モデルごとの `-c`（コンテキスト長）、同時起動の `matrix` を YAML で持てて `/v1` で到達できるため、`/v1` の num_ctx 無視を根本的に回避できる ([3rd llama-swap README](https://raw.githubusercontent.com/mostlygeek/llama-swap/main/README.md))。ただし Ollama のモデルライブラリを離れ、llama-server 用 GGUF とコマンドを自前管理することになる。Modelfile エイリアスとホスト環境変数で回るうちは Ollama のままにし、エイリアスでも context やスワップ挙動が制御できない場合の移行先と位置づけるのが妥当である（推論）。

## ローカルモデル固有のリスクと、やってはいけないこと

**Qwen3.5 のツール呼び出し（ollama#14493）。** Qwen3.5 は Qwen3-Coder の XML 形式で学習されているのに Ollama が Hermes 形式の JSON のレンダラ／パーサを使い、未閉じの `</think>` が履歴に残ってプロンプトを壊す、思考ブロック内のツール呼び出しが認識されない、などの停止が 2026 年 2 月から Aug-Sep 時点まで報告されている。35B MLX の利用者が qwen3-coder のレンダラ／パーサに替えて成功率が上がったという報告もあるが、メンテナは「完全に動かない」との主張には異議を唱えている ([issue ollama#14493](https://github.com/ollama/ollama/issues/14493))。ZeroClaw でも qwen3.5:9b と 35b-a3b で推論テキストだけが出てツールが実行されない例がある ([issue zeroclaw#3079](https://github.com/zeroclaw-labs/zeroclaw/issues/3079))。ユーザーの主モデルがまさにこれに該当するため、ツールを使う実装役にこのモデルを据える場合は、モデルと Ollama バージョンの組で必ずスモークテストを通す。公式の Hermes ローカルガイドでツール呼び出し「あり」と表に載るのは gemma4:31b のみで、gpt-oss は第三者リポジトリと #43900 の報告で使用例がある ([公式](https://hermes-agent.nousresearch.com/docs/guides/local-ollama-setup)、[3rd](https://github.com/InfiniteWhispers/HermesAgent-MultiModel))。レビュワーはツールを持たず「差分・テスト出力を読んで判定するだけ」にすれば、ツール呼び出し不具合を回避できる（推論）。ただしその場合の「テスト出力」は実装側か外部スクリプトが実行して渡す必要がある。

**/v1 の num_ctx 無視（hermes-agent#43900）。** 前節の通り、サーバー側固定が必須。思考モデルが出力予算を食う場合は `max_tokens` の設定も有効という報告がある。

**delegate_task の model 指定無視。** 前述の複数 issue に加え、Ollama Cloud 宛の delegation では #44207（P3、未解決）が報告され、`provider` と `base_url` を併記すると「custom」扱いになり API キーが落ちて 401 がコンテキスト超過に誤分類される、という仮説と、`delegation.base_url` を外す回避策が示されている ([issue #44207](https://github.com/NousResearch/hermes-agent/issues/44207))。修正 PR #89138 が最新リリースに入ったかは不明。ローカル Ollama では `provider: custom`、`base_url: http://host.docker.internal:11434/v1`、ダミーの `api_key` を使い、名前付きの組み込みプロバイダーを併記しないのが無難である（推論）。コンテナからホストの Ollama へはループバックではなくホスト名で到達させる。

**レビュワーの客観性。** 同一モデル・同一文脈のレビューは自己選好と共有盲点で形骸化しやすい。軽減策は、別モデル（できれば別系統）、新規コンテキスト、テスト実行などの外部信号、根拠を書かせてから判定させる形式、迷ったら FAIL の規則である。しかし**小型ローカルレビュワーのゴム印率を測った事例は見つからなかった**ので、これらは仮説である。30B 未満のモデルはツール利用で行動を実行したと主張しながら実際はしていない、という第三者の指摘もあり ([3rd](https://bestllmfor.com/guides/hermes-agent-local-models/))、レビュワーの「確認した」を鵜呑みにせずコマンド出力の引用を求める。

やってはいけないことは、次のとおり。同一プロファイルを 2 つのプロセス（ゲートウェイとワーカー等）で同時に使うこと。AGENTS.md にロール固有の審査基準を詰め込むこと。レビュワーに実装者の履歴や自己評価を渡すこと。往復上限なしの自動ループ。VERDICT の曖昧・欠落を PASS 扱いにすること。ゲートウェイの Telegram 会話・cron・パイプラインが同じ Ollama を異なるモデル名で叩く構成を放置すること（`MAX_LOADED_MODELS=1` では互いにスワップをぶつけ合う、推論）。`/model` による切替は設定へ永続化される旨が公式にあるため、ロール切替の手段にしないこと（[公式 Providers](https://hermes-agent.nousresearch.com/docs/integrations/providers)）。人間の承認なしにマージや本番反映まで進めること。

## 未確認事項と、実機で確かめる小さな実験計画

調査で見つからなかったものを明示する。Hermes＋Ollama＋Telegram ゲートウェイでの実装・レビュー分担の実例は見つからなかった（Zenn・Qiita・note・Reddit・HN は到達できず）。小型ローカルモデルの最小サイズや、レビュワーとしての最低限の実測も見つからなかった。20〜40GB の GGUF のロード時間（Apple Silicon）も見つからなかった。Kanban の成果物受け渡し、delegate の子が SOUL.md を継承するか、子が親の `terminal.cwd` を継承するか、トークン・コストの子から親への集計、ゲートウェイでのプラットフォーム別・チャット別のモデル上書きは未確認である。#44207 の修正や #43900 のネイティブアダプタが現行版に入ったかも不明である。

実験は次の順で、各段階を通ってから次へ進む。

| 手順 | やること | 合否の見方 |
|---|---|---|
| 1. 実コンテキスト確認 | `OLLAMA_CONTEXT_LENGTH` または Modelfile エイリアスを設定し、Hermes から長めの入力を投げて `ollama ps` を見る | 期待した context、finish_reason が `length` でない |
| 2. ツール呼び出しのスモークテスト | 実装候補モデルごとに、ファイル作成・テスト実行を 1 回ずつ依頼 | ツールが実際に実行され、結果が文面と一致 |
| 3. スワップ時間の実測 | 2 モデルを交互に 5 往復呼び、`MAX_LOADED_MODELS` 1 と 2 でロード時間・`ollama ps` を記録 | 初回トークンまでの時間、タイムアウトの有無 |
| 4. 段階 0 の通し | 同一モデルで、既知のバグを埋めた小課題を実装→テスト→レビュー（delegate） | レビュワーがバグを検出するか、根拠にコマンド出力を引用するか |
| 5. 段階 1 の通し | プロファイル 2 つと Kanban の親子タスクで同じ課題を処理 | レビュータスクが成果物を受け取れるか、`--model` が実際に効くか（`ollama ps` で確認） |
| 6. 客観性の確認 | 意図的に誤った実装を 5 件用意してレビュワーに通す | 見逃し率。同一モデルと別モデルを比較 |
| 7. 異常系 | 常に FAIL となる課題で往復上限・ESCALATE が働くか確認 | 無限ループしない、Telegram に結果が戻る |

## まとめ：構成の優先順位

本調査の新しい示唆は二つある。第一に、「どのモデルを使うか」より先に「何をコードで強制するか」が信頼性を決める。Hermes の仕組み上、ロール別モデルは Kanban 以外で確実に固定できず、往復上限・判定・停止は外部スクリプトかボードの失敗上限に出すのが堅い。第二に、ユーザーの現行主モデル（Qwen3.5 系）はツール呼び出しの未解決問題を抱えているため、パイプライン設計の前に、ツールを持たない純テキストのレビュワー役として使うか、実装役を別系統に替えるかを、段階 1 の実験で先に決めるべきである。いずれも推論に基づく設計案であり、実機検証が済むまでは暫定である。
