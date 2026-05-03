# **Cloud Run 2時間ハンズオン**

## 概要

このハンズオンは、Cloud Runの基本機能からセキュリティ・運用まで、2時間で効率的に学習できるよう調整されたバージョンです。

今回のハンズオンではリージョンは可能な限り`asia-east1`を利用します。
また、ハンズオンを実行するユーザは、基本ロールである「`Owner`」権限を持つプロジェクトを利用してください。

本ハンズオンは、Apache-2.0で配布されている [はじめてみよう Cloud Run ハンズオン](https://github.com/google-cloud-japan/gcp-getting-started-cloudrun/blob/main/tutorial.md)の内容を基に作成されています。

## Google Cloud プロジェクトの設定、確認

### **Cloud Shell の起動**

Google Cloudのプロジェクトにアクセスし、画面上部からCloud Shellを起動しましょう。



### **プロジェクトの課金が有効化されていることを確認する**

```bash
gcloud beta billing projects describe ${GOOGLE_CLOUD_PROJECT} | grep billingEnabled
```

出力結果の `billingEnabled` が **`true`** になっていることを確認してください。**`false`** の場合は、こちらのプロジェクトではハンズオンが進められません。

---

## **第1部: Cloud Run基本体験（60分）**



### **環境準備**



#### **gcloud コマンドラインツール設定**

Cloud Run の利用するリージョン、プラットフォームのデフォルト値を設定します。

```bash
gcloud config set run/region asia-east1
gcloud config set run/platform managed
```

#### **Google Cloud 機能（API）有効化設定**

```bash
gcloud services enable \
artifactregistry.googleapis.com \
run.googleapis.com \
cloudbuild.googleapis.com \
sourcerepo.googleapis.com \
container.googleapis.com \
secretmanager.googleapis.com
```

### **Cloud Runに直接デプロイ**



#### **1. アプリケーション用リポジトリを作成（Artifact Registry）**

```bash
gcloud artifacts repositories create cnsrun-app --repository-format=docker --location=asia-east1 --description="Docker repository for the-cloud-run app"
```

#### **2. docker コマンドの認証設定**

```bash
gcloud auth configure-docker asia-east1-docker.pkg.dev --quiet
```

#### **3. ローカル（Cloud Shell 上）にコンテナを作成**

```bash
(cd app/frontend && docker build -t asia-east1-docker.pkg.dev/${GOOGLE_CLOUD_PROJECT}/cnsrun-app/frontend:v1 .)
```

#### **4. Artifact Registryへプッシュ**

```bash
docker push asia-east1-docker.pkg.dev/${GOOGLE_CLOUD_PROJECT}/cnsrun-app/frontend:v1
```

#### **5. サービスアカウントの作成**

```bash
gcloud iam service-accounts create cnsrun-app-frontend --display-name "Service Account for cnsrun-frontend"
```

#### **6. Cloud Run にデプロイ**

```bash
gcloud run deploy cnsrun-frontend --image=asia-east1-docker.pkg.dev/${GOOGLE_CLOUD_PROJECT}/cnsrun-app/frontend:v1 \
--allow-unauthenticated \
--service-account=cnsrun-app-frontend
```

#### **7. 動作確認**

```bash
FRONTEND_URL=$(gcloud run services describe cnsrun-frontend --format='value(status.url)')
curl $FRONTEND_URL/frontend
```

`Hello cnsrun handson's user:D`が返却されることを確認します。

### **CI/CD設定**



#### **1. Cloud Buildのサービスアカウント作成**

```bash
gcloud iam service-accounts create cnsrun-cloudbuild --display-name "Service Account for Cloud Build in cnsrun"
gcloud projects add-iam-policy-binding ${GOOGLE_CLOUD_PROJECT} \
  --member=serviceAccount:cnsrun-cloudbuild@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com \
  --condition=None \
  --role=roles/cloudbuild.builds.builder
gcloud projects add-iam-policy-binding ${GOOGLE_CLOUD_PROJECT} \
  --member=serviceAccount:cnsrun-cloudbuild@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com \
  --condition=None \
  --role=roles/logging.logWriter
gcloud projects add-iam-policy-binding ${GOOGLE_CLOUD_PROJECT} \
  --member=serviceAccount:cnsrun-cloudbuild@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com \
  --condition=None \
  --role=roles/iam.serviceAccountUser
```

#### **2. GitHubリポジトリの接続（事前設定済み）**

GitHubとの接続設定は事前に完了しているため、スキップします。実際の環境では、Cloud BuildとGitHubの接続設定が必要です。

#### **3. Cloud Buildトリガーの作成**

```bash
REPO_NAME=$(gcloud beta builds repositories list --connection=cnsrun-app-handson --region=asia-east1 --format=json | jq -r .[].name)
```

```bash
gcloud beta builds triggers create github \
--name=cnsrun-frontend-trigger \
--region=asia-east1 \
--repository="$REPO_NAME" \
--branch-pattern=^main$ \
--build-config=app/frontend/cloudbuild_push.yaml \
--included-files=app/frontend/** \
--substitutions=_DEPLOY_ENV=main \
--service-account=projects/${GOOGLE_CLOUD_PROJECT}/serviceAccounts/cnsrun-cloudbuild@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com
```

#### **4. フロントエンドアプリケーションを修正してテスト**

アプリケーションを修正してCI/CDが動作することを確認します。

```bash
echo ${GOOGLE_CLOUD_PROJECT}
```

上記で表示されたプロジェクトIDを`app/frontend/cloudrun.yaml`の`PROJECT_ID`部分に置き換えます。

```bash
YOUR_PROJECT_ID=${GOOGLE_CLOUD_PROJECT}
sed -i -e "s/PROJECT_ID/${YOUR_PROJECT_ID}/g" app/frontend/cloudrun.yaml
```

変更をプッシュします。

```bash
git add app/frontend/cloudrun.yaml
git commit -m "feat: update project ID for 2-hour hands-on"
git push origin main
```

### **外部ALB設定**



#### **1. 自己署名証明書の作成**

```bash
openssl genrsa 2048 > private.key
openssl req -new -x509 -days 3650 -key private.key -sha512 -out cnsrun.crt -subj "/C=JP/ST=Kanagawa/L=Yokohama/O=uma-arai/OU=Container/CN=team.bit.uma.arai@gmail.com"
```

#### **2. 外部ALBコンポーネントの作成**

```bash
# グローバルIPアドレス
gcloud compute addresses create --global cnsrun-ip

# バックエンドサービス
gcloud compute backend-services create --global cnsrun-backend-services \
--load-balancing-scheme EXTERNAL_MANAGED

# URL マップ
gcloud compute url-maps create cnsrun-urlmaps \
  --default-service=cnsrun-backend-services

# SSL証明書とHTTPSプロキシ
gcloud compute ssl-certificates create cnsrun-certificate \
  --certificate ./cnsrun.crt --private-key ./private.key --global
gcloud compute target-https-proxies create cnsrun-https-proxies \
  --ssl-certificates=cnsrun-certificate \
  --url-map=cnsrun-urlmaps

# ロードバランサ
gcloud compute forwarding-rules create --global cnsrun-lb \
--target-https-proxy=cnsrun-https-proxies \
--address=cnsrun-ip \
--load-balancing-scheme=EXTERNAL_MANAGED \
--ports=443
```

#### **3. NEG の作成、バックエンドサービスへの追加**

```bash
gcloud beta compute network-endpoint-groups create cnsrun-app-neg-asia-east1 \
    --region=asia-east1 \
    --network-endpoint-type=SERVERLESS \
    --cloud-run-service=cnsrun-frontend

gcloud beta compute backend-services add-backend --global cnsrun-backend-services \
    --network-endpoint-group-region=asia-east1 \
    --network-endpoint-group=cnsrun-app-neg-asia-east1
```

#### **4. Cloud Runのアクセス制限設定**

フロントエンドアプリケーションを外部ALB経由のみアクセス可能に設定します。

`app/frontend/cloudrun.yaml`の設定を変更：

```patch
- run.googleapis.com/ingress: all
+ run.googleapis.com/ingress: internal-and-cloud-load-balancing
```

変更をプッシュ：

```bash
git add app/frontend/cloudrun.yaml
git commit -m "feat: restrict access to load balancer only"
git push origin main
```

#### **5. 疎通確認**

```bash
LB_GLOBAL_IP=$(gcloud compute addresses describe cnsrun-ip --global --format='value(address)')
watch -n 5 curl -sk https://$LB_GLOBAL_IP/frontend
```

正常に応答が返ってくることを確認したら、`Ctrl+C`で停止します。

---

## **第2部: セキュリティ・運用（55分）**



### **WAF設定（Cloud Armor）**



#### **1. セキュリティポリシーの作成**

```bash
SECURITY_POLICY_NAME=cnsrun-waf-policy
gcloud compute security-policies create $SECURITY_POLICY_NAME
```

#### **2. セキュリティルールの作成**

基本的な攻撃パターンに対するルールを設定します。

```bash
# SQLインジェクション対策
gcloud compute security-policies rules create 1001 \
--security-policy $SECURITY_POLICY_NAME  \
--description "SQL injection" \
--expression "evaluatePreconfiguredExpr('sqli-v33-stable')" \
--action=deny-403

# クロスサイトスクリプティング対策
gcloud compute security-policies rules create 1002 \
--security-policy $SECURITY_POLICY_NAME  \
--description "Cross-site scripting" \
--expression "evaluatePreconfiguredExpr('xss-v33-stable')" \
--action=deny-403

# リモートコード実行対策
gcloud compute security-policies rules create 1003 \
--security-policy $SECURITY_POLICY_NAME  \
--description "Remote code execution" \
--expression "evaluatePreconfiguredExpr('rce-v33-stable')" \
--action=deny-403
```

#### **3. バックエンドサービスへの適用**

```bash
BACKEND_SERVICE_NAME=$(gcloud compute backend-services list --format=json | jq -r .[].name | grep cnsrun)
gcloud compute backend-services update $BACKEND_SERVICE_NAME \
    --security-policy $SECURITY_POLICY_NAME \
    --global
```

#### **4. セキュリティテスト**

正常系のテスト：

```bash
LB_GLOBAL_IP=$(gcloud compute addresses describe cnsrun-ip --global --format='value(address)')
curl -i -k https://$LB_GLOBAL_IP/frontend
# HTTP 200 OK が返却されることを確認
```

異常系のテスト（XSS攻撃）：

```bash
curl -i -k https://$LB_GLOBAL_IP/frontend?test="<script>alert('XSS')</script>"
# HTTP 403 Forbidden が返却されることを確認
```

### **脆弱性スキャン設定**



#### **1. Artifact Registryの脆弱性スキャン有効化**

Google Cloud ConsoleからArtifact Registryの設定で脆弱性スキャンを有効にします。

 Artifact Registry に移動

設定 に移動し、 有効にする  をクリックします。

#### **2. 新しいイメージでスキャンテスト**

```bash
(cd app/frontend && touch dummy_scan_test && docker build -t asia-east1-docker.pkg.dev/${GOOGLE_CLOUD_PROJECT}/cnsrun-app/frontend:scan-test .)
docker push asia-east1-docker.pkg.dev/${GOOGLE_CLOUD_PROJECT}/cnsrun-app/frontend:scan-test
```

Artifact Registryコンソールで脆弱性スキャン結果を確認します。

### **SLO監視設定**



#### **1. Cloud RunでのSLO設定**

Cloud Run に移動

1. フロントエンドアプリケーション`cnsrun-frontend`を選択
2. SLO タブを選択
3. SLOを作成ボタンを押す

#### **2. SLI設定**

- 指標の選択：`可用性`
- リクエストベースを選択して続行

#### **3. SLO設定**

- コンプライアンス期間：`カレンダー`、`1暦日`
- パフォーマンス目標：`99%`

続行を押し、最終確認画面でSLOを作成を押下します。

### **リソース削除とまとめ**



#### **作成したリソースの削除**

課金を防ぐために、作成したリソースを削除します。

```bash
# Cloud Run サービス削除
gcloud run services delete cnsrun-frontend --region=asia-east1 --quiet

# ロードバランサ関連削除
gcloud compute forwarding-rules delete cnsrun-lb --global --quiet
gcloud compute target-https-proxies delete cnsrun-https-proxies --quiet
gcloud compute ssl-certificates delete cnsrun-certificate --quiet
gcloud compute url-maps delete cnsrun-urlmaps --quiet
gcloud compute backend-services delete cnsrun-backend-services --global --quiet
gcloud compute network-endpoint-groups delete cnsrun-app-neg-asia-east1 --region=asia-east1 --quiet
gcloud compute addresses delete cnsrun-ip --global --quiet

# セキュリティポリシー削除
gcloud compute security-policies delete cnsrun-waf-policy --quiet

# Artifact Registry削除
gcloud artifacts repositories delete cnsrun-app --location=asia-east1 --quiet

# Cloud Build トリガー削除
gcloud beta builds triggers delete cnsrun-frontend-trigger --region=asia-east1 --quiet

# サービスアカウント削除
gcloud iam service-accounts delete cnsrun-app-frontend@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com --quiet
gcloud iam service-accounts delete cnsrun-cloudbuild@${GOOGLE_CLOUD_PROJECT}.iam.gserviceaccount.com --quiet
```

## **お疲れ様でした！**



2時間でCloud Runの以下の内容を学習しました：

**基本機能:**
- コンテナイメージのビルドとデプロイ
- CI/CDパイプラインの構築
- 外部ロードバランサとの連携

**セキュリティ・運用:**
- WAFによるセキュリティ強化
- コンテナイメージの脆弱性スキャン
- SLOによる監視設定

これらの知識を活用して、本番環境でCloud Runを安全に運用してください！

より詳細な機能については、フルバージョンのハンズオンもご利用ください：
- [5章ハンズオン（基本編）](handson_chap5.md)
- [6章ハンズオン（実践編）](handson_chap6.md)