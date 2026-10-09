# launchs-org の k8s 環境 つまずいた点と検証の記録

更新日: 2026-10-09。構築手順は [k8s-build.md](k8s-build.md)。実際に踏んだ問題(原因と直し方)と、検証の記録。

## 落とし穴(実際に踏んだもの)

### Talos / ノードの更新

- **既存ノードに設定を丸ごと重ねるとブートが止まる**(約数十分のダウンタイム)。`worker-extensions.yaml`(`machine.files` と `kubelet.extraMounts` を含む)を、すでに入っているノードに `talosctl patch mc` すると、リストが二重になり、`EtcFileSpecs ... already exists` で `writeUserFiles` が失敗する。kubelet が起動せず NotReady のまま、35 分後に再起動、を繰り返す。
  - 直し方: JSON パッチは、マルチドキュメントの設定には使えない。`EDITOR` に重複ブロックを取り除くスクリプトを指定して `talosctl edit mc` する(テキストのまま、2 つ目の同一ブロックだけを切る。YAML を再整形すると `0o644` などの書式が壊れる)。
  - 画像だけ変えるなら `install.image` のみのパッチを使う。
- **`talosctl upgrade` の成功表示を信用しない**。ブート失敗でロールバックしても、それらしい出力で終わることがある(worker-3 で、旧イメージのまま「成功」に見えた)。`talosctl get extensions` の `schematic` で、稼働中のイメージを検証する。
- **`upgrade` は VM そのものを再起動しない**(ゲスト OS だけ)。Proxmox のデバイス(エージェントなど)の追加は、VM の停止・起動が必要。
- **ホスト名は `HostnameConfig`**(Talos 1.12 以降)。`machine.network.hostname` は `static hostname is already set` で拒否される。
- **Kubernetes のバージョンを固定する**。Talos 1.13.2 のデフォルト(1.37)は、`kubelet image is not valid: version of Kubernetes 1.37.0 is too new` で拒否される。
- **ファイアウォールの変更は `--mode try`**(自動ロールバック付き)。ロックアウトを避ける。
- **学校の外向き回線が、TLS 接続を断続的に失敗させる**(`unexpected eof`、`TLS handshake timeout`)。MTU の問題ではない(1500 まで通る)。`helm pull`、`curl` は再試行する。containerd のイメージ取得は自動で再試行する(worker-3 のイメージ取得で、約 8 分遅れた)。

### Cilium

- **設定を変えた後は、エージェント(DaemonSet)の再起動が必須**。`helm upgrade` で ConfigMap(`bpf-lb-sock-hostns-only = true`)が更新されても、**エージェントは古い設定のまま動き続けた**(実行時は `Disabled`)。ConfigMap ではなく、`cilium-dbg config -a` の実行時の値で確認する。
  - この原因が分かるまで、「設定は正しいのに gVisor から Service に届かない」状態で、的外れな調査を続けた(`socketLB.enabled=false` も試したが効果なし。戻した)。
  - 切り分けに `talosctl pcap` が有効だった(runc は veth を出る時点で宛先が変換済み、gVisor は Service の IP のまま、で socket-LB が Pod の中で動いていることに気づけた)。
- 旧 Pod 向けの k8s NetworkPolicy だけでは、**他ノードからのホスト通信**(`remote-node`)は許可されない。ノードから Harbor を引くには、`CiliumNetworkPolicy` の `fromEntities: [host, remote-node]` が要る。

### アプリ・manifest

