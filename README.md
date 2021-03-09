# argocd-sample

## 前提

ArgocdのIngressで利用するドメインは予め取得しておきます（自分はfreenomで取得しました）  
KubernetesクラスタはGKEを利用します。構築にはTerraformを使います。Terraformを使ったGCPクラスタ作成については[こちら](https://44smkn.hatenadiary.com/entry/2020/06/07/103953)から。  
external-dnsがちゃんと動くようにいくつか事前に設定をしておく必要があります。  
  
まず、`Workload Ientity`をクラスタで有効化しておきます。なのでクラスタの設定に下記のように宣言します。

```javascript
resource "google_container_cluster" "primary" {
    // 諸々を省略
    workload_identity_config {
      identity_namespace = "${local.project}.svc.id.goog"
    }
```

さらに、`Workload Ientity`を利用して、GCPサービスアカウントと紐付いたKubernetesサービスアカウントを作成するため下記のように宣言します。

```javascript
resource "google_service_account" "external-dns" {
  account_id   = "external-dns"
  display_name = "External DNS"
}

resource "google_project_iam_member" "external-dns" {
  project = local.project
  role    = "roles/dns.admin"
  member  = "serviceAccount:external-dns@infra-forklift-278011.iam.gserviceaccount.com"
  depends_on = [
    google_service_account.external-dns,
  ]
}

resource "google_service_account_iam_member" "external-dns-bindings" {
  service_account_id = "projects/infra-forklift-278011/serviceAccounts/external-dns@infra-forklift-278011.iam.gserviceaccount.com"
  role               = "roles/iam.workloadIdentityUser"
  member             = "serviceAccount:infra-forklift-278011.svc.id.goog[kube-system/external-dns]"
  depends_on = [
    google_project_iam_member.external-dns,
  ]
}
```

最後に、external-dnsで作成するドメインのホストゾーンを作成する宣言を行います。

```javascript
resource "google_dns_managed_zone" "argocd-sample-zone" {
  name        = "argocd-sample"
  dns_name    = "argocdsample-a1cac346.ml."
  description = "Automatically managed zone by kubernetes.io/external-dns"
  labels = {
    app = "argocd"
  }
}
```

修正した設定ファイルを利用し、`terraform apply`を行って前提の環境構築は完了です。

## 導入手順

検証クラスタと本番クラスタを分ける前提。なので資格情報の中身は同じでも、SealedSecretのControllerは別々に存在するので、各環境ごとにSealedSecretリソースの中身はバラバラになるはず。

### ArgoCDとSealedSecretをデプロイする

ArgoCD本体とArgoCDが利用するGitリポジトリへの資格情報をまず用意するところから始めます。  
kustomizeのバージョンはv3.9.2です。

```sh
gcloud container clusters get-credentials <RELACE_YOUR_CLUSTER>

# Sealed SecretsのControllerをデプロイ
kustomize build sealed-secrets | kubectl apply -f -

# Secretリソースの作成
kubectl create secret generic repo-secrets -n argocd --dry-run --from-literal=username='44smkn' --from-literal=password='1867hokan' -o yaml >tmp.yaml
kubeseal -o yaml <tmp.yaml >repo-secrets.yaml

rm -f tmp.yaml && mv repo-secrets.yaml argo-cd/overlays/dev
# gitで作成したSealedSecretでGitリポジトリを更新する

# argocdをデプロイ
kustomize build argo-cd/overlays/dev | kubectl apply -f -
```

### Applicationリソースを追加する

```sh
# argocdとその周辺アプリケーションのための Applicationリソースを作成
# このタイミングで先にデプロイした、sealed-secretも管理下に入れている
kustomize build applications/overlays/dev | kubectl apply -f -
```
