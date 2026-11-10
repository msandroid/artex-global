# `/btw` 検証記録

日本語 · [한국어](VALIDATION.md) · [中文](VALIDATION.zh.md)

日付: 2026-09-10。ブランチ: `codex/btw-side-question`。基準コミット（baseline）: `8dae851b9b622f2ff2631f332fde9719d0b16fba`。

> この文書は原文の中国語文書（`VALIDATION.zh.md`）を韓国語に翻訳したものです。上流（upstream）リポジトリの変更と照合しやすいように、原文はそのまま保存しています。

独立した PostgreSQL のテスト DB とデータディレクトリを使用しました。実際のモデルの認証情報は独立したテスト環境にだけ注入し、コードやこの記録には書き込んでいません。製品のデフォルトモデルも変更していません。Go 1.26.3、norma v0.3.6、Next.js 16.2.9 を使用しました。

実際のモデルとの対話、返されたオブジェクト、エンジニアリングのアサーション、Qwen の原文の審査テキストは [validation-2026-09-10.json](validation-2026-09-10.json) に保存しており、その中に API の認証情報は含まれていません。

## エンジニアリング検査

以下の項目はすべて合格しました。括弧内は根拠となるテストです。

- 構造化メッセージと、ツール引数のディープコピー: 合格（根拠 `TestCheckpointDeepCopyAndBoundaries`）。
- 要約・圧縮リクエストが上書きしないこと、完結した応答と終了状態の発行、生成途中の中途半端な応答の除外: 合格（根拠 `TestCheckpointDeepCopyAndBoundaries`、`TestSnapshotExcludesPartialStreamAndSelectsPoolMember`）。
- 実際のモデルプールのメンバーの識別情報: 合格（根拠 `TestSnapshotExcludesPartialStreamAndSelectsPoolMember`）。
- ツールの対応付け、20 組の再生、予算の削減と超過エラー: 合格（根拠 `TestBuildRequestCompactionToolPairingAndBudget`）。
- メインとサイド質問の並列実行、双方向のキャンセルの分離: 合格（根拠: ブロッキング方式の Provider、`TestMainSideConcurrencyAndIndependentCancellation`）。
- ツールの実行がないこと、ストリーミング・非ストリーミング、失敗時点で既に得られている使用量: 合格（根拠 `TestServiceNoToolsAndUsageOnFailure`）。
- 実際の norma ChatAgent とローカルの Read ツール、メインの transcript・アクティビティの分離: 合格（根拠 `TestSideActualChatCheckpointToolResultAndTranscriptIsolation` のストリーミング・非ストリーミングのサブケース）。
- 永続化、ページ分割、冪等性、再起動後の部分応答の保持: 合格（根拠 `TestSideHistoryIdempotencyPagingAndRecovery`）。
- クリアと遅れた書き込みの競合、親リソースの削除、バージョン比較: 合格（根拠 `TestSideClearLateWritersAndDeletedParent`）。
- MainAgent・Worker のアーカイブと復元（v1・v2・v3）: 合格（根拠 `TestSideTaskArchiveVersions`）。
- 3 種類の親インターフェース、認証、リソースの帰属、Worker の論理削除: 合格（根拠 `TestSideHTTPGlobalLimitTaskWorkerAndDeletion`、`TestSideCheckpointPersistsBeforeAdmissionAndRestart`）。
- 忙しいメインセッションでもサイド質問が可能、独立した SSE の再接続・切断、キャンセル、クリア: 合格（根拠 `TestSideHTTPBusyIsolationClearAndReconnect`）。
- 親セッションごとに 1 件・全体で 4 件の同時実行: 合格（根拠: 2 つの `TestSideHTTP…` のサブケース）。
- 送信前のスナップショットの保存、再起動後の続けての質問、古いセッションがスナップショットを偽造できないこと: 合格（根拠 `TestSideCheckpointPersistsBeforeAdmissionAndRestart`）。
- キャッシュ上の設定が削除されたりモデルが変わったりした場合に、処理の継続を拒否すること: 合格（根拠 `TestSideRejectsDeletedOrChangedCachedProfile`）。
- アーカイブ前にキャンセルし、最終応答と使用量が保存されるまで待つこと: 合格（根拠 `TestSideTaskDrainPersistsBeforeArchive`）。
- ストリーミングの消費者が早期にキャンセルしても、使用量は 1 回だけ記録され、サイド質問に帰属すること: 合格（根拠 `TestSideUsageRecordedOnceOnConsumerCancellation`）。
- 再起動によって自動復元された Worker・deadline の実行コンテキストが、引き続き新しいスナップショットを発行すること: 合格（根拠 `TestSideRestoredWorkerRuntimePublishesNewCheckpoint`）。
- 関連パッケージの race 検査: 合格（根拠: 下記のコマンド）。
- TypeScript と本番ビルド: 合格（根拠 `npx tsc --noEmit`、`npm run build`）。
- 新しく追加したフロントエンドモジュールの Biome 検査: 合格（根拠 `biome check`、新規モジュール 3 件）。

