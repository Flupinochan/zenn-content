---
title: "IAM Roles AnywhereでオンプレミスKubernetesからECRのImageをPullする方法"
emoji: "🍁"
type: "tech"
topics: ["kubernetes", "aws", "iam", "iamrolesanywhere", "eks"]
published: true
---

## はじめに

最近、Raspberry PiやミニPCでオンプレミス環境 (自宅サーバ) のKubernetesを構築しました

オンプレミス環境のKubernetesでセキュアにAWS認証してECRのImageをPullする方法を調べた結果、ネット上に情報が少なく、手間取ったので手順を残しておきたかった次第です

仕組みとしては、自己認証局を構築して証明書を作成し、その証明書をIAM Roles Anywhere (Trust Anchor) に登録してAWS認証します

## 対象者

- オンプレミス環境のKubernetesでECRを利用したい方
- IAM Roles Anywhereの使用例を知りたい方

:::message
EKSの場合はIAM Roleを直接付与できるため、本記事のIAM Roles Anywhereを利用する必要はありません
:::

## 前提

以下2つのKubernetes nodeがある状態で説明します

- control-plane
  - name: nipogi
  - arch: amd64
- worker
  - name: raspberry
  - arch: arm64

## 1. 変数を定義 (全ノード共通)

```bash
export CA_WORKDIR="/root/iam-roles-anywhere-ca"
export CA_HOST="kubectl get nodesで表示されるNAMEかつCAのHostname"
# 例: export CA_HOST="nipogi"
export CA_NAME="ca"
export CA_KEY="${CA_NAME}.key"
export CA_PEM="${CA_NAME}.pem"
export CA_DAYS=3650
export CA_C="JP"
export CA_O="MetalMental"
export CA_CN="IAMRolesAnywhere-RootCA"
export CA_LEAF_CONF_FILE_NAME="leaf_ext.cnf"
export TRUST_ANCHOR_NAME="MetalMentalTrustAnchor"
export TRUST_ANCHOR_ARN="作成後に記載"
# 例: export TRUST_ANCHOR_ARN="arn:aws:rolesanywhere:ap-northeast-1:012345678910:trust-anchor/56cb8347-8cc7-45dd-9f2d-a3902aae9462"
export IAM_ROLE_NAME="IAMRolesAnywhere-Role"
export AWS_ACCOUNT_ID="AWSアカウントID"
# 例: export AWS_ACCOUNT_ID="012345678910"
export IAM_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${IAM_ROLE_NAME}"
export PROFILE_NAME="IAMRolesAnywhere-Profile"
export PROFILE_ARN="作成後に記載"
# 例: export PROFILE_ARN="arn:aws:rolesanywhere:ap-northeast-1:012345678910:profile/194070ad-f9cb-471a-a732-923393c256e5"
export CONTROL_PLANE_NODE_NAME="kubectl get nodesで表示されるcontrol-plane NAME"
export CONTROL_PLANE_NODE_NAME="${CA_HOST}"
export CONTROL_PLANE_KEY="${CONTROL_PLANE_NODE_NAME}.key"
export CONTROL_PLANE_PEM="${CONTROL_PLANE_NODE_NAME}.pem"
export CONTROL_PLANE_CSR="${CONTROL_PLANE_NODE_NAME}.csr"
export WORKER_NODE_NAME="kubectl get nodesで表示されるworker NAME"
# 例: export WORKER_NODE_NAME="raspberry"
export WORKER_NODE_KEY="${WORKER_NODE_NAME}.key"
export WORKER_NODE_PEM="${WORKER_NODE_NAME}.pem"
export WORKER_NODE_CSR="${WORKER_NODE_NAME}.csr"
export WORKER_WORKDIR="/root/${WORKER_NODE_NAME}"
export AWS_SIGNING_HELPER_VERSION="1.8.4"
export ECR_CREDENTIAL_PROVIDER_VERSION="v1.37.0"
ARCH=$(uname -m)
if [ "${ARCH}" = "x86_64" ]; then
  GOARCH="amd64"
  HELPER_ARCH_URL="X86_64"
elif [ "${ARCH}" = "aarch64" ]; then
  GOARCH="arm64"
  HELPER_ARCH_URL="Aarch64"
fi
export GOARCH HELPER_ARCH_URL
export AWS_REGION="ap-northeast-1"
export PROVIDER_DIR="/etc/kubernetes/credential-providers"
export PROVIDER_BIN_DIR="/etc/kubernetes/credential-providers/bin"
export PROVIDER_YAML="${PROVIDER_DIR}/ecr-credential-provider.yaml"
export PROVIDER_BIN="${PROVIDER_BIN_DIR}/ecr-credential-provider"
export ECR_URI="ECRのImage URI"
# 例: export ECR_URI="012345678910.dkr.ecr.ap-northeast-1.amazonaws.com/aws-cli-image:latest"
```

