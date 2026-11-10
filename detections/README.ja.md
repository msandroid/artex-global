# ARTEX 検知ルール

日本語 · [한국어](README.ko.md) · [English](README.md)

> このディレクトリは、[防御・検知ガイド（../docs/defense-ko.md）](../docs/defense-ko.md) の 4 節「検知ルール」にある擬似ルールを、各自の SIEM・EDR のクエリ言語に変換してそのまま配備できる、ベンダー中立の [Sigma](https://sigmahq.io) 形式に移したものです。ここに掲載したすべての指標は**推定ではなく**、このリポジトリのソースで実際に確認した文字列や振る舞いに基づいています。すべてのルールは、自分が所有している、または書面による許可を得ているシステムを守る**防御・検知の目的にのみ**使用してください。

## 原子（atomic）ルール

- **`sigma/artex_enrich_user_agent.yml`**: ARTEX の資産エンリッチメント（`enrich/enrich.go`）が送信する、受信側の `artex-enrich/1.0` User-Agent です。ターゲット側で観測する補助的な指標です。`level: high`。
- **`sigma/artex_selfupdate_egress.yml`**: 自己更新ルーチン（`selfupdate/github.go`）が送信する、送信側（アウトバウンド）の `artex-selfupdate` User-Agent です。ホストやフォレンジックの観点からの送信（egress）指標です。`level: medium`。
- **`sigma/artex_guard_audit_framing.yml`**: ツール呼び出しがブロックされた際に監査ログへ記録される、プラットフォームガードの統制マーカー（`guard/guard.go`）です。ホストやフォレンジックの指標です。`level: high`。
- **`sigma/destructive_command_hunting.yml`**: ARTEX ガードの組み込み拒否リスト（`db/db.go` のシード）をそのまま反映した、破壊的なシェル・DB コマンドです。ARTEX 固有のシグネチャではなく、一般的なハンティングの手がかりです。`level: medium`。
- **`sigma/artex_recording_proxy_ca.yml`**: 記録用プロキシが `_ca/mitmproxy-ca-cert.pem` のレイアウトで生成する MITM CA 証明書ファイル（`traffic/traffic.go`）の作成です。ホストやフォレンジックのアーティファクトです。ファイル名そのものは単独で実行した mitmproxy と共有されるため、ハンティングの手がかりとして扱います。`level: medium`。

## 相関（correlation）ルール: 振る舞いベース

静的な文字列は変更できますが、振る舞いは隠しにくいものです。[`sigma/correlation/`](sigma/correlation/) にある Sigma の**相関**ルールは、防御ガイド（4.1〜4.2 節、4.4 節）の振る舞いベースの層を表現しています。各相関ルールは上記の原子ルールを `id` で参照しているため、参照を解決するには単一の相関ファイルではなく、`sigma/` ツリー全体を変換する必要があります（下記を参照）。

- **`sigma/correlation/artex_enrich_scan_velocity.yml`**: 1 つの送信元から短い時間枠内に送られる `artex-enrich/1.0` プローブの束です（エンリッチメントは同時実行数 4 で、レート制限なしに動作します）。単一リクエストのルールが見逃す速度を捉えます。`event_count`、`level: high`。
- **`sigma/correlation/artex_enrich_fanout.yml`**: 1 つの送信元がエンリッチメントの User-Agent を、多数の**異なる**ホストに運ぶ場合です。資産リスト全体に機械の速度で広がるパターンで、量だけでなく幅（breadth）が手がかりになります。`value_count`、`level: high`。
- **`sigma/correlation/artex_guard_block_burst.yml`**: 1 つのホストでプラットフォームガードの統制マーカーが繰り返し記録される場合です。マーカーを引用しているだけの文書ではなく、実行中の ARTEX がそのガードに引っかかっていることを示す信号です。`event_count`、`level: high`。
- **`sigma/correlation/artex_guard_marker_then_destructive.yml`**: 1 つのホストで、ガードマーカーと破壊的コマンドが時間枠内に同時に現れる場合です（防御ガイド §4.2、多段階）。ARTEX 固有のマーカーを、それ自体は一般的な破壊コマンドの信号と組み合わせることで、特異度を高めています。`temporal`、`level: high`。

しきい値と時間枠は保守的なデフォルト値です。各自のベースライン（baseline）に合わせて調整してください。§4.2 にある純粋な Web の多段階の事例（列挙 → プローブ → 認証）は、そのパターンが ARTEX 固有の単一の User-Agent に還元できないため、依然として各環境に固有の基本ルールが必要です。その出発点として使える一般的な振る舞いベースの Sigma ベーステンプレートを [防御ガイド §4.2](../docs/defense-ko.md#42-siem-상관-규칙) に置いています。ARTEX のソースで根拠を固定できないため、ここでテストされるルールツリーには含めていません。

## ネットワークルール（Suricata）

Sigma はホストとログのテレメトリを扱います。ネットワーク上で観測できる ARTEX 固有の User-Agent は 2 つあり、どちらも [`suricata/`](suricata/) に [Suricata](https://suricata.io) ルールとして収録しています。エンリッチメントのプローバーが送る `artex-enrich/1.0`（`enrich/enrich.go`）には、存在シグネチャ 1 つと高速列挙のバリアント 1 つ（sid 1000001・1000002）が、norma SDK の WebFetch ツールが攻撃段階で送る `norma/0.4`（`github.com/Autumn-27/norma/tool/webfetch.go`）には存在シグネチャ 1 つ（sid 1000003）が対応しています。その他のワーカーツール（Bash で実行する `curl`・`nmap` など）は独自の User-Agent を使うため、ARTEX 固有の指紋はありません。そのためネットワーク層は、この 2 つの UA だけに意図的に絞っています。範囲、TLS の注意点、`suricata -T` と参照用 pcap による検証方法は [`suricata/README.ko.md`](suricata/README.ko.md) を参照してください。

## ATT&CK カバレッジ

これらのルールがタグ付けしている手法は、[MITRE ATT&CK](https://attack.mitre.org/) Navigator レイヤー [`attack/artex_navigator_layer.json`](attack/) にまとめています。6 つの戦術（偵察、コマンド＆コントロール、実行、インパクト、認証情報アクセス、収集）にまたがる 8 つの手法で、各手法はルールの `attack.*` タグに根拠を置き、検知の強度（ARTEX 固有のシグネチャか、一般的なハンティングの手がかりか）でスコア付けしています。[ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/) で開くと、どのルールがどの ARTEX の振る舞いをカバーしているかを確認できます。スコアリング、手法とルールの対応、そして正直な範囲（カバレッジは網羅性ではありません）については [`attack/README.ko.md`](attack/README.ko.md) を参照してください。[一貫性テスト](tests/attack/run.sh) が、レイヤーとルールセットがずれないように維持します。

## 侵害指標リスト（機械可読）

検知ロジックではなく、原子的な指標そのものを必要とする防御者のために、[`indicators/artex_indicators.csv`](indicators/) は ARTEX が出力する固有の指紋を 1 つの CSV ファイルにまとめたものです。脅威インテリジェンスプラットフォーム、SIEM のルックアップテーブル、ホストのトリアージ用チェックリストにそのまま取り込めるよう、エンリッチメントと自己更新の User-Agent、ガードの監査マーカー、サーバー・プロキシのデフォルトエンドポイント、記録用プロキシの CA 証明書、そして PostgreSQL の探索グラフのスキーマ指紋を含めています。各行には、根拠となったソースファイルと、その上に構築されたルール（ある場合）を記録しています。同じ指標は、そのままインポートできる [MISP](https://www.misp-project.org/) イベント（[`indicators/artex_indicators.misp.json`](indicators/)）としても提供しているため、MISP を使う防御者（または MISP から STIX にエクスポートする防御者）は CSV の列を手作業で対応付ける必要がありません。ルールに根拠を持つ指紋には `to_ids` を付け、ホストのフォレンジック用のポートとスキーマ指紋には付けていません。一般的なハンティングの手がかり（破壊コマンド）と、norma SDK が共有する `norma/0.4` の WebFetch User-Agent（ARTEX 固有の文字列ではなく、Suricata sid 1000003 が捉えるネットワーク署名です）は、誤検知を避けるため、インポート用リストからは意図的に外しています。列の構成、MISP タイプの対応付け、正直な注意点、そして CSV と MISP イベントがずれないように守る一貫性テストについては [`indicators/README.ko.md`](indicators/README.ko.md) を参照してください。

## ホストのトリアージ

上記のルールは、SIEM・ネットワークセンサー・脅威インテリジェンスプラットフォームを使う防御者のためのものです。それとは別の対応者、つまり SIEM がなく、疑わしいホスト 1 台のシェルの前に座っている人のために、[`triage/artex_host_triage.py`](triage/) を用意しています。これはローカルの状態だけから「ここで ARTEX が実行されたか」に答える読み取り専用のスクリプトです。同じ指紋を点検するほか、**CSV が Sigma ルールなしで意図的に掲載している 3 つのホスト・DB 指標**（サーバーのリッスンポート、記録プロキシのエンドポイント、PostgreSQL の探索スキーマ）も点検します。この 3 つはログやネットワークでは観測できないため、ホスト上で直接確認するしかありません。さらに、記録機が子プロセスに注入する環境変数の痕跡、つまり実行中のプロセスがプロキシ変数と mitmproxy の CA 信頼変数を同時に持っているか（`agent/worker.go`）も、`/proc` または `--proc-from` のダンプから確認します。各検出結果は、対応する侵害指標の行と同じ限界を持つトリアージの手がかりです。詳細は [`triage/README.ko.md`](triage/README.ko.md) にあり、組み込みの `--self-test` は下記のマージゲートとして実行されます。

## テスト

ルールには、Docker だけで実行できる再現テストが [`tests/`](tests/) に一緒に入っています。

- **Suricata**（[`tests/suricata/run.sh`](tests/suricata/run.sh)）: まずルールファイル全体が `suricata -T --init-errors-fatal` で読み込まれることを検証します（どのキャプチャでも使われないルールであっても、パースの失敗は検出されます）。次に scapy で決定的なキャプチャを合成し、それに対して `suricata -r` を実行して、存在ルールがプローブごとに 1 回ずつ発火すること、速度ルールがしきい値を超えると発火すること、そして良性（benign）な User-Agent のキャプチャではアラートが 0 件であることを検証します。バイナリキャプチャはコミットせず、実行のたびに再生成します。
- **Sigma**（[`tests/sigma/run.sh`](tests/sigma/run.sh)）: 下記「検証と変換」の `sigma check` と `sigma convert` による検証を、実行可能なテストとして実行します。エラー 0、ツリー全体がバックエンドのクエリにコンパイルされること、各原子指標の文字列がそのクエリまで残っていること、そして相関ルールは単独では変換に失敗することを検証します。最後の検証は、相関ルールが参照する原子ルールに実際に依存していることを証明します。
- **Sigma のライブイベントマッチング**（[`tests/sigma_match/run.sh`](tests/sigma_match/run.sh)）: 上記の Sigma テストがルールの妥当性とコンパイルを証明するのに対し、このテストは原子ルールと相関ルールが実際に発火することを証明します。原子ルールごとに、代表的な悪性サンプルのイベントがルールに一致し、正常サンプルのイベントは一致しないことを検証します（例: `.mitmproxy/` 配下の単独の CA ファイルは、`_ca/` ディレクトリも必要とする記録用プロキシルールを発火させません）。相関ルールごとには、同じグループ内でしきい値が時間枠内に満たされる陽性のタイムラインで発火し、しきい値未満・時間枠超過・グループ分割・レッグ（leg）の欠落のタイムラインでは沈黙することを検証します。パースはすべて pySigma に任せ、テストはコンパイル済みの条件ツリーと集計仕様だけを辿り、どのイベントが参照先ルールに入るかを原子ルールと同じマッチャーで判定します。「実行できない検知ルールは主張にすぎない」という原則を、Suricata と同様に Sigma 側にも適用しています。
- **ATT&CK レイヤー**（[`tests/attack/run.sh`](tests/attack/run.sh)）: ATT&CK カバレッジレイヤーがルールと一貫しているかを確認します。スコア付けした手法と戦術は、ルールセットの `attack.*` タグと正確に一致している必要があり、各手法は実在するルールファイルを指していなければなりません。レイヤーを更新せずにルールを追加した場合（またはその逆）は、テストが失敗します。
- **指標の根拠（source-of-truth）**（[`tests/indicators/run.sh`](tests/indicators/run.sh)）: 各ルールが固定した指標が、今もなお上流ソースが出力する文字列そのものであることを確認します。`enrich/enrich.go` の `artex-enrich/1.0`、`selfupdate/` の `artex-selfupdate`、`guard/guard.go` のガードマーカー、`db/db.go` の破壊トークンが、ルールに今も固定されているかを見ます。他の 3 つのテストが見逃すドリフトを捉えます。すなわち、すべてのルールがコンパイルされ発火している最中に、上流の再同期によって User-Agent やマーカーが変わってしまうケースです。同じテストは、機械可読な [`indicators/artex_indicators.csv`](indicators/artex_indicators.csv) を読み直し、公開されている各行が今もソースとルールに根拠を持つことを検証するため、防御者が取り込む成果物も古くなりません。最後に、読み込むすべての上流ソースが CI ワークフローの `push`・`pull_request` のパスフィルタに含まれていることを検証します。これにより、新しく固定したソースだけを変更した PR がテストをスキップして、そのドリフトがマージゲートを通過するのを防ぎます。こうして、「このリポジトリのソースで確認した文字列に根拠を置く」（上記）という約束は、言葉ではなくガードになります。
- **MISP のエクスポート一貫性**（[`tests/misp/run.sh`](tests/misp/run.sh)）: MISP イベント（[`indicators/artex_indicators.misp.json`](indicators/artex_indicators.misp.json)）が有効な MISP ドキュメントであることを証明します。[pymisp](https://github.com/MISP/PyMISP) で読み込まれ、pymisp のオブジェクトモデルは実在しない属性タイプを拒否するため、この成果物は MISP のように見えるだけでなく、実際にインポートできます。また、上記の CSV と行単位で同期していることも検証します。同じ値、指標ごとに意図された MISP のタイプとカテゴリ、そして CSV の正直さをそのまま反映する `to_ids`・`disable_correlation` フラグが一致している必要があります（ルールに根拠がある＝対処可能なので `to_ids` オン、ホストのフォレンジック用ポート＝分類のヒントなので `to_ids` オフかつ相関を無効化）。このイベントは CSV と並べて手作業で管理しています。CSV にはない解説コメント・UUID・タグを持つためです。そのため、CSV の行を追加・削除・タイプ変更するときは、同じコミットで MISP イベントも修正する必要があり、両者が一致するまでこのテストは失敗し続けます。
- **Sigma のバックエンドの移植性**（[`tests/sigma_backends/run.sh`](tests/sigma_backends/run.sh)）: ルールが Splunk の 1 つの例を超えて変換できることを証明します。ツリー全体（原子 + 相関）が Splunk、Elasticsearch の `eql` ターゲット、Grafana Loki でコンパイルでき、5 つの原子ルールは Sigma の相関をサポートしないバックエンド（Elasticsearch の `lucene`、Microsoft の `kusto` バックエンド）でもコンパイルできます。下記「検証と変換」にあるバックエンドごとのサポート表を、再実行可能な検査で裏付けています。
- **SigmaHQ の慣例リント**（[`tests/sigma_lint/run.sh`](tests/sigma_lint/run.sh)）: SigmaHQ の検証器一式全体（`pySigma-validators-sigmahq` プラグインによるもので、通常の `sigma check` では読み込まれません）を、[`tests/sigma_lint/validators.yml`](tests/sigma_lint/validators.yml) に文書化されたベースラインに沿って実行し、問題 0 件を検証します。また、検証器一式が実際に全部実行されたこと、そして意図的に除外して文書化した 4 つのチェックだけが残っていることも確認します。そのため、ルールが新たな慣例の問題（誤った大文字小文字のタイトル、分類体系から外れたフィールド）を持ち込むと、ビルドが失敗します。

8 つのルールテストとは別に、ルールではない 3 つのゲートが、同じ CI ワークフローと [`tests/run-all.sh`](tests/run-all.sh) の中で一緒に動いています。1 つ目はハーネス同期の検査です（run-all.sh・CI・スイートのディレクトリが、同じスイートを同じ順序で呼び出しているかを確認します）。2 つ目は[ホストトリアージツール](triage/)の `--self-test`（[`tests/triage-selftest.sh`](tests/triage-selftest.sh)）です。合成ホストを作成し、すべてのトリアージ検査が発火すること、そしてクリーンなホストでは検出結果が 0 件であることを検証します。3 つ目は、防御ガイドのインライン Sigma 相関テンプレートの整合性検査（[`tests/check-defense-template.sh`](tests/check-defense-template.sh)）です。防御ガイドがそのテンプレートについて述べている事実を再実行します。`sigma check` が通ること、splunk・eql・loki に変換されること、lucene と kusto は相関ルールを拒否すること、そして韓国語版と英語版のルール本文がバイト単位で同一であることを確認します。このテンプレートはルールツリーの外に散文として置かれているため、上流の `sigma-cli` が更新されてその主張が古くなっても、他のどのスイートも検出できないからです。

各スクリプトは、いずれかのアサーションが失敗すると 0 以外のコードで終了します。[`tests/README.ko.md`](tests/README.ko.md) を参照してください。

## これらのルールを正直に読むには

- **静的な指標は変更できます。** 運用者は User-Agent を別の値に設定できるため、`artex-enrich/1.0` や `artex-selfupdate` が**見当たらないからといって、安全だという意味ではありません。** 持続するシグナルは*振る舞い*です。1 つの送信元が、偵察 → 列挙 → プローブ → 認証・インジェクションの試行と連鎖し、応答に適応しながら、休みなく動き続ける様子です。その層は防御ガイド（1 節・2 節・4.1〜4.2 節）で説明しており、上記の `sigma/correlation/` ルールが、配備可能な相関（速度、ファンアウト、ガード遮断の束、そしてガードマーカー＋破壊コマンドの多段階）として実装しています。純粋な Web の多段階の事例は、依然として各環境に固有の基本ルールが必要です。
- **破壊コマンドのルールは一般的なハンティングです。** ARTEX のガードの拒否リストを反映していますが、同じコマンドは正当な管理者も実行します。ヒットは手がかりとして扱い、環境に合わせて許可リストを用意し、それだけで ARTEX によるものと断定しないでください。
- **ポートの指標は Sigma ではなく、ホストのフォレンジック用です。** ARTEX サーバーのデフォルト `:8787` と記録プロキシ `127.0.0.1:8788`（`cmd/artex/main.go`）は、疑わしいホストで `ss`・`netstat` を使って確認するのがよいため、防御ガイドに文書化し、トリアージ用に[指標 CSV](indicators/) に掲載しています。騒がしいネットワークルールとしては配布していません。[ホストトリアージスクリプト](triage/) は、まさにこうしたホストローカルの検査（ポート、記録プロキシのアーティファクト、ログマーカー、PostgreSQL のスキーマ）を、シェルへのアクセスはあるが SIEM がない対応者の代わりに実行します。

## 検証と変換

これらのルールは [sigma-cli](https://github.com/SigmaHQ/sigma-cli)（pySigma）で検証しています。再現するには次のとおりです。

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install sigma-cli

# 構造 + ベストプラクティスの検証（期待値: エラー 0、問題 0）
sigma check detections/sigma/

# この独立したルールセット向けに文書化されたベースラインで、SigmaHQ の慣例をすべて検査（期待値: 問題 0）。
# 上の通常の `sigma check` では、これらの検証器は読み込まれません。
pip install pySigma-validators-sigmahq
sigma check --validation-config detections/tests/sigma_lint/validators.yml detections/sigma/

# 対象のクエリ言語にコンパイル。例: Splunk
sigma plugin install splunk
sigma convert -t splunk --without-pipeline detections/sigma/artex_enrich_user_agent.yml

# 相関ルールが id で参照する原子ルールを解決できるように、ツリー全体を変換
sigma convert -t splunk --without-pipeline detections/sigma/
```

ベースラインは SigmaHQ の慣例をすべて強制しますが、SigmaHQ モノレポのファイル整理体系と分類体系を定めた 4 つの検査だけは、この独立したルールセットには該当しないため除外しています。各除外とその理由は [`tests/sigma_lint/validators.yml`](tests/sigma_lint/validators.yml) に文書化されており、上記のリントテストが強制します。

### Sigma バックエンドの移植性

`sigma/correlation/` のルールは原子の基本ルールを `id` で参照するため、Sigma の相関変換をサポートするバックエンドでのみ変換されます。そのサポートはバックエンドごとに異なるため、`-t` の選択が重要です。以下の表は、固定した基準（`sigma-cli` 3.1.0、互換性のある最新のバックエンド）に対して測定したもので、[`tests/sigma_backends/run.sh`](tests/sigma_backends/run.sh) が再現します。

- **ツリー全体（原子 + 相関）を変換:** Splunk（`-t splunk`）、Elasticsearch EQL（`-t eql`）、Grafana Loki（`-t loki`）。`detections/sigma/` を直接変換すれば、相関クエリも一緒に得られます。
- **原子ルールのみ（相関はまだ未対応）:** Elasticsearch Lucene（`-t lucene`）、OpenSearch（`-t opensearch_lucene`）、そして Sentinel・Defender XDR を対象とする Microsoft の `kusto` バックエンド（`-t kusto`）。これらでは 5 つの原子ルールを変換し、相関の時間枠は製品内でネイティブに表現します（例: Sentinel のスケジュール分析の `summarize ... by bin(TimeGenerated, 30m)`）。ディレクトリ全体を渡すと、「Backend does not support correlation rules」で変換が止まります。

```sh
# 原子ルールのみ。例: Microsoft Sentinel / Defender（kusto バックエンド）
sigma plugin install kusto
sigma convert -t kusto --without-pipeline \
  detections/sigma/artex_enrich_user_agent.yml \
  detections/sigma/artex_selfupdate_egress.yml \
  detections/sigma/artex_guard_audit_framing.yml \
  detections/sigma/artex_recording_proxy_ca.yml \
  detections/sigma/destructive_command_hunting.yml
```

固定したバージョンで既知の境界: Elasticsearch の ES|QL ターゲット（`-t esql`）はガードマーカーのルールを拒否する（`String value expressions are not supported`）ため、そこでは残り 3 つの原子ルールを変換してください。また、IBM QRadar プラグイン（`ibm-qradar-aql`）は固定した pySigma と互換性がなく `--force-install` が必要なため、テストの対象外としています。環境にインストールされたバックエンドは `sigma list targets` で確認してください。

上記の例は `--without-pipeline` を使っているため、ルール本体の一般的なフィールド名（`cs-user-agent`・`cs-host`・`CommandLine`）がそのまま出力されます。製品のスキーマに合わせるには、そのフラグを外し、`-p` で処理パイプラインを適用してください（`sigma list pipelines` を参照）。ただし、製品のパイプラインはフィールド名を対応付けますが、ルールの一般的な `logsource` が指定していない出力先テーブルを追加で必要とする場合があります。たとえば `-p sentinel_asim` は、データに合った `query_table` を設定するまで「Unable to determine table name」で止まるため、配備の前に、フィールドと出力先テーブルを環境に合わせて対応付けてください。

## 貢献

検知とハードニングへの貢献を歓迎します。新しいルールは、すべての指標を観測可能な事実に根拠づけ、限界を `description` に明記し、SigmaHQ の検証器ベースラインをきれいに通過し（`sigma check --validation-config tests/sigma_lint/validators.yml`）、攻撃の手引きとして読めるような内容を含まないようにしてください。[`../CONTRIBUTING.md`](../CONTRIBUTING.md) を参照してください。
