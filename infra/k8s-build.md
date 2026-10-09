# launchs-org の k8s 環境 構築手順(Proxmox 上の Talos Linux)

更新日: 2026-10-09。launchs-org のバックエンド(PaaS)を動かす k8s クラスタを、**ゼロから作り直せる**ように、構築の順番と設定値、確認方法をまとめたもの。
秘密(パスワード・トークン・鍵)は書かない。

- Terraform とマニフェスト: [`launchs-org/terraform`](https://github.com/launchs-org/terraform) の `talos/`、`docs/talos-k8s.md`、`docs/gvisor-cilium.md`
- つまずいた点: [k8s-pitfalls.md](k8s-pitfalls.md)

## 1. 構成

| 項目 | 内容 |
|---|---|
| ホスト | iot-pve1 `192.168.10.105` / iot-pve2 `192.168.10.101`(Proxmox クラスタ `iot-cluster`)。iot-pve3 は QDevice とバックアップ用なので VM は載せない |
| OS | Talos Linux v1.13.2 |
| Kubernetes | 1.36.3(Talos 1.13 は 1.37 に未対応なので固定) |
| ストレージ | Longhorn 1.13.0(レプリカ 2、拡張・RWX)。`local-path` も残す(Harbor・PostgreSQL が使う) |
| CNI | Cilium 1.20.2(kube-proxy 置き換え、WireGuard 暗号化、`socketLB.hostNamespaceOnly=true`) |
| Ingress | Traefik 41.6.1(ClusterIP)。外部公開は Cloudflare Tunnel |
| レジストリ | Harbor 2.15.2(chart 1.19.2、`main-harbor` namespace) |
| DB | Zalando postgres-operator 2.0.3 + PostgreSQL 17(3 台) |
| ワークフロー | Temporal(chart 1.7.0) |
| サンドボックス | gVisor(`runsc`)。RuntimeClass `gvisor` |

### VM

| VM | ID | ホスト | IP | コア / RAM / ディスク |
|---|---|---|---|---|
| talos-master | 200 | iot-pve1 | 192.168.10.161 | 2 / 4GB / 32GB |
| talos-worker-1 | 201 | iot-pve1 | 192.168.10.162 | 4 / 10GB / 100GB |
| talos-worker-2 | 202 | iot-pve1 | 192.168.10.163 | 4 / 10GB / 100GB |
| talos-worker-3 | 203 | iot-pve2 | 192.168.10.164 | 4 / 12GB / 100GB |
| talos-worker-4 | 204 | iot-pve2 | 192.168.10.165 | 4 / 12GB / 100GB |

- 元の `terraform` リポジトリの `terraform.tfvars`(master 1 + worker 4、RAM 合計 56GB)は、2 ノード(各 6 コア・RAM 31GB)に収まらないため、上の値に縮小した。
- ストレージは `zfs-tank`、ネットワークは `vmbr0`(192.168.10.0/24、ゲートウェイ `.254`)。ISO は両ノードにある `talos-linux1132.iso`。
- **Proxmox の HA・ZFS 複製には登録しない**(k8s 側で冗長化するため)。夜間バックアップ `school-nightly` は全ゲスト対象なので自動で入る。
- **master は 1 台**。iot-pve1 が落ちると、k8s の API が止まる(動いている Pod は動き続ける)。

### 作業の場所

学校 LAN(192.168.10.0/24)に届く **iot-pve1 の `/root/launchs-work/`** で作業する(xvps や手元からは届かない)。

| パス | 内容 |
|---|---|
| `bin/` | `terraform` `talosctl` `kubectl` `helm`(チェックサム検証して取得した単体バイナリ。システムのパッケージには入れない) |
| `talos/` | Terraform(`siderolabs/talos`)、state、`out/talosconfig`、`out/kubeconfig`、更新用スクリプト |
| `manifest/` | `manifest` リポジトリの `renew-version` のコピー + 差分(§9)。生成した Secret を含むので、リポジトリには入れない |
| `charts/` | Helm チャートの取得物(外向きの TLS が不安定なので、先に取得して、ローカルのファイルから入れる) |


## 2. 事前準備

1. iot-pve1 に root で SSH できること(`ssh -A` で鍵を転送する。ホスト側に鍵は置かない)。
2. ツールを `bin/` に取得する。**ダウンロードは、必ずチェックサムを検証する**。
   - `terraform`(releases.hashicorp.com、`SHA256SUMS`)、`kubectl`(dl.k8s.io、`.sha256`)、`helm`(get.helm.sh、`.sha256sum`)、`talosctl`(GitHub Releases、`sha256sum.txt`)。
   - `dl.k8s.io` は iot-pve1 から TLS が失敗することがある。その場合は別の場所で取得・検証して `scp` する。
3. 学校の外向き回線は、**TLS 接続が断続的に失敗する**(`unexpected eof`、`TLS handshake timeout`)。`curl` や `helm pull` は再試行を前提にする。イメージの取得は containerd が自動で再試行するので、待てば進む。

## 3. VM を作る(`qm`、root の SSH)

Proxmox の API トークンは**作らない**(トークン作成が権限付与として拒否されたため)。`scripts/mkvm.sh`(`launchs-org/terraform` にある)を、各 Proxmox ホストで実行する。

```bash
# iot-pve1
bash mkvm.sh 200 talos-master   2 4096  32
bash mkvm.sh 201 talos-worker-1 4 10240 100
bash mkvm.sh 202 talos-worker-2 4 10240 100
# iot-pve2
bash mkvm.sh 203 talos-worker-3 4 12288 100
bash mkvm.sh 204 talos-worker-4 4 12288 100
```

- 設定: OVMF + q35、`cpu host`、バルーンなし、`scsi0` は `zfs-tank`、`ide2` に ISO、`boot order=scsi0;ide2`、`onboot=1`。
- MAC は `BC:24:11:A0:00:<ID-200>` に固定する(DHCP の IP を MAC から探すため)。
- 起動後、メンテナンスモードの Talos は **DHCP** で IP を取る。`ip neigh` と MAC で、仮の IP を調べる(§4 の `dhcp_ip`)。
- Proxmox のゲストエージェントを有効にするには、§8 を参照(**VM の停止・起動が必要**)。

## 4. Talos の設定と bootstrap(Terraform)

`launchs-org/terraform` の `talos/` を、iot-pve1 の `/root/launchs-work/talos/` に置いて実行する。

```bash
cd /root/launchs-work/talos && export PATH=/root/launchs-work/bin:$PATH
terraform init
terraform plan -out=tfplan        # 内容を確認する(追加のみのはず)
terraform apply tfplan            # 保存したプランを適用する。-auto-approve は使わない
```

- `variables.tf` の `dhcp_ip` に、§3 で調べた仮の IP を入れる。適用後は固定 IP(`ip`)に切り替わる。
- 共通の設定: CNI なし、kube-proxy 無効、システムディスク(STATE / EPHEMERAL)を LUKS2 で暗号化、`kubernetes_version = 1.36.3`。
- Talos 1.13 では、ホスト名を `HostnameConfig` ドキュメントで指定する(`machine.network.hostname` は拒否される)。
- bootstrap まで 5 分程度。**この時点では CNI がないので、全ノードが NotReady**。次の Cilium で解消する。
- 構築後に再適用するときは、`talos_machine_configuration_apply` の `node` / `endpoint` を `each.value.ip` に切り替える。

## 5. Cilium

```bash
helm install cilium charts/cilium-1.20.2.tgz -n kube-system --wait --timeout 8m \
  --set ipam.mode=kubernetes --set kubeProxyReplacement=true \
  --set k8sServiceHost=192.168.10.161 --set k8sServicePort=6443 \
  --set securityContext.capabilities.ciliumAgent="{CHOWN,KILL,NET_ADMIN,NET_RAW,IPC_LOCK,SYS_ADMIN,SYS_RESOURCE,DAC_OVERRIDE,FOWNER,SETGID,SETUID}" \
  --set securityContext.capabilities.cleanCiliumState="{NET_ADMIN,SYS_ADMIN,SYS_RESOURCE}" \
  --set cgroup.autoMount.enabled=false --set cgroup.hostRoot=/sys/fs/cgroup \
  --set encryption.enabled=true --set encryption.type=wireguard --set operator.replicas=1
# gVisor から Service に通信するための設定(§11)
helm upgrade cilium charts/cilium-1.20.2.tgz -n kube-system --reuse-values --set socketLB.hostNamespaceOnly=true
kubectl -n kube-system rollout restart ds/cilium      # ★必須。ConfigMap を変えるだけでは、エージェントに反映されない
kubectl -n kube-system exec <cilium-pod> -c cilium-agent -- cilium-dbg config -a | grep BPFSocketLBHostnsOnly   # → Enabled
```

## 6. Talos の追加設定(ファイアウォール、user namespace)

`launchs-org/terraform` の `talos/` にある YAML を、`talosctl patch mc` で当てる。**ロックアウトを避けるため、ファイアウォールは `--mode try`(自動ロールバック付き)で当てて、疎通を確認してから本適用する**。

```bash
talosctl -n <ip> -e 192.168.10.161 patch mc -p @firewall.yaml --mode try --timeout 3m
# 確認: apid が管理元から届く、全ノード Ready、Pod 間の DNS が引ける、10250 が外から遮断されている
talosctl -n <ip> -e 192.168.10.161 patch mc -p @firewall.yaml          # 本適用(全 5 台)
talosctl -n <worker-ip> -e 192.168.10.161 patch mc -p @userns.yaml     # worker のみ。再起動なし
```

- **ファイアウォール**(`firewall.yaml`): ingress は既定でブロック。許可するのは、apid 50000(管理元 iot-pve1/2 とノード)、trustd 50001・kubelet 10250・Cilium(4240/4244/4245 TCP、8472/51871 UDP)(ノードのみ)、kube-apiserver 6443(管理元・ノード・Pod ネットワーク `10.244.0.0/16`)、etcd 2379-2380(master のみ)、ICMP(学校 LAN)。
- **user namespace**(`userns.yaml`): `user.max_user_namespaces=11255`。Talos の既定は 0。rootless buildkit(railpack のビルド)が、`unshare` できず `no space left on device` で失敗するため、worker に入れる。**ノード全体の設定なので、カーネルの攻撃面が増える**。ビルド専用のノードに限定できるなら、そうするのがよい。

## 7. 基盤(Traefik、postgres-operator、Longhorn)

```bash
# Traefik: 外部公開は Cloudflare Tunnel なので ClusterIP
helm upgrade --install traefik traefik/traefik -n traefik --create-namespace --wait \
  --set service.type=ClusterIP --set ingressClass.enabled=true
# postgres-operator
helm upgrade --install postgres-operator postgres-operator-charts/postgres-operator \
  -n postgres-operator --create-namespace --wait
# local-path(Talos では /var 配下のみ書き込める。namespace は privileged)
kubectl create ns local-path-storage
kubectl label ns local-path-storage pod-security.kubernetes.io/enforce=privileged
curl -fsSL https://raw.githubusercontent.com/rancher/local-path-provisioner/v0.0.37/deploy/local-path-storage.yaml \
  | sed 's#/opt/local-path-provisioner#/var/local-path-provisioner#g' | kubectl apply -f -
kubectl patch storageclass local-path -p '{"metadata":{"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

### Longhorn(Talos の拡張が先に必要。§8)

```bash
kubectl create ns longhorn-system
kubectl label ns longhorn-system pod-security.kubernetes.io/enforce=privileged
helm install longhorn charts/longhorn-1.13.0.tgz -n longhorn-system -f longhorn-values.yaml --timeout 15m --wait
```

- `longhorn-values.yaml`: `persistence.defaultClass=false`(`local-path` をデフォルトのまま)、`defaultClassReplicaCount=2`、`defaultSettings.defaultReplicaCount=2`、`defaultDataPath=/var/lib/longhorn`。
- 前提: Talos の拡張 `iscsi-tools` と `util-linux-tools`、kubelet の `extraMounts`(`/var/lib/longhorn`、`rshared`)。**どちらも §8 の worker 用の設定に入っている**。
- `backend` の controller は、ユーザーのボリュームを **`storageClassName: longhorn`、RWX** で作る。Longhorn が無いと、PVC が `storageclass "longhorn" not found` で Pending になり、アプリが「デプロイ中」のまま止まる。
- 確認: RWX の PVC を作り、オンラインで拡張できること(1Gi → 3Gi で確認済み)。

## 8. Talos の拡張入りイメージ(Longhorn / gVisor / qemu-guest-agent)

Image Factory でスキーマティックを作る(`launchs-org/terraform` の `docs/gvisor-cilium.md` に手順)。

| 用途 | 拡張 | スキーマティック(先頭 12 桁) |
|---|---|---|
| worker | `iscsi-tools`、`util-linux-tools`、`gvisor`、`qemu-guest-agent` | `c527b6b20fb2` |
| master | `qemu-guest-agent` | `ce4c980550dd` |

インストーラは `factory.talos.dev/installer/<id>:v1.13.2`。worker の追加設定(`worker-extensions.yaml`)に、Longhorn のマウントと gVisor(`runsc`)の登録(`machine.files`)が入っている。

### 更新(既存ノード)

`launchs-org/terraform` の `scripts/talos-upgrade.sh`(検証つき)を使う。**手順と、必ず守ること**:

1. **1 台ずつ**。worker は先に drain、master は最後(master は更新中に k8s の API が数分止まる)。
2. 更新の前に、全 Pod が Running で PostgreSQL が 3 台揃っていることを確認する。
3. **既存ノードに `worker-extensions.yaml` を丸ごと `patch mc` しない**。`machine.files` と `kubelet.extraMounts` が二重になり、`EtcFileSpecs ... already exists` でブートが止まる(kubelet が起動しない)。画像だけ変えるなら `install.image` のみのパッチを使う。
4. **`talosctl upgrade` の成功表示を信用しない**。`talosctl get extensions` の `schematic` で、稼働中のイメージを検証する。
5. 設定上の `install.image` も、稼働中のイメージと揃える。

### qemu-guest-agent(Proxmox から VM の IP を見る)

1. `qm set <vmid> --agent enabled=1`
2. `talosctl upgrade` で拡張入りイメージにする
3. **VM を `qm shutdown` → `qm start` する**。`talosctl upgrade` が再起動するのはゲスト OS だけで、QEMU のプロセスは起動し直されないため、エージェントのデバイスが追加されない。
4. `qm agent <vmid> ping` で確認。起動直後は DHCP の仮の IP が一瞬表示されることがある(固定 IP に落ち着くのを待つ)。

1 回の再起動で済ませたい場合は、`talosctl upgrade --stage` でステージしてから VM を停止・起動する(master で使う想定。未検証)。

## 9. アプリ(`manifest` の `renew-version`)

**`main` ブランチではなく `renew-version`**(現行の `backend` リポジトリに対応)。`main` は旧世代(Dragonfly 構成、`0.1.7` タグのイメージは存在しない)。

### 差分(本来の定義から変えた点)

| 変更 | 理由 |
|---|---|
| イメージを `digest` で固定(`ghcr.io/launchs-org/{backend,watcher,builder,controller,frontend,archive-server}`) | `latest` は動く。再現性のため。ダイジェストは GHCR のレジストリ API から取得 |
| `rbac.yaml` の watcher の ClusterRole に `persistentvolumeclaims`、`ingressroutes` の get/list/watch を追加 | upstream に不足。無いと watcher が forbidden を出し続ける |
| `netpol-postgres-operator.yaml`(新規)を追加 | `allow-launchs-ingress` が、`postgres-operator` namespace からの接続を許可しておらず、オペレーターが DB に入れない(元の環境ではオペレーターが `default` にいた) |
| 全 Deployment に `seccompProfile: RuntimeDefault`、`allowPrivilegeEscalation: false`、`capabilities.drop: [ALL]` | kustomize の patches。`runAsNonRoot` は root 前提のイメージを壊すので付けない |
| `volume.yaml` を **RWO・`storageClassName` なし**にした(★要・元に戻す) | Longhorn を入れる前に、`local-path` で動かすため。**Longhorn が入った今は、元の RWX・`longhorn`(`archive-claim` は `longhorn` 指定)に戻すのが正しい**。auth は 1 レプリカなので、RWO のままでも動いている |

### 適用の順番

```bash
cd /root/launchs-work/manifest
kubectl create namespace launchs-org
kubectl label ns launchs-org pod-security.kubernetes.io/enforce=baseline pod-security.kubernetes.io/warn=restricted
kubectl apply -f namespace-buildkit.yaml                # buildkit namespace は PodSecurity privileged
kubectl apply -f rbac-builder.yaml
kubectl apply -n launchs-org -f secret.yaml             # ★namespace を必ず付ける。付けないと default に作られる
kubectl apply -k .
kubectl apply -f network-policy-harbor.yaml
# Temporal は別に Helm で
helm install -f temporal-values.yaml temporal charts/temporal-1.7.0.tgz --timeout 900s -n launchs-org --wait
```

- **PostgreSQL の起動が先**。postgres-operator がクラスタ(3 台)を作り、`main` `task_user` `auth` `temporal` のユーザーと、5 つの DB を作る。`status: Running` になるのを待ってから、アプリを再起動する(DB のユーザーが無い間は、アプリが SASL 認証で落ちる)。
- アプリは `rollout restart` で再起動する。Temporal が立つまで、builder / controller は接続エラーを出す。

## 10. Harbor

```bash
kubectl create namespace main-harbor
kubectl label ns main-harbor pod-security.kubernetes.io/enforce=baseline pod-security.kubernetes.io/warn=baseline
helm install harbor charts/harbor-1.19.2.tgz -n main-harbor -f harbor-values.yaml --timeout 15m
```

- **リリース名は `harbor`**(Service 名が `harbor` になり、`harbor.main-harbor` で解決できる。アプリの設定がこの名前を前提にしている)。
- 値: `expose.type=clusterIP`、TLS は自己署名(`commonName: harbor.main-harbor`、SAN も同じ)、`externalURL: https://harbor.main-harbor`、永続化は registry 30Gi・database 5Gi・redis 1Gi・jobservice 1Gi、`trivy` と `metrics` は無効。管理者パスワードは、リポジトリに入れない値ファイル(権限 600)に入れる。
- **ノードから Harbor を引くための設定**(アプリがビルドしたイメージを kubelet が pull するため):
  - 各 worker に `harbor-registry.yaml`(`launchs-org/terraform` の `talos/`)を当てる。`harbor.main-harbor` を **Harbor の ClusterIP に固定する hosts エントリ**と、**Harbor の自己署名 CA の信頼**(`tls.ca`)。証明書の検証は無効にしない(無効にする設定は、自動モードで拒否された)。
  - **Harbor の ClusterIP が変わると壊れる**(Service を作り直したとき)。
  - NetworkPolicy: `manifest` の `harbor-combined-policy` は旧環境のノード網(`10.10.11.0/24`)しか許可しない。`harbor-cnp.yaml`(CiliumNetworkPolicy)で、ノード(`host` / `remote-node`)から nginx の 8443 / 8080 を許可する。無いと、pull が `DeadlineExceeded` になる。
- **`buildkit` プロジェクト**を、手動で作る(`HARBOR_PROJECT=buildkit` として参照される)。
- **アプリ用のロボットアカウント**を作り、admin の代わりに使う。controller が作るプロジェクト別ロボットに付与する権限と同じ権限(システムレベルでプロジェクトの作成・一覧、全プロジェクトに対して `controller/src/k8s/harbor.go` の権限セット)が必要(自分が持たない権限は付与できないため)。名前は `robot$launchs-manager`、期限なし。作成時にしか表示されない secret は、リポジトリに入れない権限 600 のファイルに保管する。
- **証明書の期限は発行から 1 年(2027-10-08)**。切れると、ノードの CA 信頼とアプリからの接続が壊れる。更新時は、CA の再取得と Talos の設定の更新が必要。

## 11. gVisor

- 拡張(§8)、containerd への `runsc` の登録(worker の設定に含まれる)、**RuntimeClass `gvisor`**(handler `runsc`)の 3 点で 1 セット。

```bash
kubectl apply -f runtimeclass-gvisor.yaml
```

- 使うときは、Pod に `runtimeClassName: gvisor` を付ける。RuntimeClass が無いと、その Pod は拒否される。「全 Pod を gVisor にする」設定は、Cilium・Longhorn などホストに近いシステム Pod が壊れるので使わない。
- **Cilium の設定が必要**: gVisor は独自のネットワークスタックを持ち、`connect()` ではなくパケットを直接送る。Cilium の socket-LB(`connect()` で ClusterIP を書き換える)が効かず、Service の IP に届かない。`socketLB.hostNamespaceOnly=true` で、Pod の中ではパケット単位の変換にする。**設定後は、エージェントの再起動が必須**(§5)。
- 検証(runc と同じ結果): Pod IP、ClusterIP、Service の短縮名、`*.svc.cluster.local`、DNS、Longhorn の RWO / RWX のボリューム。
- **ビルド(rootless buildkit)を gVisor で動かす条件**: `BUILDKITD_FLAGS` に `--oci-worker-snapshotter=native` を足す(overlay が使えず、`RUN` が `/bin/sh が見つからない` で失敗する)。`sample-go-app` で、railpack のビルドと Harbor への push が成功(約 2 分)。native はコピー方式で、overlay より遅く、ディスクも使う。
- **アプリ側の対応**: `backend` のブランチ `feat/gvisor-runtime`(環境変数 `BUILD_RUNTIME_CLASS` / `APP_RUNTIME_CLASS`)。マージ、CI でのイメージ作成、`manifest` のダイジェストの更新、環境変数の追加で有効になる(未反映)。

## 12. Secret の生成

`manifest` の `setup.py` は対話式(`getpass`)で、非対話では使えない。同じ項目を、スクリプトで生成する。**値は表示しない・リポジトリに入れない**。

| 項目 | 方法 |
|---|---|
| DB のパスワード(`main` `task_user` `auth` `temporal`) | ランダム 32 文字。`postgresql-secret.yaml` は base64 |
| `DB_DSN`(auth 用) | `host=launchs-org-database-cluster.launchs-org port=5432 sslmode=require TimeZone=Asia/Tokyo user=auth dbname=authdb` とパスワード |
| `TOKEN_SECRET` `ADMIN_SESSION_KEY` `ARCHIVE_UPLOAD_TOKEN_SECRET` | `secrets.token_urlsafe(48)` |
| `JWT_PRIVATE_KEY` / `JWT_PUBLIC_KEY`、`CLI_TOKEN_PRIVATE_KEY` / `CLI_TOKEN_PUBLIC_KEY` | `openssl genpkey -algorithm ed25519`(秘密鍵は 600 の権限で保管) |
| `AdminEmail` / `AdminPassword` | 管理者のメールと、ランダムなパスワード |
| OAuth(Discord / Google / GitHub / Microsoft) | 使うものだけ設定(空でも起動する)。コールバック URL は `https://www.launchs.org/auth/oauth/<provider>/callback` |
| `HARBOR_URL` | `https://harbor.main-harbor` |
| `HARBOR_ROBOT_NAME` / `HARBOR_ROBOT_SECRET`(と `HARBOR_USERNAME` / `HARBOR_PASSWORD`) | §10 のロボット(admin は使わない) |

## 13. バックアップ(etcd)

master の更新や、設定を大きく変える前に、etcd のスナップショットを取る。

```bash
talosctl -n 192.168.10.161 -e 192.168.10.161 etcd snapshot ./etcd-$(date +%Y%m%d-%H%M%S).snapshot
```

スナップショットには Secret の内容(暗号化前の形)が含まれうるので、保管先の権限に注意する。

## 14. 外部公開(Cloudflare Tunnel)

- `cloudflared` namespace に、2 レプリカの Deployment(`launchs-org/terraform` の `talos/manifests/cloudflared.yaml`)。トークンは Secret `cloudflared-token`(標準入力で渡す。マニフェストには入れない)。
- 制限: PodSecurity は `restricted`、`runAsNonRoot`、`readOnlyRootFilesystem`、`drop ALL`。NetworkPolicy は ingress を全拒否、egress を DNS・Traefik・Cloudflare のエッジ(7844 TCP/UDP、443)だけに。
- **Cloudflare 側**: Public Hostname を `www.launchs.org` と `*.launchs.org`(ユーザーのアプリが動的に `<名前>.launchs.org` になる)に、向け先 `http://traefik.traefik.svc.cluster.local:80`。ワイルドカードは DNS レコードが自動で作られないので、CNAME を手動で追加する。
- `www.launchs.org` は Cloudflare Access で保護できる。`*.launchs.org`(ユーザーのアプリ)は、Access の対象にするかを決める。
- **止めるとき**: `kubectl -n cloudflared scale deploy/cloudflared --replicas=0`(外部からのアクセスが遮断される)。再開は `--replicas=2`。

## 15. プロジェクト namespace の egress 制限

controller が新しいプロジェクトの namespace に作る NetworkPolicy は、**egress が全許可**(LAN・ノード・k8s API サーバーに届く)。

- 既存のプロジェクトには、`egress-policy.yaml` で置き換え済み: 同じ namespace、DNS(kube-dns)、インターネット(プライベート帯 `10/8` `172.16/12` `192.168/16` `100.64/10` `169.254/16` を除く)だけを許可。
- 検証: DNS・同じ namespace・インターネットは通り、k8s API・ノード・Proxmox・他の namespace(Harbor・Temporal)は遮断された。
- **新規プロジェクトは、`backend` のブランチ `feat/egress-restrict-project-ns`(controller の修正)が反映されるまで、全許可のまま**。反映までは、新規 namespace に `egress-policy.yaml`(`__NS__` を置換)を手で当てる。

## 16. 動作確認のチェックリスト

- [ ] 全 5 ノードが Ready。`kubectl get pods -A` に、Running / Completed 以外がない。
- [ ] PostgreSQL が 3 台(master 1、replica 2)。
- [ ] `talosctl get extensions` で、稼働中のスキーマティックが、worker は `c527b6b2…`、master は `ce4c9805…`。
- [ ] Proxmox の `qm agent <vmid> ping` が、全 5 台で応答する。
- [ ] Longhorn: PVC(RWX)を作り、Pod から読み書きでき、拡張できる。
- [ ] ノードから Harbor を pull できる(`image-pull-policy=Always` のテスト Pod)。
- [ ] gVisor の Pod が起動し、`dmesg` に `Starting gVisor`。Service 名で通信できる。
- [ ] 外から: `www.launchs.org` は Cloudflare Access のログインへ、`<存在しない名前>.launchs.org` は Traefik の 404。
- [ ] ビルド: `sample-go-app` のビルドが完了し、Harbor に push され、pull してデプロイできる。