## 2. Root CA を作成

:::message
本手順ではcontrol-plane上でRoot CAを作成しています
:::

```bash
# 作業ディレクトリに移動
mkdir -p "${CA_WORKDIR}"
cd "${CA_WORKDIR}"

# Root CA用の秘密鍵を作成
openssl genrsa -out "${CA_KEY}" 4096
chmod 600 "${CA_KEY}"

# Root CA (証明書) を作成
openssl req -x509 -new -nodes -sha256 \
  -key "${CA_KEY}" \
  -out "${CA_PEM}" \
  -days "${CA_DAYS}" \
  -subj "/C=${CA_C}/O=${CA_O}/CN=${CA_CN}" \
  -addext "basicConstraints=critical,CA:TRUE" \
  -addext "keyUsage=critical,keyCertSign,cRLSign"

# 後で発行する証明書の設定
## CA:FALSE → 他の証明書に署名不可
## digitalSignature → 署名検証(認証)専用
cat > "${CA_LEAF_CONF_FILE_NAME}" << 'EOF'
basicConstraints=critical,CA:FALSE
keyUsage=critical,digitalSignature
EOF
```

:::details 実行結果
```bash
root nipogi default ~/iam-roles-anywhere-ca ❯ ls -l
total 12
-rw------- 1 root root 3272 Sep 17 03:35 ca.key
-rw-r--r-- 1 root root 1960 Sep 17 03:35 ca.pem
-rw-r--r-- 1 root root   70 Sep 17 03:35 leaf_ext.cnf
root nipogi default ~/iam-roles-anywhere-ca ❯ 
```
:::

## 3. IAM Roles Anywhere を設定

### 1. 信頼アンカー (Trust anchor) を作成

1. IAMロール画面を開く
2. 画面下のRoles Anywhereから `管理` をクリック

![1.png](/images/20260917_kubernetes-ecr-auth-iam-roles-anywhere/1.png)

`信頼アンカーを作成する` をクリック

![2.png](/images/20260917_kubernetes-ecr-auth-iam-roles-anywhere/2.png)


1. `信頼アンカー名`: 以下で表示された内容をコピペ
  ```bash
  echo "${TRUST_ANCHOR_NAME}"
  ```
2. `認証機関 (CA) ソース`: `外部証明書バンドル` にチェック
3. `外部証明書バンドル`: 以下で表示された内容をコピペ
  ```bash
  cat "${CA_PEM}"
  ```
4. `信頼アンカーを作成する` をクリック

![3.png](/images/20260917_kubernetes-ecr-auth-iam-roles-anywhere/3.png)

作成した信頼アンカーARNを変数定義しておく

```bash
export TRUST_ANCHOR_ARN="arn:aws:rolesanywhere:..."
```

![4.png](/images/20260917_kubernetes-ecr-auth-iam-roles-anywhere/4.png)

### 2. 信頼アンカー用のIAMロールを作成

IAMロール画面で `ロールを作成` をクリック

![5.png](/images/20260917_kubernetes-ecr-auth-iam-roles-anywhere/5.png)

1. `信頼されたエンティティタイプ`: `カスタム信頼ポリシー` を選択

以下2つを置き換えてコピペ

