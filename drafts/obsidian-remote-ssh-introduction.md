---
title: "Obsidian の vault を SSH リモートで直接編集できるプラグインを作った"
emoji: "🔌"
type: "tech"
topics: ["obsidian", "ssh", "plugin", "remote", "typescript"]
publish: false
publish_at: "2026-05-07T08:30:00+09:00"
qiita_id: ""
---

## はじめに

研究者として、データやログの大半は計算機サーバ側にあります。一方ノートは Obsidian で取りたい — markdown の plain text、Dataview や Templater のおかげで構造化できる、ローカル動作で速い、複数 device 間 sync は Obsidian Sync か Syncthing で何とかなる。

ただ、**「ノートはローカル / データはリモート」という二重管理**が長年の悩みでした。実験ログを書くたびに「`data/` 配下を `scp` するか… いや `rsync` で増分か… いや SyncThing でフォルダを mirror するか…」と毎回手段を選び直していました。どの選択肢も「リモートサーバが正、ローカルは作業面」という workflow とは噛み合いません。

なので作りました。

**Obsidian Remote SSH** — Obsidian の vault を **SSH ホスト上にだけ置いた状態で**、ローカル Obsidian から直接編集できるプラグインです。VS Code Remote-SSH と同じ発想を Obsidian に持ち込んだもの、と捉えてもらうと正確です。

GitHub: [sotashimozono/obsidian-remote-ssh](https://github.com/sotashimozono/obsidian-remote-ssh)

![Demo: SSH リモートの vault を Obsidian で開いて新規ノートを書く](https://raw.githubusercontent.com/sotashimozono/obsidian-remote-ssh/media/demo.gif)

GIF 内では:

1. ローカル vault に `local_demo*.md` が並んだ状態
2. **コマンドパレット** → `Remote SSH: Connect` → SSH 接続
3. **shadow window** が開き、file explorer が `remote_demo*.md` (= リモートサーバの実ファイル) に切り替わる
4. `Ctrl+N` → 新規ノート → タイピング → Obsidian の autosave がリモートに書き込み

最後の write が本物であることは E2E テストで SFTP 経由で別途検証してあります。

## なぜ作ったか — 既存解との差分

似た目的のツール / アプローチは存在しますが、研究者の workflow に決定的に欠ける要素がそれぞれありました。

### Obsidian Sync / Syncthing / iCloud / Dropbox

- **ローカルに完全 replica** を持つ前提。100 GB の実験データを vault に含めると、ラップトップが死ぬ
- リモート計算機 ↔ Obsidian の同期は人間がトリガーする必要がある
- 計算機が dev branch / production branch みたいに複数台あると、「どこが正?」 が崩壊する

### sshfs / NFS / Samba mount

- mount point が落ちるとアプリごと固まる
- `fs.watch` 系がよく死ぬ (Obsidian の reflect が来ない)
- macOS の sshfs は近年メンテが厳しい

### VS Code Remote-SSH

- 編集はできるが Obsidian の機能 (Dataview, Graph, Canvas, theme) は使えない
- markdown の preview / wikilink の解決が WYSIWYG じゃない

### rsync / git で運用

- ノート側に commit が走るたびに人間の判断が要る
- 競合解決を vault でやりたくない

このプラグインの方針:

- **ローカル replica を作らない** — vault は SSH ホスト側にだけ存在する
- 専用の **Go daemon** をリモートに常駐させ、SFTP より細粒度な RPC で読み書きする
- ローカル Obsidian は通常の Obsidian window として動作する。Dataview / Templater / Canvas / 他のコミュニティプラグインがそのまま動く
- ネットワーク断は内部キューで吸収する

## 主要機能

| | |
|---|---|
| **No local replica** | vault は SSH ホスト側のみ、ローカルは vault adapter のキャッシュ |
| **Go-powered backend** | サーバ側に常駐する Go daemon が file ops を捌く。SFTP より高速 |
| **Standard Obsidian UI** | 接続するとローカル Obsidian と区別がつかない `shadow vault` window が立つ |
| **fs.watch propagation** | リモート側の `vim` 編集等もリアルタイムで Obsidian の File Explorer に反映 |
| **3-way merge UI** | 複数マシンから同時編集して衝突したら、ancestor / mine / theirs パネルで解決 |
| **Network-resilient** | 切断中の writes を spool、再接続時に flush。冪等なファイル単位 |
| **Multi-machine** | `workspace.json` などの client-local 状態を per-host subtree に分離 |

## 使い方 (Beta)

現在 BRAT 経由でインストール可能です。

```text
1. BRAT (Beta Reviewers Auto-update Tool) を Obsidian に入れる
2. BRAT の Add Beta Plugin で「sotashimozono/obsidian-remote-ssh」を追加
3. Settings → Remote SSH → SSH Profile を作成 (host / user / port / private key)
4. Command palette → Remote SSH: Connect → プロファイル選択
```

最低限必要なリモート側の前提:

- SSH 接続できる Linux または macOS
- `~/.ssh/authorized_keys` が普通に動くこと
- (推奨) クライアントで public key 認証

詳細は [README](https://github.com/sotashimozono/obsidian-remote-ssh#readme) に書いてあります。

## 状態 — pre-1.0 / Obsidian Community Store 申請中

- v1.0.0 は cut 済み、`minAppVersion: 1.5.0`
- E2E は per-PR で本物のリモート vault に対して連結 → 切断 → 編集 → 反映 まで検証 (`continue-on-error` なし)
- Obsidian Community Plugins への登録は [obsidianmd/obsidian-releases#12390](https://github.com/obsidianmd/obsidian-releases/pull/12390) で申請中、現在 Obsidian チームのレビュー待ち

承認後は Obsidian の Community Plugins ブラウザから直接インストールできるようになります。それまでは BRAT が一番楽です。

## まとめ

「ノートはローカル、データはリモート」を 1 本化したかった。`scp` / `rsync` を毎回打ちたくなかった。Obsidian の機能を諦めたくなかった。VS Code Remote-SSH のあの体験が Obsidian にも欲しかった。

そういうモチベでこのプラグインを書きました。研究者・remote server を日常的に使う人 (ML 研究、HPC 利用、リモート NAS 運用、homelab) に刺さる predictionです。

バグ報告・プラグイン互換性レポート・feature request、何でも歓迎します。

- GitHub: <https://github.com/sotashimozono/obsidian-remote-ssh>
- Issues: <https://github.com/sotashimozono/obsidian-remote-ssh/issues>

次回はこのプラグインの内部設計 — `shadow vault` という概念と、なぜ別 Electron プロセスにしたか — について書く予定です。
