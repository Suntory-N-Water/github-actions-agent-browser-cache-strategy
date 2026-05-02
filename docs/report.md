# GitHub Actions で agent-browser を使う際のキャッシュ戦略調査レポート

## ゴール

GitHub Actions 上で [agent-browser](https://github.com/vercel-labs/agent-browser) のような重たいライブラリを使う際のキャッシュ戦略を検証し、CI の高速化を図る。

複数エージェントを並列で動かす実務ユースケースでは、インストールに時間がかかるほど CI の消費時間が膨らむ。そのため、**インストール時間(Chrome + システム依存のセットアップ)** をどれだけ短縮できるかを計測し、最適なキャッシュ戦略を明らかにする。

比較方式は「キャッシュなし vs キャッシュあり(複数戦略)」の単一ジョブ計測。

## 検証環境

| 項目 | 内容 |
|---|---|
| ランナー | `ubuntu-latest`(Ubuntu 24.04) |
| agent-browser | `npm install -g agent-browser`(グローバルインストール) |
| Chrome | Chrome for Testing 148.0.7778.97(175MB) |
| 検証サイト | https://claude-code-log.com |
| 操作内容 | `open` → `snapshot -i` → `close` |
| 計測方法 | 各アプローチを複数回(4〜9回)実行し範囲を記録。実行履歴: [ベースライン](https://github.com/Suntory-N-Water/github-actions-agent-browser-cache-strategy/actions/workflows/baseline.yml) / [Chrome キャッシュ](https://github.com/Suntory-N-Water/github-actions-agent-browser-cache-strategy/actions/workflows/cache-chrome.yml) / [最小パッケージ](https://github.com/Suntory-N-Water/github-actions-agent-browser-cache-strategy/actions/workflows/cache-min-packages.yml) / [awalsh128](https://github.com/Suntory-N-Water/github-actions-agent-browser-cache-strategy/actions/workflows/cache-apt-packages.yml) / [gerlero](https://github.com/Suntory-N-Water/github-actions-agent-browser-cache-strategy/actions/workflows/cache-apt-gerlero.yml) / [自前実装](https://github.com/Suntory-N-Water/github-actions-agent-browser-cache-strategy/actions/workflows/cache-apt-custom.yml) |

## ベースライン計測

まずキャッシュを一切行わない状態で CI を作成し、実行時間を計測した。

→ [`.github/workflows/baseline.yml`](../.github/workflows/baseline.yml)

```yaml
- name: agent-browser をインストール
  run: npm install -g agent-browser

- name: Chrome とシステム依存をインストール
  run: agent-browser install --with-deps

- name: ブラウザ操作(open → snapshot → close)
  run: agent-browser batch "open https://claude-code-log.com" "snapshot -i" "close"
```

### 結果

| ステップ | 所要時間 |
|---|---|
| agent-browser をインストール | 4秒 |
| Chrome とシステム依存をインストール | **33秒** |
| ブラウザ操作 | 2秒 |
| **合計** | **43秒** |

### 内訳の分析

CI ログのタイムスタンプから `agent-browser install --with-deps` の内訳を分析した。

| フェーズ | 所要時間 |
|---|---|
| システム依存(apt-get update + apt install) | **30秒** |
| Chrome for Testing(175MB)ダウンロード | 2.5秒 |

**ボトルネックは Chrome ではなく apt のシステム依存インストールだった。**

## キャッシュ戦略の仮説

### ボトルネックの特定

`agent-browser install --with-deps` が内部で実行する `apt-get install` のログを確認したところ、以下が判明した。

```
Running: sudo apt-get install -y libxcb-shm0 libx11-xcb1 ... (35パッケージ)
0 upgraded, 2 newly installed, 0 to remove and 53 not upgraded.
The following NEW packages will be installed:
  fonts-freefont-ttf fonts-noto-cjk
```

**35パッケージ中、実際に新規インストールされたのは `fonts-freefont-ttf` と `fonts-noto-cjk` の2パッケージのみ。** 残り33パッケージは `ubuntu-latest` に既にインストール済みだった。66.9MB のフォントデータのダウンロードが時間を要していた。

### 候補アプローチの整理

| アプローチ | 概要 | 期待効果 |
|---|---|---|
| Chrome のみキャッシュ | `~/.agent-browser/browsers/` をキャッシュ | Chrome ダウンロード(2.5秒)をスキップ |
| 最小パッケージ手動指定 | 新規必要な2パッケージのみ `apt install` | apt インストール対象を削減 |
| apt パッケージキャッシュ | `.deb` ファイルをキャッシュし `dpkg` でインストール | `apt-get update` + ダウンロードをスキップ |

## 各アプローチの検証

### Chrome バイナリのみキャッシュ

`~/.agent-browser/browsers/` を `actions/cache` でキャッシュし、agent-browser バージョンをキャッシュキーに使用した。

→ [`.github/workflows/cache-chrome.yml`](../.github/workflows/cache-chrome.yml)

#### 結果と考察

キャッシュヒット時のログ([run #25240220690](https://github.com/Suntory-N-Water/github-actions-agent-browser-cache-strategy/actions/runs/25240220690))に `✓ Chrome 148.0.7778.97 is already installed` が確認でき、Chrome ダウンロードはスキップされた。しかし `apt-get update` + システム依存インストールは毎回実行されるため、大幅な短縮にはならなかった。

| | ベースライン | キャッシュヒット(4回計測) |
|---|---|---|
| 合計 | 36〜45秒 | 36〜46秒 |

### 最小パッケージの手動指定

#### 調査：`--with-deps` は何をインストールしているか

ベースライン CI のログ([run #25240037766](https://github.com/Suntory-N-Water/github-actions-agent-browser-cache-strategy/actions/runs/25240037766))を解析し、`apt-get install` の実行結果を確認した。

```
fonts-noto-color-emoji is already the newest version (2.047-0ubuntu0.24.04.1).
...(他33パッケージも同様に already the newest version)
The following NEW packages will be installed:
  fonts-freefont-ttf fonts-noto-cjk
0 upgraded, 2 newly installed, 0 to remove and 53 not upgraded.
Need to get 66.9 MB of archives.
After this operation, 108 MB of additional disk space will be used.
Get:2 http://azure.archive.ubuntu.com/ubuntu noble/main amd64 fonts-freefont-ttf all 20211204+svn4273-2 [5641 kB]
Get:3 http://azure.archive.ubuntu.com/ubuntu noble/main amd64 fonts-noto-cjk all 1:20230817+repack1-3 [61.2 MB]
Fetched 66.9 MB in 1s (51.6 MB/s)
```

ubuntu-latest には Chromium の依存の大半が既にインストール済みであり、実際に必要なのは `fonts-freefont-ttf` と `fonts-noto-cjk` の2パッケージのみと判明。

この知見をもとに `agent-browser install --with-deps` を使わず、不足する2パッケージだけを直接指定する方式に切り替えた。

→ [`.github/workflows/cache-min-packages.yml`](../.github/workflows/cache-min-packages.yml)

#### 結果と考察

`apt-get update` は引き続き実行されるため、削減効果は限定的だった。フォント2パッケージに絞ってもダウンロードとインストールで約20秒かかる。Chrome キャッシュの有無で変わらず、毎回ほぼ同じ時間がかかる。

| | ベースライン | 実測範囲(5回計測) |
|---|---|---|
| 合計 | 36〜45秒 | 24〜30秒 |

### apt パッケージキャッシュ

`.deb` ファイルそのものをキャッシュし、ヒット時は `apt-get update` なしで `dpkg` から直接インストールする方式を検証した。

#### ライブラリ調査

既存のライブラリとして以下を調査した。

##### awalsh128/cache-apt-pkgs-action

最も利用実績が多い(★346)apt キャッシュアクション。動作確認では最速(ヒット時4秒)だったが、内部で `actions/cache@v4`(Node.js 20)を使用しており、Node.js 20 廃止問題を抱えている。修正 PR([#193](https://github.com/awalsh128/cache-apt-pkgs-action/pull/193)・[#198](https://github.com/awalsh128/cache-apt-pkgs-action/pull/198))がオープン中だが、メンテナーの動きが止まっており採用リスクがある。

→ [`.github/workflows/cache-apt-packages.yml`](../.github/workflows/cache-apt-packages.yml)

##### gerlero/apt-install

Node.js を一切使わない composite + Bash 実装で、`actions/cache@v5`(Node.js 24)を使用しており廃止問題がない。しかしキャッシュヒット時も `apt-get update` を実行するため、ヒット時でも20秒かかった。

→ [`.github/workflows/cache-apt-gerlero.yml`](../.github/workflows/cache-apt-gerlero.yml)

##### 自前実装

`gerlero/apt-install` の仕組みを参考に、**キャッシュヒット時は `apt-get update` を完全にスキップ**する composite action を自作した。`actions/cache/restore@v5` と `actions/cache/save@v5`(Node.js 24)のみを使用し、外部依存なしで構成した。

→ [`.github/actions/apt-cache/action.yml`](../.github/actions/apt-cache/action.yml)
→ [`.github/workflows/cache-apt-custom.yml`](../.github/workflows/cache-apt-custom.yml)

#### 結果と考察

| 実装 | 実測範囲(4回計測) | Node.js 問題 | 備考 |
|---|---|---|---|
| awalsh128 | 13〜19秒 | ⚠️ node20 廃止予定 | - |
| gerlero | 24〜36秒 | なし | ヒット時も apt-get update が走るため遅い |
| **自前実装** | **16〜19秒** | **なし** | - |

速度は awalsh128(13〜19秒)と自前実装(16〜19秒)がほぼ同等。awalsh128 は Node.js 20 廃止問題を抱えているため、同等の速度を保ちつつ廃止リスクがない自前実装が最もバランスが良い結果となった。

## 計測結果まとめ

2回目以降(キャッシュ温まり後)の実測範囲。各アプローチ4〜9回実行。

| アプローチ | キャッシュ状態 | 実測範囲 | 備考 |
|---|---|---|---|
| ① ベースライン | キャッシュなし(毎回) | 36〜45秒 | - |
| ② Chrome のみ | Chrome ヒット・apt 毎回 | 36〜46秒 | ほぼ改善なし |
| ③ 最小パッケージ | Chrome ヒット・apt 毎回 | 24〜30秒 | - |
| ④ awalsh128 | Chrome + apt ヒット | 13〜19秒 | ⚠️ node20 廃止予定 |
| ⑤ gerlero | Chrome + apt ヒット | 24〜36秒 | ヒット時も apt-get update が走る |
| ⑥ **自前実装** | **Chrome + apt ヒット** | **16〜19秒** | **node.js 不使用** |

## 結論と注意点

### 採用アプローチ

**自前実装の apt キャッシュ(アプローチ⑥)を採用する。**

理由：
- キャッシュヒット時にベースライン比 50〜60% 削減(36〜45秒 → 16〜19秒)
- `actions/cache@v5`(Node.js 24)のみを使用しており廃止リスクがない
- 外部アクションへの依存なし・composite action 約40行のシンプルな実装
- キャッシュキーに手動バスト用バージョン番号を持ち、緊急時のリセットが容易

### 注意点・今後の課題

**awalsh128 について**
Node.js 20 が GitHub Actions ランナーから削除される 2026年9月16日 以降、動作しなくなる可能性がある。修正 PR がマージされれば再評価の余地はあるが、現時点では採用しない。

**ubuntu-latest のバージョンアップについて**
今回の「新規必要パッケージは2つだけ」という結果は ubuntu-24.04 時点の話。ubuntu-latest が更新されてもインストールステップ自体は失敗しない。ただし、今まで ubuntu-latest に含まれていたパッケージが削除された場合、Chrome が起動できずブラウザ操作ステップで落ちる可能性はある(コアなシステムライブラリが削除されることは考えにくいため現実的なリスクは低い)。なお、インストール対象パッケージを変更した場合は `inputs.packages` が変わるためキャッシュキーが自動的に変わる。`cache-version` の手動バストが必要になるのは、同じパッケージ構成のままキャッシュを強制リセットしたい場合のみ。

**システム依存の完全キャッシュについて**
今回は `.deb` ファイルのキャッシュで apt 関連の時間を削減したが、`apt-get update` 自体(約10秒)はキャッシュミス時に毎回発生する。さらなる短縮にはカスタムランナーイメージ(GitHub Team/Enterprise プランが必要)の検討が有効。
