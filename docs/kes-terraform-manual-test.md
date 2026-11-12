# Terraform Provider KES 手动复测文档

本文档记录 **`terraform-provider-ksyun` 中 KES / KES8 相关代码在真实 OpenAPI 上的全部实测**：调用过哪些 API、每个请求的完整参数、真实响应，以及可一键复现的探针源码与 Terraform 配置。

> **凭据约定**：文中所有 AK/SK 一律写作 `your ak` / `your sk` 占位符。真实凭据从环境变量读取，**任何情况下不得硬编码或提交到仓库**。
>
> ```bash
> export KSYUN_ACCESS_KEY='your ak'
> export KSYUN_SECRET_KEY='your sk'
> ```

---

## 0. 环境准备

### 0.1 手工调用 API（Go SDK 探针）

| 项 | 值 |
|---|---|
| Go | 1.25.5（`/d/go/go1.25.5/bin/go`） |
| SDK | `github.com/kingsoftcloud/sdk-go/v2 v2.2.32` |
| 离线构建 | `GOFLAGS=-mod=mod GOPROXY=off GOSUMDB=off` |
| 地域 | `cn-beijing-6` |
| 传输 | `HTTP`（内网域名，公网不可达） |

探针统一骨架（每个探针目录下 `go.mod` 只依赖 SDK）：

```go
cpf := profile.NewClientProfile()
cpf.HttpProfile.ReqMethod = "POST"
cpf.HttpProfile.Endpoint = "kes.inner.api.ksyun.com" // 或 kmr.inner.api.ksyun.com
cpf.HttpProfile.Scheme = "HTTP"
cred := common.NewCredential(os.Getenv("KSYUN_ACCESS_KEY"), os.Getenv("KSYUN_SECRET_KEY"))
client, err := kes.NewClient(cred, "cn-beijing-6", cpf)
```

运行方式：

```bash
cd /d/tmp/<probe>
set -a && . /d/tmp/kes_cred.env && set +a
export PATH=/d/go/go1.25.5/bin:$PATH
GOFLAGS=-mod=mod GOPROXY=off GOSUMDB=off go run .
```

### 0.2 走 Terraform 复现（dev_overrides）

```bash
# 1. 编译 provider 到 dev 覆盖目录
cd /d/workspace/terraform-provider-ksyun
GOFLAGS=-mod=mod GOPROXY=off GOSUMDB=off go build -o /d/tmp/tfprov/terraform-provider-ksyun.exe .

# 2. /d/tmp/tfkes8b/terraform.rc
# provider_installation {
#   dev_overrides { "ksyun" = "D:/tmp/tfprov" }
#   direct {}
# }

# 3. 执行
cd /d/tmp/tfkes8b
set -a && . /d/tmp/kes_cred.env && set +a
TF_CLI_CONFIG_FILE=/d/tmp/tfkes8b/terraform.rc /d/tmp/terraform.exe apply -auto-approve -no-color
```

`terraform.exe` 为 v1.9.8。**不要**执行 `terraform init`——dev_overrides 模式下 init 会因找不到 lock 而失败，直接 apply 即可。

---

## 1. 实测 API 总览

| # | API | 路径 / Version | 用途 | 探针 |
|---|---|---|---|---|
| 1 | `kes:LaunchCluster` | `kes/2020-12-15` | KES7 标准版创建 | `kes7probe` / `kes7charg` / `ptprobe` |
| 2 | `kes:ListClusters` | `kes/2020-12-15` | 集群列表（KES7/KES8 共用） | `accountprobe` |
| 3 | `kes:DescribeCluster` | `kes/2020-12-15` | 单集群详情 | `queryprobe` |
| 4 | `kes:ListInstanceGroups` | `kes/2020-12-15` | 实例组列表 | `queryprobe` |
| 5 | `kmr:LaunchCluster` | `kmr/2021-09-02` | KES8 创建（`ClusterType="KES"`） | `nohdrprobe` / `hdrprobe` / `chargprobe` |
| 6 | `kmr:ListClusterVersions` | `kmr/2021-09-02` | 版本元数据（**全部 402**） | `verprobe2` |