2. `TRUST_ANCHOR_ARN`: `echo "${TRUST_ANCHOR_ARN}"` で表示された内容
3. `CA_CN`: `echo "${CA_CN}"` で表示された内容

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
                    "aws:PrincipalTag/x509Issuer/CN": "CA_CN"
                }
            }
        }
    ]
}
```

4. `次へ` を選択

![6.png](/images/20260917_kubernetes-ecr-auth-iam-roles-anywhere/6.png)

`許可ポリシー`: オンプレミス環境で利用したい権限を付与 (以下例)

- `AmazonEC2ContainerRegistryReadOnly`: ECRからPullするため必須
- `AmazonS3FullAccess`: PodからS3にアクセスする場合 (本手順ではECRへの認証のみのため省略しても問題ありません)

`次へ` を選択

![7.png](/images/20260917_kubernetes-ecr-auth-iam-roles-anywhere/7.png)

`ロール名`: 以下を入力し、`ロールを作成` をクリック

```bash
echo ${IAM_ROLE_NAME}
```

![8.png](/images/20260917_kubernetes-ecr-auth-iam-roles-anywhere/8.png)


### 3. プロファイルを作成

`プロファイルを作成` をクリック

![9.png](/images/20260917_kubernetes-ecr-auth-iam-roles-anywhere/9.png)

1. `プロファイル名`: 以下で表示された内容をコピペ

```bash
echo ${PROFILE_NAME}
```

2. `ロール`: 作成した信頼アンカー用のIAMロールを指定
3. `セッションポリシー`: 特定のPodに付与したい権限に絞る ※なにも指定しなくても良い
   - `AmazonEC2ContainerRegistryReadOnly`
   - `AmazonS3FullAccess`
4. `プロファイルを作成` をクリック

![10.png](/images/20260917_kubernetes-ecr-auth-iam-roles-anywhere/10.png)


プロファイルARNを環境変数に定義

```bash
export PROFILE_ARN="xxx"
```

![11.png](/images/20260917_kubernetes-ecr-auth-iam-roles-anywhere/11.png)

## 4. 証明書を発行 (各ノード)

### 1. control-plane の場合

```bash
cd "${CA_WORKDIR}"
# PrivateKeyを作成
openssl genrsa -out "${CONTROL_PLANE_KEY}" 4096
chmod 600 "${CONTROL_PLANE_KEY}"
# CSRを作成
openssl req -new -key \
  "${CONTROL_PLANE_KEY}" \
  -out "${CONTROL_PLANE_CSR}" \
  -subj "/C=${CA_C}/O=${CA_O}/CN=${CONTROL_PLANE_NODE_NAME}"
# PEMを作成
openssl x509 -req -in \
  "${CONTROL_PLANE_CSR}" \
  -CA "${CA_PEM}" \
  -CAkey "${CA_KEY}" \
  -out "${CONTROL_PLANE_PEM}" \
  -days "${CA_DAYS}" \
  -sha256 \
  -CAcreateserial \
  -extfile "${CA_LEAF_CONF_FILE_NAME}"
```

:::details 実行結果
```bash
root nipogi default ~/iam-roles-anywhere-ca ❯ ls -l
total 28
-rw------- 1 root root 3272 Sep 17 03:35 ca.key
-rw-r--r-- 1 root root 1960 Sep 17 03:35 ca.pem
-rw-r--r-- 1 root root   41 Sep 17 12:10 ca.srl
-rw-r--r-- 1 root root   70 Sep 17 03:35 leaf_ext.cnf
-rw-r--r-- 1 root root 1626 Sep 17 12:10 nipogi.csr
-rw------- 1 root root 3272 Sep 17 12:10 nipogi.key
-rw-r--r-- 1 root root 1935 Sep 17 12:10 nipogi.pem
root nipogi default ~/iam-roles-anywhere-ca ❯
```
:::

### 2. worker の場合

#### workerのShell (CSR作成)

```bash
mkdir -p "${WORKER_WORKDIR}"
cd "${WORKER_WORKDIR}"
# PrivateKeyを作成
openssl genrsa -out "${WORKER_NODE_KEY}" 4096
chmod 600 "${WORKER_NODE_KEY}"
# CSRを作成
openssl req -new \
  -key "${WORKER_NODE_KEY}" \
  -out "${WORKER_NODE_CSR}" \
  -subj "/C=${CA_C}/O=${CA_O}/CN=${WORKER_NODE_NAME}"
chown "$(whoami)" "${WORKER_NODE_CSR}"
# CAのあるホストへCSRを転送
scp "${WORKER_NODE_CSR}" "${CA_HOST}:/tmp/"
```

:::details 実行結果
```bash
# worker側でcsrを作成
root@raspberry:~/raspberry# ls -l
total 8
-rw-r--r-- 1 root root 1630 Sep 17 12:21 raspberry.csr
-rw------- 1 root root 3272 Sep 17 12:21 raspberry.key
root@raspberry:~/raspberry#

