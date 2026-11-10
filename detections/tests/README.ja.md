# ARTEX 検知ルールのテスト

日本語 · [English](README.md) · [한국어](README.ko.md)

[`../`](../) 配下の検知ルールが実際に発火するか、そしてそれと同じくらい重要な、良性（benign）トラフィックに対して沈黙するかどうかを、再現可能な形で証明する回帰テストです。実行して確かめられない検知ルールは、単なる主張にすぎません。これらのテストは、ルールファイルや防御ガイドに書かれた主張を、レビュアーがソースから再び実行して確認できるものに変えます。

バイナリのパケットキャプチャはリポジトリに入れていません。キャプチャは**実行のたびに決定論的に生成**し、終わったら削除するため、テストは不透明な固定ファイルではなく読めるソースとして配布され、リポジトリを肥大化させません。

## すべてのスイートを一度に実行: [`run-all.sh`](run-all.sh)

[`run-all.sh`](run-all.sh) は、以下の 8 つのスイートを CI と同じ順序で 1 つのコマンドにまとめて実行するため、8 つの `run.sh` スクリプトを手作業で一つずつ呼び出す必要はありません。前のスイートが失敗しても各スイートは最後まで実行され、スクリプトは最後にスイートごとの PASS/FAIL を 1 行ずつ要約として出力します。いずれかが失敗した場合は 0 以外のコードで終了します。

スイートを実行する前に、ハーネスの自己点検（[`check-harness-sync.sh`](check-harness-sync.sh)）を先に実行します。この点検は、上記のスイート一覧、[CI](../../.github/workflows/detections.yml) のスイートごとのステップ、そしてディスク上のスイートディレクトリ、この 3 つが異なるスイートや異なる順序を指している場合に、実行を失敗として終わらせます。これは 8 つのスイートが自分では見ることのできない唯一の死角です。3 箇所のうち 1 箇所にしか配線されていないスイート（例: `run-all.sh` の項目なしに CI のステップだけが追加されたもの、あるいはどちらにも入っていないディレクトリ）は、スイートごとのテストがすべて通っていても、ローカルで緑だった `run-all.sh` を緑の CI を意味しないものにしてしまうからです。この点検は 9 つ目のスイートではなくゲートなので、下の要約には現れず、検知スイートは 8 つのままです。

ゲートはさらに 2 つ、同じ形で実行されます。スイートのループの前に実行され、要約には現れず、スイートとしても数えません。1 つは[ホスト分類ツール](../triage/)の `--self-test`（[`triage-selftest.sh`](triage-selftest.sh)）、もう 1 つは防御テンプレートの整合性チェック（[`check-defense-template.sh`](check-defense-template.sh)）です。後者は、防御ガイドがインラインの Sigma 相関テンプレートについて主張している事実（`sigma check` の通過、splunk・eql・loki への変換、lucene・kusto での相関の拒否、韓国語版と英語版のルール本文がバイト単位で同一であること）を再び実行します。そのテンプレートは [`../sigma/`](../sigma/) のルールツリーの外側に、散文として置かれているため、上流の `sigma-cli` が上がって主張が古くなっても、他のどのスイートも検出できないからです。

```sh
detections/tests/run-all.sh
```

想定される出力（抜粋）:

```
===== detection suites summary =====
  PASS  sigma
  PASS  sigma_match
  PASS  sigma_lint
  PASS  sigma_backends
  PASS  suricata
  PASS  attack
  PASS  indicators
  PASS  misp
RESULT: PASS
```

失敗が 1 つでもあると 0 以外のコードで終了するため、そのまま pre-commit フックに組み込めます。すぐに使える例がリポジトリ最上位の [`.pre-commit-config.yaml`](../../.pre-commit-config.yaml) にあります。`pip install pre-commit && pre-commit install` でインストールすると、検知ルールやそのルールが固定している上流のソースファイルを変更するコミットで、ランナーが発火します。CI と同じ範囲です。個々のスイートが認識するイメージ・バージョンの上書き（`PYTHON_IMAGE`、`SIGMA_CLI_VERSION`、`SIGMAHQ_VALIDATORS_VERSION`、`SURICATA_IMAGE`）は、ランナーがそのまま引き継ぐため、そのいずれかを export すれば、すべてのスイートに一度に適用されます。

