# ARTEX ホスト分類（トリアージ）

日本語 · [English](README.md) · [한국어](README.ko.md)

[`artex_host_triage.py`](artex_host_triage.py) は、ARTEX が実行された疑いのある **ホスト一台の上で直接** 実行する、読み取り専用のトリアージ（分類）スクリプトです。このディレクトリのほかの資料は、SIEM（[Sigma](../sigma/)）・ネットワークセンサー（[Suricata](../suricata/)）・脅威インテリジェンス基盤（[侵害指標](../indicators/)）を運用する防御者向けのものです。このスクリプトは、それとは別の対応者、すなわち SIEM のない状態で疑わしいホストのシェルの前に立ち、ローカルの状態だけで「ここで ARTEX は動いたか」を素早く根拠をもって答えなければならない人を想定しています。

このスクリプトは、ディレクトリのほかの資料が含む指紋をそのまま点検するほか、**[侵害指標一覧](../indicators/artex_indicators.csv)が意図的に Sigma ルールなしで残した 3 つのホスト・DB 指標**も点検します。その 3 つはログやネットワークでは観測されず、ホストで直接確認するしかないものです（`server-listen-port`、`recording-proxy-endpoint`、`postgres-exploration-schema`）。

## 何を点検するか

すべての点検項目は、このリポジトリのソースで確認した文字列やパスに基づいており、各発見には対応する Sigma ルールや侵害指標の行と同じ限界を併記します。

- **リスンポート**: `:8787`（管理 UI）と `127.0.0.1:8788`（記録プロキシ）を確認します。両値は [`cmd/artex/main.go`](../../cmd/artex/main.go) の `--addr`・`--proxy` フラグの既定値です。実行中のホストでは `ss`・`netstat`・`lsof` の出力を解析し、`--ports-from` で渡したファイルから読み込むこともできます。
- **記録プロキシの成果物**: レコーダーが初回実行時に作成する中間者（MITM）CA ファイル `<データディレクトリ>/traffic/_ca/mitmproxy-ca-cert.pem` と、その隣の `_index/index.sqlite`・`_blobs/` を確認します（[`traffic/traffic.go`](../../traffic/traffic.go)。データディレクトリの既定値は実行ファイルの隣の `data/` です）。この CA は、行き来する HTTP(S) を復号して記録する中間者型のトラフィックレコーダーの信頼の起点です（MITRE ATT&CK T1557）。
- **ログマーカー**: ログファイルから、補強プローブの User-Agent `artex-enrich/1.0`（[`enrich/enrich.go`](../../enrich/enrich.go)）、自己更新送信の User-Agent `artex-selfupdate`（[`selfupdate/github.go`](../../selfupdate/github.go)）、プラットフォームガードの監査マーカー（[`guard/guard.go`](../../guard/guard.go)）を探します。ガードマーカーは非 ASCII のフレーミングまで原文のまま扱い、grep が実際に一致するようにしてあります。ログ回転で `.gz`・`.bz2`・`.xz` に圧縮された過去のログも展開して点検するため、ホストのログ履歴全体をカバーします。ただし Python の標準ライブラリにコーデックがない形式（`.zst`・`.lz4`）は点検せず、**スキップしたファイルとして報告**します。黙って問題なしとは扱わないので、そのようなファイルは先に展開するか、手作業で `grep` して個別に確認してください。
- **PostgreSQL の探索スキーマ**: ARTEX リポジトリの二重グラフテーブル（`exploration_nodes`・`_edges`・`_anchors` と `assets`・`companies`・`activity`、そして `agent_prompts` のシード）を確認します（[`db/schema.sql`](../../db/schema.sql)）。DSN を渡せば `psql` で照会し、`psql` がなければ手作業で実行できる読み取り専用クエリをそのまま出力します。
- **実行中プロセスの環境変数の注入**: プロキシ変数（`HTTP_PROXY`・`HTTPS_PROXY`・`ALL_PROXY`）と、mitmproxy CA（`mitmproxy-ca-cert.pem`）を指すツールチェーンの CA 信頼変数（`SSL_CERT_FILE`・`CURL_CA_BUNDLE`・`REQUESTS_CA_BUNDLE`・`GIT_SSL_CAINFO`・`NODE_EXTRA_CA_CERTS`）を**同時に**持つプロセスを探します。ARTEX は生成するすべてのワーカーツールに、まさにこれらの変数を注入します（[`agent/worker.go`](../../agent/worker.go) の `proxyEnv`、[`agent/proxyenv_test.go`](../../agent/proxyenv_test.go) が検証）。**変数名がソースにハードコードされている**（値だけ変更可能）ため、この指紋は運用者がバイナリ名やポートを変えても残り、リスンポート一つよりも特異的です。実行中の Linux ホストでは `/proc` を読み、オフライン・フォレンジックイメージでは `--proc-from` で取り込んだ環境変数のダンプを読みます。プロキシと CA が両方そろっていれば高い重大度、mitmproxy CA だけ、または ARTEX 既定のプロキシエンドポイント（`127.0.0.1:8788`）だけなら中程度の重大度として報告しますが、mitmproxy CA のない企業プロキシは手がかりに上げません。

