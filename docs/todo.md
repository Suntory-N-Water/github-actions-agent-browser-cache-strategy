# やることリスト

## ゴール

GitHub Actions 上で agent-browser を使う際のキャッシュ戦略(CI高速化)を検証し、結果を `docs/report.md` にまとめる。

- キャッシュなし vs キャッシュあり でインストール時間を比較
- 単一ジョブでの計測(並列ジョブは対象外)
- 1回の実行で明確な差が出るかどうかを確認する

## タスク

- [x] agent-browser の仕様を確認
- [x] ベースライン CI 作成・実行(キャッシュなし)
- [x] Chrome 保存パスを特定(`~/.agent-browser/browsers/`)
- [x] キャッシュ戦略を調査・仮説立案
- [x] 各キャッシュ戦略の CI を作成・実行・比較
- [ ] 計測結果を `docs/report.md` にまとめる

## 計測結果サマリー(キャッシュヒット時)

| アプローチ | 合計 | インストールステップ | 備考 |
|---|---|---|---|
| ① ベースライン(キャッシュなし) | 43秒 | 33秒 | - |
| ② Chrome のみキャッシュ | 35秒 | 27秒 | Chrome ダウンロードをスキップ |
| ③ 最小パッケージ手動指定 | 34秒 | 20秒 | 新規パッケージ2つのみ apt install |
| ④ awalsh128 apt キャッシュ | 16秒 | 4秒 | node20 廃止問題あり |
| ⑤ gerlero apt キャッシュ | 36秒 | 20秒 | ヒット時も apt-get update が走る |
| ⑥ **自前実装 apt キャッシュ** | **19秒** | **6秒** | node.js 不使用・外部依存なし |

## メモ

- `agent-browser install --with-deps` の内訳: システム依存(apt)30秒 + Chrome ダウンロード2.5秒
- ubuntu-latest に既にインストール済みのパッケージが33個。新規必要なのは `fonts-freefont-ttf` と `fonts-noto-cjk` の2つのみ
- 自前実装はキャッシュヒット時に `apt-get update` をスキップするため高速
- gerlero/apt-install はキャッシュヒット時も `apt-get update` を実行するため遅い
- awalsh128 は最速だが Node.js 20 廃止問題(2026年9月16日にランナーから削除予定)
- カスタムランナーイメージは Team/Enterprise プランが必要(Free プラン不可)