# CA側で転送されてきたcsrを確認
root nipogi default ~/iam-roles-anywhere-ca ❯ ls -l /tmp | grep .csr
-rw-r--r-- 1 root root 1630 Sep 17 12:22 raspberry.csr
root nipogi default ~/iam-roles-anywhere-ca ❯ 
```
:::

#### control-planeのShell

```bash
cd "${CA_WORKDIR}"
# 転送されてきたCSRに署名しPEMを作成
openssl x509 -req \
  -in "/tmp/${WORKER_NODE_CSR}" \
  -CA ${CA_PEM} \
  -CAkey ${CA_KEY} \
  -out "/tmp/${WORKER_NODE_PEM}" \
  -days "${CA_DAYS}" \
  -CAcreateserial \
  -sha256 \
  -extfile "${CA_LEAF_CONF_FILE_NAME}"
# 作成したPEMをworkerに転送
scp "/tmp/${WORKER_NODE_PEM}" "${WORKER_NODE_NAME}:/tmp/"
```

:::details 実行結果
```bash
# CA側でpemを作成
root nipogi default ~/iam-roles-anywhere-ca ❯ ls -l /tmp | grep .pem
-rw-r--r-- 1 root root 1939 Sep 17 12:25 raspberry.pem
root nipogi default ~/iam-roles-anywhere-ca ❯ 

# worker側で転送されてきたpemを確認
root@raspberry:~/raspberry# ls -l /tmp | grep .pem
-rw-r--r-- 1 root root 1939 Sep 17 12:25 raspberry.pem
root@raspberry:~/raspberry#
```
:::

#### workerのShell (PEM配置)

```bash
mv "/tmp/${WORKER_NODE_PEM}" "${WORKER_WORKDIR}/"
```

:::details 実行結果
```bash
# pemが適切な場所に格納されたことを確認
root@raspberry:~/raspberry# ls -l "${WORKER_WORKDIR}"
total 12
-rw-r--r-- 1 root root 1630 Sep 17 12:21 raspberry.csr
-rw------- 1 root root 3272 Sep 17 12:21 raspberry.key
-rw-r--r-- 1 root root 1939 Sep 17 12:25 raspberry.pem
root@raspberry:~/raspberry#
```
:::

## 5. aws_signing_helper を設定 (各ノード)

```bash
# aws cliをインストール
apt install -y unzip
if [ "${GOARCH}" = "amd64" ]; then
  AWSCLI_URL="https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip"
else
  AWSCLI_URL="https://awscli.amazonaws.com/awscli-exe-linux-aarch64.zip"
fi
curl "${AWSCLI_URL}" -o "awscliv2.zip"
unzip awscliv2.zip
./aws/install
aws --version
```

```bash
# aws_signing_helperをダウンロード
export AWS_SIGNING_HELPER_PATH="/usr/local/bin/aws_signing_helper"
curl -o "${AWS_SIGNING_HELPER_PATH}" "https://rolesanywhere.amazonaws.com/releases/${AWS_SIGNING_HELPER_VERSION}/${HELPER_ARCH_URL}/Linux/Amzn2023/aws_signing_helper"
chmod +x "${AWS_SIGNING_HELPER_PATH}"

## control-planeの場合
export WORKDIR="${CA_WORKDIR}"
export NODE_PEM="${CONTROL_PLANE_PEM}"
export NODE_KEY="${CONTROL_PLANE_KEY}"

## workerの場合
export WORKDIR="${WORKER_WORKDIR}"
export NODE_PEM="${WORKER_NODE_PEM}"
export NODE_KEY="${WORKER_NODE_KEY}"

