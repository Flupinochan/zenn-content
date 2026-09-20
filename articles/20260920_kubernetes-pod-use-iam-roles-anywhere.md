---
title: "IAM Roles AnywhereでオンプレミスKubernetes PodからAWSリソースを利用する方法"
emoji: "🍂"
type: "tech"
topics: ["kubernetes", "aws", "iam", "iamrolesanywhere", "eks"]
published: true
---

## はじめに

前回はオンプレミス環境のKubernetesからECRのImageをPullしました

今回はKubernetesのPodからAWSリソースを利用します

具体的にはaws cliが実行可能なImageでPodを起動し、aws s3 lsコマンドが実行できることを確認します

仕組みはECRのImageをPullする時と同様で、Pod用の証明書 (Secret) を作成し、その証明書に対してIAMロールの信頼関係を設定することです

## 対象者

- オンプレミス環境のKubernetes PodでAWSリソースを利用したい方
- IAM Roles Anywhereの使用例を知りたい方

:::message
EKSの場合はIAM Roleを直接付与できるため、本記事のIAM Roles Anywhereを利用する必要はありません
:::

## 前提

@[card](https://zenn.dev/metalmental/articles/20260917_kubernetes-ecr-auth-iam-roles-anywhere)

:::message
上記で信頼アンカー (Trust Anchor) を作成していることを前提とします
:::

また、以下2つのKubernetes nodeがある状態で説明します

- control-plane
  - name: nipogi
  - arch: amd64
- worker
  - name: raspberry
  - arch: arm64

## 1. 変数を定義 (共通)

### 1. 前回の記事で作成した値を定義

```bash
export CA_WORKDIR="/root/iam-roles-anywhere-ca"
export CA_NAME="ca"
export CA_KEY="${CA_NAME}.key"
export CA_PEM="${CA_NAME}.pem"
export CA_DAYS=3650
export CA_C="JP"
export CA_O="MetalMental"
export CA_LEAF_CONF_FILE_NAME="leaf_ext.cnf"
export AWS_REGION="ap-northeast-1"
export AWS_ACCOUNT_ID="AWSアカウントIDを指定"
# 例: export AWS_ACCOUNT_ID="012345678910"
export TRUST_ANCHOR_ARN="前回の記事で作成したものを記載"
# 例: export TRUST_ANCHOR_ARN="arn:aws:rolesanywhere:ap-northeast-1:012345678910:trust-anchor/56cb8347-8cc7-45dd-9f2d-a3902aae9462"
```

### 2. 新規に作成する値を定義

```bash
export NAMESPACE="default"
export WORKLOAD_NAME="backend"
export WORKLOAD_CN_NAME="${WORKLOAD_NAME}"
export WORKLOAD_KEY="${WORKLOAD_NAME}.key"
export WORKLOAD_CSR="${WORKLOAD_NAME}.csr"
export WORKLOAD_PEM="${WORKLOAD_NAME}.pem"
export WORKLOAD_SECRET_NAME="iam-roles-anywhere-credential-helper-sidecar-${WORKLOAD_NAME}"
export WORKLOAD_SECRET_YAML="${WORKLOAD_SECRET_NAME}.yaml"
export WORKLOAD_CONTAINER_NAME="app"
export IAM_ROLE_NAME="${WORKLOAD_CN_NAME}-Role"
export ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${IAM_ROLE_NAME}"
export PROFILE_NAME="${WORKLOAD_CN_NAME}-Profile"
export PROFILE_ARN="作成した後に記載"
# 例: export PROFILE_ARN="arn:aws:rolesanywhere:ap-northeast-1:012345678910:profile/0031f6f5-0017-4597-a0bf-903d7d567c3d"
export POD_TEST_YAML="pod-test.yaml"
```

## 2. AWSにアクセス可能な環境での作業

### 1. 信頼アンカー用のIAMロール作成

IAMロール画面で `ロールを作成` をクリック

![1.png](/images/20260920_kubernetes-pod-use-iam-roles-anywhere/1.png)

1. `信頼されたエンティティタイプ`: `カスタム信頼ポリシー` を選択

以下2つを置き換えてコピペ

2. `TRUST_ANCHOR_ARN`: `echo "${TRUST_ANCHOR_ARN}"` で表示された内容
3. `WORKLOAD_CN_NAME`: `echo "${WORKLOAD_CN_NAME}"` で表示された内容

```json
{
    "Version":"2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": [
                    "rolesanywhere.amazonaws.com"
                ]
            },
            "Action": [
                "sts:AssumeRole",
                "sts:TagSession",
                "sts:SetSourceIdentity"
            ],
            "Condition": {
                "ArnEquals": {
                    "aws:SourceArn": [
                        "TRUST_ANCHOR_ARN"
                    ]
                },
                "StringEquals": {
                    "aws:PrincipalTag/x509Subject/CN": "WORKLOAD_CN_NAME"
                }
            }
        }
    ]
}
```

4. `次へ` を選択

![2.png](/images/20260920_kubernetes-pod-use-iam-roles-anywhere/2.png)

`許可ポリシー`: `AmazonS3FullAccess` (S3へのアクセス権限) を選択し、`次へ` を選択

![3.png](/images/20260920_kubernetes-pod-use-iam-roles-anywhere/3.png)

`ロール名`: 以下で表示された内容を入力し、`ロールを作成` をクリック

```bash
echo ${IAM_ROLE_NAME}
```

![4.png](/images/20260920_kubernetes-pod-use-iam-roles-anywhere/4.png)


### 2. プロファイルを作成

IAMロール画面の右下の `管理` をクリック

![11.png](/images/20260920_kubernetes-pod-use-iam-roles-anywhere/11.png)

`プロファイルを作成` をクリック

![5.png](/images/20260920_kubernetes-pod-use-iam-roles-anywhere/5.png)

1. `プロファイル名`: 以下で表示された内容をコピペ

```bash
echo ${PROFILE_NAME}
```

2. `ロール`: 作成した信頼アンカー用のIAMロールを指定
3. `プロファイルを作成` をクリック

![6.png](/images/20260920_kubernetes-pod-use-iam-roles-anywhere/6.png)

作成したプロファイルARNを変数定義

```bash
export PROFILE_ARN="arn:aws:rolesanywhere:ap-northeast-1:012345678910:profile/0031f6f5-0017-4597-a0bf-903d7d567c3d"
```

![12.png](/images/20260920_kubernetes-pod-use-iam-roles-anywhere/12.png)

## 3. Kubernetes環境での作業

### 1. Workload (Pod) 用の証明書を作成

```bash
cd "${CA_WORKDIR}"

# PrivateKey作成
openssl genrsa -out "${WORKLOAD_KEY}" 4096
chmod 600 "${WORKLOAD_KEY}"

# CSR作成
openssl req -new \
  -key "${WORKLOAD_KEY}" \
  -out "${WORKLOAD_CSR}" \
  -subj "/C=${CA_C}/O=${CA_O}/CN=${WORKLOAD_NAME}"

# PEM作成
openssl x509 -req -in \
  "${WORKLOAD_CSR}" \
  -CA "${CA_PEM}" \
  -CAkey "${CA_KEY}" \
  -out "${WORKLOAD_PEM}" \
  -days "${CA_DAYS}" \
  -sha256 \
  -CAcreateserial \
  -extfile "${CA_LEAF_CONF_FILE_NAME}"
```

:::details 実行結果
```bash
root nipogi default ~/iam-roles-anywhere-ca ❯ ls -l
total 40
-rw-r--r-- 1 root root 1630 Sep 20 05:40 backend.csr
-rw------- 1 root root 3272 Sep 20 05:40 backend.key
-rw-r--r-- 1 root root 1935 Sep 20 05:40 backend.pem
-rw------- 1 root root 3272 Sep 19 23:22 ca.key
-rw-r--r-- 1 root root 1960 Sep 19 23:22 ca.pem
-rw-r--r-- 1 root root   41 Sep 20 05:40 ca.srl
-rw-r--r-- 1 root root   70 Sep 19 23:22 leaf_ext.cnf
-rw-r--r-- 1 root root 1626 Sep 19 23:32 nipogi.csr
-rw------- 1 root root 3272 Sep 19 23:32 nipogi.key
-rw-r--r-- 1 root root 1935 Sep 19 23:32 nipogi.pem
root nipogi default ~/iam-roles-anywhere-ca ❯
```
:::

### 2. 証明書をもとにSecretを作成

```bash
cd "${CA_WORKDIR}"

kubectl create secret tls "${WORKLOAD_SECRET_NAME}" \
  --cert="${WORKLOAD_PEM}" \
  --key="${WORKLOAD_KEY}" \
  -n "${NAMESPACE}" \
  -o yaml \
  --dry-run=client > "${WORKLOAD_SECRET_YAML}"

kubectl apply -f "${WORKLOAD_SECRET_YAML}"
```

:::details 実行結果
```bash
root nipogi default ~/iam-roles-anywhere-ca ❯ kubectl get secret "${WORKLOAD_SECRET_NAME}"
NAME                                                   TYPE                DATA   AGE
iam-roles-anywhere-credential-helper-sidecar-backend   kubernetes.io/tls   2      18s
root nipogi default ~/iam-roles-anywhere-ca ❯
```
:::

### 3. 動作確認

#### 1. aws-cli Podを作成

```bash
cat > "${POD_TEST_YAML}" << EOF
apiVersion: v1
kind: Pod
metadata:
  name: "${WORKLOAD_NAME}"
  namespace: ${NAMESPACE}
spec:
  initContainers:
    - name: iamra-sidecar
      image: public.ecr.aws/rolesanywhere/credential-helper:latest
      restartPolicy: Always
      command: ["aws_signing_helper"]
      args:
        - "serve"
        - "--certificate"
        - "/iamra/tls.crt"
        - "--private-key"
        - "/iamra/tls.key"
        - "--trust-anchor-arn"
        - "\$(TRUST_ANCHOR_ARN)"
        - "--profile-arn"
        - "\$(PROFILE_ARN)"
        - "--role-arn"
        - "\$(ROLE_ARN)"
      env:
        - name: TRUST_ANCHOR_ARN
          value: "${TRUST_ANCHOR_ARN}"
        - name: PROFILE_ARN
          value: "${PROFILE_ARN}"
        - name: ROLE_ARN
          value: "${ROLE_ARN}"
      volumeMounts:
        - name: iamra-certs
          mountPath: /iamra
          readOnly: true
  containers:
    - name: "${WORKLOAD_CONTAINER_NAME}"
      image: public.ecr.aws/aws-cli/aws-cli:latest
      command: ["sleep", "3600"]
      env:
        - name: AWS_EC2_METADATA_SERVICE_ENDPOINT
          value: "http://127.0.0.1:9911/"
  volumes:
    - name: iamra-certs
      secret:
        secretName: "${WORKLOAD_SECRET_NAME}"
EOF

kubectl apply -f "${POD_TEST_YAML}"
kubectl get pods -o wide -w
```

#### 2. Podでaws s3 lsコマンドを実行

```bash
kubectl exec "${WORKLOAD_NAME}" -n "${NAMESPACE}" -c "${WORKLOAD_CONTAINER_NAME}" -- aws s3 ls --region "${AWS_REGION}"
```

::: details 実行結果
```bash
# manifestを確認
root nipogi default ~/iam-roles-anywhere-ca ❯ cat pod-test.yaml 
apiVersion: v1
kind: Pod
metadata:
  name: "backend"
  namespace: default
spec:
  initContainers:
    - name: iamra-sidecar
      image: public.ecr.aws/rolesanywhere/credential-helper:latest
      restartPolicy: Always
      command: ["aws_signing_helper"]
      args:
        - "serve"
        - "--certificate"
        - "/iamra/tls.crt"
        - "--private-key"
        - "/iamra/tls.key"
        - "--trust-anchor-arn"
        - "$(TRUST_ANCHOR_ARN)"
        - "--profile-arn"
        - "$(PROFILE_ARN)"
        - "--role-arn"
        - "$(ROLE_ARN)"
      env:
        - name: TRUST_ANCHOR_ARN
          value: "arn:aws:rolesanywhere:ap-northeast-1:012345678910:trust-anchor/c2197007-2131-4f0b-9f8b-00fa9a12e164"
        - name: PROFILE_ARN
          value: "arn:aws:rolesanywhere:ap-northeast-1:012345678910:profile/0031f6f5-0017-4597-a0bf-903d7d567c3d"
        - name: ROLE_ARN
          value: "arn:aws:iam::012345678910:role/backend-Role"
      volumeMounts:
        - name: iamra-certs
          mountPath: /iamra
          readOnly: true
  containers:
    - name: "app"
      image: public.ecr.aws/aws-cli/aws-cli:latest
      command: ["sleep", "3600"]
      env:
        - name: AWS_EC2_METADATA_SERVICE_ENDPOINT
          value: "http://127.0.0.1:9911/"
  volumes:
    - name: iamra-certs
      secret:
        secretName: "iam-roles-anywhere-credential-helper-sidecar-backend"
root nipogi default ~/iam-roles-anywhere-ca ❯

# podを作成
root nipogi default ~/iam-roles-anywhere-ca ❯ cat > "${POD_TEST_YAML}" << EOF
kubectl apply -f "${POD_TEST_YAML}"
kubectl get pods -o wide -w
pod/backend created
NAME      READY   STATUS     RESTARTS   AGE   IP       NODE        NOMINATED NODE   READINESS GATES
backend   0/2     Init:0/1   0          0s    <none>   raspberry   <none>           <none>
backend   0/2     Init:0/1   0          0s    <none>   raspberry   <none>           <none>
backend   0/2     Init:0/1   0          1s    <none>   raspberry   <none>           <none>
backend   1/2     PodInitializing   0          2s    10.244.151.173   raspberry   <none>           <none>
backend   2/2     Running           0          3s    10.244.151.173   raspberry   <none>           <none>
root nipogi default ~/iam-roles-anywhere-ca took 6s ❯

# podにアクセスしてaws s3 lsコマンドを実行
root nipogi default ~/iam-roles-anywhere-ca took 6s ❯ kubectl exec "${WORKLOAD_NAME}" -n "${NAMESPACE}" -c "${WORKLOAD_CONTAINER_NAME}" -- aws s3 ls --region "${AWS_REGION}"
2026-04-28 22:52:12 amplify-d25csu3vso9tmw-...
2026-04-29 06:20:42 amplify-d25csu3vso9tmw-...
2026-04-28 23:11:48 aws-glue-assets-...
root nipogi default ~/iam-roles-anywhere-ca took 2s ❯ 
```
:::

:::message
Pod ManifestファイルにShellの環境変数からARNを取得し直接記載していますが、実務ではConfigMapで定義しておくとよいと思います
:::

## トラブルシューティング

| 症状                                    | 原因                                                   | 対処                                                           |
| --------------------------------------- | ------------------------------------------------------ | -------------------------------------------------------------- |
| `aws: [ERROR]: date value out of range` | 古いProfileARNないし存在しないProfileARNを指定していた | プロファイル作成後に発行された正しいARNを`PROFILE_ARN`に再設定 |

## おわりに

自分はDatabaseが苦手です

Databaseは基本的にお金がかかるサービスであり、あまり個人で利用していなかったからです

AWS RDSは当然高くて利用しません

無料のSupabase等のSaaSはスペックが低かったり、定期的にアクセスしたり、ログインしたりしないと停止してしまう、などの問題があり、継続的に利用できませんでした

自宅Kubernetesを用意した理由の一つが、PostgreSQLやOpenSearchを無料で制限無しに利用できることです

個人開発でDatabaseを利用しやすくなり、Databaseへの習熟度も上がると考えました

Redisも積極的に利用したいと考えているので楽しみです!

また、話が変わりますが、TermiusとTailscaleをスマホにインストールして、スマホから自宅サーバ (Kubernetes) にアクセスしてみました

便利だなぁと思いました (小並感)

使ったことのない方はぜひ試してみてください

P.S. `IAM Roles Anywhere` が `ロール（任意の場所）` と翻訳されていて分かりづらい...

## 参考URL

- [Connect your on-premises Kubernetes cluster to AWS APIs using IAM Roles Anywhere](https://aws.amazon.com/jp/blogs/security/connect-your-on-premises-kubernetes-cluster-to-aws-apis-using-iam-roles-anywhere/)
- [Get temporary security credentials from IAM Roles Anywhere](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/credential-helper.html)