- **`kubectl apply -f secret.yaml` に namespace を付けないと、`default` に作られる**(認証情報が default に入った。気づいて削除済み)。`-n launchs-org` を必ず付ける。
- **`manifest` の NetworkPolicy は、旧環境の前提**(postgres-operator が `default` にいる、ノード網が `10.10.11.0/24`)。`postgres-operator` namespace から DB に入れず、`could not init db connection: i/o timeout` でクラスタが `CreateFailed` になる。
- **PostgreSQL のユーザー作成前に、アプリを起動すると SASL 認証で落ちる**。クラスタが `Running` になってから、アプリを `rollout restart` する。
- **Dragonfly(Redis)operator は不要**(`renew-version` は使わない)。`main` ブランチ向けの構成なので、撤去した。
- **イメージの `0.1.7` タグは、`ghcr.io/launchs-org/backend` などには存在しない**(`new-*` にはある)。`latest` と `sha-*` だけ。ダイジェストで固定した。
- **`new-*` イメージは `backend-new` リポジトリ(5/31 で更新停止)由来**で、現行(`backend` リポジトリ、`main` から継続的にマージ)ではない。比較の結果(2026-10-09): `backend` の方が新しく、機能が多く、現行のデプロイ(`main` の PR #158 からビルド)と一致する。履歴は 3 リポジトリ(`backend_old` 4/9、`backend-new` 5/23、`backend` 6/8)で、共通のコミットがゼロ(それぞれ作り直し)。
- **`volume.yaml` の `longhorn` / RWX**: Longhorn を入れる前は、RWO・`local-path` に変えて動かしていた。Longhorn が入った今は、元に戻すべき(構築手順 §9)。
- **ビルドの「`no space left on device`」はディスクの話ではない**。`user.max_user_namespaces=0`(Talos の既定)で、rootless buildkit の `unshare` が ENOSPC を返す。
- **ビルドした Pod の「デプロイ中」が終わらない**: ノードが `harbor.main-harbor` を解決できない / Harbor に届かない(`ImagePullBackOff`、`DeadlineExceeded`)。構築手順 §10 の設定(hosts、CA、CiliumNetworkPolicy)。バックオフ中の Pod は、削除すれば即座に再試行される。

### 作業のしかた

- **作業は 1 つのセッションだけで行う**。同じ作業を行うセッションが並行して動いていて、drain、VM の停止・起動、`talosctl upgrade` が二重に走った(worker-2 に 2 回、master の更新が知らないうちに始まっていた)。
  - 接続のたびに別プロセスを起動する設定だと起きやすい。**接続のたびに、同じ `screen` / `tmux` のセッションに attach する形にする**。
  - 作業の前に、`ps` で他の作業用のプロセスや、実行中のスクリプト(`ga-*.sh`、`talosctl upgrade`)がないことを確認する。
- **途切れた操作の結果を、実際の状態で確認する**。前のセッションが途切れたときの呼び出しは、あとから実行されることがある。再実行する前に、`qm status`、ノードの状態、ログで確かめる。
- **バックグラウンドのスクリプトは、完了を `pgrep` ではなく結果の検証で判断する**。

## 検証の記録(2026-10-08〜09)

| 検証 | 結果 |
|---|---|
| Longhorn の RWX を 1Gi → 3Gi にオンライン拡張 | 成功(Pod 内の `df` にも反映) |
| gVisor(`runsc`)の Pod 起動 | 成功(`dmesg` に `Starting gVisor`) |
| gVisor から Pod IP / ClusterIP / Service 名 / `*.svc.cluster.local` / DNS | **Cilium エージェントの再起動後**に、runc と同じ結果で成功(それまでは ClusterIP が全滅) |
| gVisor + Longhorn の RWO / RWX | 成功 |
| Pod の user namespace(`hostUsers: false`) | ボリュームなし・RWO は成功。RWX は失敗(NFS が idmap 非対応) |
| railpack のビルドを gVisor で実行(`sample-go-app`) | `--oci-worker-snapshotter=native` で成功。Harbor への push も成功(約 2 分) |
| プロジェクト namespace の egress 制限 | DNS・同一 namespace・インターネットは通り、k8s API・ノード・Proxmox・他 namespace は遮断 |
| ノードからの Harbor の pull | CA の信頼と CiliumNetworkPolicy の後に成功(4.3 秒) |
| 全ノードの更新(拡張入りイメージ + qemu-guest-agent) | 5 台とも成功。稼働中のスキーマティックと、Proxmox のエージェントの応答で検証 |