> **关键结论**：KES7 与 KES8 是**两条不同的创建路径**，但**读路径共用 `kes/2020-12-15`**。`ListClusters` 不区分创建路径，两者都能列出来。

---

## 2. 逐 API 复测

### 2.1 `kes:LaunchCluster`（KES7 标准版）

**Body 参数**（`kes7probe`）：

```json
{
  "ClusterName": "probe-ver-<version>",
  "MainVersion": "<version>",
  "Scenario": "General",
  "AvailabilityZone": "cn-beijing-6a",
  "ChargeType": "Monthly",
  "PurchaseTime": 1,
  "ProjectId": 0,
  "VpcDomainId": "00000000-0000-0000-0000-000000000000",
  "VpcSubnetId": "00000000-0000-0000-0000-000000000000",
  "SecurityGroupId": "00000000-0000-0000-0000-000000000000"
}
```

> VPC / 子网 / 安全组**故意填全 0 的假 UUID**：后端对版本校验在 VPC 校验之前，版本通过后才会报 `'InstanceGroups' should be list of dict, got None.`——用错误信息的变化来判定版本是否被接受，属于无副作用的探测手法。

**实测响应（12 个版本）**：

| MainVersion | 响应 |
|---|---|
| `8.19.13` / `8.19.0` / `8.17.0` / `8.15.0` / `8.11.0` / `8.9.0` / `8.5.0` / `8.0.0` | `402 'MainVersion' invalid, got <v>.` |
| `OpenSource-8.19.13` / `OpenSource-7.4.2` | `402 'ClusterName' pattern invalid, got probe-ver-OpenSource-8-19-13.`（**版本已通过**，卡在集群名不允许 `-` 连写的正则） |
| `7.4.2` / `7.10.0` | `402 'InstanceGroups' should be list of dict, got None.`（**版本已通过**） |

**结论**：KES7 原生接口**只接受 7.x**，任何 8.x 都被 `'MainVersion' invalid` 拒绝。这就是 KES8 必须改走 `kmr/2021-09-02` 的直接原因。

---

### 2.2 `kes:LaunchCluster` —— ChargeType 白名单（KES7）

**Body**：同上，`MainVersion="7.4.2"`，仅 `ChargeType` 变化（`kes7charg`）。

**实测响应**：

| ChargeType | 响应 |
|---|---|
| `Monthly` | `402 'InstanceGroups' should be list of dict, got None.`（**通过**） |
| `Daily` | `402 'InstanceGroups' should be list of dict, got None.`（**通过**） |
| `Minutely` | `402 'ChargeType' invalid, got Minutely.` |
| `Hourly` | `402 'ChargeType' invalid, got Hourly.` |
| `Yearly` | `402 'ChargeType' invalid, got Yearly.` |
| `PrePaid` | `402 'ChargeType' invalid, got PrePaid.` |
| `PostPaid` | `402 'ChargeType' invalid, got PostPaid.` |
| `monthly`（小写） | `402 'ChargeType' invalid, got monthly.` |

**结论**：白名单 = **`{Monthly, Daily}`**，**大小写敏感**。

---

### 2.3 `kes:LaunchCluster` —— PurchaseTime 语义

**Body**：同上，`MainVersion="7.4.2"`，组合 `ChargeType` × `PurchaseTime`（`ptprobe`）。

**实测响应**：

| 组合 | 响应 |
|---|---|
| `Daily` + 不传（nil） | `402 'InstanceGroups' should be list of dict, got None.`（**通过**） |
| `Daily` + `0` | `402 'PurchaseTime' should grea...`（**拒绝**） |
| `Daily` + `1` | `402 'InstanceGroups' should be list of dict, got None.`（**通过**） |
| `Monthly` + 不传（nil） | `402 'InstanceGroups' should be list of dict, got None.`（**通过**） |
| `Monthly` + `0` | `402 'PurchaseTime' should grea...`（**拒绝**） |

