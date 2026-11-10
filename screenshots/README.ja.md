# スクリーンショットの案内 (Screenshots)

日本語 · [한국어](README.md) · [English below](#english)

このフォルダには、性質の異なる 2 種類の画面キャプチャが入っています。混同しないよう、まずそれぞれが何であるかを明記します。

- **`en/`: デフォルトの英語 UI をキャプチャした ARTEX の実際の画面。** このリポジトリが提供する成果物です。メインドキュメントの [`../README.md`](../README.md) と [`../README.ja.md`](../README.ja.md) がこの画像を使用しています。提供用の英語画面だけを見たい場合は、[`en/`](en/) フォルダを開いてください。
- **ルートの `*.png`: 上流オリジナル（中国語版）の UI キャプチャ。** 上流リポジトリと 1:1 で照合できるように**原文のまま保存**しており、保存された中国語の README [`../README.zh.md`](../README.zh.md) がこの画像を参照しています。日本語・英語のドキュメントでは使用していません。
- **`wx.png`: 上流の原作者による WeChat 公式アカウントの宣伝画像。** [`../README.zh.md`](../README.zh.md) がそのまま正しく表示されるよう、合わせて保存しています。このフォークとは無関係です。

つまり、ルートにまず中国語の画面が見えるのは上流を保存しているためであり、**提供用の英語画面は [`en/`](en/) 内にあります。** このリポジトリが元の画像も並べて保持している理由（ローカライズの透明性・上流との照合）は、防御・検知のポジショニングと同じ文脈にあります。詳しい背景はルートの README を参照してください。

---

## English

This folder holds two separate sets of captures. To keep them from looking mixed, here
is what each one is.

- **`en/` — real UI of ARTEX in the default English display language.** This is what the fork delivers. The
  main docs [`../README.md`](../README.md) and [`../README.ko.md`](../README.ko.md) use
  these images. For the delivered screens, open [`en/`](en/).
- **The `*.png` files in the root — upstream (Chinese edition) UI captures.** They are
  **preserved as-is** so the fork can be diffed against the upstream repository 1:1, and
  the preserved Chinese README [`../README.zh.md`](../README.zh.md) references them. The
  Korean and English docs do not use them.
- **`wx.png` — the upstream author's WeChat channel promo image.** Kept so that
  [`../README.zh.md`](../README.zh.md) still renders unchanged. It is unrelated to this
  fork.

In short, the Chinese screens you see first in the root are there for upstream
preservation; the **delivered English screens live in [`en/`](en/).** See the root README
for why this fork keeps the originals alongside (localization transparency and upstream
comparison).
