# ARTEX 検知ルール（Sigma / ホスト・ログ・SIEM）

日本語 · [English](README.md) · [한국어](README.ko.md)

このディレクトリは ARTEX の検知セットのうち、ホスト・ログ・SIEM の階層を担当します。ここに収めた [Sigma](https://sigmahq.io) ルールは、防御ガイド（[韓国語版](../../docs/defense-ko.md) · [English](../../docs/defense-en.md)）の 4 節にある擬似ルールを、ベンダー中立の形式で正式化したものです。各自の SIEM・EDR のクエリ言語に変換して使ってください。すべての指標は推定ではなく、このリポジトリのソースで実際に確認した文字列や振る舞いに基づいています。ネットワーク階層は [`../suricata/`](../suricata/) にあり、ATT&CK レイヤー、指標 CSV・MISP エクスポート、ホスト分類スクリプトを含む検知セット全体は [`../README.ko.md`](../README.ko.md) が索引しています。すべてのルールは、自分が所有する、または書面による許可を得たシステムを守る **防御・検知の目的にのみ** 使用してください。

## 原子的（atomic）ルール

ルール一つが、観測可能な事実一つに対応します。個別に変換しても、ツリー全体の一部として変換しても構いません。

- **[`artex_enrich_user_agent.yml`](artex_enrich_user_agent.yml)**: *ARTEX Asset Enrichment Probe User-Agent*。資産の補強（`enrich/enrich.go`）が送る、インバウンドの `artex-enrich/1.0` User-Agent です。対象側で観測できる補助指標です。`level: high`。
- **[`artex_selfupdate_egress.yml`](artex_selfupdate_egress.yml)**: *ARTEX Self-Update Egress User-Agent*。自己更新ルーチン（`selfupdate/github.go`）が送信するアウトバウンドの `artex-selfupdate` User-Agent です。ホスト・フォレンジックの egress 指標です。`level: medium`。
- **[`artex_guard_audit_framing.yml`](artex_guard_audit_framing.yml)**: *ARTEX Platform Guard Audit-Log Framing*。ツール呼び出しがブロックされたときに監査ログへ記録される、プラットフォームガードの統制マーカーです（`guard/guard.go`）。ホスト・フォレンジックの指標です。`level: high`。
- **[`artex_recording_proxy_ca.yml`](artex_recording_proxy_ca.yml)**: *ARTEX Recording-Proxy MITM CA Certificate Artifact*。記録プロキシが `_ca/mitmproxy-ca-cert.pem` のバッチで生成する MITM CA ファイルです（`traffic/traffic.go`）。ホスト・フォレンジックの成果物ですが、ファイル名自体が単独で動く mitmproxy とも共有されるため、探索の手がかり（hunting lead）として扱います。`level: medium`。
- **[`destructive_command_hunting.yml`](destructive_command_hunting.yml)**: *Destructive Command Execution (ARTEX Guard-List Hunting)*。ARTEX ガードの組み込み拒否リスト（`db/db.go` のシード）を反映した、破壊的なシェル・DB コマンドです。ARTEX 固有のシグネチャ**ではなく**、一般的な探索の手がかりです。`level: medium`。

## 相関（correlation）ルール（振る舞いベース）· [`correlation/`](correlation/)

静的な文字列は変えられますが、振る舞いは隠しにくいものです。これらの Sigma **相関** ルールは、防御ガイドの 4.1〜4.2 節と 4.4 節にある振る舞いベースの階層を正式化したものです。各ルールは上の原子的ルールの一つを `id` で参照するため、相関ファイル一つだけではなく **`sigma/` ツリー全体を変換して** 初めて参照が解決されます（[Sigma テスト](../tests/sigma/)がまさにこの依存関係を検証しています）。

- **[`correlation/artex_enrich_scan_velocity.yml`](correlation/artex_enrich_scan_velocity.yml)**: *Enrichment Scan Velocity*。一つの送信元が短い時間枠の中で `artex-enrich/1.0` のプローブを連打する場合です（補強は並行度 4 で、速度制限なしに動きます）。単発ルールが見逃す速度を捉えます。`event_count`、`level: high`。
- **[`correlation/artex_enrich_fanout.yml`](correlation/artex_enrich_fanout.yml)**: *Enrichment Fan-Out*。一つの送信元が補強用 User-Agent を、異なる複数のホストにばらまく場合です。信号はリクエスト量ではなく、接触した異なるホストの数であり、資産一覧を機械的な速度で舐める幅を捉えます。`value_count`、`level: high`。
- **[`correlation/artex_guard_block_burst.yml`](correlation/artex_guard_block_burst.yml)**: *Guard-Block Burst*。一つのホストでプラットフォームガードの統制マーカーが繰り返し現れる場合です。マーカーを引用しただけの文書ではなく、実際に稼働中の ARTEX 実行が自身のガードに触れている状況を示します。`event_count`、`level: high`。
- **[`correlation/artex_guard_marker_then_destructive.yml`](correlation/artex_guard_marker_then_destructive.yml)**: *Guard Marker With Destructive Command*。ガードマーカーと破壊的コマンドが、一つのホストで一つの時間枠の中に一緒に現れる場合です（防御ガイド 4.2 節、多段階）。ARTEX 固有のマーカーを、本来は一般的な破壊的コマンドの信号と組み合わせるため、特異度が上がります。`temporal`、`level: high`。

閾値と時間枠は保守的な既定値です。環境のベースラインに合わせて調整してください。純粋な Web の多段階ケース（列挙 → プロービング → 認証）は、依然として環境ごとの既定ルールが必要です。そのパターンは ARTEX 固有の User-Agent 一つには還元できないためです。出発点として使える一般的な振る舞いベースの基本テンプレートは [防御ガイド 4.2 節](../../docs/defense-ko.md)にありますが、ARTEX のソースに根拠を置けないため、この検証済みツリーでは意図的に除きました。

## 範囲と正直さ：デプロイ前にお読みください

- **静的な指標は変えられます。** 運用者は User-Agent を変えたり CA ファイルを消したりできるため、原子的な指標がないからといって安全という意味では**ありません**。長持ちするシグナルは `correlation/` ルールが基準とする振る舞いです。一つの送信元が、偵察から列挙、プロービング、認証・インジェクションの試行へと進み、応答に適応し、休みなく動き続ける流れがそれです。
- **破壊的コマンドのルールは一般的な探索です。** ARTEX ガードの拒否リストを反映していますが、同じコマンドを正当な管理者も実行します。ヒットは手がかりとして扱い、自分の環境で許可リストによって除外し、それだけで ARTEX だと断定しないでください。
- **ポートとスキーマはネットワークではなく、ホスト・フォレンジックの対象です。** サーバーの既定ポート `:8787`、記録プロキシ `127.0.0.1:8788`（`cmd/artex/main.go`）、そして PostgreSQL の探索グラフスキーマは、疑わしいホストで直接確認するほうが適切です。そのため、騒がしいルールの代わりに [指標 CSV](../indicators/) と [ホスト分類スクリプト](../triage/)で提供しています。
- **`logsource` とフィールド名は一般的な値です。** ルールは一般的な `category`・`product` のログソースとフィールド名（`cs-user-agent`、`CommandLine`、`TargetFilename`）を使います。変換時にはパイプライン（`-p`）で自分の製品のスキーマへマッピングしてください。下記のバックエンドの説明を参照してください。

## 検証と変換

[sigma-cli](https://github.com/SigmaHQ/sigma-cli)（pySigma）で検証しています。リポジトリのルートで実行します。

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install sigma-cli

# 構造 + ベストプラクティスの検証（期待: 0 errors, 0 issues）
sigma check detections/sigma/

# このルールセットの文書化された基準線として、SigmaHQ の慣習全体をチェック（期待: 0 issues）
pip install pySigma-validators-sigmahq
sigma check --validation-config detections/tests/sigma_lint/validators.yml detections/sigma/

# 相関ルールが参照する原子的ルールを解決できるよう、ツリー全体を変換
sigma plugin install splunk
sigma convert -t splunk --without-pipeline detections/sigma/
```

バックエンドごとに相関ルールのサポート状況が異なるため、`-t` の選択が重要です。Splunk、Elasticsearch EQL、Grafana Loki はツリー全体を変換しますが、Elasticsearch Lucene、OpenSearch、Microsoft `kusto` バックエンドは原子的ルールの 5 つだけを変換します（時間枠は製品側でネイティブに表現します）。バックエンドごとの実測表と `--without-pipeline`・`-p` のフィールドマッピングの説明は [`../README.ko.md`](../README.ko.md) にあり、[`../tests/sigma_backends/`](../tests/sigma_backends/) がこれを再現します。

## テスト

[`../tests/`](../tests/) 以下にある、再現可能な 4 つのスイートがこれらのルールを扱います。それぞれ Docker さえあれば実行できます。[`sigma/`](../tests/sigma/) は検証とツリー全体のコンパイル、そして相関ルールが単独では変換に失敗することを検証します。[`sigma_match/`](../tests/sigma_match/) は、ルールが悪性サンプルで実際に発火し、良性サンプルでは沈黙することを確認します。[`sigma_backends/`](../tests/sigma_backends/) は 5 種のバックエンドでの移植性を、[`sigma_lint/`](../tests/sigma_lint/) は SigmaHQ 検証器の基準線全体（0 issues）を確認します。詳しくは [`../tests/README.ko.md`](../tests/README.ko.md) を参照してください。

## 貢献

検知への貢献を歓迎します。新しいルールは、すべての指標を観測可能な事実に基づかせ、限界を `description` に明記し、SigmaHQ 検証器の基準線をクリーンに通過し（`sigma check --validation-config ../tests/sigma_lint/validators.yml .`）、攻撃の手引きとして読める内容を含まないようにしてください。[`../../CONTRIBUTING.md`](../../CONTRIBUTING.md)、[`../suricata/`](../suricata/) のネットワーク階層、そして [`../README.ko.md`](../README.ko.md) を参照してください。