**结论**：后端**接受字段缺失**（`nil`），但**拒绝显式 0**（`'PurchaseTime' should between 1-36`）。这正是 provider 中 `purchase_time` 从 `Required` 改为 `Optional` 且 `v > 0` 才下发的依据：

```go
// ksyun/service_ksyun_kes.go
if v := d.Get("purchase_time").(int); v > 0 {
    kesReq.PurchaseTime = ptrInt(v)
}
```

---

### 2.4 `kes:ListClusters`（读路径，KES7/KES8 共用）

**Body**（`accountprobe`）：

```json
{ "Marker": "limit=1&offset=0" }
```

**实测响应**（节选）：

```json
{
  "Clusters": [
    {
      "ClusterId": "e759e450-d3ba-43f6-b6d2-b8dcac136a07",
      "ClusterName": "klog-test",
      "MainVersion": "OpenSource-7.4.2",
      "InstanceGroups": [
        { "Id": "98d44ccc-...", "InstanceGroupType": "DATA",        "ResourceType": "KEC", "InstanceType": "ES.SE9.4C8G" },
        { "Id": "890ec073-...", "InstanceGroupType": "COORDINATOR", "ResourceType": "KEC", "InstanceType": "ES.SE9.4C8G" },
        { "Id": "07ef69e5-...", "InstanceGroupType": "WARM",        "ResourceType": "KEC", "InstanceType": "ES.SE9.4C8G" }
      ],
      "EnableEip": true,
      "Region": "cn-beijing-6",
      "AvailabilityZone": "cn-beijing-6a",
      "VpcDomainId": "da03d553-13f5-47af-bd10-2ee916b01124",
      "VpcSubnetId": "0f0c2e65-8fd6-4a40-b749-3d946c74ec07",
      "ChargeType": "Daily",
      "ClusterStatus": "RUNNING",
      "CreateTime": "2026-08-12 09:04:39",
      "UpdateTime": "2026-11-11 22:46:53",
      "ServingMinutes": 132309
    }
  ],
  "Total": 4,
  "RequestId": "e030c0e2-e3cb-4192-a9f5-cffba367b21a",
  "StatusCode": 200
}
```

**关键点**：`MainVersion` 返回的是 **`OpenSource-7.4.2`**，带 `OpenSource-` 前缀。这是 `main_version` 永久 diff 的直接来源——用户配置写 `7.4.2`，state 里是 `OpenSource-7.4.2`，`ForceNew` 字段每次 plan 都判定为"需重建"。

---

### 2.5 `kes:DescribeCluster` + `kes:ListInstanceGroups`

**Body**（`queryprobe`）：

```json
// DescribeCluster
{ "ClusterId": "e759e450-d3ba-43f6-b6d2-b8dcac136a07" }
// ListInstanceGroups
{ "ClusterId": "e759e450-d3ba-43f6-b6d2-b8dcac136a07" }
```

**DescribeCluster 实测响应**：

```json
{
  "ClusterId": "e759e450-d3ba-43f6-b6d2-b8dcac136a07",
  "ClusterName": "klog-test",
  "ClusterType": "KES",
  "MainVersion": "OpenSource-7.4.2",
  "InstanceGroups": [
    { "Id": "98d44ccc-...", "InstanceGroupType": "DATA",        "ResourceType": "KEC", "InstanceType": "ES.SE9.4C8G", "VolumeSize": 40,  "VolumeType": "ESSD_PL0" },
    { "Id": "890ec073-...", "InstanceGroupType": "COORDINATOR", "ResourceType": "KEC", "InstanceType": "ES.SE9.4C8G", "VolumeSize": 40,  "VolumeType": "ESSD_PL0" },
    { "Id": "07ef69e5-...", "InstanceGroupType": "WARM",        "ResourceType": "KEC", "InstanceType": "ES.SE9.4C8G", "VolumeSize": 100, "VolumeType": "ESSD_PL0" }
  ],
  "EnableEip": true,
  "Region": "cn-beijing-6",
  "AvailabilityZone": "cn-beijing-6a",
  "VpcDomainId": "da03d553-13f5-47af-bd10-2ee916b01124",
  "VpcSubnetId": "0f0c2e65-8fd6-4a40-b749-3d946c74ec07",
  "ChargeType": "Daily",
  "ClusterStatus": "RUNNING",
  "CreateTime": "2026-08-12 09:04:39",
  "UpdateTime": "2026-11-11 22:46:53",
  "ServingMinutes": 132311,
  "ProxyPort": 28291,
  "Tags": [],
  "RequestId": "6fbc7fc5-67de-4873-beb2-2d075d4b837f",
  "StatusCode": 200
}
```

