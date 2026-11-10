# ARTEX 検知ルール（Suricata / ネットワーク）

日本語 · [English](README.md) · [한국어](README.ko.md)

[Sigma ルール](../sigma/)のネットワーク階層の相棒です。この [Suricata](https://suricata.io) シグネチャは、ネットワーク上で観測される 2 種類の ARTEX の成果物を扱います。すべての指標は推定ではなく、このリポジトリのソースで確認した文字列や振る舞いに基づいています。ホスト・ログ・SIEM の階層は [`../sigma/`](../sigma/) にあり、全体像は防御ガイド（[韓国語版](../../docs/defense-ko.md) · [English](../../docs/defense-en.md)）が説明しています。

## ルール: [`artex.rules`](artex.rules)

- **sid 1000001**: `ARTEX enrichment prober User-Agent`。User-Agent が `artex-enrich/` で始まるインバウンドの HTTP `GET` です（資産補強プローブ `enrich/enrich.go:233`）。単発のリクエストの存在を示す指標です。`classtype: attempted-recon`。
- **sid 1000002**: `ARTEX enrichment prober high-rate enumeration`。同じ User-Agent が、`detection_filter` の閾値である **送信元ごとに 300 秒で 30 リクエスト** を超える場合です。単発ルールが見逃す、機械的な速度で浴びせる頻度です。Sigma 相関ルール `artex_enrich_scan_velocity` を反映しています。`classtype: attempted-recon`。
- **sid 1000003**: `ARTEX worker WebFetch User-Agent`。User-Agent が `norma/` で始まるインバウンドの HTTP リクエストです（norma SDK の WebFetch ツール `github.com/Autumn-27/norma/tool/webfetch.go`）。この UA は norma の旧バージョン（v0.1.0–v0.4.3、検証済み）全体にわたってハードコードされており、記録プロキシがリクエストヘッダーを変更しないため（`traffic/traffic.go`）、対象ホストの通信上にそのまま届きます。補強プローブと異なり、**攻撃段階**（能動的な脆弱性プロービング）で発火します。`classtype: attempted-recon`。

## 範囲と正直さ：デプロイ前にお読みください

- **ネットワーク上で観測される ARTEX の User-Agent は 2 種類です。** 補強プローブは偵察段階で `artex-enrich/1.0`（`enrich/enrich.go:233`）を、norma SDK の WebFetch ツールは攻撃段階で `norma/0.4`（`github.com/Autumn-27/norma/tool/webfetch.go`）を送信します。記録プロキシ（`traffic/traffic.go`）はリクエストヘッダーを変更しないため、両方の UA が対象のネットワーク上に届きます。それ以外のワーカーツール（Bash の子プロセスである `curl`、`nmap` など）は独自の User-Agent を使うため、一般的なスキャナーのシグネチャと、[`../sigma/`](../sigma/) の振る舞いベースの SIEM ルールで検知してください。
- **User-Agent は平文でしか見えません。** トラフィックが平文 HTTP である場合、または TLS を終端するプロキシ・WAF で検査される場合に現れます。エンドツーエンドの TLS はこれを暗号化するため、実際に HTTP リクエストを見られる場所に配置してください。
- **静的な User-Agent は運用者が変更できるため**、ないからといって安全という意味では**ありません**。長持ちするシグナルは振る舞い、すなわち速度と幅です。sid 1000002（および Sigma の相関階層）が速度を基準にしている理由、そして純粋な Web の多段階検知に環境ごとの既定ルールが必要な理由はここにあります。
- **意図的に除いたもの。** 自己更新の User-Agent `artex-selfupdate` は GitHub への HTTPS で送られるため、ネットワーク上では観測されません（TLS SNI だけで警報を出すには一般的すぎます）。監査の統制マーカーは対象に向かうトラフィックではなく運用者側のログ成果物なので、[`../sigma/artex_guard_audit_framing.yml`](../sigma/artex_guard_audit_framing.yml) で検知してください。サーバーポート `:8787` と記録プロキシ `127.0.0.1:8788`（`cmd/artex/main.go`）は、ネットワークシグネチャではなくホスト・フォレンジック用（`ss`・`netstat`）です。
- **指標一つは、このリポジトリのソースではなく固定された依存関係に基づきます。** 冒頭で、すべての指標がこのリポジトリのソースで確認されたものだと述べましたが、sid 1000003 だけがその例外です。このルールの `norma/` User-Agent は norma SDK（`github.com/Autumn-27/norma/tool/webfetch.go`）にハードコードされており、ARTEX はこの SDK をリポジトリのツリーにベンダリングせず、`go.mod` で `v0.4.3` に固定した依存関係として取り込んでいます。そのため、この文字列はリポジトリ内のファイルではなく、固定された上流の依存関係（v0.1.0–v0.4.3 にわたってハードコード、検証済み）に根拠を置きます。指標の根拠を確認するテストが、ARTEX 自身が出力する文字列は読み取れても、この値だけは読み取れない理由がここにあります。同じ但し書きは[指標一覧](../indicators/README.ko.md)にも記してあり、上の「このリポジトリのソースで確認」という文が例外なく成り立つよう、ここにも記しておきます。

## 検証とテスト

Suricata 8 で検証しています。読み込みテストはトラフィックを必要とせず、常に実行できます。

```sh
# 文法 + エンジンへの読み込みテスト（期待: "Configuration provided was successfully loaded"）
docker run --rm -v "$PWD/detections/suricata":/r -w /r jasonish/suricata:latest \
  suricata -T -S artex.rules -l /tmp --init-errors-fatal
```

`--init-errors-fatal` は、パースはできても初期化に失敗するルールも致命的なエラーにし、黙って捨てられたシグネチャがあれば読み込みテストが通らないようにします。

ルールが実際に発火することを確認するには、再現可能な回帰テストが [`../tests/suricata/`](../tests/suricata/) にあります。まず同じ読み込みチェックを実行し、scapy で決定的なキャプチャを合成して、その上で `suricata -r` を実行し、警報数を検証します。Docker さえあれば実行できます。

```sh
detections/tests/suricata/run.sh
```

sid 1000001 はプローブごとにちょうど一度発火し（35 フローのキャプチャで 35 回）、sid 1000002 は 300 秒で 30 の閾値を超え（Suricata 8.0.7 で **5** 回の警報、31〜35 番目のフロー）、同じキャプチャを良性のブラウザ User-Agent で流すと警報が **0** であることを検証します。シグネチャが特異的であることを確認するものです。詳しくは [`../tests/README.ko.md`](../tests/README.ko.md) を参照してください。代わりに自分のトラフィックで確認するには、ローカルサーバーに対してループバックの `curl -A 'artex-enrich/1.0'` をキャプチャし、警報を読んでください。

```sh
suricata -r enrich.pcap -S artex.rules -l out && \
  grep -c '"signature_id":1000001' out/eve.json    # 存在: プローブごとに一回
```

## 貢献

検知への貢献を歓迎します。新しいルールは、すべての指標を観測可能な事実に基づかせ、限界をコメントに明記し、`suricata -T` をクリーンに通過し、攻撃の手引きとして読める内容を含まないようにしてください。[`../../CONTRIBUTING.md`](../../CONTRIBUTING.md)、[`../sigma/`](../sigma/) の Sigma 階層、そして [`../README.ko.md`](../README.ko.md) を参照してください。