# ~/.aws/configを作成 (すでにある場合は追記するようにしてください)
mkdir -p ~/.aws
cat > ~/.aws/config << EOF
[profile ecr]
credential_process = ${AWS_SIGNING_HELPER_PATH} credential-process --certificate ${WORKDIR}/${NODE_PEM} --private-key ${WORKDIR}/${NODE_KEY} --trust-anchor-arn ${TRUST_ANCHOR_ARN} --profile-arn ${PROFILE_ARN} --role-arn ${IAM_ROLE_ARN} --region ${AWS_REGION}
EOF
```

:::details 実行結果
```bash
# aws cliを確認
root nipogi default ~/iam-roles-anywhere-ca ❯ aws --version
aws-cli/2.36.44 Python/3.14.6 Linux/7.0.0-31-generic exe/x86_64.ubuntu.26
root nipogi default ~/iam-roles-anywhere-ca ❯ 
root nipogi default ~/iam-roles-anywhere-ca ❯ 
# aws_signing_helperを確認
root nipogi default ~/iam-roles-anywhere-ca ❯ ls -l "${AWS_SIGNING_HELPER_PATH}"
-rwxr-xr-x 1 root root 12094568 Sep 17 12:38 /usr/local/bin/aws_signing_helper
root nipogi default ~/iam-roles-anywhere-ca ❯ 
root nipogi default ~/iam-roles-anywhere-ca ❯ 
# aws configを確認
root nipogi default ~/iam-roles-anywhere-ca ❯ cat ~/.aws/config
[profile ecr]
credential_process = /usr/local/bin/aws_signing_helper credential-process --certificate /root/iam-roles-anywhere-ca/nipogi.pem --private-key /root/iam-roles-anywhere-ca/nipogi.key --trust-anchor-arn arn:aws:rolesanywhere:ap-northeast-1:012345678910:trust-anchor/56cb8347-8cc7-45dd-9f2d-a3902aae9462 --profile-arn arn:aws:rolesanywhere:ap-northeast-1:012345678910:profile/194070ad-f9cb-471a-a732-923393c256e5 --role-arn arn:aws:iam::012345678910:role/IAMRolesAnywhere-Role --region ap-northeast-1
root nipogi default ~/iam-roles-anywhere-ca ❯
```
:::

## 6. ecr-credential-provider を設定 (各ノード)

```bash
# ecr-credential-providerをダウンロード
mkdir -p "${PROVIDER_BIN_DIR}"
curl -L -o "${PROVIDER_BIN}" "https://storage.googleapis.com/k8s-staging-provider-aws/releases/${ECR_CREDENTIAL_PROVIDER_VERSION}/linux/${GOARCH}/ecr-credential-provider-linux-${GOARCH}"
chmod 755 "${PROVIDER_BIN}"

# CredentialProviderConfigを作成
cat > "${PROVIDER_YAML}" << EOF
apiVersion: kubelet.config.k8s.io/v1
kind: CredentialProviderConfig
providers:
  - name: ecr-credential-provider
    apiVersion: credentialprovider.kubelet.k8s.io/v1
    matchImages:
      - "*.dkr.ecr.*.amazonaws.com"
    defaultCacheDuration: "12h"
    env:
      - name: AWS_PROFILE
        value: ecr
      - name: AWS_CONFIG_FILE
        value: /root/.aws/config
      - name: HOME
        value: /root
EOF
echo "KUBELET_EXTRA_ARGS=\"--image-credential-provider-config=${PROVIDER_YAML} --image-credential-provider-bin-dir=${PROVIDER_BIN_DIR}\"" >> /etc/default/kubelet

