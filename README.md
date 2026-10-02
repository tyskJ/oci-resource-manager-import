![](./doc/001samune.png)

# OCI Resource Managerで既存リソースをimportしたい 〜terraform importとの違いと実践手順〜

- [詳細](https://qiita.com/tyskJ/items/d961731ee474eca14e18)

## 構成図

![](./doc/drawio/architecture.drawio.png)

## デプロイ - Terraform -

### 作業環境 - ローカル -

- macOS Tahoe ( v26.6.2 )
- Visual Studio Code 1.137.0
- oci cli 3.71.0
- Python 3.14.2
- Terraform v1.15.7 on darwin_arm64

---

### フォルダ構成

- [こちら](./folder.md) を参照

---

### 前提条件

- `manage all-resources IN TENANCY` を付与した IAM グループに所属する IAM ユーザーが作成されていること
- 以下コマンドを実行し、_ADMIN_ プロファイルを作成していること (デフォルトリージョンは _ap-tokyo-1_ )

```bash
oci session authenticate
```

---

### 事前作業(1)

#### 1. 各種モジュールインストール

- [GitHub](https://github.com/tyskJ/common-environment-setup) を参照

---

### 事前作業 - ローカル -

#### 1. 定義ファイルの圧縮

OCI Resource Manager に Terraform Configuration をアップロードするため、`envs` ディレクトリを ZIP ファイルに圧縮します。

まず、作成する ZIP ファイル名を定義します。

```bash
ZIP_FILE="stack.zip"
```

Terraform Configuration を圧縮します。

```bash
zip -r "${ZIP_FILE}" envs
```

#### 2. 関数定義

Resource Manager の Job は非同期で実行されるため、Job の完了を待機する `wait_job` 関数を定義します。

Job の状態を10秒間隔で確認し、`SUCCEEDED` になるまで待機します。  
`FAILED` または `CANCELED` となった場合は、エラー内容を表示して処理を終了します。

```bash
wait_job() {
  local JOB_ID=$1
  local STATE
  local PREV_STATE=""

  while true; do
    if ! STATE=$(oci resource-manager job get \
      --job-id "${JOB_ID}" \
      --profile ADMIN \
      --auth security_token \
      --query 'data."lifecycle-state"' \
      --raw-output); then
      echo "ERROR: Failed to get job state."
      return 1
    fi

    if [[ "${STATE}" != "${PREV_STATE}" ]]; then
      echo "State: ${STATE}"
      PREV_STATE="${STATE}"
    fi

    case "${STATE}" in
      SUCCEEDED)
        return 0
        ;;

      FAILED|CANCELED)
        echo
        echo "ERROR: Job ${STATE}"

        oci resource-manager job get \
          --job-id "${JOB_ID}" \
          --profile ADMIN \
          --auth security_token \
          --query 'data.{
            ErrorCode: "failure-details".code,
            ErrorMessage: "failure-details".message
          }'

        return 1
        ;;

      ACCEPTED|IN_PROGRESS|CANCELING)
        echo "Waiting 10 seconds..."
        sleep 10
        ;;

      *)
        echo "ERROR: Unexpected job state: ${STATE}"
        return 1
        ;;
    esac
  done
}
```

---

### 実作業 - OCI Resource Manager -

#### 1. スタック作成

Terraform Configuration を管理する Resource Manager Stack を作成します。

まず、Stack を作成するルート・コンパートメント（Tenancy）の OCID を取得します。

```bash
TENANCY_ID=$(oci iam compartment list \
  --lifecycle-state ACTIVE \
  --include-root \
  --profile ADMIN \
  --auth security_token \
  --query "data[?\"compartment-id\"==null].id | [0]" \
  --raw-output)
```

続いて、Resource Manager Stack の作成に使用する各種パラメータを定義します。

リージョンは、OCI CLI の `ADMIN` プロファイルから取得します。

```bash
REGION=$(awk -F= '/\[ADMIN\]/{f=1} f && /^region=/{print $2; exit}' ~/.oci/config)
SYSTEM_NAME="oci-resource-manager-import"
TF_VER="1.5.x"
STACK_NAME="${SYSTEM_NAME}-stack"
```

Terraform へ渡す Variable を `terraform.tfvars.json` として作成します。

```bash
cat <<EOF > terraform.tfvars.json
{
  "tenancy_ocid": "${TENANCY_ID}",
  "region": "${REGION}",
  "system_name": "${SYSTEM_NAME}",
  "vcn_cidr": "10.0.0.0/16"
}
EOF
```

準備した Terraform Configuration と Variable を使用して Resource Manager Stack を作成します。

```bash
oci resource-manager stack create \
  --compartment-id "${TENANCY_ID}" \
  --display-name "${STACK_NAME}" \
  --description "Dev Stack" \
  --config-source "${ZIP_FILE}" \
  --working-directory "envs" \
  --terraform-version "${TF_VER}" \
  --variables file://terraform.tfvars.json \
  --wait-for-state "ACTIVE" \
  --profile ADMIN \
  --auth security_token
```

#### 2. Plan Job 作成

作成した Stack に対して Plan Job を実行し、Terraform によってどのような変更が行われるか確認します。

まず、作成した Stack の OCID を取得します。

```bash
STACK_ID=$(oci resource-manager stack list \
  --all \
  --compartment-id "${TENANCY_ID}" \
  --display-name "${STACK_NAME}" \
  --profile ADMIN \
  --auth security_token \
  --query 'data[0].id' \
  --raw-output)
```

Plan Job を作成し、後続処理で利用する Job OCID を取得します。

```bash
PLAN_JOB_ID=$(oci resource-manager job create-plan-job \
  --stack-id "${STACK_ID}" \
  --display-name "${STACK_NAME}-plan" \
  --profile ADMIN \
  --auth security_token \
  --query 'data.id' \
  --raw-output)

echo "PLAN_JOB_ID=${PLAN_JOB_ID}"

wait_job "${PLAN_JOB_ID}"
```

Plan Job が完了したら、Terraform の実行ログを確認します。

`jq` でログ本文のみを取得し、`sed` で Resource Manager が付与する日時やログレベルを除外して、Terraform の出力を見やすくしています。

```bash
oci resource-manager job get-job-logs-content \
  --job-id "${PLAN_JOB_ID}" \
  --profile ADMIN \
  --auth security_token \
  | jq -r '.data' \
  | sed -E 's/^[0-9]{4}\/[0-9]{2}\/[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}\[TERRAFORM_CONSOLE\] \[INFO\] ?//'
```

#### 3. Deploy

Plan の内容に問題がなければ、作成した Plan Job を指定して Apply Job を実行します。

`FROM_PLAN_JOB_ID` を指定することで、確認済みの Plan と同じ内容を Apply します。

```bash
APPLY_JOB_ID=$(
  oci resource-manager job create-apply-job \
    --stack-id "${STACK_ID}" \
    --execution-plan-strategy FROM_PLAN_JOB_ID \
    --execution-plan-job-id "${PLAN_JOB_ID}" \
    --display-name "${STACK_NAME}-apply" \
    --profile ADMIN \
    --auth security_token \
    --query 'data.id' \
    --raw-output
)

echo "APPLY_JOB_ID=${APPLY_JOB_ID}"

wait_job "${APPLY_JOB_ID}"
```

Apply Job が `SUCCEEDED` となれば、初期構成のデプロイは完了です。

#### 4. Import用リソース作成

Import の動作を確認するため、Terraform 管理外の Subnet を OCI CLI から作成します。

この Subnet は既存 Resource Manager Stack の Terraform Configuration には含まれていないため、この時点では Terraform 管理外のリソースとなります。

> [!NOTE]
>
> `COMPARTMENT_OCID` は、作成する Subnet のコンパートメント OCID に変更してください。  
> `VCN_OCID` は、作成する Subnet を配置する VCN の OCID に変更してください。

```bash
COMPARTMENT_OCID="<compartment-ocid>"
VCN_OCID="<vcn-ocid>"
SUBNET_NAME="private-subnet"
SUBNET_CIDR="10.0.1.0/24"

oci network subnet create \
  --compartment-id "${COMPARTMENT_OCID}" \
  --vcn-id "${VCN_OCID}" \
  --display-name "${SUBNET_NAME}" \
  --cidr-block "${SUBNET_CIDR}" \
  --prohibit-public-ip-on-vnic true \
  --profile ADMIN \
  --auth security_token \
  --wait-for-state AVAILABLE
```

#### 5. Resource Discovery 設定

Resource Discovery を実行するため、OCI Terraform Provider をローカル環境へ配置します。

まず、利用する OCI Terraform Provider のバージョンとアーキテクチャを定義します。

```bash
OCI_PROVIDER_VERSION="9.3.0"
OCI_PROVIDER_ARCH="arm64"
```

OCI Terraform Provider を一時的に展開するためのディレクトリを作成します。

```bash
OCI_PROVIDER_TMP_DIR="/tmp/oci-provider"
mkdir -p "${OCI_PROVIDER_TMP_DIR}"
```

指定したバージョン・アーキテクチャの OCI Terraform Provider をダウンロードします。

```bash
curl -L \
  -o "${OCI_PROVIDER_TMP_DIR}/terraform-provider-oci_${OCI_PROVIDER_VERSION}_darwin_${OCI_PROVIDER_ARCH}.zip" \
  "https://releases.hashicorp.com/terraform-provider-oci/${OCI_PROVIDER_VERSION}/terraform-provider-oci_${OCI_PROVIDER_VERSION}_darwin_${OCI_PROVIDER_ARCH}.zip"
```

ダウンロードした ZIP ファイルを展開します。  
展開に成功した場合は、不要になった ZIP ファイルを削除します。

```bash
OCI_PROVIDER_ZIP="${OCI_PROVIDER_TMP_DIR}/terraform-provider-oci_${OCI_PROVIDER_VERSION}_darwin_${OCI_PROVIDER_ARCH}.zip"

unzip \
  "${OCI_PROVIDER_TMP_DIR}/terraform-provider-oci_${OCI_PROVIDER_VERSION}_darwin_${OCI_PROVIDER_ARCH}.zip" \
  -d "${OCI_PROVIDER_TMP_DIR}" \
  && rm -f "${OCI_PROVIDER_ZIP}"

ls -l "${OCI_PROVIDER_TMP_DIR}"
```

展開した OCI Terraform Provider を `/usr/local/bin` 配下へ配置します。

```bash
sudo mv \
  "${OCI_PROVIDER_TMP_DIR}"/terraform-provider-oci_* \
  /usr/local/bin/
```

配置した OCI Terraform Provider のパスを取得します。

```bash
OCI_PROVIDER_BIN=$(ls /usr/local/bin/terraform-provider-oci*)
echo "${OCI_PROVIDER_BIN}"
```

Resource Discovery を簡単に実行できるよう、`tf-oci` という名前でシンボリックリンクを作成します。

```bash
sudo ln -sfn \
  "${OCI_PROVIDER_BIN}" \
  /usr/local/bin/tf-oci

ls -l /usr/local/bin/tf-oci
```

最後に、Resource Discovery が利用できることを確認します。

以下のコマンドで、Resource Discovery が対応しているサービスおよび Terraform Resource の一覧を確認できます。

```bash
tf-oci -command=list_export_services
tf-oci -command=list_export_resources
```

#### 6. Import リソースの Terraform Configuration 生成

> [!NOTE]
>
> `SUBNET_OCID` は、Import 対象の Subnet OCID に変更してください。

```bash
SUBNET_OCID="<subnet-ocid>"
```

Resource Discovery の出力先ディレクトリを定義します。

```bash
DISCOVERY_OUTPUT_DIR="$(pwd)/resource-discovery"
```

既存の出力が残っている場合に備えて、出力先ディレクトリを再作成します。

```bash
rm -rf "${DISCOVERY_OUTPUT_DIR}" \
  && mkdir -p "${DISCOVERY_OUTPUT_DIR}"
```

Resource Discovery が OCI CLI のセッション認証を利用できるよう、環境変数を設定します。

```bash
export OCI_AUTH="SecurityToken"
export OCI_CONFIG_FILE_PROFILE="ADMIN"
export TF_VAR_region="ap-tokyo-1"
```

Import 対象の Subnet を指定して Resource Discovery を実行します。

```bash
tf-oci \
  -command=export \
  -ids="oci_core_subnet:${SUBNET_OCID}" \
  -output_path="${DISCOVERY_OUTPUT_DIR}"
```

生成されたファイルを確認します。

```bash
find "${DISCOVERY_OUTPUT_DIR}" -type f
```

#### 7. Resource Block 追加

Resource Discovery によって生成された `resources.tf` から、Import 対象の Subnet に該当する Resource Block を確認します。

```bash
cat "${DISCOVERY_OUTPUT_DIR}/resources.tf"
```

生成された Resource Block を、既存 Stack の構成に合わせて `envs/main.tf` へ追加します。

例えば、以下のような Resource Block が生成された場合、

```hcl
resource "oci_core_subnet" "export_subnet" {
  cidr_block                 = "10.0.1.0/24"
  compartment_id             = "<compartment-ocid>"
  display_name               = "private-subnet"
  prohibit_public_ip_on_vnic = true
  vcn_id                     = "<vcn-ocid>"
}
```

既存 Stack の Variable や Resource Reference に合わせて修正します。

```hcl
resource "oci_core_subnet" "private" {
  cidr_block                 = "10.0.1.0/24"
  compartment_id             = var.tenancy_ocid
  display_name               = "private-subnet"
  prohibit_public_ip_on_vnic = true
  vcn_id                     = oci_core_vcn.main.id
}
```

> [!NOTE]
>
> Resource Discovery で生成された Resource Block は、そのまま利用するのではなく、既存 Stack の構成に合わせて調整します。

Resource Block の転記が完了したら、Resource Discovery の生成物を削除します。

```bash
rm -rf "${DISCOVERY_OUTPUT_DIR}"
```

#### 8. import block 追加

既存の Subnet と、追加した Terraform Resource を紐付けるため、`import` block を追加します。

```hcl
import {
  to = oci_core_subnet.private
  id = "<subnet-ocid>"
}
```

`id` に変数は利用できないため、Import 対象となる Subnet の OCID を直接指定します。

#### 9. スタック更新

Terraform Configuration を再度圧縮します。

```bash
zip -r "${ZIP_FILE}" envs
```

Resource Manager Stack を更新します。

```bash
oci resource-manager stack update \
  --stack-id "${STACK_ID}" \
  --config-source "${ZIP_FILE}" \
  --working-directory "envs" \
  --terraform-version "${TF_VER}" \
  --variables file://terraform.tfvars.json \
  --wait-for-state "ACTIVE" \
  --profile ADMIN \
  --auth security_token \
  --force
```

#### 10. Plan Job 作成

Stack の更新が完了したら、再度 Plan Job を実行します。

今回は `import` block を追加しているため、対象 Subnet が新規作成ではなく Import として認識されることを確認します。

```bash
PLAN_JOB_ID=$(oci resource-manager job create-plan-job \
  --stack-id "${STACK_ID}" \
  --display-name "${STACK_NAME}-plan" \
  --profile ADMIN \
  --auth security_token \
  --query 'data.id' \
  --raw-output)

echo "PLAN_JOB_ID=${PLAN_JOB_ID}"

wait_job "${PLAN_JOB_ID}"
```

Plan の実行ログを確認します。

```bash
oci resource-manager job get-job-logs-content \
  --job-id "${PLAN_JOB_ID}" \
  --profile ADMIN \
  --auth security_token \
  | jq -r '.data' \
  | sed -E 's/^[0-9]{4}\/[0-9]{2}\/[0-9]{2} [0-9]{2}:[0-9]{2}:[0-9]{2}\[TERRAFORM_CONSOLE\] \[INFO\] ?//'
```

以下のように、対象 Subnet が `will be imported` と表示されることを確認します。

```text
# oci_core_subnet.private will be imported
```

また、意図しない新規作成・変更・削除が発生していないことを確認します。

```text
Plan: 1 to import, 0 to add, 0 to change, 0 to destroy.
```

#### 11. Apply Job 作成

Plan の内容に問題がなければ、確認した Plan Job を指定して Apply Job を実行します。

```bash
APPLY_JOB_ID=$(oci resource-manager job create-apply-job \
  --stack-id "${STACK_ID}" \
  --execution-plan-strategy FROM_PLAN_JOB_ID \
  --execution-plan-job-id "${PLAN_JOB_ID}" \
  --display-name "${STACK_NAME}-apply" \
  --profile ADMIN \
  --auth security_token \
  --query 'data.id' \
  --raw-output)

echo "APPLY_JOB_ID=${APPLY_JOB_ID}"

wait_job "${APPLY_JOB_ID}"
```

Apply が成功すると、既存 Subnet と Terraform Resource が Resource Manager の Terraform State 上で紐付けられます。

#### 12. Import 結果確認

Import が完了したら、再度 Plan Job を実行します。

Terraform Configuration・Terraform State・実際の OCI リソースに差分がなく、以下の状態となれば Import 完了です。

```text
0 to add, 0 to change, 0 to destroy
```

---

### 後片付け - ローカル -

#### 1. 環境削除

検証で Resource Manager から作成したリソースを削除するため、Destroy Job を実行します。

```bash
DESTROY_JOB_ID=$(oci resource-manager job create-destroy-job \
  --stack-id "${STACK_ID}" \
  --execution-plan-strategy AUTO_APPROVED \
  --display-name "${STACK_NAME}-destroy" \
  --profile ADMIN \
  --auth security_token \
  --query 'data.id' \
  --raw-output)

echo "DESTROY_JOB_ID=${DESTROY_JOB_ID}"

wait_job "${DESTROY_JOB_ID}"
```

Destroy Job が完了したら、不要になった Resource Manager Stack を削除します。

```bash
oci resource-manager stack delete \
  --stack-id "${STACK_ID}" \
  --force \
  --wait-for-state DELETED \
  --profile ADMIN \
  --auth security_token
```

---

### 参考資料

#### リファレンス

- [リソース検出 - Oracle Cloud Infrastructure ドキュメント](https://docs.oracle.com/ja-jp/iaas/Content/ResourceManager/Concepts/resource-discovery.htm)
- [リソース検出の設定 - Oracle Cloud Infrastructure ドキュメント](https://docs.oracle.com/ja-jp/iaas/Content/dev/terraform/tutorials/tf-resource-discovery-setup.htm)
