# 貢献ガイド（Contributing）

日本語 · [English](CONTRIBUTING.en.md) · [한국어](CONTRIBUTING.md)

ARTEX グローバル版（`artex-global`）にご関心をお寄せいただき、ありがとうございます。この文書は、貢献を始める前に知っておいていただきたい範囲・方針・手順を日本語でまとめたものです。貢献を送る前に、必ず [使用範囲と法的責任](#使用範囲と法的責任) と [ローカライズ方針](#ローカライズ方針) をお読みください。

- バグを報告したい、または機能を提案したい場合 → [Issue テンプレート](https://github.com/jiwoochris/artex-global/issues/new/choose) をご利用ください。
- 翻訳・ローカライズの誤りを見つけた場合 → 「翻訳・ローカライズの誤り」Issue テンプレートをご利用ください。
- セキュリティの脆弱性を見つけた場合 → **公開 Issue には投稿せず**、[SECURITY.md](SECURITY.md) の手順に従ってください。
- すべての参加者は [行動規範（CODE_OF_CONDUCT.md）](CODE_OF_CONDUCT.md) を守る必要があります。

---

## 使用範囲と法的責任

ARTEX は、LLM マルチエージェントが **自律的に** ペネトレーションテストを実行する攻撃型セキュリティツールです。貢献者も、利用者と同じ範囲の制限を受けます。

- コードを検証する際は、**ご自身が所有している対象、または書面による明示的な許可を得ている対象**、あるいは **ローカルの隔離環境**（Docker で起動した OWASP Juice Shop や DVWA のような、意図的に脆弱に作られ、ご自身が所有する対象）に対してのみツールを実行してください。
- 許可された範囲を超えて、実環境・本番環境・リモートのシステムに対してスキャン・検知・エクスプロイトを行うコード、またはそうした利用を助長する変更は受け付けません。
- 大韓民国において、権限なく他者の情報通信網に侵入したり障害を引き起こしたりする行為は、「情報通信網利用促進及び情報保護等に関する法律」（정보통신망법）違反となります。収集・露出した個人情報には「個人情報保護法」（개인정보보호법）が適用されます。詳しい告知は [README](README.ja.md#️-最初にお読みください--許可された利用と法的告知) にあります。

貢献として提出されたコードや文書がどのように使われるかについての法的責任は、それを実行する利用者ご本人が負います。このリポジトリは「現状のまま（AS IS）」で提供されます。

---

## ローカライズ方針

このリポジトリの存在理由は、原本 [Autumn-27/ARTEX](https://github.com/Autumn-27/ARTEX)（現在アクセス不可、2026-10 確認）の **判断性能をそのまま保ちつつ、利用者に見える成果物だけを設定した言語に変えること** です。この方針から外れる翻訳の貢献は、性能を低下させるおそれがあるため受け付けません。

- **エージェントの内部推論プロンプト（行動指示の本体）は翻訳しないでください。** 原文（中国語）でベンチマークされた動作を維持する必要があります。この本体は `agent/promptcatalog.go` と DB シード（`agent_prompts`）にあります。翻訳はエージェントの判断にドリフトを生じさせます。
- **利用者に表示される成果物だけを、設定した言語に強制します。** 検知結果（`report_finding`）、事実の要約（`record_fact`）、最終レポート、チャット応答がこれに該当します。この強制は、`agent/prompt.go` の `langDirective()` というコードで固定された末尾指示によって、各ロールの system プロンプトの末尾に付加されます。出力言語を変更するには、この関数を修正してください。
- **コマンド・ペイロード・コード・URL・ログの原文は翻訳しません。** 分析に必要な原本なので、そのまま残します。
- **原文の中国語は保存します。** 文書は `README.zh.md`、UI の文字列は `web/messages/zh.json` に原文のまま残し、上流リポジトリの変更と照合しやすくしています。韓国語の翻訳は `web/messages/ko.json` に記入します。
- UI の文字列を新たに翻訳する際は、ハードコードせず、メッセージファイルのキーとして追加してください。
- **出力言語の強制はハードな上限ではなく、プロンプトによる誘導です。** `langDirective()` は出力言語を設定した言語にするよう **指示** するだけで、強制的に固定するものではありません。そのため、言語への忠実度はモデルの能力・ロール・文脈によって変わります。ローカライズの変更を検証する際は、能力のあるモデル（フロンティア級）を使ってください。低価格・小型のモデルでは、レポートや要約が原文（中国語）に戻ることがあるため、翻訳が正しく適用されたかを低価格モデルの出力だけで判断しないでください。OpenAI 系モデルを使う場合の `max_tokens` 設定の落とし穴は、[README の「モデル選択と出力言語」の節](README.ja.md#モデル選択と出力言語) にまとめてあります。
- **上流（upstream）の変更に追従する手順は、メンテナ向けの文書にあります。** 原本 ARTEX が更新された際に、保存すべき資産と翻訳対象を見極めて反映する方法、および翻訳の対称性とドリフトを検査する手順は、[MAINTAINING.md](MAINTAINING.md) にまとめてあります。

---

## 開発環境

このプロジェクトは、**Go バックエンド**（単一バイナリにフロントエンドを内蔵）と **Next.js フロントエンド** で構成されています。

主要機能の設計意図は `docs/` の設計文書にまとめてあります。脆弱性とトラフィックの証拠を結びつける機能（レポートエージェントの自動バインディング、`report_finding` の `traffic_refs` など）を扱う場合は、まず [脆弱性の複数トラフィック証拠の設計文書](docs/finding-traffic-evidence-ko.md) をお読みください。原文（中国語）は同じフォルダの `finding-traffic-evidence-zh.md` に保存されています。

### 必要なバージョン

- Go 1.26 以上（`go.mod` 基準）
- Node.js 22 以上（リリースワークフロー基準）
- Docker と Docker Compose（ローカルでの実行・検証用）

### バックエンド（Go）

ローカルに Go がインストールされている場合は、リポジトリのルートで次を実行します。

```bash
go build ./...
go vet ./agent/
go test ./agent/
```

ローカルに Go がない場合は、Docker で同じように検証できます。モジュールとビルドキャッシュを named volume に置くと、再実行が速くなります。

```bash
docker run --rm -v "$PWD":/src -w /src \
  -v artexko-gomod:/go/pkg/mod -v artexko-gocache:/root/.cache/go-build \
  golang:1.26 sh -c 'go build ./... && go vet ./agent/ && go test ./agent/'
```

#### DB 統合テスト（postgres が必要）

上記の `go test ./agent/` は、PostgreSQL に接続しないと実行される **DB 統合テストを黙ってスキップします。** `agent`・`config`・`db`・`evidence`・`llmrec`・`server` の 6 パッケージには、実際のデータベースがあって初めて動くテストが含まれています。環境変数 `ARTEX_PG_DSN` がなく、設定ファイルにも `database` の項目がない場合、それらのテストは `--- SKIP` となり、パッケージは `ok` で終わります。そのため、これら 6 パッケージを修正した後に DSN なしで検証すると、**ローカルでは成功（ok）しても PR の `go-db` ジョブが失敗する** ことがあります。

これらのテストをローカルで実行するには、PostgreSQL を起動し、`ARTEX_PG_DSN` を渡します。以下は、CI と同じ `postgres:16-alpine` を隔離ネットワーク上で起動して実行する例です。上記と同じ named volume を再利用します。

```bash
# 1) 隔離ネットワークと空の postgres を起動します（CI と同じイメージ・アカウント）。
docker network create artexko-db 2>/dev/null || true
docker run -d --name artexko-pg --network artexko-db \
  -e POSTGRES_USER=artex -e POSTGRES_PASSWORD=artex -e POSTGRES_DB=artex \
  postgres:16-alpine
until docker exec artexko-pg pg_isready -U artex -d artex >/dev/null 2>&1; do sleep 1; done

# 2) DSN を渡して DB 統合パッケージを実行します（DSN の host はコンテナ名です）。
#    修正したパッケージだけを実行するには ./agent/ の部分を config・db・evidence・llmrec・server に置き換えます。
docker run --rm --network artexko-db -v "$PWD":/src -w /src \
  -v artexko-gomod:/go/pkg/mod -v artexko-gocache:/root/.cache/go-build \
  -e ARTEX_PG_DSN='postgres://artex:artex@artexko-pg:5432/artex?sslmode=disable' \
  golang:1.26 sh -c 'go test ./agent/ -count=1'

# 3) 後片付けをします。
docker rm -f artexko-pg && docker network rm artexko-db
```

CI の `go-db` ジョブは、これら 6 パッケージをそれぞれ独自の postgres に隔離して、マージ前に必ず実行します（`.github/workflows/ci.yml`）。DB 統合パッケージを修正した場合は、PR を出す前に上記の方法でそのパッケージを直接確認することをお勧めします。

### フロントエンド（web）

```bash
cd web
npm ci
npm run dev          # 開発サーバー
npm run build        # 本番ビルド
npm run build:static # 静的エクスポートのビルド（マージゲート・TypeScript の型検査を含む）
npm run check        # Biome の lint・フォーマット検査（参考情報・既存の負債があるため、まだマージゲートではない）
npm run check:fix    # 自動修正
```

ローカルに Node（要求バージョン 20 以上）がない場合や、インストールされたネイティブ依存関係がプラットフォームと合わない場合は、Docker で同じように検証できます。`node_modules` を named volume に置くと、ホストのプラットフォーム用バイナリ（例: `@parcel/watcher`・Biome）と混ざらず、再実行も速くなります。このリポジトリの web 検証は、CI と同じこの方式で行っています。

```bash
docker run --rm -v "$PWD":/src -w /src/web \
  -v artexko-web-nm:/src/web/node_modules \
  node:22 bash -c 'npm ci && npm run build:static'
```

コミット前のフォーマットと lint は Biome で管理しています。`lint-staged` がステージされたファイルに対して `biome check --write` を自動的に実行します。

### 全体の実行（Docker Compose）

```bash
cp .env.example .env     # POSTGRES_PASSWORD を設定
docker compose up -d     # artex + postgres を起動 → http://localhost:8787
```

---

## 貢献手順

1. まず **Issue を立ててください。** 大きな変更は、作業を始める前に Issue で方向性を合わせておくと良いでしょう。小さな修正（誤字・リンク・明白なバグ）は、そのまま PR を送っても構いません。
2. リポジトリを **フォーク** して、トピックブランチを作成します。ブランチ名は `feat/...`、`fix/...`、`docs/...`、`i18n/...` のように、変更の性質を先頭に置きます。
3. 変更を作成し、**該当範囲の検証を自分で実行します。** Go の変更であれば、上記の `build`・`vet`・`test` を通してください。web の変更であれば、`npm run build:static`（マージゲートであり、TypeScript の型検査も同時に行います）を通してください。`npm run check`（Biome）は上流から引き継いだ既存の lint 負債が残っているため、まだマージゲートではなく、`web.yml` では参考情報としてのみ実行されます。したがって、すべてを通す必要はありません。代わりに、**自分の変更が新たなエラーを加えていないか** だけを確認してください（コミット時には、`lint-staged` がステージされたファイルにのみ `biome check --write` を自動適用します）。文書（`.md`）を変更した場合は、`python3 -I scripts/check-doc-links.py` を実行して、リポジトリ内のリンク・画像参照と、文書内のアンカー（`#見出し`）リンクが壊れていないかを確認します。アンカーは GitHub と同じ規則で見出しから slug を作って照合するため、見出しの文字を変えたのにその見出しを指していたアンカーリンクを一緒に直さないと、ここで検出されます（CI の `docs` ワークフローが同じ検査をマージゲートとして強制します）。この検査はリポジトリルートの [`.pre-commit-config.yaml`](.pre-commit-config.yaml) にも `docs` フックとして含まれているため、`pre-commit install` を実行しておくと、コミット時に自動的に実行されます（Python の標準ライブラリのみを使い、ネットワークにはアクセスしないため、Docker は不要です）。
4. **PR を開きます。** タイトルと説明は [PR テンプレート](.github/PULL_REQUEST_TEMPLATE.md) に従い、何をなぜ変えたか、どのように検証したかを記載してください。UI を変更した場合は、スクリーンショットを添付してください。
5. 利用者に見える変更（機能・ローカライズ・文書・検知ルールなど）であれば、[変更履歴（CHANGELOG.md）](CHANGELOG.md) の `[Unreleased]` の節に 1 行を追加します。内部リファクタリングやテストだけの変更は省略して構いません。

### コミットメッセージ

既存のコミット履歴の慣例に従います。形式は `type(scope): 説明` で、説明は韓国語で書きます。

- `type`: `feat` · `fix` · `docs` · `chore` · `refactor` · `test` · `i18n` など
- `scope`: 変更した領域（`agent` · `web` · `server` など）。省略可能

例は次のとおりです。

```
feat(agent): 사용자 노출 출력을 한국어로 강제 (langDirective)
docs: 한국어 README 작성, 원본은 README.zh.md 로 보존
i18n(web): 대시보드 네비게이션 라벨 한국어 번역
```

---

## 検知ルール・検知テストへの貢献

このリポジトリには、ARTEX のような自律型 AI 攻撃を **防御・検知** するためのルールも [`detections/`](detections/) に含まれています。配布可能な [Sigma](https://sigmahq.io) ルール（[`detections/sigma/`](detections/sigma/)）、ネットワーク用の [Suricata](https://suricata.io) ルール（[`detections/suricata/`](detections/suricata/)）、[MITRE ATT&CK](https://attack.mitre.org/) のカバレッジレイヤー（[`detections/attack/`](detections/attack/)）、そしてこれらのルールが実際に発火することを再現可能な形で証明するテスト（[`detections/tests/`](detections/tests/)）で構成されています。検知ルールを新たに送る場合、または修正する場合は、以下の契約を守ってください。八つのテストスイートがこの契約の多くを機械的に強制しているため、ルールだけを変更してテストやレイヤーを更新しなければ、テストが失敗します。

- **すべての指標を観測可能な事実に基づかせます。** ルールが使う文字列・User-Agent・振る舞いのしきい値は、このリポジトリのソースで実際に確認できるものでなければならず、推測で作ってはいけません。根拠となるソースファイルをルール内に明記してください（例: `artex-enrich/1.0` の指標は `enrich/enrich.go` で確認できます）。指標一致テスト（[`detections/tests/indicators/`](detections/tests/indicators/)）が、各指標が上流ソースとルールの両方に今も存在するかを検査します。そのため、上流との再同期でソースの文字列が変わった場合、ルールも一緒に修正しなければテストが失敗します。機械可読な指標リスト（[`detections/indicators/artex_indicators.csv`](detections/indicators/artex_indicators.csv)）を変更した場合は、その指標をそのまま含む MISP イベント（[`detections/indicators/artex_indicators.misp.json`](detections/indicators/artex_indicators.misp.json)）も更新してください。MISP エクスポートのテスト（[`detections/tests/misp/`](detections/tests/misp/)）が、二つのファイルが行単位で一致していること、そしてそのイベントが pymisp で読み込める有効な MISP 文書であることを強制します。
- **限界を正直に記載します。** Sigma ルールは `description` に、Suricata ルールはコメントに、そのルールが検知できない場合や誤検知の可能性を記載します。ARTEX 固有のシグネチャではなく、一般的なハンティングの手がかり（例: 破壊的コマンド）である場合は、そのように明記し、一度の一致だけで攻撃者を ARTEX と断定しないようにします。
- **静的検証を通します。** Sigma ルールは SigmaHQ の検証器の基準でエラー 0 を通過しなければなりません（`sigma check --validation-config detections/tests/sigma_lint/validators.yml`）。標準の `sigma check` は pySigma のコア検証器だけを実行するため、タイトルの表記、フィールド・ログソースの分類、参照リンクといった SigmaHQ の慣例は、この基準によってのみ検出されます。四つの例外は、単独のルールセットには合わない SigmaHQ モノレポの慣例であり、その理由は [`detections/tests/sigma_lint/validators.yml`](detections/tests/sigma_lint/validators.yml) に記載しています。Suricata ルールは `suricata -T` でエラーなく読み込まれなければなりません。
- **再現可能なテストを一緒に送ります。** ルールが発火すること（または構造が有効であること）を、[`detections/tests/`](detections/tests/) 配下のテストで証明します。入力はバイナリをリポジトリに入れず、毎回決定論的に生成します。エンジンのバージョンに依存しない性質（発火の存在・誤検知のなさ）は正確に検証し、バージョンによって揺れる数値は下限として検証して、基準値は別途記録します。Sigma の相関ルールを追加または修正する場合は、バックエンドの移植性テスト（[`detections/tests/sigma_backends/`](detections/tests/sigma_backends/)）が、そのルールが複数のバックエンドに変換されることを確認するため、[`detections/README.md`](detections/README.md) のバックエンド対応の説明と食い違わないようにしてください。
- **ATT&CK レイヤーを一緒に更新します。** ルールに `attack.*` タグを追加または変更した場合は、[`detections/attack/artex_navigator_layer.json`](detections/attack/artex_navigator_layer.json) の技術・スコアも合わせて更新してください。整合性テストがルールとレイヤーの双方向の一致を強制するため、レイヤーにないルールのタグや、ルールにないレイヤーの技術があると失敗します。
- **攻撃の手引きと読める内容を入れないでください。** このリポジトリの検知資料は、防御・検知の位置づけだけを維持します。エクスプロイトの実行方法や検知回避の手法のように、攻撃を助ける記述は受け付けません。

八つのテストスイートは Docker さえあればそのまま実行でき、生成物はリポジトリにコミットしません。各スクリプトは、一つでもアサーションが失敗すると 0 以外のコードで終了するため、CI や pre-commit フックにそのまま組み込めます。

```bash
detections/tests/sigma/run.sh           # Sigma: sigma check + バックエンド変換 + 指標の保持
detections/tests/sigma_match/run.sh     # Sigma: 原子ルールが悪性サンプルで発火し、正常サンプルで沈黙すること
detections/tests/sigma_lint/run.sh      # Sigma: SigmaHQ の慣例をすべて含む検証器 + 文書化された基準
detections/tests/sigma_backends/run.sh  # Sigma の移植性: 相関ルールが複数のバックエンドで変換されること
detections/tests/suricata/run.sh        # Suricata: pcap の合成 → suricata -r → アラート数のアサーション
detections/tests/attack/run.sh          # ATT&CK: レイヤーとルールの双方向の整合
detections/tests/indicators/run.sh      # 指標: ルールの固定指標 ↔ 上流ソースの双方向一致
detections/tests/misp/run.sh            # MISP: 指標 CSV ↔ MISP イベントの同期 + pymisp の有効性
```

八つをまとめて実行するには [`detections/tests/run-all.sh`](detections/tests/run-all.sh) を使ってください。CI と同じ順序で八つを順次実行し、前のスイートが失敗しても残りを最後まで実行したうえで、スイートごとの PASS/FAIL の要約を出力します。一つでも失敗すると 0 以外のコードで終了します。このランナーを pre-commit フックとして組み込む設定例は、リポジトリルートの [`.pre-commit-config.yaml`](.pre-commit-config.yaml) にあります。`pip install pre-commit && pre-commit install` でインストールすると、検知ルールやそのルールが固定している上流ソースが変わるコミットでのみ（CI と同じ範囲）ランナーが実行され、ルールとテストの不一致をプッシュ前に検出します。同じ設定ファイルには、文書内の内部リンク・画像・アンカーを検査する `docs` フック（上記の貢献手順 3 番の `check-doc-links.py`）も含まれています。

これら八つのテストは、リポジトリの CI（[`.github/workflows/detections.yml`](.github/workflows/detections.yml)）が、`detections/` 配下が変更されたプッシュ・PR のたびに実行します。指標一致テストは、その指標が参照している上流ソースファイル（`enrich/`・`selfupdate/`・`guard/`・`db/`・`cmd/artex/main.go`）が変わったときにも実行されるため、上流との再同期で User-Agent・マーカー・既定ポートが変わり、ルールが静かに古くなる場合も検出できます。したがって、ルールだけを変更してテストやレイヤーを更新しなかった変更、SigmaHQ の慣例を破ったルール、またはソースと食い違うルールは、マージ前に CI で赤く表示されます。

ルールの索引と、各ルールの根拠・限界は [`detections/README.md`](detections/README.md) に、テストのアサーション項目と実行方法は [`detections/tests/README.md`](detections/tests/README.md) にまとめてあります。

---

## ライセンス

このプロジェクトは **GNU Affero General Public License v3.0（AGPL-3.0）** のもとで配布されます。貢献物を提出した場合、その貢献物も **AGPL-3.0 のもとで公開されることに同意したもの** とみなされます。特に、このプロジェクトを改変してネットワーク経由で（例: オンラインサービスとして）利用者に提供する場合は、その利用者に対応する完全なソースコードを公開する必要があります。全条項は [LICENSE](LICENSE) ファイルにあります。