## Suricata: [`suricata/`](suricata/)

[`suricata/run.sh`](suricata/run.sh) は、[`../suricata/artex.rules`](../suricata/artex.rules) のネットワークルールをエンドツーエンドで実行し、5 つの性質をアサートします。

- **有効性**: ルールファイル全体を `suricata -T --init-errors-fatal` で読み込むため、下記のどのキャプチャにも触れないルールであっても、パースや初期化に失敗すれば検出します。単なる `suricata -r` はそのようなルールを飛ばしても 0 で終了するため、この読み込みチェックは、Sigma スイートの `sigma check` による有効性のアサーションに相当する Suricata 側の仕組みです。
- **存在性（強化プローブ）**: sid `1000001` が強化プローブごとにちょうど 1 回発火します。
- **速度**: ソースごとに 300 秒で 30 リクエストという `detection_filter` のしきい値を超えると、sid `1000002` が発火します。
- **存在性（WebFetch）**: sid `1000003` が norma の WebFetch リクエストごとにちょうど 1 回発火し、同じキャプチャで強化プローブの sid は沈黙します。2 つのネットワークシグネチャがそれぞれ発火するだけでなく、互いに特異的であることを確認しています。
- **特異性**: 他の条件は同じで User-Agent だけを良性のブラウザに変えたキャプチャは、ARTEX の警報を**一つも**出しません。

