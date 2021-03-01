# argocd-sample

## 導入手順

検証クラスタと本番クラスタを分ける前提。なので資格情報の中身は同じでも、SealedSecretのControllerは別々に存在するので、各環境ごとにSealedSecretリソースの中身はバラバラになるはず。

### SealedSecretをデプロイする

ArgoCDが利用するGitリポジトリへの資格情報をまず用意するところから始めます。  

```sh
# Sealed SecretsのControllerをデプロイ
kustomize build apps/secrets | kubectl apply -l needed-for-initialization!=false -f -

# kubeseal のインストール
VERSION=v0.15.0
wget https://github.com/bitnami-labs/sealed-secrets/releases/download/${VERSION}/kubeseal-linux-amd64 -O kubeseal
sudo install -m 755 kubeseal /usr/local/bin/kubeseal
kubeseal --version

# Secretリソースの作成
kubectl create secret generic repo-secrets --dry-run=client --from-literal=username='<REPLACE_YOUR_USERNAME>' --from-literal=password='<REPLACE_YOUR_PASSWORD>' -o yaml >tmp.yaml
kubeseal -o yaml <tmp.yaml >repo-secrets.yaml

rm -f tmp.yaml
mv repo-secrets.yaml argocd/overlays/dev
```

### ArgoCDをデプロイする

```sh
kustomize build argocd/overlays/dev | kubectl apply -f -
```