**ListInstanceGroups 实测响应**（节选）：

```json
{
  "InstanceGroups": [
    {
      "Id": "98d44ccc-dfe1-4ee0-9399-f3eb5c0388c8",
      "InstanceGroupType": "DATA",
      "ResourceType": "KEC",
      "InstanceTypeCode": "ES.SE9.4C8G",
      "InstanceCount": 3,
      "VolumeType": "CLOUD_ESSD0",
      "VolumeSize": 40,
      "VolumeCount": 1,
      "VpcId": "da03d553-13f5-47af-bd10-2ee916b01124",
      "VpcSubnetId": "0f0c2e65-8fd6-4a40-b749-3d946c74ec07",
      "AvalabilityZone": "cn-beijing-6a",
      "MultiInstanceCount": 1,
      "Instances": [
        {
          "Id": "cd349430-119b-424f-982d-503b4daded9e",
          "InstanceGroupId": "98d44ccc-dfe1-4ee0-9399-f3eb5c0388c8",
          "InstanceId": "e85c00e5-8443-4a86-a826-322d1fc9ef85",
          "InstanceName": "kes-e759e450-gn-73699608-data-1-1",
          "InternalIp": "172.31.30.176",
          "Volumes": ["60486a75-1872-4b05-b8de-408c5fdaebc1"]
        }
      ]
    }
  ]
}
```

> **注意拼写**：响应字段是 `AvalabilityZone`（少一个 `i`），不是 `AvailabilityZone`。这是后端既有拼写，provider 解析时需按原样映射。

---

### 2.6 `kmr:LaunchCluster`（KES8 创建路径）

**Body**（`nohdrprobe`，与 provider 实际下发的 body 完全一致）：

```json
{
  "ClusterType": "KES",
  "ClusterName": "probe-nohdr",
  "MainVersion": "8.19.13",
  "Distribution": "ElasticSearch",
  "Scenario": "General",
  "ProjectId": 0,
  "DatabaseInfo": {
    "DatabaseName": "",
    "Endpoint": "",
    "IsVerified": false,
    "Password": "",
    "RdsInstanceId": "",
    "Type": "",
    "Username": ""
  },
  "VpcDomainId": "47e19917-1232-4f11-842d-ed4efa74a703",
  "VpcSubnetId": "ae78943b-6895-4947-a485-1e433e9e0792",
  "SecurityGroupId": "36bd25c6-a3e5-4b95-9499-d4fedf722d9d",
  "ChargeType": "Monthly",
  "XpackEnabled": true,
  "XpackUsername": "elastic",
  "XpackPassword": "ProbeOnly123!",
  "Services": ["elasticsearch"],
  "InstanceGroups": [
    {
      "InstanceGroupType": "MASTER",
      "InstanceType": "ES.I3.2C4G",
      "InstanceTypeCode": "ES.I3.2C4G",
      "ResourceType": "KEC",
      "InstanceCount": 3,
      "InstanceGroupIndex": 1,
      "AvailabilityZone": "cn-beijing-6a",
      "AvailabilityZoneIndex": 1,
      "VpcId": "47e19917-1232-4f11-842d-ed4efa74a703",
      "VpcSubnetId": "ae78943b-6895-4947-a485-1e433e9e0792",
      "ChargeType": "Monthly",
      "VolumeType": "SSD3.0",
      "VolumeSize": 100,
      "VolumeCount": 1,
      "SystemDiskType": "SSD3.0",
      "SystemDiskSize": 50
    }
  ]
}
```