# kubeletを再起動
systemctl daemon-reload
systemctl restart kubelet
systemctl status kubelet
```

:::details 実行結果
```bash
root nipogi default ~/iam-roles-anywhere-ca ❯ ls -l ${PROVIDER_DIR}
total 8
drwxr-xr-x 2 root root 4096 Sep 16 23:06 bin
-rw-r--r-- 1 root root  408 Sep 17 12:57 ecr-credential-provider.yaml
root nipogi default ~/iam-roles-anywhere-ca ❯ 
root nipogi default ~/iam-roles-anywhere-ca ❯ 
root nipogi default ~/iam-roles-anywhere-ca ❯ ls -l ${PROVIDER_BIN_DIR}
total 14056
-rwxr-xr-x 1 root root 14385314 Sep 17 12:56 ecr-credential-provider
-rw-r--r-- 1 root root      408 Sep 16 23:06 ecr-credential-provider.yaml
root nipogi default ~/iam-roles-anywhere-ca ❯ 
root nipogi default ~/iam-roles-anywhere-ca ❯ 
root nipogi default ~/iam-roles-anywhere-ca ❯ cat /etc/default/kubelet 
KUBELET_EXTRA_ARGS="--image-credential-provider-config=/etc/kubernetes/credential-providers/ecr-credential-provider.yaml --image-credential-provider-bin-dir=/etc/kubernetes/credential-providers/bin"
root nipogi default ~/iam-roles-anywhere-ca ❯ 
root nipogi default ~/iam-roles-anywhere-ca ❯ 
root nipogi default ~/iam-roles-anywhere-ca ❯ systemctl status kubelet
● kubelet.service - kubelet: The Kubernetes Node Agent
     Loaded: loaded (/usr/lib/systemd/system/kubelet.service; enabled; preset: enabled)
    Drop-In: /usr/lib/systemd/system/kubelet.service.d
             └─10-kubeadm.conf
     Active: active (running) since Thu 2026-09-17 12:58:12 UTC; 32s ago
 Invocation: 1230c7c82b6746f3829c76507124ab4a
       Docs: https://kubernetes.io/docs/
   Main PID: 2252801 (kubelet)
      Tasks: 13 (limit: 13481)
     Memory: 50.4M (peak: 50.7M)
        CPU: 3.270s
     CGroup: /system.slice/kubelet.service
             └─2252801 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf --config=/var/lib/kubelet/confi>

Sep 17 12:58:13 nipogi kubelet[2252801]: I0917 12:58:13.555263 2252801 reconciler_common.go:251] "operationExecutor.VerifyControllerAttachedVolume started for volume \"kubelet>
Sep 17 12:58:13 nipogi kubelet[2252801]: I0917 12:58:13.555343 2252801 reconciler_common.go:251] "operationExecutor.VerifyControllerAttachedVolume started for volume \"policys>
Sep 17 12:58:13 nipogi kubelet[2252801]: I0917 12:58:13.555434 2252801 reconciler_common.go:251] "operationExecutor.VerifyControllerAttachedVolume started for volume \"xtables>
Sep 17 12:58:13 nipogi kubelet[2252801]: I0917 12:58:13.555550 2252801 reconciler_common.go:251] "operationExecutor.VerifyControllerAttachedVolume started for volume \"var-lib>
Sep 17 12:58:13 nipogi kubelet[2252801]: I0917 12:58:13.555610 2252801 reconciler_common.go:251] "operationExecutor.VerifyControllerAttachedVolume started for volume \"sys\" (>
Sep 17 12:58:13 nipogi kubelet[2252801]: I0917 12:58:13.555650 2252801 reconciler_common.go:251] "operationExecutor.VerifyControllerAttachedVolume started for volume \"root\" >
Sep 17 12:58:13 nipogi kubelet[2252801]: I0917 12:58:13.555698 2252801 reconciler_common.go:251] "operationExecutor.VerifyControllerAttachedVolume started for volume \"xtables>
Sep 17 12:58:13 nipogi kubelet[2252801]: I0917 12:58:13.570145 2252801 server.go:177] "Pod update broadcasted" podUID="167882adb6ac5f1da095629cd19ee677" type="MODIFIED"
Sep 17 12:58:14 nipogi kubelet[2252801]: I0917 12:58:14.045356 2252801 server.go:177] "Pod update broadcasted" podUID="40f0378a902e204c2362282e573f0c39" type="MODIFIED"
Sep 17 12:58:14 nipogi kubelet[2252801]: I0917 12:58:14.062986 2252801 server.go:177] "Pod update broadcasted" podUID="e688cf553d3d13c37d4231aca82288a7" type="MODIFIED"
root nipogi default ~/iam-roles-anywhere-ca ❯
```
:::

## 動作確認

imageにECRのURIを指定してPodをapply

```bash
cat > /tmp/ecr-test.yaml << EOF
apiVersion: v1
kind: Pod
metadata:
  name: ecr-test
spec:
  containers:
    - name: test
      image: ${ECR_URI}
      command: ["sleep", "3600"]
EOF

