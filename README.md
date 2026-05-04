# MySyncLore

> This is a personal fork of [SyncLore](https://github.com/sotashimozono/SyncLore).
> Upstream documentation is preserved in [docs/UPSTREAM_README.md](docs/UPSTREAM_README.md).

Markdown 記事を Zenn と Qiita の両方に同時公開するための個人 repo。
`drafts/<slug>.md` を書いて main に push すると、両プラットフォームに反映されます。

## セットアップ済み事項

このリポジトリは SyncLore template から fork 後 `npm run init:fork -- --apply` で
My 化済みです。upstream のデモ記事 (synclore-*.md) は
`drafts/archive/upstream-synclore/` に退避されています。

## 使い方の概要

1. `drafts/template.md` をコピーして新しい slug の draft を作成
2. `publish: true` (または `publish_at: <ISO-8601>`) を設定して push
3. GitHub Actions の `sync.yml` が Zenn / Qiita 両方に同時公開

詳細な仕様 (予約公開・取り下げ・rename・wiki-link など) は
[docs/UPSTREAM_README.md](docs/UPSTREAM_README.md) を参照してください。

## License

| 対象 | ライセンス |
| --- | --- |
| コード・スクリプト | [MIT License](./LICENSE) (upstream SyncLore 由来) |
| 記事本文 | 各記事の著者に帰属 |