**A/B 实测（是否携带 `X-Ksc-Source: kmr`）**：

```
withHeader=false status=400 err=<nil>
  {"RequestId": "6d92b3a2-b3c5-4f59-9179-0d372cbed0b6", "Error": {"Code": 400, "Message": "User have no permission to buy kmr resource"}}
withHeader=true  status=400 err=<nil>
  {"RequestId": "afa0819a-2b44-4767-b645-f5332ee8d1bc", "Error": {"Code": 400, "Message": "User have no permission to buy kmr resource"}}
```

**结论**：`X-Ksc-Source: kmr` **对授权没有任何影响**，带与不带结果完全一致。`service_ksyun_kes_vector.go` 原注释声称"without this header it rejects the call"是**错误的**，该 header 已移除。

**header × host × ClusterType 矩阵**（`hdrprobe`，4 组）：

```
kes-host / src=kmr  type=KES 8.19.13   status=400  User have no permission to buy kmr resource
kmr-host / src=kmr  type=KES 8.19.13   status=400  User have no permission to buy kmr resource
kmr-host / src=kmr  type=KMR bad-ver   status=400  User have no permission to buy kmr resource
kmr-host / src=user type=KMR bad-ver   status=400  User have no permission to buy kmr resource
```

**结论**：`400 User have no permission to buy <product> resource` 出现在**所有**组合里，与 header、host、`ClusterType` 全部无关——这是**账号级购买权限**的拦截，不是请求格式问题。请求体已通过网关的格式校验（否则会先报 `'XXX' invalid`）。

---

### 2.7 `kmr:LaunchCluster` —— ChargeType 白名单（kmr 路径）

**Body**：同 2.6，`Distribution="Hadoop"`、`MainVersion="5.3.1"`，仅 `ChargeType` 变化（`chargprobe`）。

**实测响应**：

| ChargeType | status | 响应 |
|---|---|---|
| `Monthly` | 400 | `User have no permission to buy kmr resource`（**通过**） |
| `Daily` | 400 | `User have no permission to buy kmr resource`（**通过**） |
| `Minutely` | 402 | `'ChargeType' invalid, got Minutely.` |
| `Hourly` | 402 | `'ChargeType' invalid, got Hourly.` |
| `Yearly` | 402 | `'ChargeType' invalid, got Yearly.` |
| `PrePaid` | 402 | `'ChargeType' invalid, got PrePaid.` |
| `PostPaid` | 402 | `'ChargeType' invalid, got PostPaid.` |
| `monthly` | 402 | `'ChargeType' invalid, got monthly.` |
| `daily` | 402 | `'ChargeType' invalid, got daily.` |

**结论**：kmr 路径与 KES7 路径的 `ChargeType` 白名单**完全一致** = `{Monthly, Daily}`，且**大小写敏感**。这是 provider 侧 `validation.StringInSlice([]string{"Monthly", "Daily"}, false)` 的依据。

---

### 2.8 `kmr:ListClusterVersions`

**Body**：`{ "DistributionVersion": "<v>" }`（`verprobe2`，4 种取值）。

**实测响应**：

```
DistributionVersion=""              402  'DistributionVersion' should be <class 'str'>, got None.
DistributionVersion="ElasticSearch" 402  'DistributionVersion' invalid, got ElasticSearch.
DistributionVersion="Elasticsearch" 402  'DistributionVersion' invalid, got Elasticsearch.
DistributionVersion="KES"           402  'DistributionVersion' invalid, got KES.
```

**结论**：该接口不接受任何 KES 侧的发行版标识。**版本元数据不在 OpenAPI 层**，而在 Oasis 的 `cluster_version` 表中（见第 4 节）。

