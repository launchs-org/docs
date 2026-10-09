# インフラ(運用者向け)

launchs-org のバックエンドを動かす k8s クラスタ(Proxmox 上の Talos Linux)の、構築と運用の記録です。
利用者向けの使い方は [usage](../usage/) にあります。

- [k8s-build.md](k8s-build.md) — **構築手順**。ゼロから作り直せる順番、設定値、確認方法
- [k8s-pitfalls.md](k8s-pitfalls.md) — **つまずいた点**(原因と直し方)と、検証の記録

Terraform とマニフェストは [`launchs-org/terraform`](https://github.com/launchs-org/terraform) の `talos/` にあります。
秘密情報(パスワード・トークン・鍵)は、このリポジトリには含めません。