[`suricata/gen_pcap.py`](suricata/gen_pcap.py) は [scapy](https://scapy.net) でキャプチャを作成します。1 つの固定されたソースから出る N 個の独立した平文 HTTP のリクエスト/レスポンスのフローを、それぞれ指定した User-Agent を載せて、固定の基準タイムスタンプから 1 秒間隔で並べます。ファイルに書き出すだけで、パケットを送信したりネットワークに触れたりはしません。

### 実行

Docker さえあれば動きます。scapy と Suricata はどちらもコンテナ内で動作します。

```sh
detections/tests/suricata/run.sh
```

想定される出力（抜粋）:

```
  PASS  ruleset loads with zero parse/init errors (suricata -T)
  PASS  sid 1000001 presence: one alert per probe  (got 35, want eq 35)
  PASS  sid 1000002 velocity: fires past 30-in-300s  (got 5, want ge 1)
  PASS  sid 1000003 presence: one alert per WebFetch request  (got 8, want eq 8)
  PASS  enrich sids stay silent on norma traffic (specificity)  (got 0, want eq 0)
  PASS  benign browser UA produces no ARTEX alerts  (got 0, want eq 0)
RESULT: PASS
```

アサーションが 1 つでも失敗すると、スクリプトは 0 以外のコードで終了するため、そのまま CI や pre-commit フックに組み込めます。内部にミラーを置いている場合は、`SURICATA_IMAGE` / `PYTHON_IMAGE` でイメージを上書きしてください。

### 速度アラート数を正確な値ではなく下限でアサートする理由

`run.sh` は、存在性のアラート数（`1000001 == 35`・`1000003 == 8`）と良性のアラート数（`== 0`）は正確にアサートします。これらはエンジンのバージョンに依存しないからです。一致するリクエストごとに 1 件のアラート、別の User-Agent では不一致です。速度ルールのアラート数は、特定の Suricata リリースが境界で `detection_filter` のしきい値をどう扱うかに依存するため、テストは `>= 1` をアサートし、基準値は別途記録しています。**Suricata 8.0.7** では、基準実行が sid `1000002` に対して **5** 件のアラートを出します（300 秒に 30 のしきい値を超えた後の 31〜35 番目のフロー）。

## Sigma: [`sigma/`](sigma/)

[`sigma/run.sh`](sigma/run.sh) は [`../sigma/`](../sigma/) 配下の Sigma ルールを、構造的に、そして [sigma-cli](https://github.com/SigmaHQ/sigma-cli)（pySigma）でコンパイルして検証し、5 つの性質をアサートします。

- **有効性**: `sigma check` がツリー全体でエラー 0、条件エラー 0、問題 0 を報告します。
- **コンパイル**: `sigma convert -t splunk` がツリー全体をエラーなくバックエンドのクエリ言語に変換します。
- **指標の保持**: 各アトミックな指標文字列（`artex-enrich/1.0`、`artex-selfupdate`、ガードマーカー、そして記録用プロキシの CA ファイル名 `mitmproxy-ca-cert.pem`）が、コンパイル後のクエリにそのまま残ることを確認します。そのため、ルールが自分の根拠とする文字列を黙って失うことはありません。
- **相関ルールのコンパイル**: [`../sigma/correlation/`](../sigma/correlation/) の振る舞いルールが破棄されず、`event_count` / `value_count` の集計を出力することを確認します。
- **相関ルールが実際に機能すること**: 相関ルール 1 つだけを*単独で*変換すると失敗します。そのルールは `id` でアトミックな基盤ルールを参照しており、この参照は飾りではなく強制されるからです。これは、上記の Suricata の特異性のアサーションに相当する Sigma 側の仕組みです。

これは、[`../README.ko.md`](../README.ko.md) で説明している構造 + コンパイルによる検証を、実行可能でアサートする形にしたものです。下記の対になるスイート [`sigma_match/`](sigma_match/) が、アトミックルールと相関ルールの両方について*マッチング*の側面を追加します。代表的な悪性イベント（またはタイムライン）が各ルールを発火させ、正常なイベントは発火させないことを確認するため、Sigma ルールも Suricata ルールと同様に、再現可能な検証テストと再現可能なマッチングテストの両方を持つようになりました。（手製の粗いマッチャーがルールを骨抜きにしうるという従来の懸念は、解析をすべて pySigma に任せることで解消しました。信頼モデルは次の節で説明します。）

### 実行

Docker さえあれば動きます。sigma-cli と splunk バックエンドはコンテナ内で動作し、リポジトリには何も書き込みません。

```sh
detections/tests/sigma/run.sh
```

想定される出力（抜粋）:

```
  PASS  sigma check: 0 errors, 0 condition errors, 0 issues
  PASS  whole tree converts to splunk (exit 0)
  PASS  indicator present: artex-enrich/1.0
  PASS  correlation rule fails to convert alone — it requires its atomic base rule
RESULT: PASS
```

アサーションが 1 つでも失敗すると、スクリプトは 0 以外のコードで終了するため、そのまま CI や pre-commit フックに組み込めます。sigma-cli は基準バージョン（`3.1.0`）に固定されています。内部にミラーを置いている場合は、`SIGMA_CLI_VERSION` でバージョンを、`PYTHON_IMAGE` でイメージを上書きしてください。

## Sigma リアルタイムイベントマッチング: [`sigma_match/`](sigma_match/)

[`sigma_match/run.sh`](sigma_match/run.sh) は、[`../sigma/`](../sigma/) 配下の Sigma ルールが、アトミックルールと [`../sigma/correlation/`](../sigma/correlation/) の相関ルールの両方を含めて、マッチするイベントに対して実際に*発火*し、正常なイベントに対しては沈黙することを証明します。Suricata スイートがネットワークルールに与えている「実行して確かめられない検知ルールは主張にすぎない」という保証を、ホストやログのレイヤーのルールにまで広げたものです。アトミックルールの集合と相関ルールの集合、それぞれについて次の 6 つの性質をアサートします。

- **ルールとサンプルの対応**: すべてのアトミックルールには [`events/<名前>.json`](sigma_match/events/) のサンプルファイルがあり、すべてのサンプルファイルはルールにたどり着けます。サンプルなしで追加されたルールは、検証を受けずに通り過ぎる代わりに、ここで失敗します。
- **真陽性（true positive）**: 各ルールが自分の悪性サンプルのイベントをすべてマッチさせます。
- **真陰性（true negative）**: 各ルールが自分の正常サンプルのイベントを 1 つもマッチさせません。たとえば `.mitmproxy/` 配下の単独の `mitmproxy-ca-cert.pem` は、記録用プロキシのルールを発火させ**ません**。そのルールの `|all` 修飾子が、ARTEX が使う `_ca/` ディレクトリも併せて要求するからで、この判別を証明するのがまさにマッチングテストです。
- **相関ルールとタイムラインの対応**: すべての相関ルールには [`events/correlation/<名前>.json`](sigma_match/events/correlation/) のタイムラインファイルがあり、すべてのタイムラインはルールにたどり着けます。タイムラインの各イベントは、相対秒を持つ `ts` フィールドを備えています。
- **相関ルールの真陽性**: 時間枠の中で、あるグループがしきい値を満たす陽性のタイムラインで各ルールが発火します。たとえば、1 つのソース（`c-ip`）から 10 分以内に 20 個の異なるホストへ広がるリクエストは、収集ファンアウトのルールを発火させます。
- **相関ルールの真陰性**: しきい値未満、しきい値は満たしたが時間枠を外れている、グループが分かれている、時間相関で片方のレッグが欠けている、の各場合は沈黙します。特に、リクエスト量は多くても広がり（異なるホストの数）が小さいバーストは、ファンアウトのルールを発火させ**ません**。判定の信号は量ではなく広がりであり、この判別を証明するのがまさにマッチングテストです。

信頼モデルは次のとおりです。各ルールを解析するのは手書きのコードではなく pySigma です。アトミックルールは修飾子と条件をツリーにコンパイルし（`|contains` → ワイルドカードの値、`|all` → AND、`1 of selection_*` → OR）、相関ルールは集計仕様（種別・group-by・時間枠・しきい値の条件・参照するアトミックルール）にコンパイルします。[`check.py`](sigma_match/check.py) はそのツリーと仕様をたどるだけで、相関ルールがどのイベントを取り込むかの判定は、アトミックルールと同じマッチャーで行います。そのため、権威ある Sigma のロジックは pySigma の中に残ります。明示的にサポートしていない構文に出会った場合は、黙って通過させず例外を投げます（fail-closed）。範囲と限界はスクリプトの冒頭に明記しています。相関ルールの時間枠は、標準的なスライディングウィンドウ（マッチするイベントごとに `timespan` の長さの窓を取る）として解釈しており、実際の SIEM のウィンドウの方式とは異なる場合があります。マッチングは**大文字小文字を区別しません**（`sigma/` スイートが対象とする splunk バックエンドの既定値であり、破壊的コマンドのルールの誤検知に関するコメント自体がこれを前提としています）。キーワードのマッチングは全文の部分文字列検索です。これはルールのフィールド・値・条件・集計のロジックに対する回帰テストであり、フィールドの正規化が異なる可能性のある各自の SIEM での検証を代替するものではありません。

### 実行

Docker さえあれば動きます。pySigma はコンテナ内で動作し、リポジトリには何も書き込みません。

```sh
detections/tests/sigma_match/run.sh
```

想定される出力（抜粋）:

```
  PASS  rule/sample pairing: 5 atomic rules, 5 event files, no orphans
  PASS  artex_enrich_user_agent: 1/1 positive events matched
  PASS  artex_recording_proxy_ca: 2/2 benign events correctly not matched
  PASS  rule/timeline pairing: 4 correlation rules, 4 timeline files, no orphans
  PASS  artex_enrich_fanout: fired — 20 distinct hosts from one source within the 10-minute window
  PASS  artex_enrich_fanout: quiet — high volume, low breadth: 25 requests from one source but only 4 distinct hosts
RESULT: PASS
```

アサーションが 1 つでも失敗すると、スクリプトは 0 以外のコードで終了するため、そのまま CI や pre-commit フックに組み込めます。pySigma は基準バージョン（`2.0.0`）に固定されています。内部にミラーを置いている場合は、`PYSIGMA_VERSION` でバージョンを、`PYTHON_IMAGE` でイメージを上書きしてください。

## Sigma バックエンドの移植性: [`sigma_backends/`](sigma_backends/)

[`sigma_backends/run.sh`](sigma_backends/run.sh) は、ルールが Sigma テストで実際に実行している単一の Splunk の例を超えても変換できることを証明し、[`../README.ko.md`](../README.ko.md) にあるバックエンドごとのサポート表を正直に保ちます。Sigma の相関ルールの変換はバックエンドによって異なるため、README は、どの `-t` のターゲットがツリー全体を受け付け、どのターゲットがアトミックルールだけを受け付けるかを防御者に伝えています。これは再び実行して初めて信頼できる主張です。2 つの性質をアサートします。どちらも肯定形で、実際に回帰があった場合にのみ失敗します。

- **相関ルールの移植性**: ツリー全体（アトミック + 相関）が Splunk、Elasticsearch の `eql` ターゲット、Grafana の `loki` で変換され、強化の指標が各クエリにそのまま残ります。相関ルールが Splunk 専用ではないことを示しています。
- **アトミックのみのフォールバックの動作**: 5 つのアトミックルールは、`lucene` と Microsoft の `kusto` バックエンドでも変換されます。これらのバックエンドは、固定されたバージョンでは Sigma の相関ルールの変換をサポートしていないため、そのバックエンドを使う防御者はアトミックルールを配備し、相関の時間枠はそのバックエンド固有の機能で表現できます。

「バックエンド X は相関ルールを扱えない」という否定形の主張は、意図的にアサートしていません。そうすると、バックエンドが*改善*されたときに赤いビルドになってしまうからです。正直な限界は README に記載しており、このテストのコマンドがそれを再現します。[`sigma_backends/check.sh`](sigma_backends/check.sh) はコンテナ内の半分です。固定された sigma-cli と 4 つのバックエンドをインストールし、読み取り専用でマウントしたルールツリーを読み込みます。

### 実行

Docker さえあれば動きます。sigma-cli とバックエンド群はコンテナ内で動作し、リポジトリには何も書き込みません。

```sh
detections/tests/sigma_backends/run.sh
```

想定される出力（抜粋）:

```
  PASS  whole tree (atomic + correlation) converts on 'eql', enrich indicator survives
  PASS  five atomic rules convert on 'kusto', enrich indicator survives
RESULT: PASS
```

アサーションが 1 つでも失敗すると、スクリプトは 0 以外のコードで終了します。sigma-cli は固定されており（`3.1.0`、`SIGMA_CLI_VERSION` で上書き可能）、バックエンドのプラグインは互換性のある最新版でインストールされます。そのため、このスイートは上流のバックエンドのリリースに最も敏感です。サポートを落としたプラグインはビルドを赤くしますが、これは固定バージョンと README の表を合わせて更新せよという合図です。

## SigmaHQ 慣習リント: [`sigma_lint/`](sigma_lint/)

[`sigma_lint/run.sh`](sigma_lint/run.sh) は、README と `CONTRIBUTING.md` にある「`sigma check` をきれいに通過する」という約束に、pySigma のコアなチェックだけでなく SigmaHQ の慣習も含まれるようにします。素の `sigma check` は `pySigma-validators-sigmahq` プラグインを読み込まないため、タイトルの大文字小文字、フィールド名の分類体系、ログソースの分類体系、参照リンクの慣習が検査されずに通り過ぎてしまいます。このスイートはそのプラグインをインストールし、[`sigma_lint/validators.yml`](sigma_lint/validators.yml) に文書化したベースラインに合わせて、検査セット全体を実行します。2 つの性質をアサートします。

- **文書化したベースラインがクリーン**: `validators.yml` とともに `sigma check` を実行すると、エラー 0、問題 0 を報告します。
- **検査セット全体が生きており、文書化した除外だけが残る**: 除外なしにすべての SigmaHQ バリデーターを実行しても問題が報告されますが、その各問題は、`validators.yml` が意図的にオフにしている 4 つのチェックのいずれかに該当します（それ以外はありません）。これは空振りを防ぐガードです。プラグインの読み込みに失敗していれば、フルの実行は何も報告せず、1 つ目の性質が誤った理由で通ってしまうため、既知の除外が確実に現れることを要求します。

4 つの除外は、SigmaHQ のモノレポのファイル整理の仕方（ログソースの接頭辞を付けたファイル名と `correlation_` のファイル名）と、分類体系（汎用の `application` ログソース、製品名のない `process_creation`）、そしてブランチ参照ではなく永続リンク（permalink）を使う参照の慣習を反映しています。いずれも、自分のリポジトリにある生きたドキュメントを参照する、小さく独立したルール集合には当てはまりません。各除外の根拠は `validators.yml` 内に併記しています。*それ以外の*すべての SigmaHQ のチェックは強制されるため、新しい慣習の問題を含むルール（大文字小文字の誤ったタイトル、分類体系から外れたフィールド名）はビルドを赤くします。`pySigma-validators-sigmahq` は固定されており（`0.21.0`、`SIGMAHQ_VALIDATORS_VERSION` で上書き可能）、バージョンを上げると新しい慣習が表面化することがありますが、それはルールや文書化したベースラインを更新せよという合図です。

### 実行

Docker さえあれば動きます。sigma-cli と検証プラグインはコンテナ内で動作し、リポジトリには何も書き込みません。

```sh
detections/tests/sigma_lint/run.sh
```

想定される出力（抜粋）:

```
  PASS  sigma check with the documented baseline: 0 errors, 0 issues
  PASS  every reported issue is one of the four documented exclusions
RESULT: PASS
```

## ATT&CK レイヤー: [`attack/`](attack/)

[`attack/run.sh`](attack/run.sh) は、[`../attack/artex_navigator_layer.json`](../attack/artex_navigator_layer.json) の [ATT&CK カバレッジレイヤー](../attack/)が、そのカバーすると主張するルールと食い違っていないかを確認します。ルール集合と食い違ったカバレッジレイヤーは、ないほうがましなので、このテストは「これらのルールがこれらの ATT&CK テクニックをカバーする」を、レビュアーがソースから再び実行して確認できるものに変えます。アサートするのは次のとおりです。

- **有効なレイヤー**: ファイルが JSON としてパースでき、必須の ATT&CK Navigator v4.x フィールドを備えており、すべての項目が正しい形式のテクニック ID と有効な ATT&CK 戦術を持ちます。
- **双方向の一致**: スコアを付けたテクニックが、Sigma ルールの `attack.*` テクニックタグと*正確に*一致します。レイヤーに欠けているルールのテクニックも、ルールにないレイヤーのテクニックもありません。戦術も同様に一致します。
- **根拠がある**: スコアを付けたすべてのテクニックのコメントが、実在するルールファイルを指しているため、レイヤーが名前を変えられたり削除されたりしたルールを引用することはできません。

これは発火テストではなく一貫性チェックです。検知バックエンドは不要で、Python の標準ライブラリがあれば十分です。そのため、Sigma や Suricata のテストと違って、バージョンに依存するアラート数はありません。[`attack/check.py`](attack/check.py) はコンテナ内の半分です。読み取り専用でマウントした検知ツリーを読み込み、何も書き込みません。

### 実行

Docker さえあれば動きます。チェックは Python のコンテナで動作し、リポジトリには何も書き込みません。

```sh
detections/tests/attack/run.sh
```

想定される出力（抜粋）:

```
  PASS  scored techniques match the rule set exactly (8: T1059, T1105, T1485, T1489, T1557, T1561.002, T1592, T1595)
  PASS  scored tactics match the rule set exactly (collection, command-and-control, credential-access, execution, impact, reconnaissance)
RESULT: PASS
```

アサーションが 1 つでも失敗すると、スクリプトは 0 以外のコードで終了するため、そのまま CI や pre-commit フックに組み込めます。内部にミラーを置いている場合は、`PYTHON_IMAGE` でイメージを上書きしてください。

## 指標の根拠（source-of-truth）: [`indicators/`](indicators/)

[`indicators/run.sh`](indicators/run.sh) は、上記の 3 つのテストではできない 1 つのことを証明します。各ルールが固定している指標が、今もなお ARTEX 自身のソースが実際に出力する文字列であるかどうかです。Sigma のテストは、指標がルールからクエリへの*コンパイル*を経ても残ることを証明し、ATT&CK のテストは、レイヤーがルールのタグと一致することを証明し、Suricata のテストは、ネットワークルールが生成したキャプチャで*発火*することを証明します。しかし、指標がどのソースファイルに由来すると主張しているのかを、さかのぼって確認するものはありません。これらすべてが見落とす劣化は、プローブの User-Agent を `artex-enrich/2.0` に上げたり、ガードマーカーを書き直したりする上流の再同期です。そのときもルールはすべてコンパイルされ、レイヤーは依然として一致し、pcap テストも依然として発火します。しかし配備されたルールは、実際の ARTEX のトラフィックに対して黙ってマッチを止めてしまいます。各指標について、次の双方向のアサーションを行います。

- **ソースが今も出力している**: 値が、それを生成する上流のソースファイルに存在します（`enrich/enrich.go` の `artex-enrich/1.0`、`selfupdate/` の `artex-selfupdate`、`guard/guard.go` のガードマーカー）。値がなければ、ルールがまだ追いついていない上流の変更を意味します。
- **ルールが今も固定している**: 値が、その上に構築されたルールに存在するため、ルールの編集が指標をソースから黙って切り離すことはできません。Suricata のルールは `startswith` の接頭辞で確認します。これは、そのルールが実際にワイヤー上で照合する方法と同じです。
- **ブロックリストとの対応**: 破壊的コマンドのトークン（`rm -rf`、`mkfs`、`DROP DATABASE`、`FLUSHALL`）が、ARTEX のガードのブロックリスト（`db/db.go`）と、それを反映したハンティングルールの両方に現れます。これらは固有の指紋ではなく一般的なハンティングの手がかりなので、テストはルールが実際に主張する対応関係だけをアサートします。
- **公開リストが根拠を保つ**: 防御者が取り込む成果物である、機械可読な指標リスト [`detections/indicators/artex_indicators.csv`](../indicators/artex_indicators.csv) を行単位で読み直します。すべての値は、引用したソースファイルに今も存在し、引用したルールに固定されていなければならず、テストが根拠を確認したすべての指紋がこのリストに現れなければなりません。そのため、公開された CSV が、自分が由来すると主張するソースからどの方向にも黙ってずれることはできません。
- **固定された各ソースで両ゲートが発火する**: テストが読むすべての上流ソースは、それを実行する 2 つのゲートに含まれます。CI ワークフローの `push`・`pull_request` の paths フィルタ（[`.github/workflows/detections.yml`](../../.github/workflows/detections.yml)）と、ローカルの pre-commit フックの `files` 正規表現（[`.pre-commit-config.yaml`](../../.pre-commit-config.yaml)）です。必要な集合は指標そのものから導出されるため、新しいソースを固定しながら（以前の `cmd/artex/main.go` の移植がそうだったように）*両方の*ゲートに配線しなければ、ここで失敗します。そうしないと、そのソースだけを変更したコミットは、そのソースを取りこぼしたゲートではテストが飛ばされてしまいます。CI では、マージゲートを緑で通過させ、フックでは「CI と同じソース範囲」と約束しておきながら、ローカルでは最後まで捕まらないことになります。

これは [`../README.ko.md`](../README.ko.md) の約束（「ここにあるすべての指標は推測ではなく、このリポジトリのソースで確認した文字列に根拠を置く」）と、CONTRIBUTING の最初の貢献契約を、レビュアーが再び実行して確認できるガードに変えるものです。ATT&CK のテストと同様に、検知バックエンドは不要で、Python の標準ライブラリがあれば十分です。[`indicators/check.py`](indicators/check.py) は、ルールツリー、公開された指標リスト、固定されたソースパッケージ、そしてそれを発火させる 2 つのゲート（CI ワークフローと pre-commit 設定）を、読み取り専用でマウントして読み込み、何も書き込みません。

### 実行

Docker さえあれば動きます。チェックは Python のコンテナで動作し、リポジトリには何も書き込みません。

```sh
detections/tests/indicators/run.sh
```

想定される出力（抜粋）:

```
  PASS  enrichment prober User-Agent: 'artex-enrich/1.0' emitted by enrich/enrich.go
  PASS  detections/sigma/artex_enrich_user_agent.yml pins 'artex-enrich/1.0'
  PASS  'FLUSHALL' present in both db/db.go and detections/sigma/destructive_command_hunting.yml
  PASS  enrich-user-agent: 'artex-enrich/1.0' grounded in enrich/enrich.go
  PASS  tested fingerprint 'artex-enrich/1.0' is published in the list
  PASS  .github/workflows/detections.yml push paths covers cmd/artex/main.go
  PASS  .pre-commit-config.yaml files covers cmd/artex/main.go
RESULT: PASS
```

アサーションが 1 つでも失敗すると、スクリプトは 0 以外のコードで終了するため、そのまま CI や pre-commit フックに組み込めます。内部にミラーを置いている場合は、`PYTHON_IMAGE` でイメージを上書きしてください。

## MISP エクスポートの一貫性: [`misp/`](misp/)

[`misp/run.sh`](misp/run.sh) は、指標の 2 つ目の公開形式、すぐに取り込める MISP イベント [`detections/indicators/artex_indicators.misp.json`](../indicators/artex_indicators.misp.json) を扱います。上記の指標テストが CSV をソースに根拠づけたままにするのに対し、このテストは、防御者が実際に脅威インテリジェンス基盤に取り込む成果物である MISP イベントが、CSV からずれないようにします。アサートするのは次のとおりです。

- **本当に MISP である**: イベントは [pymisp](https://github.com/MISP/PyMISP) で取り込まれます。そのオブジェクトモデルは、`type` が実際の MISP タイプではない属性を拒否します。もっともらしく見えても無効なタイプは、ここで失敗するため、「有効な MISP」であることは単に主張されるのではなく、MISP サーバーが使うのと同じライブラリによって証明されます。
- **CSV と行単位で同期している**: すべての CSV の行が、意図したタイプ・カテゴリを持つ MISP 属性にちょうど 1 つ対応します（`http.user-agent` → `user-agent`、ガードマーカーの `string` → `pattern-in-file`、`port` → `port`、`ip-dst|port` → 合成値 `ip|port` を持つ `ip-dst|port`、探索スキーマの `other` → `other`）。また、CSV の行に対応しない MISP 属性は 1 つもありません。イベントは CSV と並べて手作業で管理しているため、`artex_indicators.misp.json` を同じコミットで合わせて更新せずに CSV の行を追加・削除・タイプ変更すると失敗します。
- **`to_ids` が `rule` 列を反映している**: ルールに裏付けられた指標は `to_ids: true` であり、ルールのないホストのフォレンジック行は `disable_correlation: true` とともに `to_ids: false` です。CSV が含意する内容と異なるようにフラグを反転させると失敗するため、MISP イベントが、どの指紋が実行可能かを黙って水増ししたり減らしたりして主張することはできません。
- **ガードマーカーがバイト単位で保持され**、`detections/**` が CI の paths フィルタに含まれているため、CSV やイベントを変更すると、このスイートが発火します。

上記の純粋な標準ライブラリのテストと違い、このスイートはコンテナ内に固定された `pymisp` をインストールします（ホストには何もインストールしません）。[`misp/check.py`](misp/check.py) は CSV、MISP イベント、CI ワークフローを読み取り専用でマウントして読み込み、何も書き込みません。

### 実行

Docker さえあれば動きます。pymisp はコンテナにインストールされ、リポジトリには何も書き込みません。

```sh
detections/tests/misp/run.sh
```

アサーションが 1 つでも失敗すると、スクリプトは 0 以外のコードで終了します。内部にミラーを置いている場合は、`PYTHON_IMAGE` でイメージを、`PYMISP_VERSION` で固定されたライブラリを上書きしてください。

## 貢献

新しい検知ルールは、それが発火することを示すテストがあると、より強固になります。テストは自分の入力を決定論的に生成し、エンジンのバージョンに依存しない性質は正確にアサートし（それより弱い性質は、基準値を記録した下限で）、攻撃の手引きとして読めてしまう内容は避けます。[`../../CONTRIBUTING.md`](../../CONTRIBUTING.md) と、[`../README.ko.md`](../README.ko.md) のルール索引を参照してください。

8 つのスイートはすべて、`detections/` に触れるすべての push や pull request で CI によって実行されます（[`../../.github/workflows/detections.yml`](../../.github/workflows/detections.yml) を参照）。さらに、指標テストは、それが固定している上流のソースファイル（`enrich/`、`selfupdate/`、`guard/`、`db/`、`cmd/artex/main.go`）が変更されたときにも実行されます。そのため、指標を落とす、ATT&CK レイヤーとずれる、文書化したバックエンドで変換が止まる、SigmaHQ の慣習を破る、ソースとの同期がずれる、MISP イベントを CSV とずれたままにする、あるいはワークフローがまだ監視していない新しいソースを固定する、といったルールの変更は、マージされる前にビルドを赤くします。
