---
title: "remote サーバ上の研究データを Obsidian で直接管理する — Obsidian-Remote-SSH の使い方"
emoji: "🔬"
type: "tech"
topics: ["obsidian", "research", "workflow", "ssh", "data"]
publish: false
publish_at: "2026-05-13T08:30:00+09:00"
qiita_id: ""
---

## はじめに

これまで [紹介記事](https://zenn.dev/sotashimozono/articles/obsidian-remote-ssh-introduction) と [内部設計](https://zenn.dev/sotashimozono/articles/obsidian-remote-ssh-architecture) について書きました。今回は実際にどう使ってるかという運用面の話です。

`obsidian-remote-ssh` がどんな問題を解いてくれるか、自分の使い方を交えて書きます。

対象読者:

- 計算機サーバ上に実験データやログを大量に持っている人
- ノートを Obsidian で取りたいが「どこに置くか」で悩んでる人
- ML 研究、HPC 利用、homelab、複数マシンを行き来する人

## 私の典型的な workflow

### 環境

- 計算機 (Panza) — 36 コアの自宅サーバ。実験ジョブをここで回す。データの「正」
- laptop (MacBook) — 出先で書く。電車移動中に「次の実験のメモ」を書く端末
- desktop (Mac mini) — 自宅で書く。腰据えて分析・執筆する端末

3 つの machine から同一の vault にアクセスしたい。けれど vault には 100 GB の実験データ (`data/`)、5 GB の plot 画像 (`plots/`)、生成された論文 PDF (`papers/`) を含めたい。これを既存の sync 系 plugin (`remotely-save` 等) で sync するのは:

- laptop に 105 GB を mirror できない (ストレージ的にも、転送速度的にも)
- ジョブが新しい結果を吐いた瞬間に Obsidian で見たい (ファイル数千個の sync を待ちたくない)

なので `obsidian-remote-ssh` で vault は Panza 上にだけ置く、という構成にしています。

```text
Panza:/home/souta/vault/
├── notes/                  # 実験ノート、論文ドラフト
│   ├── 2026-01-15-tpqmps-bench.md
│   ├── 2026-02-03-thermal-mps-paper.md
│   └── ...
├── data/                   # 計算ジョブが書く出力
│   ├── runs/2026-01-15/
│   └── ...
├── plots/                  # matplotlib の出力
│   └── 2026-01-15-cv-vs-temperature.png
└── papers/                 # 関連論文の PDF (BibliFetch.jl で集める)
    └── 10.1103_physrevb.99.214433.pdf
```

laptop でも Mac mini でも、Obsidian を開いて Panza に SSH 接続するだけ。machine 側に何も置かれない。

## 具体例 1: 実験ノートをそのまま機械上に書く

新しいジョブを投げる前にノートを切ります:

````markdown
# 2026-01-15 TPQ-MPS bench at L=20

## 背景
前回 L=18 で...

## 設定
- bond dim: 256
- temperature: [0.05, 0.1, 0.2, 0.5, 1.0]
- Trotter step: 0.005

## ジョブ
```bash
$ julia --project=. -t 36 -e 'include("scripts/run_tpqmps.jl"); main(L=20, betas=[20,10,5,2,1])'
```

## 結果
[[2026-01-15-cv-vs-temperature]]
````

このノートは Panza 上の `vault/notes/2026-01-15-tpqmps-bench.md` に直接書かれます。laptop を閉じても、ノートは Panza に存在し続けます。実験ジョブが走ってる間に laptop の電源が切れても問題なし。

## 具体例 2: 計算結果を Obsidian で見る

ジョブが完走すると `data/runs/2026-01-15/cv_results.csv` と `plots/2026-01-15-cv-vs-temperature.png` が書かれます。

`fs.watch` が走ってるので、Obsidian の File Explorer に数秒以内に新しいファイルが現れます。ノートに `![[plots/2026-01-15-cv-vs-temperature.png]]` と書けば inline で表示される。

ここでポイント:

- 5 GB の plot ディレクトリを laptop に sync してない。表示時に必要な範囲だけ pull (`read_range` RPC)
- 計算ジョブが新しい plot を吐いた瞬間にノートで見える
- 別マシン (Mac mini) で同じノートを開けば、同じ plot がそのまま見える

## 具体例 3: Dataview で実験を集約

`notes/` 配下のノートに frontmatter で実験 metadata を入れておきます:

```yaml
---
date: 2026-01-15
system: TPQ-MPS
L: 20
bond_dim: 256
temperatures: [0.05, 0.1, 0.2, 0.5, 1.0]
status: complete
---
```

トップレベルに `_index.md` を置いて Dataview で集約:

````markdown
## 2026 年の実験一覧

```dataview
TABLE date, system, L, bond_dim, status
FROM "notes"
WHERE date >= date("2026-01-01")
SORT date DESC
```
````

これも全部 Panza 側のファイル読みなので、local ディスクが空でもまったく問題ない。

## 複数マシン同時利用 — workspace.json の罠

laptop で開いてる pane 構成と Mac mini で開いてる pane 構成は別であってほしい。`workspace.json` を機械間で共有してしまうと、片方の機械で window を分割した瞬間にもう片方も追従してしまう。

このプラグインは path mapper で `<vault>/.obsidian/workspace.json` を per-client subtree にリダイレクトしています。

```text
laptop から:    vault/.obsidian/workspace.json
                ↓ (実ファイルは)
                vault/.obsidian/user/macbook/workspace.json

Mac mini から:  vault/.obsidian/workspace.json
                ↓ (実ファイルは)
                vault/.obsidian/user/mac-mini/workspace.json
```

両方が同じ仮想パスを読み書きしているように見えるが、実体は機械別。layout が干渉しない。

## SSH 周りのセットアップ

最低限の前提:

```bash
# クライアント側 (laptop など)
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@panza.lan

# テスト
ssh user@panza.lan ls
```

これが普通に通れば、Obsidian Settings → Remote SSH → Add Profile で:

- Host: `panza.lan`
- Port: `22`
- Username: `souta`
- Auth: `Private Key`
- Private key path: `/Users/souta/.ssh/id_ed25519`

を設定して `Connect` を押すだけ。

### Jumphost (踏み台) 経由

研究機関の SSH は jumphost 経由が多いと思います。`~/.ssh/config` で:

```ssh-config
Host panza
  HostName panza-internal.example.ac.jp
  User souta
  ProxyJump jumphost.example.ac.jp
```

を書いておけば、プラグインの Host を `panza` だけに指定すれば自動的に jumphost を経由します。

### ssh-agent

agent 経由の認証は対応しています。私は `1Password` の SSH agent を使ってますが、何の問題もなく動いています。

## 何が辛いか

正直に書きます。

- 大量の小さいファイル一覧の初回 walk — 10000 ファイルあるとさすがに数秒かかる。`list_with_stat` の batch が効くのは 100-500 ファイル単位
- plugin の中で `fs.readFileSync` を直叩きするもの — ごく一部 (主にレガシーな自作 plugin) で互換性問題。Dataview / Templater / Excalidraw / Tasks などメジャーどころは 100% 動く
- Mobile (iPad/iPhone) はサポートしない — 設計上 desktop only (`isDesktopOnly: true`)

## まとめ

「ノートは local、データは remote」という分裂を解消できる。vault は SSH ホスト側に置きっぱなし、local replica なし、Obsidian の通常の編集体験そのまま。

実例 ↑ に近い workflow を持つ方は、ぜひ BRAT 経由でインストールして、issue / discussion で「動いた」「動かなかった」を教えてもらえると嬉しいです。

- GitHub: <https://github.com/sotashimozono/obsidian-remote-ssh>
- Issues: <https://github.com/sotashimozono/obsidian-remote-ssh/issues>
- 過去記事: [紹介](https://zenn.dev/sotashimozono/articles/obsidian-remote-ssh-introduction) / [内部設計](https://zenn.dev/sotashimozono/articles/obsidian-remote-ssh-architecture)

次回は本プラグインの E2E テスト — Playwright + CDP screencast で Electron を回す方法 — について書きます。