kubectl apply -f /tmp/ecr-test.yaml
kubectl describe pod ecr-test
kubectl logs ecr-test
```

:::details 実行結果
Dockerfileを作成
```dockerfile
FROM public.ecr.aws/aws-cli/aws-cli:latest
ENTRYPOINT ["aws"]
```

ECRにImageを配置
```bash
# ECRにログイン
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin 012345678910.dkr.ecr.ap-northeast-1.amazonaws.com
# buildしてECRにpush (クロスプラットフォーム対応の例)
docker buildx build --platform linux/amd64,linux/arm64 -t 012345678910.dkr.ecr.ap-northeast-1.amazonaws.com/aws-cli-image:latest --push .
```

ImageのPullが成功し、Podが正常に起動することを確認
```bash
# Podをapplyし、Runningであることを確認
root nipogi default ~/iam-roles-anywhere-ca ❯ kubectl apply -f /tmp/ecr-test.yaml
pod/ecr-test created
root nipogi default ~/iam-roles-anywhere-ca ❯ 
# EventsでSuccessfully pulled imageを確認
root nipogi default ~/iam-roles-anywhere-ca ❯ kubectl describe pod ecr-test 
Name:             ecr-test
# ~~~ 省略 ~~~
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  97s   default-scheduler  Successfully assigned default/ecr-test to raspberry
  Normal  Pulling    96s   kubelet            spec.containers{test}: Pulling image "012345678910.dkr.ecr.ap-northeast-1.amazonaws.com/aws-cli-image:latest"
  Normal  Pulled     96s   kubelet            spec.containers{test}: Successfully pulled image "012345678910.dkr.ecr.ap-northeast-1.amazonaws.com/aws-cli-image:latest" in 442ms (442ms including waiting). Image size: 136402954 bytes.

root nipogi default ~/iam-roles-anywhere-ca ❯ 
# STATUSでRunningを確認
root nipogi default ~/iam-roles-anywhere-ca ❯ kubectl get pods -o wide -w
NAME       READY   STATUS    RESTARTS   AGE   IP               NODE        NOMINATED NODE   READINESS GATES
ecr-test   1/1     Running   0          10s   10.244.151.191   raspberry   <none>           <none>
root nipogi default ~/iam-roles-anywhere-ca took 3s ❯
```
:::

以上で終わりです

ただし、PodからAWSのリソースにアクセスすることはまだできない状態です

そのため、次回はPodでAWSのリソースにアクセスする例を投稿します!

## トラブルシューティング

| 症状                                                                     | 原因                                       | 対処                                 |
| ------------------------------------------------------------------------ | ------------------------------------------ | ------------------------------------ |
| `AccessDeniedException: Untrusted certificate. Insufficient certificate` | 証明書のKey UsageにDigital Signatureが無い | `leaf_ext.cnf`で再署名               |
| `no basic auth credentials`                                              | kubeletがcredential providerを呼べていない | `ps aux \| grep kubelet`でフラグ確認 |

## おわりに

最近は自宅サーバにはまっています

まさか、オンプレ回帰するとは思っていませんでした...w

そもそもなんで今さら自宅サーバを構築しているのかというと、Tailscale (Funnel) の存在を知り、固定されていない動的IPの自宅サーバでもインターネットに公開できることに気づいたからです

あと、AWS ECS Fargateが便利でずっと使っていたのですが、最近は、マイクロサービスとかgRPCに興味を持ち、Kubernetesが自宅サーバにあったほうがコストを気にせず気軽に試せるから、という理由もあります

また、先日CKAを取得できました!

試験よりもその前の試験環境チェック (自宅の部屋に変なものが置いていないか) の方が大変で試験開始まで1時間半くらいかかりました

マイクが必要なのにヘッドホンが許されないとか意味が分からなかった...

どうすればよいのかと聞いてもアドバイスは何ももらえず、試行錯誤した結果、カメラにマイクが付属していることに気づき、それを利用したら解決しました

CKAを受けようと思っている方はマイク付きのカメラを用意するよう気を付けてください

CKAの試験費は高額なので受験できなかったらどうしようかとあせりました...

もう二度と受けません...w

## 参考URL
- [Getting started with IAM Roles Anywhere](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/getting-started.html)
- [The IAM Roles Anywhere trust model](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/trust-model.html)
- [Get temporary security credentials from IAM Roles Anywhere](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/credential-helper.html)
- [Configure a kubelet image credential provider](https://kubernetes.io/docs/tasks/administer-cluster/kubelet-credential-provider/)
- [cloud-provider-aws](https://github.com/kubernetes/cloud-provider-aws)
