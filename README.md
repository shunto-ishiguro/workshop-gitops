# GitOps Workshop 手順書

このリポジトリは Argo CD を使った GitOps デプロイ自動化のワークショップ用リポジトリです。

## 前提条件

以下のツールがインストール済みであること：

- Docker CE (27.0+)
- KinD (Kubernetes in Docker)
- kubectl (1.32+)
- Helm (3.16+)
- Git

---

## Step 1: KinD クラスターの作成

```bash
kind create cluster --name my-cluster
```

クラスターが正常に起動したか確認：

```bash
kubectl get nodes
```

`STATUS` が `Ready` になっていれば OK。

---

## Step 2: Argo CD のインストール（Helm）

### 2-1. Helm リポジトリの追加

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
```

### 2-2. Argo CD をインストール

```bash
helm install argocd argo/argo-cd -n argocd --create-namespace
```

### 2-3. Pod が起動するのを確認

```bash
kubectl get pod -n argocd -w
```

全ての Pod が `Running` になるまで待つ（Ctrl+C で終了）。

---

## Step 3: Argo CD UI にアクセス

### 3-1. ポートフォワード

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 3-2. 管理者パスワードの取得

別のターミナルで実行：

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

### 3-3. ブラウザでログイン

- URL: https://localhost:8080
- ユーザー名: `admin`
- パスワード: 上記で取得した文字列

> **注意**: Chrome で「この接続ではプライバシーが保護されません」という警告が表示された場合は、画面上の何もクリックせずにキーボードで `thisisunsafe` と入力してください。画面が自動的に遷移します。

---

## Step 4: Argo CD Application の適用

このリポジトリにある `argocd-application-private.yaml` を使って、Argo CD に Nginx アプリケーションを登録する。

```bash
kubectl apply -f argocd-application-private.yaml
```

### 適用される内容

Argo CD がこのリポジトリの `manifests/nginx/` 配下を監視し、以下のリソースを自動デプロイする：

- **Deployment** (`manifests/nginx/deployment.yaml`): Nginx を 3 レプリカで起動
- **Service** (`manifests/nginx/service.yaml`): ClusterIP で 80 番ポートを公開

### 同期状態の確認

```bash
kubectl get applications -n argocd
```

`STATUS` が `Synced`、`HEALTH` が `Healthy` になれば成功。

デプロイされたリソースの確認：

```bash
kubectl get pods,svc,deployment -l app=nginx
```

---

## Step 5: GitOps の動作確認

### 5-1. Git で変更してみる

例えば `manifests/nginx/deployment.yaml` の `replicas` を `3` から `5` に変更して push する：

```bash
git add manifests/nginx/deployment.yaml
git commit -m "Update nginx replicas to 5"
git push
```

Argo CD が自動で変更を検知し、クラスターに反映される。

### 5-2. セルフヒーリングの確認

手動でレプリカ数を変更しても、Argo CD が Git の状態に自動復元する：

```bash
kubectl scale deployment nginx --replicas=1
kubectl get pods -l app=nginx -w
```

しばらくすると Argo CD が元のレプリカ数に戻す。

---

## リポジトリ構成

```
workshop-gitops/
├── README.md                        # この手順書
├── argocd-application-private.yaml  # Argo CD Application 定義
└── manifests/
    └── nginx/
        ├── deployment.yaml          # Nginx Deployment (3 レプリカ)
        └── service.yaml             # Nginx Service (ClusterIP)
```

---

## 参考: GitOps のメリット

- **監査性**: 全ての変更が Git のコミット履歴に残る
- **再現性**: 同じ設定でどの環境にもデプロイ可能
- **ロールバック**: `git revert` で簡単に以前の状態に戻せる
- **セルフヒーリング**: 手動変更を自動で修正し、Git の状態を維持