---

## 3. Terraform 端到端复测

### 3.1 KES7 新建（`/d/tmp/tfkes7new/main.tf`）

```hcl
terraform {
  required_providers {
    ksyun = { source = "ksyun" }
  }
}

provider "ksyun" {
  region   = "cn-beijing-6"
  endpoint = "kes.inner.api.ksyun.com"
}

resource "ksyun_kes_cluster" "probe" {
  cluster_name      = "tf-newcreate-probe"
  main_version      = "7.4.2"
  scenario          = "general"
  availability_zone = "cn-beijing-6a"
  charge_type       = "Daily"
  project_id        = 0
  vpc_domain_id     = "da03d553-13f5-47af-bd10-2ee916b01124"
  vpc_subnet_id     = "0f0c2e65-8fd6-4a40-b749-3d946c74ec07"
  security_group_id = "36bd25c6-a3e5-4b95-9499-d4fedf722d9d"

  instance_groups {
    instance_group_type = "DATA"
    instance_type       = "ES.SE9.4C8G"
    resource_type       = "KEC"
    instance_count      = 3
    volume_type         = "ESSD_PL0"
    volume_size         = 40
    volume_count        = 1
    system_disk_type    = "SSD3.0"
    system_disk_size    = 50
  }

  instance_groups {
    instance_group_type = "COORDINATOR"
    instance_type       = "ES.SE9.4C8G"
    resource_type       = "KEC"
    instance_count      = 3
    volume_type         = "ESSD_PL0"
    volume_size         = 40
    volume_count        = 1
    system_disk_type    = "SSD3.0"
    system_disk_size    = 50
  }

  instance_groups {
    instance_group_type = "WARM"
    instance_type       = "ES.SE9.4C8G"
    resource_type       = "KEC"
    instance_count      = 2
    volume_type         = "ESSD_PL0"
    volume_size         = 100
    volume_count        = 1
    system_disk_type    = "SSD3.0"
    system_disk_size    = 50
  }
}
```

> `charge_type = "Daily"` 且**不写** `purchase_time`——正是 2.3 验证过的合法组合。

**实测结果**：

```
ACTION:LaunchCluster;REQ:&{ClusterName:... MainVersion:... ChargeType:... PurchaseTime:<nil> ...}
Error: error on creating KES cluster: [KsyunSDKError] [HttpCode:400 Err:Request failed]
  {"RequestId": "cc4b4818-3596-4d68-8bf2-1ecb69e4c699", "Error": {"Code": 400, "Message": "User have no permission to buy kes resource"}}
```

`PurchaseTime:<nil>` 证明 provider 侧 `v > 0` 的判断生效，`Daily` 场景不再误传 `0`。

失败点是**账号无 KES 购买权限**，与请求格式无关（格式错误会返回 402 的 `'XXX' invalid`）。

### 3.2 KES8 新建（`/d/tmp/tfkes8b/main.tf`）

```hcl
resource "ksyun_kes_cluster_vector" "probe" {
  cluster_name      = "tf-probe-minutely"
  main_version      = "8.19.13"
  distribution      = "ElasticSearch"
  scenario          = "General"
  charge_type       = "Daily"
  project_id        = 0
  vpc_domain_id     = "47e19917-1232-4f11-842d-ed4efa74a703"
  vpc_subnet_id     = "ae78943b-6895-4947-a485-1e433e9e0792"
  security_group_id = "36bd25c6-a3e5-4b95-9499-d4fedf722d9d"
  xpack_username    = "elastic"
  xpack_password    = "ProbeOnly123!"

  instance_groups {
    instance_group_type     = "MASTER"
    instance_type           = "ES.I3.2C4G"
    instance_type_code      = "ES.I3.2C4G"
    resource_type           = "KEC"
    instance_count          = 3
    instance_group_index    = 1
    availability_zone       = "cn-beijing-6a"
    availability_zone_index = 1
    vpc_id                  = "47e19917-1232-4f11-842d-ed4efa74a703"
    vpc_subnet_id           = "ae78943b-6895-4947-a485-1e433e9e0792"
    charge_type             = "Daily"
    volume_type             = "SSD3.0"
    volume_size             = 100
    volume_count            = 1
    system_disk_type        = "SSD3.0"
    system_disk_size        = 50
  }
}
```

