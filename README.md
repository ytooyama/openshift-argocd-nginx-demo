# Argo CD nginx CDテスト

既存Argo CDから同一クラスターのnginx-testへDeploymentとServiceを自動同期します。
OpenShift/CRC、またはArgo CD導入済みのMicroShift向けです。実環境での動作・権限・イメージ取得は未検証です。

## 0. in-cluster参照権限の付与
一般ユーザーでArgo CDにログインしin-clusterを参照できない場合は、必要に応じて既存のRBACポリシーに次のルールを追記してください。
OpenShift Localを使った場合は developerですが、実際の環境の対象のユーザー名に置き換えてください。設定方法はマニュアルを参照してください。

```text
p, developer, clusters, get, *, allow
```

このルールは、Argo CDに登録されたすべてのクラスターの参照を許可します。デプロイ権限やOpenShift側の管理権限を付与するものではありません。

## 1. 準備

前提: ocでログイン済み、Argo CD稼働済み、クラスターからGitHubとregistry.access.redhat.comに接続可能。
実行者にはApplication作成権限、準備担当者にはNamespace作成権限が必要です。
default AppProjectで対象リポジトリ・配置先・Deployment/Serviceが許可されている必要があります。

```bash
tar -xzf argocd-nginx-test.tar.gz
cd argocd-nginx-test
```

application.yamlのrepoURLを実際のGitHubリポジトリURLに変更し、metadata.namespaceを実際のArgo CDのNamespaceに合わせます（初期値openshift-gitops）。
以下がGitHubリポジトリのルートになるように配置・コミット・pushします。
公開リポジトリを想定します。非公開の場合はArgo CDへのリポジトリ認証情報登録が必要です。

```text
application.yaml
nginx/nginx.yaml
README.md
```

## 2. Namespaceと権限確認

```bash
oc get namespace nginx-test
# 存在しない場合のみ作成
oc create namespace nginx-test
ARGOCD_NS=openshift-gitops  # 実際のArgo CDのNamespace
oc get deployment,statefulset -n "$ARGOCD_NS" -o custom-columns='KIND:.kind,NAME:.metadata.name,SA:.spec.template.spec.serviceAccountName'
```

Red Hat公式資料で既定インスタンスのSA名はopenshift-gitops-argocd-application-controllerと確認しています。
導入方法・インスタンス名で異なるため、上の出力で実際のApplication ControllerのSA名を確認してください。
ApplicationSet ControllerやServerのSAを選ばないでください。

```bash
CONTROLLER_SA='<確認したApplication ControllerのSA名>'
oc get serviceaccount "$CONTROLLER_SA" -n "$ARGOCD_NS"
oc auth can-i --list -n nginx-test --as="system:serviceaccount:${ARGOCD_NS}:${CONTROLLER_SA}"
```

--asには偽装権限が必要です。Forbiddenの場合はSAの権限不足と断定せず管理者に確認してください。
同一クラスターへの接続にこのSAを使用し、Deployment/Service等の管理権限不足を確認できた場合に限り、権限付与可能な管理者が次を実行します。
これは検証Namespace内のadmin権限を付与する例で、最小権限の定義ではありません。

```bash
oc create rolebinding argocd-nginx-test --clusterrole=admin --serviceaccount="${ARGOCD_NS}:${CONTROLLER_SA}" -n nginx-test --dry-run=client -o yaml | oc apply -f -
```

SAのNamespaceはArgo CD側、RoleBindingのNamespaceはnginx-testです。
Operatorの管理対象Namespace制限やArgo CDのクラスター接続設定によっては、このRoleBindingだけでは不十分です。その場合は管理者に設定を確認してください。

## 3. 適用と確認

```bash
oc apply -f application.yaml
oc get applications.argoproj.io nginx-test -n "$ARGOCD_NS" -w
```

Synced / Healthyを確認したらCtrl+Cで監視を終了します。

```bash
oc rollout status deployment/nginx -n nginx-test --timeout=180s
oc get pods,svc -n nginx-test
oc port-forward -n nginx-test service/nginx 8080:80
```

別ターミナルで実行します。HTTP 200とArgo CD nginx CD test OKが期待値です。

```bash
curl -i http://127.0.0.1:8080/
```

## 4. Git変更によるCD確認

GitHub上のnginx/nginx.yamlのspec.replicasを1から2に変更してコミットします。
Gitの定期確認後に同期するため、即時反映とは限りません。

```bash
oc get applications.argoproj.io nginx-test -n "$ARGOCD_NS" -w
# Synced / HealthyになったらCtrl+C
oc get deployment nginx -n nginx-test
oc get pods -n nginx-test -l app=nginx
```

READYが2/2、nginx Podが2個ともReadyならCD反映を確認できます。
prune/selfHealは有効です。Gitから削除した管理対象リソースは同期時に削除され、手動変更はGitの状態へ戻されます。

## 構成・参照資料

UBI 9 nginx 1.26、8080、非root実行を使用し、固定UID・特権・追加SCCは指定していません。
イメージの既定コマンドは使用方法の表示なので/usr/libexec/s2i/runを明示しています。
latestは検証用途の可変タグです。

- [Red Hat: 既定Application Controller SA](https://docs.redhat.com/en/documentation/red_hat_openshift_gitops/1.19/html-single/managing_cluster_configuration/index)
- [UBI nginx定義（任意UID対応・8080・既定コマンド）](https://github.com/sclorg/nginx-container/blob/master/1.26/Dockerfile.rhel9)
- [nginx起動スクリプト](https://github.com/sclorg/nginx-container/blob/master/1.26/s2i/bin/run)
