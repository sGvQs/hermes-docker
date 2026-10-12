# ターミナル不調を断つ Hermes Docker 設定の見直し

**結論から言うと、原因は「コンテナ内の Hermes が兄弟コンテナ(docker バックエンド)を起動しようとして失敗している」可能性が最も高く、最短の解決策は `terminal.backend: local` に切り替えて docker.sock のマウントを外すことである。** 現在の compose には効果が確認できない設定が複数ある。`HERMES_DOCKER_RUN_AS_ROOT` は公式ドキュメントにもソースにも存在が確認できず、公式にあるのは `docker exec` 用の `HERMES_DOCKER_EXEC_AS_ROOT` だけである。`terminal.docker_run_as_host_user=true` は環境変数名として書式が誤っており、正しくは `TERMINAL_DOCKER_RUN_AS_HOST_USER` で、しかも docker バックエンドの子コンテナにしか効かない。`~/.hermes` の二重マウントは、子コンテナのパス解決問題(未修正の既知バグ)を解決しない。同種の事例は GitHub 上に多数あり、権限エラー、バインドマウントのパス不一致、ヘルスチェックの誤検知の3系統に分かれる。ただし、調査では「ターミナルが使えない」という文言そのものの報告例は見つからず、ユーザーの実エラーメッセージとログ確認が確定診断に必要である。

## ターミナルツールは「バックエンド」の可用性で決まる

Hermes のターミナルツールは `terminal.backend` に `local / docker / ssh / modal / daytona / vercel_sandbox / singularity` を取り、ソース上の既定値は `local` で、コンテナ内かどうかの自動判定は行わない ([Configuration docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration), [terminal_tool.py](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/tools/terminal_tool.py))。`local` ならコマンドは Hermes コンテナ自身の中で実行され、追加設定は不要である。一方 `docker` バックエンドは、Hermes がコンテナ「内」から別の兄弟コンテナ(サンドボックス)を作る構成になり、docker CLI と到達可能なデーモンが必須となる。イメージには `docker-cli` が入っているのでクライアントは存在するが、デーモンはない ([Dockerfile](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/Dockerfile))。