**实测结果**：

```
ACTION:LaunchCluster;REQ:&{ClusterType:... Distribution:... Services:[elasticsearch] ...}
  DatabaseInfo:map[DatabaseName: Endpoint: IsVerified:false Password: RdsInstanceId: Type: Username:]
Error: error on creating KES vector cluster: [HttpCode:400]
  {"RequestId": "270b920d-7f0a-4550-8cf5-0d0c74252dee", "Error": {"Code": 400, "Message": "User have no permission to buy kmr resource"}}
```

同样卡在账号权限；`DatabaseInfo` 全零占位按设计下发（kmr 网关先做类型检查再按 `ClusterType` 路由）。

### 3.3 import 场景的 `main_version` 稳定性（`/d/tmp/tfkes7imp/main.tf`）

以已存在的 `klog-test`（`MainVersion=OpenSource-7.4.2`）配置 `main_version = "7.4.2"` 做 import / 刷新。

**实测 plan**：

```
- `main_version` 不在 diff 中（StateFunc 归一化生效，`OpenSource-7.4.2` 与 `7.4.2` 收敛）
- 触发替换的是 `project_id` / `scenario` / `security_group_id` 在 API 响应中缺失，
  以及 `instance_groups` 被清空
```

**结论**：`mainVersionStateFunc` 修复的是**配置侧新值的归一化**（`terraform-plugin-sdk v1.7.0` 的 `StateFunc` 只在 `diffString` 中对 config 侧生效），配合 `ReadAndSetClusterResource` 里对响应值的 `TrimPrefix`，两侧同时归一。少了任一侧都会出现永久 diff。

---

## 4. `main_version must be in [] for ElasticSearch` 的根因

**报错来源**：Oasis Python 层，`oasis/utils/validation/completion.py`：

```python
distribution = params.get('distribution', '')
cvs_res = await get_cluster_versions(product.upper(), distribution)
cluster_versions = [cv.get('Version', '') for cv in cvs_res.get('Versions', [])]
if main_version not in cluster_versions:
    raise Exception(f"main_version must be in {cluster_versions} for {distribution}")
```

**为什么是空列表 `[]`**：`get_cluster_versions("KES", "ElasticSearch")` 查不到任何版本行。

**元数据链路**：

```
Oasis get_cluster_versions(product.upper(), distribution)
  → Redis 缓存 /oasis/clusterversions/{cluster_type}/{distribution_version}/   (TTL 3600s)
  → gringotts_client.ListClusterVersions
  → GG server/http/kop.go:123  ClusterMetaHandler
  → db.GetClusterVersionMap  按 cluster_type + distribution_version 过滤
  → 表 gg_cluster_versions
```

**缺失的数据**（`gringotts-framework/docs/kes/init_es819_metadata.sql`）：

```sql
INSERT INTO `gg_cluster_versions`
  (`id`, `name`, `cluster_type`, `distribution_version`, `type`, ...)
VALUES
  (@cv_id, '8.19.13', 'KES', 'ElasticSearch', 'public', ...);
```

配套还需写入 `gg_service_meta`（`service_name='elasticsearch'`, `version='8.19.13'`）与 4 条 `gg_component_meta`（MASTER / DATA / COORDINATOR / WARM）+ 1 条 KIBANA。9.4.8 对应 `init_es948_metadata.sql`。

**注意 `distribution_version` 的大小写**：必须是 `ElasticSearch`（驼峰），与 `kes.DistributionElasticSearch` 常量一致。2.8 已实测 `kmr:ListClusterVersions` 对 `ElasticSearch` 也报 `invalid`——说明 OpenAPI 层不承载版本元数据，**唯一的修复位置是 GG 的 `gg_cluster_versions` 表**。

