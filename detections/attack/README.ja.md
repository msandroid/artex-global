# ARTEX ATT&CK カバレッジ

日本語 · [English](README.md) · [한국어](README.ko.md)

このリポジトリの検知ルールがタグ付けしている [MITRE ATT&CK](https://attack.mitre.org/)（Enterprise）のテクニックを、Navigator レイヤーとしてまとめたものです。[Sigma ルール](../sigma/)の `attack.*` タグから手作業で作成しており、すべてのテクニックは、指標がこのリポジトリのソースで確認した文字列、あるいは振る舞いであるルールに根拠を置いています。推定で入れた項目はありません。[一貫性テスト](../tests/attack/run.sh)が、レイヤーとルールが互いに食い違わないように守ります。

- **`artex_navigator_layer.json`**: ATT&CK Navigator v4.5 形式のレイヤーです。

## スコアの意味

ここでのカバレッジとは「このリポジトリが、そのテクニックをタグ付けする検知を提供している」という意味であり、「そのテクニックが完全にカバーされている」という意味ではありません。スコアは、検知の強度を意図的に正直な値で付けています。

- **100: ARTEX 固有のシグネチャまたは振る舞い。** ARTEX にしか存在しない静的な指標（`artex-enrich/1.0`・`artex-selfupdate` の User-Agent、ガードの監査マーカー）か、その上に構築した振る舞いのルール（強化の速度・ファンアウト、ガードのブロックの集中）です。
- **50–65: 一般的なハンティングの手がかり。** ARTEX のガードのブロックリストを反映した、破壊的コマンドのハンティングです。同じコマンドは正当な管理者も実行するため、良性の活動でも検知が発火します。ヒットは手がかりとして扱い、断定の根拠にしないでください。65 は、相関ルールがそのコマンドを ARTEX のガードマーカーと組み合わせて特異度を高めた場合を指します。

## 扱うテクニック

6 つの戦術にまたがる 8 つのテクニックです。各テクニックは、それをタグ付けしているルールに対応します。

- **偵察(Reconnaissance): T1595 (Active Scanning), T1592 (Gather Victim Host Information).**
  [`sigma/artex_enrich_user_agent.yml`](../sigma/artex_enrich_user_agent.yml)、
  [`sigma/correlation/artex_enrich_scan_velocity.yml`](../sigma/correlation/artex_enrich_scan_velocity.yml)、
  [`sigma/correlation/artex_enrich_fanout.yml`](../sigma/correlation/artex_enrich_fanout.yml)、そして
  [Suricata ルール](../suricata/artex.rules)（sid 1000001 / 1000002）です。
- **命令・制御(Command and Control): T1105 (Ingress Tool Transfer).**
  [`sigma/artex_selfupdate_egress.yml`](../sigma/artex_selfupdate_egress.yml) です。
- **実行(Execution): T1059 (Command and Scripting Interpreter).**
  [`sigma/artex_guard_audit_framing.yml`](../sigma/artex_guard_audit_framing.yml)、
  [`sigma/correlation/artex_guard_block_burst.yml`](../sigma/correlation/artex_guard_block_burst.yml)、
  [`sigma/correlation/artex_guard_marker_then_destructive.yml`](../sigma/correlation/artex_guard_marker_then_destructive.yml) です。
- **インパクト(Impact): T1485 (Data Destruction), T1561.002 (Disk Wipe: Disk Structure Wipe), T1489 (Service Stop).**
  [`sigma/destructive_command_hunting.yml`](../sigma/destructive_command_hunting.yml) であり、T1485 は
  [`sigma/correlation/artex_guard_marker_then_destructive.yml`](../sigma/correlation/artex_guard_marker_then_destructive.yml) によっても補強されます。
- **資格情報へのアクセス・収集(Credential Access / Collection): T1557 (Adversary-in-the-Middle).**
  [`sigma/artex_recording_proxy_ca.yml`](../sigma/artex_recording_proxy_ca.yml) であり、ワーカーツールのトラフィックを復号・記録するために ARTEX 内蔵のトラフィック記録器（`traffic/traffic.go`）がインストールする MITM のルート CA アーティファクトを狙った、ホスト・フォレンジックのハンティングの手がかりです。

## 使い方

1. [ATT&CK Navigator](https://mitre-attack.github.io/attack-navigator/) を開きます。
2. **Open Existing Layer → Upload from local** を選び、`artex_navigator_layer.json` を選択します（またはこのリポジトリの raw ファイル URL を指定します）。
3. スコアを付けたテクニックが検知の強度に応じて色分けされて表示され、各テクニックには根拠となったルールファイルと防御ガイドの節を記したコメントが付いています。

## 範囲と正直さ

- **カバレッジは網羅性ではありません。** ここでスコアを付けたテクニックは、ルールがそれをタグ付けしているという意味であり、そのテクニックのあらゆる変形を検知するという意味ではありません。ネットワーク上で ARTEX 固有の User-Agent によって捉えられるシグナルは、偵察段階の強化プローブ（`artex-enrich/1.0`）と、攻撃段階の norma SDK の WebFetch（`norma/0.4`）の 2 つだけで、それ以外の攻撃トラフィックはツールの既定の指紋に従います。長く有効な検知は振る舞いに基づくものです（防御ガイド [한국어](../../docs/defense-ko.md) · [English](../../docs/defense-en.md) の 1〜2 節・4.1〜4.2 節を参照）。純粋な Web の多段階事例では、依然として環境ごとの既定ルールが必要です。
- **静的な指標は変更できます。** 運用者は User-Agent を別の値に設定できるため、タグ付けされた指標が見つからないからといって安全だという意味ではありません。ルールファイルにも同じ注意点を記してあります。

## 検証と貢献

[一貫性テスト](../tests/attack/run.sh)を実行してください。Docker さえあれば動作し、レイヤーがスコアを付けたテクニック・戦術がルールの `attack.*` タグと正確に一致すること、そしてすべてのテクニックが実在するルールファイルに根拠を持つことを検証します。

```sh
detections/tests/attack/run.sh
```

ルールを追加したり、タグを付け直したりした場合は、このレイヤーも合わせて更新してください。ルールのテクニックがレイヤーにない、またはレイヤーのテクニックがルールにない場合、テストは失敗します。[`../README.ko.md`](../README.ko.md) と [`../../CONTRIBUTING.md`](../../CONTRIBUTING.md) を参照してください。