使い捨てのデータベースに `ARTEX_PG_DSN` を設定すると、自動検査を再現できます（本番 DB を指定しないでください）。

```sh
go test -race ./agent ./db ./server ./sidequestion ./llmrec ./llmpool \
  -run 'Test(Side|Checkpoint|Snapshot|BuildRequest|Service|MainSide|CaptureRun|TaskArchive|CompleteForwards|StopIntent|CancelIntent)' -count=1
cd web
npx tsc --noEmit
npx biome check src/lib/side-questions.ts src/hooks/use-side-questions.ts src/components/side-question-workspace.tsx
npm run build
```

Go のリグレッションテスト全体はすべて成功（green）しているわけではありません。`server` パッケージの既存のテスト 2 件が、一時ディレクトリのクリーンアップ段階で失敗し、どちらも `TempDir RemoveAll … directory not empty` を報告しています。

- `TestInheritedActivityDetailAndRelationDeletion`
- `TestTaskMetadataPatchReturnsRenameAndPin`

変更していない上記の基準コミットからソースを取り出し、同じ隔離環境で `server` パッケージを再実行したところ、この 2 件のクリーンアップ失敗も同様に再現しました。基準コミットの実行では、別途 `TestCoreTaskLifecyclePG` の対象ノード数のアサーションも失敗しましたが、最終的な修正後の `server` のリグレッションでは、そのアサーションの失敗は起きませんでした。その他のパッケージは合格し、今回のサイド質問関連のケースと race 検査も合格しています。基準コミットの問題を今回の検収の合格として扱っておらず、隠れた問題を隠すために既存のアサーションを変更することもしていません。

Next.js のビルド出力には、複数の lockfile・workspace ルートの推定に関する既存の警告が出ていますが、ビルドは完了し、すべてのページが正常に生成されています。

## ブラウザ検査

Codex 組み込みブラウザを使って、独立したローカルの Go サービスと Next.js の開発サーバーに接続しました。デスクトップと 390 × 844 の狭い画面で、次の手動・自動操作を行い、スクリーンショットとブラウザログを確認しました。

- 通常のチャットの実行中に `/btw` を入力すると、本文とサイド質問が同時に表示され、デスクトップのサイドバーも正常です。
- 続けて追加の質問をしました。サイド質問を停止しても、すでに生成された部分は残り、本文の流れは続きます。
- パネルを閉じてもリクエストは継続し、再度開くと完了した回答を復元します。ページを再読み込みした後、内容なしで `/btw` と入力すると履歴を復元します。
- 狭い画面の Drawer で、入力・ボタン・履歴・閉じる操作が正常で、横方向のはみ出しもありません。
- クリアは確認ポップアップを表示し、クリア後は履歴が消えますが、メインの transcript とスナップショットは残ります。
- タスクの MainAgent と 2 つの Worker にそれぞれ質問して切り替えたところ、エージェントのタブと履歴が混ざることはありませんでした。
- ブロッキング方式のローカルモデルのフィクスチャで Worker を実行し続けた状態で、Worker のメイン入力欄から `/btw` を送信しました。サイド質問を停止した後も、Worker はリアルタイムの実行状態と自身の一時停止ボタンを表示し続け、サイド質問は部分応答を保存していました。
- ブラウザのエラー・警告ログは空です。