**处理动作**（GG 侧，非 provider）：

```bash
mysql -h <gg-db> -u <user> -p <db> < docs/kes/init_es819_metadata.sql
# 清 Redis 缓存，避免 3600s 内继续读到空结果
redis-cli --scan --pattern '/oasis/clusterversions/KES/*' | xargs -r redis-cli DEL
```

---

## 5. 探针清单

| 目录 | 作用 | 依赖 |
|---|---|---|
| `/d/tmp/nohdrprobe` | KES8 LaunchCluster 的 `X-Ksc-Source` A/B 对照 | 无 |
| `/d/tmp/hdrprobe` | header × host × ClusterType 4 组矩阵 | 无 |
| `/d/tmp/kes7probe` | KES7 12 个版本接受性 | 无 |
| `/d/tmp/kes7charg` | KES7 `ChargeType` 8 种取值 | 无 |
| `/d/tmp/chargprobe` | kmr `ChargeType` 9 种取值 | 无 |
| `/d/tmp/ptprobe` | `ChargeType` × `PurchaseTime` 5 种组合 | 无 |
| `/d/tmp/accountprobe` | `ListClusters` 账号可见集群 | 无 |
| `/d/tmp/verprobe2` | kmr `ListClusterVersions` 4 种发行版 | 无 |
| `/d/tmp/queryprobe` | `DescribeCluster` + `ListInstanceGroups` | 无 |

所有探针均为**只读或可在网关层被拒**的请求，不会实际创建资源（账号无购买权限，创建请求在权限层即被拦截）。

---

## 6. 本次测试驱动的代码修改

| 文件 | 修改 | 依据 |
|---|---|---|
| `ksyun/service_ksyun_kes.go` | 新增 `mainVersionStateFunc`（TrimPrefix `OpenSource-`） | 2.4：`ListClusters` 返回带前缀 |
| `ksyun/service_ksyun_kes.go` | `purchase_time` 改为 `Optional`，`v > 0` 才下发 | 2.3：nil 通过、0 被拒 |
| `ksyun/service_ksyun_kes.go` | `ReadAndSetClusterResource` 对 `MainVersion` 做 TrimPrefix | 2.4 / 2.5 |
| `ksyun/service_ksyun_kes_vector.go` | 同上两处归一化 | 2.4 / 2.5 |
| `ksyun/service_ksyun_kes_vector.go` | **移除** `req.SetHeaders({"X-Ksc-Source": "kmr"})` 及错误注释 | 2.6：A/B 结果完全一致，header 无作用 |
| `ksyun/resource_ksyun_kes_cluster.go` | `charge_type` 加 `StringInSlice{Monthly, Daily}` | 2.2：其余值 402 |
| `ksyun/resource_ksyun_kes_cluster_vector.go` | `charge_type`（含 `instance_groups` 内）同上 | 2.7：kmr 路径白名单一致 |
| `ksyun/resource_ksyun_kes_charge_type_test.go` | 4 个单测覆盖上述约束 | 无需网络 |

单元测试：

```bash
GOFLAGS=-mod=mod GOPROXY=off GOSUMDB=off go test ./ksyun/ -run TestKes -v
# --- PASS: TestKesClusterChargeTypeValidation
# --- PASS: TestKesClusterVectorChargeTypeValidation
# --- PASS: TestKesResourcesInternalValidate
# --- PASS: TestKesMainVersionStateFunc
```

---

## 7. 待跟进（非 provider 侧）

1. **账号级购买权限**：`User have no permission to buy kes/kmr resource` 需产品侧为测试账号开通 KES / KMR 购买权限，否则端到端创建无法完成。
2. **版本元数据**：按第 4 节在 GG 执行 `init_es819_metadata.sql` / `init_es948_metadata.sql` 并清 Redis 缓存，`main_version must be in []` 才会消失。
