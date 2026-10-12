# 環境の事実は AGENTS.md と SOUL.md に書く

**結論: 「Docker コンテナ内で動き、docker コマンドは使えない」という事実は、`terminal.cwd` が指すディレクトリの `AGENTS.md`(または `.hermes.md`)に短い「事実＋代替行動」の形で書くのが本筋であり、cwd 解決の不確実性に備えて SOUL.md にも 2〜3 行の保険を置くのが現実的な最適解である。** Hermes はプロジェクト文脈ファイルを 1 種類だけ(先勝ち)読み、探索の起点は HERMES_HOME ではなく作業ディレクトリである一方、SOUL.md は HERMES_HOME 直下からのみ常に読まれる。現状の `~/.hermes/AGENTS.md`(空ディレクトリ)は二重に無効である。ディレクトリである上に、仮にファイルでも `/opt/data` が cwd でなければ探索対象にならない。Hermes が自分の環境を知らない原因は、この「置いた場所と読まれる場所のずれ」である可能性が高い(推論)。反映の確認は Telegram 上の `/context` が最も確実で、編集後は `/new` で新セッションを開始する。Docker Desktop 固有では、WAL による state.db 破損リスクと Ollama の既定コンテキスト長(4096 の報告)によるシステムプロンプトの先頭切り捨てが、「指示を無視する」ように見える別原因になり得る。なお、公式ドキュメントが明示していない点(SOUL.md/AGENTS.md の再読込タイミング、ゲートウェイの cwd 解決の詳細)が複数残っており、第 5 節に未確認事項として列挙した。

出典ラベルは次のとおり。**[公式]**=公式ドキュメント、**[ソース]**=リポジトリのコード、**[issue]**=GitHub issue、**[コミュニティ]**=第三者記事、**[推論]**=調査者の推論。公式ドキュメントはいずれも要約モデル経由で取得しており、逐語引用ではなくパラフレーズ水準である。

## 各ファイルは読み込み元も条件も異なる