制御可能なフィクスチャは、並行実行のタイミングを正確に検証するために使ったもので、実際のモデルの出力速度には依存していません。デバッグ中の 2 回の Worker 実行の検査では、有効な並行実行の区間が生じませんでした（タスクがすでに終了していた、または応答が早く終わっていた）。フィクスチャを修正して再実行し、合格しました。この初期の操作は有効な合格とはみなしていません。

## 実際のモデルとの対話

まず `grok-4.6` を検出しました。OpenAI 互換インターフェースは `http://127.0.0.1:12580/tingly/openai` です。検出結果は HTTP 200 で、モデル名 `grok-4.6` と `READY` を返し、2.82 秒かかりました。第一候補が利用可能だったため、Tingly の `glm` や Zhipu の `glm-5.3` のフォールバックチェーンは有効にしていません。これら 2 つの予備サービスは今回検証していません。

- メインセッションの実行中に資産・目標・マーカーを質問したところ、`redhaze.top`、トップページの読み取りと目標の要約、`BTW-REAL-0910` を返し、サイド質問は完了しました（16.97 秒）。
- メインセッションがトップページの読み取りを終えた後、ツールの根拠を質問したところ、WebFetch 200、curl のリダイレクト 301 → 302 → 200、ページタイトルを正確に引用しました（7.24 秒）。
- サイド質問が Bash でテストファイルを作成するよう求めたところ、実行を拒否し、対象ファイルは作成されませんでした（7.74 秒）。
- 完了後のサイド質問がメインのコンテキストを変えないこと: メインの transcript の SHA-256 とメインのアクティビティ記録は変わらず一致し、サイド質問のツール実行回数は 0 でした。
- Go サービスを実際に停止・再起動した後に続けて質問したところ、以前のサイド質問の履歴 3 件を保持しており、永続化したスナップショットからすぐに資産・マーカー・タイトルに答え、メインエージェントは再実行しませんでした。
- 新しいセッションで Grok の非ストリーミング設定を使ったところ、資産と `ATOMIC-0910` を正確に答え、使用量を返して保存しました（input 11734、output 138、cache_read 11520）。

資産の事例では、メインセッションが WebFetch と Bash/curl で公開されたトップページを読み取りました。ランディングページは `https://id.redhaze.top/home` で、ページが返したタイトルは「红幕科技 RedHaze Group · 全球综合集团门户」です（モデルが返した原文のため、そのまま引用しています）。Bash はレスポンスをローカルのテストファイルに一時的に保存しただけで、リモートへの書き込みは実行していません。この事実は「サイド質問がツールを実行しなかった」という点とは別に確認しています。

メインの transcript の検証値: `e7e61f135a4a120954b539f357e8c4205d7d5cd7460dcaf3dc0fd066463e1d00`。

**使用量の制限:** Tingly の Grok ストリーミング応答は usage を返しませんでした。`stream_options.include_usage=true` を直接送って検証したところ、HTTP 200、データフレーム 12 個、usage フレーム 0 個でした。したがって、ストリーミングテストでの 0 は、エンドポイントが使用量を提供しないことを意味するのであり、課金がなかったと解釈してはなりません。非ストリーミングの使用量と、フィクスチャの失敗・キャンセル時の使用量は、いずれも正しく保存されました。

## Qwen による審査

審査モデルは `qwen-flash`、OpenAI 互換インターフェースは `https://dashscope.aliyuncs.com/compatible-mode/v1`、HTTP 200 です。最初の 3 つの実際のサイド質問の対話、メインセッションのツールの根拠、エンジニアリングのアサーションを提供したところ、`verdict: accept` と `concerns: []` を返しました。回答が資産・マーカー・ページ読み取りの証拠と一致しており、サイド質問のツール拒否が制約に合致していると判断しています。審査の使用量: prompt 6625、completion 312、total 6937。

今回の Qwen 審査の範囲には、後から追加したサービスの再起動と非ストリーミングのテストは含まれていません。Qwen の「書き込みなし」という一般化は広すぎます。メインセッションの curl は実際にローカルの応答の一時ファイルを作成しており、これは前の部分で明記しています。並行性、ツール実行 0 回、transcript の分離はエンジニアリングのアサーションによって判断しており、モデルによる審査は回答の品質評価を補助するものにすぎません。
