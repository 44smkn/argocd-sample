# argocd-sample

## 導入手順

検証クラスタと本番クラスタを分ける前提。なので資格情報の中身は同じでも、SealedSecretのControllerは別々に存在するので、各環境ごとにSealedSecretリソースの中身はバラバラになるはず。

### SealedSecretをデプロイする

ArgoCDが利用するGitリポジトリへの資格情報をまず用意するところから始めます。  

```sh
# kustomize/kubeseal はapply済の想定
# CRDリソースを先にデプロイ（ここ上手くやりたい）
kustomize build "github.com/argoproj/argo-cd/manifests/crds?ref=v1.8.6" | kubectl apply -f -

# Sealed SecretsのControllerをデプロイ
kustomize build apps/secrets | kubectl apply -f -

# Secretリソースの作成
kubectl create secret generic repo-secrets --dry-run --from-literal=username='<REPLACE_YOUR_USERNAME>' --from-literal=password='<REPLACE_YOUR_PASSWORD>' -o yaml >tmp.yaml
kubeseal -o yaml <tmp.yaml >repo-secrets.yaml

rm -f tmp.yaml && mv repo-secrets.yaml argocd/overlays/dev
# gitで作成したSealedSecretでGitリポジトリを更新する
```

### ArgoCDをデプロイする

```sh
kustomize build argocd/overlays/dev | kubectl apply -f -
```