起動時の要件チェックは `docker version` を5秒タイムアウトで実行し、非ゼロ終了なら「Docker is installed but the Docker daemon is not running」系のメッセージでツールを無効化する ([terminal_tool_backends.py](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/tools/terminal_tool_backends.py))。この文言が「ターミナルが使えない」というモデルの言い換えとして現れている、と推測するのが合理的である(推測。公式の文言とモデルの言い回しの対応は未確認)。重要なのは、docker.sock を実際に渡していても、`hermes` ユーザーがソケットのグループに属していなければ同じ失敗になる点である。コンテナは root で起動し s6 のフックで `hermes`(UID 10000)に降格するが、フックはソケットの GID に合わせたグループへ `hermes` を自動追加する仕組みを持つ ([stage2-hook.sh](https://raw.githubusercontent.com/NousResearch/hermes-agent/main/docker/stage2-hook.sh))。この自動処理が入る前のバージョンでは、`group_add` が PID 1 にしか効かず、`exec gosu hermes` でグループが失われて「permission denied ... docker.sock」となる報告がある ([#16703](https://github.com/NousResearch/hermes-agent/issues/16703))。

## 現在の compose に潜む四つの問題

第一に、`HERMES_DOCKER_RUN_AS_ROOT=1` は存在を確認できなかった。公式ドキュメントの環境変数表にも `docker.py` にもなく、検索できた `HERMES_DOCKER_*` は `HERMES_DOCKER_BINARY` と `HERMES_DOCKER_EXEC_AS_ROOT` のみである ([Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker))。なお、リポジトリ全体の grep は未実施で、「存在しない」ではなく「確認できない」が正確である。意図が「root で動かしたい」なら、イメージは既に root で起動してから降格する設計であり、不要または逆効果になりうる。root でゲートウェイを直接動かすには `HERMES_ALLOW_ROOT_GATEWAY=1` が別途必要で、リスクがある ([Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker))。

第二に、`terminal.docker_run_as_host_user=true` は環境変数として無効である。この設定は `terminal:` 配下の真偽値キー(既定 false)で、環境変数形式は `TERMINAL_<KEY大文字>`、つまり `TERMINAL_DOCKER_RUN_AS_HOST_USER` である ([Configuration docs](https://hermes-agent.nousresearch.com/docs/user-guide/configuration))。ドットを含む名前は多くのシェルで扱えず、Hermes にも読まれない可能性が高い。さらに、有効化しても効果は `docker run` に `--user $(id -u):$(id -g)` を付けるだけで、サンドボックス用の子コンテナにしか関係しない。local バックエンドでは無意味である。

第三に、二重マウントはパス問題を解決しない。Docker デーモンはバインドマウントのソースを「デーモンのあるホスト側」で解決するため、Hermes が組み立てる `/opt/data/...` は Mac ホスト(正確には Docker Desktop の VM)上の同名パスを指し、`~/.hermes` ではない ([hermes-agent-docker](https://code.mehalter.com/mehalter/hermes-agent-docker))。実際、サンドボックスのスキル/キャッシュのマウントが空になる、`docker_volumes` に足すと「Duplicate mount point」になる事例が報告され、2026年10月10日時点で修正はマージされていない ([#50798](https://github.com/NousResearch/hermes-agent/issues/50798), [#40368](https://github.com/NousResearch/hermes-agent/issues/40368))。`/workspace/hermes` へのもう一つのマウントは、コンテナ内で同じディレクトリを別名で見せるだけで、デーモン側のパス解決には寄与しない。加えて、公式は同一データディレクトリを複数プロセスで使うことを避けるよう警告している ([Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker))。

第四に、macOS の Docker Desktop では bind mount(virtiofs/9p)上の SQLite WAL が静かに壊れうると公式が警告しており、名前付きボリューム `hermes-data:/opt/data` か `database.journal_mode: delete` が推奨されている ([Docker docs](https://hermes-agent.nousresearch.com/docs/user-guide/docker))。ターミナルエラーの直接原因ではないが、開発が不安定になる別要因として切り分けたい。なお Docker Desktop for Mac では、マウントされる `/var/run/docker.sock` は Mac のファイルではなく Linux VM 内のソケットで、コンテナ root なら通常使えるが、非 root は権限エラーになりやすい ([Docker forums](https://forums.docker.com/t/permission-for-v-var-run-docker-sock/132976))。

## 診断は「ログ → ソケット → 設定ファイル」の順で確定させる

最初に `docker logs hermes-agent --tail 100` と、Hermes にターミナルを使わせたときの正確なエラー文を取得する。次に、コンテナ内で `docker exec hermes-agent docker version` を実行し、エラー文で原因を振り分ける。調査ノートの診断表に基づく解釈は次のとおりである。

| 症状 | 原因 | 対処 |
|---|---|---|
| `permission denied ... docker.sock` | `hermes` がソケットのグループに属さない | `docker exec -u root hermes-agent stat -c '%u:%g %a' /var/run/docker.sock` で GID を確認し `group_add` を追加、または local へ切替 |
| `Cannot connect to the Docker daemon` | ソケット未マウント、`DOCKER_HOST` 誤り | ソケットのマウントと環境変数を確認 |
| 子コンテナのマウントが空 | ホスト/コンテナのパス不一致 | local へ切替、または下記のパス同一化 |
| `docker: command not found` | 古い、またはカスタムイメージ | `which docker` で確認しイメージ更新 |

(診断コマンド自体は標準的な Docker 操作で、ノートでは一次情報による裏付けのない手順である。)あわせて、見落としやすい点として、`~/.hermes/config.yaml` の `terminal.backend` が `hermes setup` により `docker` に設定されていないかを確認する。これが `docker` だと、compose 側の環境変数が何であれ、ソケット周りが整うまでターミナルは失敗し続ける。また `hermes doctor` は socket-proxy 構成で「daemon not running」と誤表示する不具合が v0.18.2 で報告され、9月5日に修正されているため、古いイメージならこの表示を鵜呑みにしない ([#72927](https://github.com/NousResearch/hermes-agent/issues/72927))。

## 修正案は「local 化」を第一選択にする

開発用途でコマンドをコンテナ内で実行できれば足りるなら、`local` バックエンドが最も確実である。ソケットが不要になり、権限・パス不一致・ヘルスチェックの3系統の失敗がすべて消え、セキュリティ面でも有利になる。docker.sock へのアクセスは、読み取り専用マウントでもデーモンホスト上で実質 root と同等であり、Mac では VM と共有ディレクトリ(`/Users` など)が影響範囲になる ([hermes-agent-docker](https://code.mehalter.com/mehalter/hermes-agent-docker), [Docker docs](https://docs.docker.com/desktop/setup/install/mac-permission-requirements/))。LLM エージェントに渡す権限としては大きい。ただし「local がコンテナ自体をサンドボックスにする」という整理は、ノートでも一次情報にはなく推論であり、検証が必要である。

```yaml
  agent:
    image: nousresearch/hermes-agent:<固定したX.Y.Zタグ>   # latest は避ける
    container_name: hermes-agent
    command: gateway run
    init: false            # 既定エントリポイントを上書きしない
    ports:
      - "127.0.0.1:9119:9119"
      - "8642:8642"
    volumes:
      - "hermes-data:/opt/data"          # 名前付きボリューム (WAL破損回避)
      - "${HOME}/Documents/ai-workspace:/root/workspaces"
      # 削除: ~/.hermes:/workspace/hermes, docker.sock, HERMES_DOCKER_RUN_AS_ROOT
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      - TERMINAL_ENV=local              # backend を明示 (config.yaml より確認)
      - HERMES_UID=501                  # bind mount を使う場合のみ。Mac の uid
      - HERMES_GID=20
      - HERMES_DASHBOARD=1
      - HERMES_PROFILE=docker
      - DASHBOARD_HOST=127.0.0.1
      - HERMES_PROVIDER_URL=http://host.docker.internal:11434/v1
      - SEARXNG_URL=http://searxng:8080
      - HERMES_API_TIMEOUT=1800
      - TELEGRAM_BOT_TOKEN=${TELEGRAM_BOT_TOKEN}
      - TELEGRAM_ALLOWED_USERS=${TELEGRAM_ALLOWED_USERS}
volumes:
  hermes-data:
```

(`TERMINAL_ENV` が実際のバックエンド指定名である点は環境変数リファレンスに基づく ([Env vars](https://hermes-agent.nousresearch.com/docs/reference/environment-variables))。`~/.hermes` の既存データを使い続けたい場合は bind mount のまま `hermes-data` を置き換えず、`database.journal_mode: delete` を設定する。ホストの `~/.hermes` を CLI と共有したい事情があるなら、WAL リスクを承知で bind mount を一本に絞る。)

docker バックエンドをどうしても使う場合は、(1) 修正済みの新しいイメージを使い、非 root の `user:` を指定しない(`user: "10000:10000"` と `group_add` の組み合わせは v0.15.0/v0.15.1 で s6 の起動ループを起こした) ([#34648](https://github.com/NousResearch/hermes-agent/issues/34648))、(2) `docker exec -u root` でソケットの GID を確認し必要なら `group_add`、(3) `HERMES_HOME` と `docker_volumes` をホストの絶対パスに揃える(例: `${HOME}/.hermes:${HOME}/.hermes` と `HERMES_HOME=${HOME}/.hermes`)、という順で試す。(3) はノートでも推論であり、イメージの `VOLUME /opt/data` と衝突するため、検証前提である。公式の Hermes はホスト上で docker バックエンドを動かす構成を想定しており、コンテナ内からの利用手順は文書化されていない。

## 補足と未確認事項

compose の `SEARXNG_SECRET_KEY=$(openssl rand -hex 32)` は、compose がシェル展開しないため文字列としてそのまま渡る(一般的な compose の挙動に基づく補足で、ノートの調査対象外)。`.env` に実値を書くほうが安全である。`/var/run/docker.sock` を前提にする限り、Docker Desktop の「Allow the default Docker socket to be used」設定と再起動後のシンボリックリンク消失にも注意が要る ([Docker docs](https://docs.docker.com/desktop/setup/install/mac-permission-requirements/))。

## Conclusion

見えてきたのは、これは単一バグというより「Docker 内で動く Hermes に Docker 越しのサンドボックスを持たせる」設計上の摩擦だという点である。環境変数で root やホストユーザーを強制しても、問題の本体であるデーモンへの到達性とパス解決は変わらない。今回の compose で追加した設定の多くは、効くことが確認できないか、効く対象が別の層にある。

なお本レポートには限界がある。情報は主に GitHub issue 要約と公式ドキュメントの取得結果で、文言は逐語的に検証されていない。「ターミナルが使えない」という正確な文言の事例、Mac mini + Ollama + Telegram 構成の同種報告、Docker Desktop for Mac 上の現在のソケット所有者/モードは見つからなかった。確定には、ユーザー環境の `docker logs`、`docker exec ... docker version`、`config.yaml` の `terminal.backend` の3点が必要であり、可能ならリポジトリを clone して `grep -rn HERMES_DOCKER_RUN_AS_ROOT` で存否を確かめるべきである。
