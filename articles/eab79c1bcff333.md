---
title: "OpenTofuでプライベートエンドポイント利用した ASR環境を構築"
emoji: "🐦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["OpenTofu", "Terraform", "Azure"]
published: true
published_at: 2024-05-10 18:03 
---

https://zenn.dev/mnod/articles/869495687fd9ab の続き。

ストレージアカウントとの間、ASRとの間の通信を、プライベートエンドポイント利用したものにする。
検証で構築するときプライベートエンドポイントは、金額的にはたいしたことがないかもしれないけど、自費で払うと考えると意外と高い。

## 参考
https://learn.microsoft.com/ja-jp/azure/site-recovery/azure-to-azure-how-to-enable-replication-private-endpoints

## 構築資材

https://zenn.dev/mnod/articles/910f0d0b32e8ee にある asr_main.tf の代わりに下記の asr_main_pep.tf を使う

:::details asr_main_pep.tf ファイル
@[gist](https://gist.github.com/mnod/0ed9ec48287d3a785a1e648911720b37?file=asr_main_pep.tf)
:::

## もろもろ
- カーネルダウングレードのためには、一時的にインターネットへ抜ける設定(80/tcp、443/tcp)を追加する。レプリケーション設定の前に削除する。
- 閉域で使う前提ならパブリックIPアドレスは不要。ちゃんとプロキシを設定する。
- WindowsのVMでデータディスク が サイトリカバリで認識されずレプリケートできないとき、OSからマウントしてみたら認識された。
- サイトリカバリ設定削除したら、次回設定できないことがあった。他人と共同作業していたときのため、再現できていない。ロックがなんとかというエラーメッセージの内容だった。もしかしたらディスクの削除が契機だったかもと推測している。
- Proxy経由で通信するとき、hypervrecoverymanager.windowsazure.com、login.microsoftonline.com、blob.core.windows.net は除外。ProxyInfo.conf ファイルは Mobility Serviceが利用するプロキシ設定を記載することができる。
- 自前のDNSを利用するとき、プライベートエンドポイントの名前解決ができるようにする。
