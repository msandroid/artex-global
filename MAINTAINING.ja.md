# 上流同期と翻訳ドリフトの防止（メンテナ向けガイド）

日本語 · [English](MAINTAINING.en.md) · [한국어](MAINTAINING.md)

この文書は、**メンテナ** が原本リポジトリ [Autumn-27/ARTEX](https://github.com/Autumn-27/ARTEX)（現在アクセス不可、2026-10 確認）の変更に追従しながら、多言語ローカライズを維持するための手順をまとめたものです。貢献の範囲・法的責任・ローカライズ方針は [CONTRIBUTING.ja.md](CONTRIBUTING.ja.md) に、利用者向けの案内は [README.ja.md](README.ja.md) にあるため、この文書は、その方針を **実際にどのように運用するか** に集中します。

ローカライズの核心目標は、一文でまとめられます。原本の **判断性能をそのまま保ちつつ、利用者に見える成果物だけを設定した言語に変えること** です。上流が更新されるたびにこの境界は崩れやすいため、以下の手順と検査で翻訳ドリフトを防ぎます。

---

## 1. ローカライズの構造を一目で見る

このリポジトリは、上流 ARTEX を **フォーク** し、その履歴の上に韓国語のローカライズコミットを積み重ねた構造です。上流 `main` のすべてのコミットがこのリポジトリの履歴に含まれており、その上にローカライズコミットが追加されています。したがって、上流の変更を取り込む作業は、「上流 `main` との差分を確認し、保存すべきものと翻訳すべきものを見極めて反映する作業」になります。

成果物は三つに分かれます。

- **原文のまま残す資産**（下記 2 節）。翻訳すると性能や上流との照合が壊れます。
- **コードで固定された出力言語の強制**。`agent/prompt.go` の `langDirective()` が、各ロールの system プロンプトの末尾に「ユーザーに見える出力は韓国語で書け」という指示を付け加えます。
- **韓国語に翻訳するユーザー向けの文字列**。UI は `web/messages/ko.json` に、サーバーのユーザー応答文言は各 Go ファイルの名前付き定数に置きます。

---

## 2. 原文を保存する資産（翻訳禁止）

次の資産は翻訳せず、原文（中国語または英語）のまま維持します。上流の変更がこれらの資産に触れた場合は、**翻訳せずにそのまま反映** します。

- **エージェントの内部推論プロンプト（推論プロンプト本体）。** `agent/promptcatalog.go` と DB シード `agent_prompts` にある行動指示の本体です。原文（中国語）でベンチマークされた動作を維持する必要があるため、翻訳すると判断にドリフトが生じます。
- **表示とエージェントへの入力を兼ねる文字列。** アクティビティタイムラインに表示されると同時に、プランナーやレポーターの入力コンテキストにフィードバックされる一部の文言（作業中断の理由、インターセプトの遮断メッセージ、トラフィック証拠のヘルパーなど）は、一つのレコードが二つの用途を兼ねるため、原文を保存します。判定の根拠は `work/DECISIONS-FOR-JIWOO.md` の「推論プロンプトの境界記録」に案件ごとに記載されています。
- **原本の中国語の文書・文字列。** 文書は `README.zh.md`、UI の文字列は `web/messages/zh.json` に原文のまま残し、上流の変更と照合しやすくします。韓国語の翻訳は `web/messages/ko.json` にのみ記入します。
- **コマンド・ペイロード・コード・URL・識別子・ログの原文。** 分析に必要な原本なので翻訳しません。Go コードのコメントも優先順位が最も低く、上流との照合が終わるまで原文のままにします。

---

## 3. 上流追跡の基準点

上流のリモートが次のように設定されている必要があります。なければ追加してください。

```bash
git remote add upstream https://github.com/Autumn-27/ARTEX
git remote -v   # upstream が表示されることを確認
```

現在、ローカライズへの反映を済ませた上流の基準コミットは次のとおりです。

- **基準 = `d003372`**（上流 `main`、2026-10-03、PR #189 `fix/sse-same-origin` のマージ）。

この値は「このコミットまでの上流の変更はすべてこのリポジトリに取り込まれている」ことを意味します。上流の変更を新たに反映するたびに、この基準を 7 節の方法で更新します。

---

## 4. 上流の変更を取り込む手順

### 4.1 上流を取得し、差分を確認します

```bash
git fetch upstream
git rev-list --count d003372..upstream/main        # 未反映の上流コミット数
git log --oneline d003372..upstream/main           # 未反映のコミット一覧
```

`git fetch` は上流のリモート追跡ブランチだけを更新するため、作業ツリーと `HEAD` には影響しません。未反映のコミットが 0 であれば、上流と同期済みの状態なので、これ以上の作業はありません。

### 4.2 変更されたファイルを分類します

未反映のコミットがどのファイルに触れたかを確認し、2 節の保存資産と翻訳対象に分けます。

```bash
git log --name-status --oneline d003372..upstream/main
```

分類の基準は次のとおりです。

- `agent/promptcatalog.go`・`agent_prompts` のシード、および 2 節の表示兼入力の文字列が変わった場合 → **翻訳せずにそのまま反映** します。
- Go バックエンドのロジック（`db/`・`llmrec/`・`server/` など）が変わった場合 → ロジックはそのまま反映しますが、**新たに追加されたユーザー応答の文言**（`writeErr` など）がないか確認し、韓国語の定数に翻訳します。
- UI（`web/src/**`）が変わり、**新しい画面の文字列** が生まれた場合 → ハードコードせず、`web/messages/zh.json`（原文）と `web/messages/ko.json`（翻訳）に同じキーで追加します。
- **検知ルールが固定している上流の指標**（`enrich/enrich.go` のプローブ User-Agent、`selfupdate/` の自己更新 User-Agent、`guard/guard.go` の監査マーカー、`db/db.go` の破壊的コマンドの deny リスト、`cmd/artex/main.go` の既定のリッスン・記録プロキシのポート）が変わった場合 → `detections/` の Sigma・Suricata ルールと ATT&CK レイヤー、そして `detections/indicators/artex_indicators.csv` の値も新しい値に合わせます。これらの指標は翻訳対象ではなく **検知の根拠** であり、上流が値を変えるとルールが静かに古くなります。4.2 の最後の分類基準に従い、5.4 の指標一致テストがこの食い違いを自動的に検出します。

### 4.3 反映します

機能単位でマージするか、選別して反映したうえで、4.2 で見極めた新しい文字列を韓国語に翻訳します。マージの過程で `ko.json`・`zh.json` のキーが食い違ったり、ユーザーに見える箇所に原文が混入したりしやすいため、反映直後に必ず 5 節の検査を実行してください。

> **例（2026-10-05 時点の未反映コミット）。** `git fetch upstream` の結果、上流 `main` は `b55ceb1` まで進んでおり、基準 `d003372` と比べて 2 件のコミット（`86729b6` モデルフォールバック承認トークンの計量機能 + マージコミット `b55ceb1`）が未反映です。このコミットは `db/llm_usage.go`・`llmrec/llmrec.go`・`server/intercept.go`・`server/server.go` といった Go ロジックと、`web/src/app/(main)/system/intercept/page.tsx`・`web/src/lib/api.ts`・`web/src/lib/mock/handler.ts`・`web/src/lib/types.ts` に触れています。したがってメンテナは、Go ロジックはそのまま反映し、intercept 設定ページに新しく生まれた画面の文字列だけを `ko.json`・`zh.json` のキーとして抽出・翻訳すれば十分です。（この二つのコミットは、この文書を書いた時点ではまだ反映していないため、基準は `d003372` のままにしてあります。）

---

## 5. 翻訳の対称性と翻訳ドリフトの検査

上流の反映や翻訳作業の後には、次の五つを確認します。

### 5.1 ko ↔ zh メッセージの対称性と利用者に見える CJK

`ko.json` と `zh.json` のキーが正確に一致し、`ko.json` の値に中国語の漢字が残っていないことを確認します。次のスクリプトが三つの数値を出力します。

```bash
python3 - <<'PY'
import json, re
ko = json.load(open('web/messages/ko.json'))
zh = json.load(open('web/messages/zh.json'))
def flatten(d, p=''):
    out = {}
    if isinstance(d, dict):
        for k, v in d.items(): out.update(flatten(v, p + '/' + k))
    elif isinstance(d, list):
        for i, v in enumerate(d): out.update(flatten(v, p + '/' + str(i)))
    else: out[p] = d
    return out
fk, fz = flatten(ko), flatten(zh)
han = re.compile(r'[㐀-鿿]')
print('ko leaf keys :', len(fk))
print('zh leaf keys :', len(fz))
print('key symdiff  :', len(set(fk) ^ set(fz)))        # 0 であるべき
print('ko vals w/CJK:', sum(1 for v in fk.values() if isinstance(v, str) and han.search(v)))  # 0 であるべき
PY
```

基準値（2026-10-05）: `ko leaf keys = 2950`、`zh leaf keys = 2950`、`key symdiff = 0`、`ko vals w/CJK = 0`。キーの数は上流の反映によって増えることがありますが、ko と zh は常に同じで、`key symdiff` と `ko vals w/CJK` は常に 0 でなければなりません。

### 5.2 推論プロンプト資産の原文保存の確認

推論プロンプト本体は中国語の原文を維持するため、次の検査で **漢字の行数が 0 に落ちた場合**、推論プロンプトが誤って翻訳され汚染された兆候です。

```bash
python3 -c "import re; han=re.compile(r'[㐀-鿿]'); t=open('agent/promptcatalog.go').read(); print('promptcatalog.go CJK lines =', sum(1 for l in t.splitlines() if han.search(l)))"
```

基準値（2026-10-05）: `promptcatalog.go CJK lines = 70`。この数が大きく減っていれば、推論プロンプト本体が翻訳されていないかを確認します。

### 5.3 ビルド成果物に原文が漏れていないか

UI を静的にエクスポートした後、プリレンダリングされた HTML に中国語が見えていれば、翻訳漏れです。

```bash
cd web && npm ci && NEXT_EXPORT=1 npm run build   # out/ を生成
# out/**/*.html の可視テキストに含まれる中国語の漢字が 0 であることを確認
```

### 5.4 検知指標が上流ソースと今も一致しているか

`detections/` のルールは、上流が実際に出力する文字列（プローブの User-Agent、自己更新の User-Agent、監査マーカー、破壊的コマンドの deny リスト）を根拠にしています。上流の再同期によってこれらの値が変わると、翻訳の検査はすべて通るのに、配布されたルールだけが静かにマッチしなくなります。次のテストは、各指標が上流ソースとルールの両方に今も存在するかを双方向で確認するため、再同期の後に併せて実行してください。

```bash
detections/tests/indicators/run.sh   # Docker で隔離実行。RESULT: PASS なら一致
```

失敗した場合は、どの指標が食い違ったか、そしてその方向（上流のソースが変わったのか、ルールが変わったのか）を出力するため、4.2 の最後の分類基準に従って、ルールとレイヤーを新しい値に合わせます。このテストは、リポジトリの CI（[`.github/workflows/detections.yml`](.github/workflows/detections.yml)）でも、ルールツリーや上記の上流ソースファイルが変更されたプッシュ・PR のたびに自動的に実行され、再同期によるドリフトをマージゲートで検出します。

新しい指標を追加する際に **新しい上流ソースファイルを固定した場合**（例: `cmd/artex/main.go` のポート指標を追加したときのように）、そのファイルをワークフローの `push`・`pull_request` の `paths` フィルタにも必ず追加してください。追加し忘れると、そのソースだけを変更した PR では指標テストが発火せず、ドリフトがマージゲートを静かに通過してしまいます。この同期も指標テストが自動で確認します（五番目の検査 "CI triggers this test when any pinned source changes"）。テストが読む `detections/` 以外のソースがすべて両方の `paths` ブロックに列挙されていなければテストが失敗するため、ソースの固定と CI の発火条件が食い違ったまま マージされることはありません。

### 5.5 検知テストのツール固定版を上げるとき

検知テストは、`sigma-cli`・SigmaHQ 検証器プラグイン（`pySigma-validators-sigmahq`）・Suricata イメージを固定バージョンで実行します（各 `run.sh` の既定値。環境変数で上書きできます）。この固定版を上げると、上流ソースではなく **ツール側のドリフト** が生じることがあります。特に SigmaHQ の検証器はバージョンアップのたびに新しい慣例チェックを追加するため、`detections/tests/sigma_lint/run.sh` が新しい問題を赤く示すことがあります。その場合は、ルールを新しい慣例に合わせるか、単独のルールセットに合わない慣例であれば、その理由を [`detections/tests/sigma_lint/validators.yml`](detections/tests/sigma_lint/validators.yml) の除外リストに追記します。バックエンドのプラグインが対応を変えた場合は、`sigma_backends` テストが同じシグナルを出します。

---

## 6. ビルドとテストによる最終検証

反映・翻訳の後は、[CONTRIBUTING.ja.md の開発環境](CONTRIBUTING.ja.md#開発環境) の手順どおりに、バックエンドとフロントエンドを検証します。ローカルに Go がない場合は、Docker で同じように実行できます。

```bash
docker run --rm -v "$PWD":/src -w /src \
  -v artexko-gomod:/go/pkg/mod -v artexko-gocache:/root/.cache/go-build \
  golang:1.26 sh -c 'go build ./... && go vet ./... && go test ./... -count=1'
```

利用者に見える文言を翻訳する際は、その文言を検証する回帰テスト（`*_localized_test.go`）も一緒に用意しておき、後の上流変更が再び中国語を持ち込んだときにテストで検出できるようにします。翻訳の検証は必ず能力のある（フロンティア級の）モデルで行ってください。低価格・小型のモデルは出力が原文に戻ることがあるため、翻訳が適用されたかをその出力だけで判断してはいけません。

---

## 7. 基準の更新記録

上流の変更を反映し、検証まで完了したら、この文書の 3 節にある **基準コミットの値を新しい上流コミットに更新** し、その変更を同じコミットか後続のコミットに含めます。こうしておけば、次のメンテナは「どこまで反映されたか」をこの文書一か所で確認できます。

コミットメッセージは [CONTRIBUTING.ja.md のコミットメッセージの規則](CONTRIBUTING.ja.md#コミットメッセージ) に従います。たとえば、上流同期のコミットは次のように記載します。

```
chore(upstream): 상류 d003372..b55ceb1 반영 (intercept 토큰 계량) + 신규 UI 문자열 번역
```

---

## 8. レビューと検証の規則（よくある落とし穴）

上流の反映・翻訳・文書の補強を点検する際に、メンテナが繰り返し陥りやすい落とし穴を二つ記しておきます。どちらも「検査の方法そのものが誤っていて、問題のないものを壊れたと誤認する」ケースなので、不要な差し戻しを防ぐために規則として固定します。

### 8.1 リポジトリの CI 状態は、リポジトリを指定して確認します

このリポジトリは上流 ARTEX のフォークであるため、ローカルの `git remote` には `origin`（jiwoochris/artex-global）と `upstream`（Autumn-27/ARTEX）の両方が登録されています（3 節参照）。この状態で `gh` コマンドにリポジトリを指定しないと、`gh` が **上流リポジトリを既定値として選び**、私たちのワークフローが存在しない上流の実行結果を表示することがあります。すると、上流の CI が緑であるのを見て **私たちの CI が通ったと誤解** したり、私たちのワークフロー（`ci.yml`・`detections.yml`）を「HTTP 404 … not found」と誤って判断したりする可能性があります。

そのため、CI を確認する際は常にリポジトリを明示してください。

```bash
gh run list -R jiwoochris/artex-global --workflow ci.yml --limit 5
gh run list -R jiwoochris/artex-global --workflow detections.yml --limit 5
```

一度設定しておけば、`-R` を省略しても私たちのリポジトリを既定で見るようにできます。ただし、この設定は **ローカルの gh の設定** であり、リポジトリにはコミットされないため、新しいマシンや新しいチェックアウトでは改めて指定する必要があります。

```bash
gh repo set-default jiwoochris/artex-global
gh repo set-default --view   # jiwoochris/artex-global が表示されることを確認
```

### 8.2 文書の外部リンクは、ブラウザのように GET で確認します

防御ガイド（[`docs/defense-ko.md`](docs/defense-ko.md)・[`defense-en.md`](docs/defense-en.md)）の 7 節には、国内の公式チャネル（boho.or.kr・fsec.or.kr・pipc.go.kr）へのリンクが掲載されています。このリンクが生きているかを確認する際に、`curl -I`（HEAD リクエスト）や既定の User-Agent だけで確認すると、**正常なリンクを壊れていると誤認** します。国内の公共・セキュリティ機関のサイトは、次の三つの理由から単純な確認を拒否するためです。

- **HEAD リクエストを拒否します。** たとえば fsec.or.kr は `curl -I`（HEAD）に対して 400 を返します。
- **既定の `curl` User-Agent をブロックします。** fsec.or.kr と pipc.go.kr は、既定の UA で送った GET リクエストにも 400 を返します（ブラウザの UA で送ると 200 になります）。
- **別のアドレスにリダイレクトします。** pipc.go.kr は `www.pipc.go.kr` から `pipc.go.kr/np/` へ二回リダイレクトするため、リダイレクトを追跡しないと最終状態を見落とします。

したがって、リンクの確認は **ブラウザの User-Agent で、GET で、リダイレクトを追跡しながら** 行います。

```bash
UA='Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/140.0 Safari/537.36'
for u in https://www.boho.or.kr https://www.fsec.or.kr https://www.pipc.go.kr; do
  curl -sS -L -A "$UA" -o /dev/null -w "$u -> %{http_code} %{url_effective}\n" "$u"
done
```

最終的なステータスコードが 200 であれば、リンクは有効です。400・403 が返ってきた場合は、リンクが壊れているのではなく **確認方法がサーバーのアクセスポリシーに阻まれている** 可能性をまず疑い、HEAD・既定 UA・リダイレクト非追跡といった要因を一つずつ取り除いて再確認します。（2026-10-06 の確認時点では、三つのリンクとも上記の方法で 200 となり、pipc.go.kr は二回のリダイレクトの後に 200 でした。）

この手動の手順は、`scripts/check-external-links.py` がそのまま自動化しています。追跡対象のすべての `.md` から、コードフェンスとインラインコードの外にある外部リンクを集め（予約ホストやプレースホルダーのホストは除外）、上記のようにブラウザ UA・GET・リダイレクト追跡で状態を確認します。ネットワークエラー・5xx・429 は再試行して、一時的な不調と本当の障害を区別します。結果は四つに分類されます。OK（2xx・3xx）、RESTRICTED（401・403・405・429。ホストは生きているが確認方法が阻まれている）、ALLOWED（`scripts/external-links-allowlist.txt` に記載された、私たちが修正できない上流から引き継いだ壊れたリンク）、DOWN（404・410・5xx・接続エラー。壊れている可能性が高い）。

- ネットワークなしで点検対象だけを事前に確認: `python3 -I scripts/check-external-links.py --list`
- リリース時や定期点検（新たに壊れたリンクがあれば異常終了）: `python3 -I scripts/check-external-links.py --strict`

外部リンクの生存は flaky なので、**マージゲートには入れません**。代わりに、非ブロッキングのワークフロー [`external-links`](.github/workflows/external-links.yml) が毎週月曜日と手動実行で `--strict` を実行し、allowlist にない DOWN が新たに発生した場合に赤く表示します。原本の文字を保存するファイルが引き継いだ壊れたリンク（例: `CHANGELOG.zh.md` が謝辞として挙げている、既に消えた貢献者アカウント）は私たちでは修正できないため、理由とともに allowlist に記載し、strict 検査から除外します。

---

## 9. リリース発行パイプライン

バージョンタグ（`v*`）をプッシュすると、[`.github/workflows/release.yml`](.github/workflows/release.yml) が五つのプラットフォーム向けのバイナリと、条件を満たした場合にはマルチアーキテクチャの Docker イメージを作成します。このフォークはまだリリースタグを切ったことがなく、このワークフローは一度も実行されていないため、この節では、パイプラインが何を前提とし何を生成するのか、そしてその前提が現在のリポジトリ構造と合っているかを整理します。タグをプッシュすると公開リポジトリに GitHub Release が作成されるため、リリースを切る作業は公開の判断が確定した後に行ってください。

### 9.1 リリースを切る方法

`v` で始まるタグをプッシュするとワークフローが起動します。

```bash
git tag v0.3.15
git push origin v0.3.15
```

### 9.2 パイプラインの処理内容

ワークフローは五つのジョブに分かれています。

- **frontend。** フロントエンドを静的に一度エクスポートし（`web/out`）、その成果物を `web-dist` アーティファクトとしてアップロードします。下の binaries ジョブは、ターゲットごとにこの成果物を受け取って再利用します。
- **binaries。** 五つのターゲット（linux amd64・arm64、darwin amd64・arm64、windows amd64）をクロスコンパイルし、ターゲットごとに zip にまとめます。linux amd64 のバイナリには `artex -h` のスモークテストを実行し、バイナリが実際に動くことを確認します。
- **release。** すべての zip を集めて `SHA256SUMS` のチェックサムを作成し、GitHub Release を作成して zip とチェックサムを添付します。
- **docker。** binaries がクロスコンパイルした linux バイナリを受け取り、マルチアーキテクチャ（amd64・arm64）のイメージをビルドして、GitHub Container Registry の `ghcr.io/jiwoochris/artex-global` にリリースタグと `latest` でアップロードします。ログインはワークフロー既定の `GITHUB_TOKEN`（ジョブ権限 `packages: write`）で行うため、別途用意するシークレットはありません。

### 9.3 ビルドの前提がリポジトリ構造と合っているか

パイプラインは次の三つの前提の上で動作します。これらの前提が現在のリポジトリ構造とすべて合っているかを、binaries ジョブをローカルで直接再現して確認しました。

- **フロントエンドの埋め込み。** frontend ジョブがアップロードした `web-dist`（= `web/out` の内容）を binaries ジョブが `server/webui/dist` に受け取り、`server/webui_embed.go` の `//go:embed all:webui/dist` がその場所をバイナリに埋め込みます。そのため binaries ジョブはフロントエンドを再ビルドせず、`ARTEX_SKIP_FRONTEND=1` を指定して [`build.sh`](build.sh) を呼び出します。
- **バイナリとパッケージのパス。** `build.sh --target <os>/<arch>` は、`dist/artex-<os>-<arch>/artex` のバイナリと、`dist/` 配下の zip パッケージを作成します。zip には、バイナリとともに起動スクリプト（Linux・macOS は `start.sh`、Windows は `start.bat`）、`skills/`、`config.example.json`、`README.md` が含まれます。
- **Docker イメージへのバイナリのコピー。** binaries ジョブは linux バイナリを `bin-linux-<arch>` アーティファクトとして別にアップロードし、docker ジョブがそれを `dist/<arch>/artex` として受け取ります。[`Dockerfile`](Dockerfile) の `COPY dist/${TARGETARCH}/artex` が、マルチアーキテクチャビルドで buildx が各プラットフォームに合わせて設定する `TARGETARCH` を使ってそのパスを取り込みます。[`.dockerignore`](.dockerignore) は `dist/` を除外していないため、バイナリはビルドコンテキストに含まれます。

### 9.4 Docker イメージの名前空間

イメージは、このフォークが所有する `ghcr.io/jiwoochris/artex-global` で公開し、`docker-compose.yml` もこのイメージを取得します（2026-10-11 の決定、`work/DECISIONS-FOR-JIWOO.md` の 8 番の項目）。上流の `autumn27/artex` は Docker Hub から削除されており、このフォークには push 権限もないため使用しません。GHCR のパッケージは最初の公開時に非公開で作成されることがあるため、最初のリリースの後にパッケージの設定で公開（public）になっているかを確認してください。非公開のままだと、`docker compose pull` が認証エラーで失敗します。

### 9.5 タグを切らずにローカルで事前検証する

公開リリースを切らずにパイプラインの前提だけを確認したい場合は、binaries ジョブをローカルで再現します。ローカルに Go がない場合は、Docker で同じように実行できます。

```bash
# 1) フロントエンドの静的エクスポート（release.yml の frontend ジョブに相当）
cd web && npm ci && npm run build:static && cd ..
# 2) binaries ジョブがアーティファクトを受け取る場所に配置
rm -rf server/webui/dist && mkdir -p server/webui/dist && cp -a web/out/. server/webui/dist/
# 3) 一つのターゲットを binaries ジョブと同じ環境でビルド
docker run --rm -v "$PWD":/app -w /app \
  -e ARTEX_SKIP_FRONTEND=1 -e ARTEX_SKIP_NPM_CI=1 \
  -e ARTEX_COMPRESS=0 -e ARTEX_PACKAGE=1 -e ARTEX_PACKAGE_DIR=dist \
  -e ARTEX_BUILD_VERSION=v0.0.0-local \
  golang:1.26 bash -c 'apt-get update && apt-get install -y zip && ./build.sh --target linux/amd64'
# 4) 成果物を確認: dist/artex-linux-amd64/artex · dist/*.zip · dist/SHA256SUMS
```

`dist/artex-linux-amd64/artex` は静的リンクされた ELF で、`-h` を指定すると使い方を出力して終了コード 0 で終わります。これが binaries ジョブのスモークテストが確認する動作です。ビルド成果物（`dist/`・`server/webui/dist/`）はリポジトリにコミットしないでください（`.gitignore` で除外されています）。
