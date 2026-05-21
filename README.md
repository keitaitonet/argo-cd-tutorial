# argo-cd-tutorial

Argo CD + k3s + GitHub Actions の自宅ラボ用チュートリアル。

`app/index.html` を書き換えて `git push` すると、自宅 VM 上の nginx Pod が自動的に新しい内容に置き換わるところまでをゴールとする。

## アーキテクチャ

```
push (app/**)                                        ┌────────────────────┐
   ─► GitHub Actions ─► build & push image ─► GHCR ─►│  ghcr.io/.../...   │
                    │                                 └─────────▲──────────┘
                    │                                           │ pull
                    └─► manifests/kustomization.yaml の         │
                        image tag を書き換えて commit & push     │
                                       │                        │
                                       ▼                        │
                          ┌─────────────────────────┐           │
                          │ k3s on 自宅 Ubuntu VM   │           │
                          │  └─ Argo CD が Git 監視 ├───────────┘
                          │     ↓ auto-sync         │
                          │  nginx-hello Pod        │
                          └─────────────────────────┘
```

## リポジトリ構成

```
app/                       nginx の Docker ビルドコンテキスト
manifests/                 Kubernetes manifests (Kustomize)
argocd/application.yaml    Argo CD Application CRD
.github/workflows/         CI (build → push → manifest bump)
```

---

## 章1. リポジトリを GitHub に push（Mac 側）

1. GitHub 上で空の `keitaitonet/argo-cd-tutorial` を作成（README 等は入れない）
2. ローカルから push

   ```bash
   git remote add origin git@github.com:keitaitonet/argo-cd-tutorial.git
   git add -A
   git commit -m "chore: initial tutorial scaffold"
   git push -u origin main
   ```

3. GitHub の Actions タブを開き、`Build & Bump` が成功するのを確認
4. GitHub の Profile → Packages から `nginx-hello` を開き、**Package settings → Change visibility → Public** に変更
   - これで VM 側に imagePullSecret 不要

push 後、`manifests/kustomization.yaml` の `newTag` が `sha-xxxxxxx` に書き換わるコミットが自動的に増える。これが正常動作のサイン。

---

## 章2. VM に k3s をインストール

VM (Ubuntu) で以下を実行。

```bash
curl -sfL https://get.k3s.io | sh -

# kubectl を sudo なしで使えるようにする
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

kubectl get nodes
```

k3s には Traefik と ServiceLB が同梱されているので、追加インストール不要で IngressRoute が使える。

---

## 章3. Argo CD をインストール

VM で実行。

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 全 Deployment が ready になるまで待つ
kubectl -n argocd wait --for=condition=available --timeout=300s deployment --all

# 初期 admin パスワード
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

### UI へアクセス（port-forward）

VM 側で LAN に開けたまま待機させる：

```bash
kubectl -n argocd port-forward --address 0.0.0.0 svc/argocd-server 8080:443
```

Mac のブラウザで `https://<VM-IP>:8080` を開く。

- username: `admin`
- password: 上で取り出した値

自己署名証明書なので警告が出るが「詳細設定 → 移動」で進めばよい。

---

## 章4. Argo CD に Application を登録

VM で実行。

```bash
kubectl apply -f https://raw.githubusercontent.com/keitaitonet/argo-cd-tutorial/main/argocd/application.yaml
```

`argocd` namespace に `nginx-hello` Application が現れ、`manifests/` 配下を読み込んで自動 sync する。UI 上で緑（Synced / Healthy）になればOK。

---

## 章5. アプリにアクセス（Traefik IngressRoute）

Mac の `/etc/hosts` に追記：

```
<VM-IP>  nginx-hello.local
```

ブラウザで `http://nginx-hello.local` を開くと、`app/index.html` の内容が表示される。

---

## 章6. 自動デプロイを試す

`app/index.html` の `<h1>` などを書き換えて push する：

```bash
# 例
sed -i '' 's/Hello from Argo CD + k3s/Hello v2/' app/index.html
git add app/index.html
git commit -m "feat: update index.html"
git push
```

順に以下が起きる：

1. GitHub Actions `Build & Bump` が走る（数十秒〜1分）
2. GHCR に新しい sha tag の image が push される
3. 同じ workflow が `manifests/kustomization.yaml` の `newTag` を書き換えて commit
4. Argo CD が Git の変更を検知（poll 間隔は約3分、UI 上の `Refresh` で即時化可）
5. 自動で sync され、Pod が rolling update される
6. ブラウザを reload すると新しい内容が表示される

---

## トラブルシュート

- **Pod が `ImagePullBackOff`**: GHCR の image を public にしたか確認
- **Argo CD が動かない**: `kubectl -n argocd get pods` で repo-server, application-controller が Running か確認
- **IngressRoute が効かない**: `kubectl -n nginx-hello get ingressroute` と `kubectl -n kube-system get svc traefik` で確認、`/etc/hosts` の VM-IP が合っているか

## 参考

- Argo CD: https://argo-cd.readthedocs.io/
- k3s: https://docs.k3s.io/
- GHCR + Actions: https://docs.github.com/en/actions/tutorials/publish-packages/publish-docker-images