発見は**分類のための手がかりであり、断定ではありません。** また、何も引っかからなかったとしても安全という意味ではありません。運用者はバイナリ名を変えたり、データディレクトリを移したり、ポートを変えたりできるからです。

## プラットフォーム対応

このスクリプトは純粋な Python 3（標準ライブラリのみ）なので、Python 3 が動く環境ならどこでも動作します。Linux（CI の自己テスト）と macOS で実際に実行して確認しました。OS に依存する点検は 2 つあり、どちらも失敗せずにきれいに縮退します。

- **実行中のポート点検**: `ss` → `netstat` → `lsof` の順に試し、出力が得られた最初のツールを使います。Linux では `ss`・`netstat` が使われ、`ss` がなく `netstat` が Linux 形式の `-ltnp` フラグを受け付けない macOS・BSD（この場合は出力なしで終了）では、`lsof -nP -iTCP -sTCP:LISTEN` に切り替えて同じ方法で解析します。リアルタイムに走査する代わりに保存済みの一覧を読むには `--ports-from` を使ってください。
- **実行中プロセスの環境変数の点検**: `/proc` を読むため Linux でのみ動きます。`/proc` がないホスト（macOS・BSD）では、問題なしではなく**スキップ**として報告するため、Linux ホストでダンプを取り、`--proc-from` に渡してください（使い方を参照）。

残りの点検（記録プロキシの成果物・ログマーカー・PostgreSQL スキーマ）はファイルシステム・ログファイル・（DSN があれば）`psql` を読むため、OS に依存しません。

## 使い方

```sh
# ホストを最初から最後まで点検します
detections/triage/artex_host_triage.py \
    --data-dir /opt/artex/data \
    --log /var/log/syslog --log-dir /var/log/artex \
    --pg-dsn "$ARTEX_PG_DSN"

# 機械が読める形式で出力し、一つでも引っかかれば非ゼロで終了します
detections/triage/artex_host_triage.py --data-dir /opt/artex/data --json --exit-code

# オフライン・フォレンジックイメージ: 取得したプロセス環境変数のダンプを読みます
#   ホストでダンプを作成する方法:
#   for p in /proc/[0-9]*; do echo "# $p"; tr '\0' '\n' < "$p/environ"; echo; done > proc_env_dump.txt
detections/triage/artex_host_triage.py --proc-from proc_env_dump.txt

# 再現可能なフィクスチャの自己テスト（ホストの状態には触れません）
detections/triage/artex_host_triage.py --self-test
```

このスクリプトは Python 3 の標準ライブラリだけを使います。インストールは不要で、ネットワークは使わず、`--self-test` が使う自前の一時ディレクトリ以外には何も書き込みません。ホストの状態（開いているポート、データディレクトリ、ログファイル、そして DSN を渡したときだけデータベース）を読み、発見事項を出力します。終了コードは既定で `0` です（ゲートではなく分類用途のため）。`--exit-code` を付けると、指標が一つでも引っかかった場合に `1` で終了します。

## 正直さをどう保つか

`--self-test` は合成ホストを作ります。仕込んだ CA・索引・ブロブストアを持つデータディレクトリ、各マーカーを含むログ、ポート一覧、取得したプロセス環境変数のダンプを作成し、すべての点検がその上で発火することを検証します。続いて、クリーンなホスト・正常なログ・企業プロキシのプロセスでは発見が **0** 件であること（誤検知がないこと）を検証します。この自己テストは [`detections` CI ワークフロー](../../.github/workflows/detections.yml)に組み込まれており、[`detections/tests/run-all.sh`](../tests/run-all.sh) が再実行します。そのため、ある点検が壊れた場合や、指標が grep するソース文字列からずれた場合は、マージゲートで失敗します。動かして確かめられない検知は主張にすぎない、という原則に従っています。