Hermes のシステムプロンプトは、安定層(SOUL.md=identity、ツール/モデル指針)、文脈層(プロジェクト文脈ファイル、git スナップショット、platform hints)、揮発層(スキル索引、MEMORY/USER スナップショット、日時、`# Hermes runtime environment` ブロック)の順で組み立てられ、1 度構築したらキャッシュされる ([公式: Prompt assembly](https://hermes-agent.nousresearch.com/docs/developer-guide/prompt-assembly))。ただし同ページはスキルや platform_hints の層について記述が揺れている([公式]の内部不整合)。以下が早見表である。

| 場所 | 誰が書くか | 読み込み元と条件 | 上限 | 反映タイミング | 確度 |
|---|---|---|---|---|---|
| `$HERMES_HOME/SOUL.md`(Docker では `/opt/data/SOUL.md`) | 人 | HERMES_HOME 直下のみ。cwd では探索しない。空・読めない場合は既定の identity にフォールバック。サブエージェントでは読まない | 文脈ファイルと同じ cap | 次セッション(推論) | [公式: Personality](https://hermes-agent.nousresearch.com/docs/user-guide/features/personality) |
| `.hermes.md` / `HERMES.md` | 人 | cwd から git ルートまで上方探索(git ルートなしなら cwd のみ)。YAML frontmatter は除去。最優先 | 同上 | 次セッション(推論) | [ソース: prompt_builder.py](https://github.com/NousResearch/hermes-agent/blob/main/agent/prompt_builder.py) |
| `AGENTS.md`(`AGENTS.override.md`) | 人 | cwd 基準。git リポジトリ内なら git ルートから cwd まで合成、深い方が優先。リポジトリ外は cwd のみ。override は docs 間で記載が割れ実装未確認 | `context_file_max_chars`(既定は窓の 0.06 倍、20,000〜500,000 字)、先頭 70%＋末尾 20% を残す | 次セッション(推論) | [公式: Context files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files) |
| `CLAUDE.md` / `.cursorrules` | 人 | 上記が全て無いときだけ(先勝ちで 1 種類のみ) | 同上 | 同上 | 同上 |
| サブディレクトリの AGENTS.md 等 | 人 | ツールがそのディレクトリに触れたとき、ツール結果に追記。親 5 階層まで、1 ディレクトリ 1 回、32,000 字上限 | 32,000 字 | その場 | 同上 |
| `memories/MEMORY.md` | エージェント | セッション開始時の凍結スナップショット | 2,200 字 | 書込は即時にディスクへ、プロンプトへは次セッション | [公式: Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory) |
| `memories/USER.md` | エージェント | 同上 | 1,375 字 | 同上 | 同上 |
| `skills/**/SKILL.md` | 人/エージェント | 索引(name/description)のみ常時。本体は `skill_view` で必要時。プロジェクト `.hermes/skills/` は `hermes skills trust` 後 | 本体 約 24k 字で警告 | 索引はセッション中固定。`/reload-skills` で更新 | [公式: Skills](https://hermes-agent.nousresearch.com/docs/user-guide/features/skills) |
| cron ジョブのプロンプト(`cron/jobs.json`) | 人 | 毎回新規セッション。文脈ファイルは `workdir` 指定時のみ。プロンプトは作成/更新時にインジェクションスキャン | なし | 次回実行 | [公式: Cron](https://hermes-agent.nousresearch.com/docs/user-guide/features/cron) |
| config.yaml の `agent.system_prompt` / `platform_hints` / `agent.personalities` | 人 | system_prompt は personality 未選択時のみ有効。platform_hints は `append`/`replace` でプラットフォーム別に指針を差し替え | なし | 設定により異なる(未確認) | [公式: Personality, Prompt assembly](https://hermes-agent.nousresearch.com/docs/user-guide/features/personality) |

この表から読み取れる設計上の含意が 3 つある。第 1 に、プロジェクト文脈ファイルは**1 種類しか読まれない**ので、`.hermes.md` が存在すれば AGENTS.md も CLAUDE.md も黙って無視される。同一ディレクトリに複数置いて合算されることを期待してはならない([推論]、先勝ち規則は[公式])。第 2 に、SOUL.md だけが cwd に依存しない。第 3 に、記憶(MEMORY/USER)はエージェント自身が書く小さな凍結スナップショットであり、人が管理する設定の置き場ではない。公式ガイダンスも「memory は what、skills は how」とし、毎セッション必要な事実だけを記憶に置くとしている ([公式: Tips](https://hermes-agent.nousresearch.com/docs/guides/tips))。

スキャナにも注意が要る。プロジェクト文脈ファイルは「ignore previous instructions」型の上書き指示、「do not tell the user」、`cat .env` や `curl ... $API_KEY`、隠し HTML コメント、不可視 Unicode を含むと `[BLOCKED: ...]` に置換され、内容が読み込まれない。SOUL.md はユーザー作成なら警告のみでロードされる ([公式: Context files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files))。環境の記述に「`.env` を cat するな」のような禁止例を書くとブロック対象になり得るので避けるべきである([推論])。

## 環境の事実は cwd 直下の AGENTS.md に置き、SOUL.md に保険を掛ける

最大の落とし穴は、SOUL.md と AGENTS.md で「どこから読むか」が違うことである。メンテナ自身が「SOUL.md は HERMES_HOME から、AGENTS.md は設定された作業ディレクトリから読まれる」と述べており、ゲートウェイでは `terminal.cwd` が `TERMINAL_CWD` に橋渡しされる([issue #37480](https://github.com/NousResearch/hermes-agent/issues/37480))。同 issue では、`TERMINAL_CWD` を設定すると `hermes prompt-size` が 0 B から約 16 KB に増え、AGENTS.md が発見されたことが確認されている。設定ページは `terminal.cwd` の既定値を `"."`、コメントを「Gateway/cron working directory (CLI always uses launch dir)」としている ([公式: Configuration](https://hermes-agent.nousresearch.com/docs/user-guide/configuration))。したがって、Docker ゲートウェイで `.`(=プロセスの起動ディレクトリ)が何になるかは**公式に確認できていない**。ユーザーの `~/.hermes/AGENTS.md` を正しいファイルにしても、cwd が `/opt/data` でなければ読まれない([推論])。

推奨は次のとおりである。`config.yaml` に `terminal.cwd: /opt/data/workspace` を明示する(stage2 が `workspace/` を作成する ([ソース: stage2-hook.sh](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/docker/stage2-hook.sh))ので新規ディレクトリ不要)。ホストでは `~/.hermes/workspace/AGENTS.md` として実ファイルを置く。ここは `/opt/data` 直下より安全でもある。ただし local バックエンドのエージェントは同一ユーザーで `/opt/data/.env` を読めるので、cwd を変えてもシークレットは隔離されない点は変わらない([推論])。cwd をここへ変えるとエージェントの作業ファイルの出力先も変わるため、既存のタスクで相対パスに依存していないか確認する。

役割分担は次の方針とする。SOUL.md は人格・口調・応答言語(日本語で返す等)・「分からないことは分からないと言う」程度に限り、パスやコマンドは入れない。Hermes 公式の分担も SOUL.md=identity/tone、AGENTS.md=コマンド・ポート・パス・ワークフローである ([公式: Which File Does What](https://hermes-agent.nousresearch.com/docs/user-guide/which-file-does-what))。この原則に従えば環境事実は AGENTS.md の仕事である。それでも SOUL.md に 2〜3 行を重ねる理由は、今回の問題が「最初から認識していない」ことであり、cwd 解決が未確認な以上、cwd に依存しない唯一のチャネルである SOUL.md が最も確実だからである。重複は内容の乖離リスクを持つため、SOUL.md 側は「コンテナ内で動く。docker は使えない。詳細は AGENTS.md」の要約に徹する。公式は他層と重複する内容を記憶に置かないよう勧めており、同じ理由で複製は最小限にとどめる([公式: Memory](https://hermes-agent.nousresearch.com/docs/user-guide/features/memory))。

記述の仕方は、小型のローカルモデル(3B active の MoE)向けに寄せる。ベンダー各社は一貫して、短く、具体的で、検証可能な規則にせよと言う。Claude Code は 200 行未満、Cursor は 500 行未満を目安とし、「削っても誤りが起きないなら削れ」と述べる ([Claude Code best practices](https://code.claude.com/docs/en/best-practices), [Cursor rules](https://cursor.com/docs/context/rules))。ETH Zurich の研究は、文脈ファイルが総じて成功率を上げず推論コストを 20% 以上増やす一方、リポジトリ概要は役立たず、ツール固有の指示はよく使われたと報告している ([arXiv 2602.11988](https://arxiv.org/abs/2602.11988))。ただし対象はフロンティアモデルが中心で、小型ローカルモデルの直接的な証拠はない。つまり、環境事実は「非自明で、間違えると失敗する」情報の典型であり、書く価値が最も高い一方、長文化は逆効果である。以下は [推論] に基づく記述テンプレート案で、AGENTS.md の冒頭に置く。

```markdown
# 実行環境(確認日: YYYY-MM-DD / 根拠: docker-compose.yml)
- あなたは Docker コンテナ(Linux)の中で動いている。ホストは macOS で、直接は見えない。
- docker コマンドは使えない(docker.sock も無い)。docker が必要な作業は、ユーザーに実行コマンドを提示して依頼する。
- コマンドはこのコンテナ内で直接実行される(terminal.backend=local)。
- 作業ファイルは /opt/data/workspace に置く。永続データは /opt/data(=HERMES_HOME)。
- LLM は Ollama(ホスト側)。localhost はこのコンテナ自身を指す。
- できないことは「できない」と述べ、代替手順を1つ示す。

# 運用ルール(要点のみ)
- 返答は日本語。
- (プロジェクト固有の規則を 10〜20 行以内で)
```

SOUL.md 側の保険は次の程度にする。

```markdown
## 前提
あなたは Docker コンテナ内で動く。docker コマンドは使えない。環境の詳細は AGENTS.md を参照する。
```

書き方の要点は、各制約を「事実＋代替行動」の 1 行にすること、リテラルなパスとコマンドを使うこと、重要な 3〜5 行を冒頭に置くこと(「lost in the middle」では冒頭と末尾が強い ([Liu et al.](https://arxiv.org/abs/2307.03172)))、強調を乱発しないことである。禁止の言い回しは「やるな」よりも「代わりにこうする」へ書き換える。この点は実務者記事が主な根拠で確度は低い([コミュニティ])。HTML コメントは、スキャナが「隠しコメント」を検知対象とするため Hermes では避ける([公式]＋[推論])。

環境事実の真の出所は docker-compose.yml であり、AGENTS.md はその派生コピーである。「確認日」行を設け、compose を変えたら必ず見直す。古い事実は、エージェントが信じ込むため最も危険である。Hermes 公式も「stale context is worse than no context」と述べる ([公式: Context files](https://hermes-agent.nousresearch.com/docs/user-guide/features/context-files))。さらに、プロンプトは助言にすぎないので、docker CLI を使わせない措置を物理側でも取る。前回の調査どおり、docker.sock をマウントせず `terminal.backend: local` に固定すれば、誤った推測は静かな失敗ではなく明確なエラーになる(前回レポートの結論に依拠)。

## 反映の確認は /context から始め、編集後は /new を挟む

確認手段は 2 つある。`/context [all]`(別名 `/ctx`)は、カテゴリ別のトークン概算と、`.hermes.md`、AGENTS.md、CLAUDE.md、`.cursorrules`、SOUL.md の各ファイルが loaded / truncated / skipped のどれかを表示する。読み取り専用でローカル処理であり、メッセージング上でも使える ([公式: Slash commands](https://hermes-agent.nousresearch.com/docs/reference/slash-commands))。`hermes prompt-size [--platform <name>] [--json]` は、システムプロンプトとツールスキーマのバイト内訳をオフラインで出す ([公式: CLI reference](https://hermes-agent.nousresearch.com/docs/reference/cli-commands))。コマンド名は `prompt` ではなく `prompt-size` である。Docker では `docker exec hermes hermes prompt-size --platform telegram` の形になる(公式の形式からの推論)。ただしこの CLI 実行はゲートウェイの `terminal.cwd` を反映するとは限らない(CLI は起動ディレクトリを使う)ので、**真のゲートウェイ視点は Telegram 上の `/context` で見る**のが正しい。補助シグナルとして、SOUL.md が空・不可読なら identity が「You are Hermes Agent, built by Nous Research.」に戻る ([公式: Personality](https://hermes-agent.nousresearch.com/docs/user-guide/features/personality))。文脈ファイルがブロック・切り詰めされると `[BLOCKED: ...]` や `[...truncated AGENTS.md: kept ...` というマーカーが出る。加えて、runtime environment ブロックに cwd が載るので、Telegram で「いまの作業ディレクトリは?」と尋ねるのも有効である([公式: Prompt assembly])。

反映タイミングについて、公式は一般則を示していない。記憶は「書込は即時、プロンプトは次セッション」と明記され、スキル索引はセッション中固定、システムプロンプトはキャッシュ保持のため安定させると説明される([公式: Memory, Skills, Tips])。ゲートウェイのセッションは自動リセットされず、`/new` または `/reset` で新規セッションになる ([公式: Messaging](https://hermes-agent.nousresearch.com/docs/user-guide/messaging))。一方、`model.context_length` と `compression.*` は稼働中でも次メッセージから反映される([公式: Configuration])。以上から SOUL.md/AGENTS.md の編集は、`/new` の後に反映される可能性が高く、ゲートウェイがエージェントをキャッシュしている場合は再起動が要る可能性がある([推論]。明文なし)。手順としては「編集 → `/new` → `/context` で loaded を確認 → 反映されなければ `docker compose restart hermes`」が安全である。`terminal.cwd` のような config.yaml の変更は再起動を前提にする(非ホット項目は再起動と仮定。[推論])。

確認時には内容そのものも検証する。Telegram で「あなたは今どこで動いていて、docker は使えますか」と尋ね、答えが AGENTS.md の文言と一致するか見る。不一致なら、(a) 文脈ファイルが読まれていない、(b) 読まれたが Ollama 側で切り捨てられた、(c) モデルが無視した、の順で切り分ける。(b) の確認は次節に示す。

## Docker 固有の落とし穴は空ディレクトリ・WAL・権限・コンテキスト長

**空ディレクトリ。** 存在しないホストファイルを単一ファイルとして bind mount すると、Docker はそこにディレクトリを作る(Docker の一般挙動)。ユーザーの `~/.hermes/AGENTS.md` が空ディレクトリになったのはこの典型だと考えられる(推論)。stage2 は SOUL.md の種まきで `[ ! -f ... ]` を検査し、ディレクトリだと「無い」と見なすか、シンボリックリンク先を拒否する ([ソース: stage2-hook.sh](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/docker/stage2-hook.sh))。種まき対象は `.env`、`config.yaml`、`SOUL.md` のみで、AGENTS.md は種まきされない。対処は、コンテナを止めて `rmdir ~/.hermes/AGENTS.md` で空ディレクトリを消し(中身が空であることを確認)、compose から該当の単一ファイル mount を外し、実ファイルを `~/.hermes/workspace/AGENTS.md` に作成することである。単一ファイルの bind mount は、ホスト側のファイルを先に作ってから行う。ただしそもそも `~/.hermes:/opt/data` を丸ごとマウントしているなら、個別 mount は不要で、二重マウントは混乱の元になる。

**WAL。** Docker Desktop for Mac のバインドマウントは virtiofs で、state.db の WAL が破損し得る。公式は、新規 DB は DELETE ジャーナルへフォールバックし、既存 DB は `PRAGMA journal_mode=DELETE` を実行して `database.journal_mode: delete` を設定する、または名前付きボリュームを使うよう案内している ([公式: Docker](https://hermes-agent.nousresearch.com/docs/user-guide/docker))。ユーザーは bind mount なので該当する(推論)。ただし記憶は state.db とは別の MD ファイルなので、MEMORY.md/USER.md の読み書き自体は WAL の影響を受けない。影響するのはセッション履歴と `session_search` である。

**権限。** stage2 は `cron`、`sessions`、`memories`、`skills`、`workspace`、`profiles` などの許可リストだけを再帰的に chown し、`/opt/data` 直下は非再帰である([ソース: stage2-hook.sh])。したがって、ホストの root または別 UID で作った `AGENTS.md` や `workspace` 配下のファイルは、Hermes ユーザー(UID 10000、`HERMES_UID`/`PUID` で変更可)から読めない場合がある。`/opt/data/workspace` は許可リスト内なので再帰 chown されるが、直下の AGENTS.md を cwd に使うと自動修復されない。読めないファイルは `/context` で skipped になるか、無言で空扱いになる(推論)。公式は、全体を world-readable にすることを禁じている(認証情報を含むため)。

**Ollama のコンテキスト長。** 複数のガイドが、Ollama の既定を 4096 トークンとし、超過分(システムプロンプトの先頭)が警告なしで切り捨てられると報告する。OpenAI 互換 `/v1` エンドポイントは `num_ctx` を無視する場合があり、サーバー側で `OLLAMA_CONTEXT_LENGTH` を設定する必要がある ([fast.io](https://fast.io/resources/ollama-context-window/))。Hermes 側でも、プロンプトが窓を超えると出力が空・切り詰めになる報告がある ([issue #43900](https://github.com/NousResearch/hermes-agent/issues/43900))。これらはベンダーブログと issue の水準で、使用中の Ollama バージョンでは未確認である。重要なのは、Hermes 自体のツール定義とスキル索引(約 3k トークン)だけで 4096 を超えかねない点で、その場合 SOUL.md と AGENTS.md は先頭から落ちる。「環境を最初から認識しない」症状は、この切り捨てでも説明できる([推論])。ホスト側で `ollama ps` により実効コンテキスト長を確認し、`model.context_length` を Hermes 側にも合わせて設定する。また、`context_file_max_chars` は窓の 0.06 倍(下限 20,000 字)で算出されるため、窓の設定が誤っていると cap も狂う([ソース])。

その他の注意として、同じデータディレクトリに 2 つのゲートウェイを立てない、`entrypoint:` を上書きしない、`HERMES_HOME_MODE` の挙動はイメージ依存である、といった点は公式 Docker ページに従う。前回調査で扱った docker.sock と terminal.backend の件は繰り返さない。

## 推奨構成と未確認事項

推奨するホスト側の構成を以下に示す(`~/.hermes` が `/opt/data`)。

| ホスト | コンテナ | 内容 | 管理者 |
|---|---|---|---|
| `~/.hermes/SOUL.md` | `/opt/data/SOUL.md` | 人格・口調・言語＋環境の保険 3 行 | 人(Git 管理を推奨) |
| `~/.hermes/config.yaml` | `/opt/data/config.yaml` | `terminal.backend: local`、`terminal.cwd: /opt/data/workspace`、`model.context_length`、必要なら `memory.write_approval: true` | 人 |
| `~/.hermes/workspace/AGENTS.md` | `/opt/data/workspace/AGENTS.md` | 実行環境、運用ルール(20〜60 行) | 人(エージェントからは読み取り専用が理想) |
| `~/.hermes/memories/` | 同左 | MEMORY.md/USER.md | エージェント(人は定期監査のみ) |
| `~/.hermes/skills/` | 同左 | 繰り返し手順(5 ステップ以上) | エージェント/人 |

運用面では、(1) 手書きの事実を MEMORY.md に手で種まきせず AGENTS.md に置く(記憶は 2,200 字で、日本語は 1 字あたりのトークンが多いため実質さらに小さい。[推論])、(2) AGENTS.md を長手順の置き場にせず、手順はスキルへ逃がす、(3) `memory.write_approval: true` で「ホストの Mac 上にいる」といった誤った環境認識が記憶に固定されるのを防ぐ([推論]。公式は `write_approval` を設定項目として記載)、(4) エージェントが書き換えられるファイルは次セッションで再読込される以上、永続的なインジェクション面になり得るため、AGENTS.md は読み取り専用にし、書込先を memories と workspace に限る([推論])、(5) API キーやトークンを SOUL/AGENTS/MEMORY に書かない、という方針を勧める。スキルについては、`description` が索引で選択される唯一の手掛かりなので具体的に書き、使わないものはアーカイブする。キュレーター(v0.12.0)は既定で `enabled: true` だが、対象はエージェント作成スキルのみで、統合は既定オフである ([公式: Curator](https://hermes-agent.nousresearch.com/docs/user-guide/features/curator))。cron は文脈ファイルを既定で読まないため、環境事実が必要なジョブには `workdir` を指定するか、プロンプト自体に必要な 1〜2 行を含める。プロファイルを使う場合、SOUL.md は `profiles/<name>/` 配下だが、AGENTS.md は場所ではなく `terminal.cwd` で決まる([公式: Profiles](https://hermes-agent.nousresearch.com/docs/user-guide/profiles)＋[推論])。

確認できていない点は次のとおりである。第 1 に、Docker ゲートウェイでの `terminal.cwd` 既定(`.`)の実際の解決先と `MESSAGING_CWD` の有無。第 2 に、SOUL.md/AGENTS.md の再読込に `/new` で足りるのか再起動が要るのか(公式に記載なし。issue #37480 の報告者の解決も未確認のまま「not planned」で閉じられている)。第 3 に、`AGENTS.override.md` が main で実装されているか(docs 間で矛盾)。第 4 に、`platform_hints` で環境事実を渡す方式の挙動と再読込の条件。第 5 に、Hermes が AGENTS.md の HTML コメントを除去するか(Claude Code は除去するが Hermes は未記載)。第 6 に、qwen3.5 35B-A3B のような MoE モデルでの指示追従に関する実測データ(今回の調査では見つからず、小型モデル向けの書き方は一般研究からの外挿)。第 7 に、使用中の Ollama とイメージのバージョンでの実効 `num_ctx` の挙動。第 8 に、日本語コンテンツのトークン効率(Qwen トークナイザでの実測)。

検証の最短手順は次の 4 ステップである。まず、`rmdir ~/.hermes/AGENTS.md` の後に `~/.hermes/workspace/AGENTS.md` を作成し、`config.yaml` に `terminal.cwd: /opt/data/workspace` を設定して `docker compose restart`、次に Telegram で `/new`、続いて `/context` で AGENTS.md と SOUL.md が loaded であることを確認し、最後に環境についての質問で挙動を確かめる。`/context` が AGENTS.md を skipped や未検出とする場合は、cwd の解決先、権限(UID 10000 から読めるか)、スキャナのブロックの順に疑う。

## 結論

今回の課題の核心は、記述内容の巧拙ではなく「Hermes の 5 つの入口はそれぞれ読み込み元が異なり、AGENTS.md だけが HERMES_HOME ではなく cwd に依存する」というアーキテクチャ上の非対称にある。空ディレクトリ化した `~/.hermes/AGENTS.md` は、この非対称と Docker の単一ファイル bind mount の挙動が重なった結果で、記述テンプレートを磨く前に配置の修復が先決である。環境事実の置き場は、cwd 直下の AGENTS.md を正本とし、cwd に依存しない SOUL.md の数行を保険とする二段構えが現時点での最善である。

もう 1 つの含意は、プロンプト改善より先に「プロンプトが実際にモデルへ届いているか」を確認すべきだという点である。`/context` による Hermes 側の確認と、`ollama ps` による Ollama 側の実効コンテキスト長の確認の 2 段で、「読まれていない」「切り捨てられた」「無視された」を分離できる。小型ローカルモデルでは、この切り分けを経ずに指示文を書き足すと、文脈をさらに圧迫して悪化する恐れがある。
