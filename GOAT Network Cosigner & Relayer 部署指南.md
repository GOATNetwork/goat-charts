# GOAT Network Cosigner & Relayer 迁移部署指南

## 一、创建 VPC（适用于新环境）

1. 登录 AWS 控制台，进入 **VPC 控制台 > VPC 向导 > 创建 VPC**
2. 填写以下参数：
   - 名称：`goat-network-cosigner-relayer（或其他标识环境的名称）
   - IPv4 CIDR 块：`12.2.0.0/16`
   - 可用区数量：1
   - 公有子网：1 个
   - 私有子网：1 个
   - NAT 网关：每个可用区一个
3. 其他保持默认，点击 **创建 VPC**

---

## 二、打通 VPC 对等连接

1. 由 **对等连接请求方** 提供以下信息：

   - AWS 账户 ID
   - VPC ID（上面创建的 VPC）

2. **被请求方** 登录 AWS 控制台：

   - 打开 **VPC 控制台 > 对等连接**
   - 找到刚才的请求，点击 **接受**

---

## 三、配置路由表打通网络

1. 打开 **VPC 控制台 > 路由表**
2. 找到刚才创建的 VPC 的私有子网对应的路由表（通常是名称中包含 `private`）
3. 编辑路由表，添加一条路由：
   - 目标地址（Destination）：`12.0.0.0/16`（对方 VPC CIDR）
   - 目标（Target）：选择刚才创建的 **对等连接**
   - 保存
---

## 四、使用 Terraform 创建 EC2 和资源

### 1. 准备 AKSK

- 使用拥有管理员权限的 IAM 用户，在控制台创建一组访问密钥（Access Key + Secret Key）

### 2. 安装 Terraform（以 Mac 为例）

```bash
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
terraform -v
```

### 3. 下载terraform配置代码

```bash
git clone https://github.com/GOATNetwork/aws-terraform-fireblocks-cosigner.git
cd aws-terraform-fireblocks-cosigner
```

### 4. 配置参数

分别进入以下目录并复制示例变量文件：

```bash
cd cosigner
cp terraform.tfvars.example terraform.tfvars
# 修改其中 Fireblocks 配置等参数

cd ../relayer
cp terraform.tfvars.example terraform.tfvars
# 修改 relayer 参数
```

---

## 五、运行 Terraform 创建资源

### 1. 创建密钥对（EC2 控制台 > 密钥对）

- `cosigner-nitro-prod`（下载并妥善保存私钥）
- `relayer-prod`（下载并妥善保存私钥）

### 2. 执行 Terraform 部署命令

```bash
cd cosigner
terraform init
terraform plan
terraform apply

cd ../relayer
terraform init
terraform plan
terraform apply
```

---

## 六、三套环境 VPC 对等连接互通

以 yuan 和 Eric 的网络为例：

1. **yuan 发起对 Eric 的对等连接请求**

   - 提供账户 ID 和 VPC ID
   - Eric 登录控制台，同意对等连接

2. **设置双方私网的路由表**

   - yuan 设置到 Eric 的路由：目标 CIDR `12.2.0.0/16`
   - Eric 设置到 yuan 的路由：目标 CIDR `12.1.0.0/16`
   - 都选择对等连接为目标

---

## 七、安装 Cosigner-Nitro

### 1. 使用 AWS 控制台 >  点击 `cosigner` 对应EC2 > 连接 > 会话管理器 登录

### 2. 在`cosigner`的ec2上执行以下命令，生成fireblocks.csr和fireblocks_secret.key，备份保存
```bash
openssl req -new -newkey rsa:4096 -nodes -keyout fireblocks_secret.key -out fireblocks.csr
```
### 3. 登录fireblocks网站，上传刚才生成的`fireblocks.csr`
![示例图片](1.jpg)
### 4. 本地生成 Callback 密钥对

```bash
openssl genrsa -out callback_private.pem 2048
openssl rsa -in callback_private.pem -outform PEM -pubout -out callback_public.pem
```

### 5. 配置 Callback 到 Cosigner

```bash
sudo cosigner callback-update
```

按提示填写：

- 回调地址：`http://${relayer-ip}:8080/api/fireblocks/cosigner`  <span style="color:red"># 配置新增relayer的IP地址</span>
- 粘贴 `callback_public.pem` 内容

### 6. 安装 Fireblocks Nitro 客户端

```bash
wget https://fb-customers-nitro.s3.amazonaws.com/nitro-cosigner-v2.0.2.tar.gz
tar -zxf nitro-cosigner-v2.0.2.tar.gz
./install.sh
```

安装过程中按提示输入：

- Fireblocks Pair Token    #上传fireblocks.csr文件得到Token
- S3 配置                   #terraform输出的s3 name
- KMS CMK ARN              #terraform输出的kms信息和arn拼接出arn地址

---

## 八、安装 GOAT Relayer

### 1. 使用 AWS 控制台 >  点击 `relayer` 对应EC2 > 连接 > 会话管理器 登录

### 2. 安装 Docker

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ubuntu
```

### 3. 准备 relayer 数据

```bash
mkdir -p data/keys
# 解压数据库压缩包到 data 目录
tar zxvf db.tar.gz -C data
```

### 4. 创建 `.env` 文件和 `docker-compose.yml`

**docker-compose.yml：**

```yaml
version: '3.8'

services:
  relayer:
    image: goatnetwork/goat-relayer:v20250624-385
    container_name: goat-relayer
    env_file:
      - .env
    network_mode: host
    volumes:
      - /data/db:/app/db
    restart: unless-stopped

  tss-node:
    image: goatnetwork/goat-network-tss:v20250624-37
    container_name: tss-node
    env_file:
      - .env
    network_mode: host
    volumes:
      - /data/keys:/app/data/keys
    restart: no
```

**.env：**
```yaml
LIBP2P_BOOT_NODES="/ip4/12.0.161.146/tcp/4001/p2p/12D3KooWACrp8YgYGKvX2obqgz6DoucWT8iEpe8gPb8dMev829wr"
HTTP_PORT=8080
RPC_PORT=52051
LIBP2P_PORT=4001
BTC_RPC=18.220.237.51:8332
BTC_RPC_USER=goat
BTC_RPC_PASS=goat
BTC_START_HEIGHT="902515"
BTC_CONFIRMATIONS=6
BTC_NETWORK_TYPE=mainnet
BTC_MAX_NETWORK_FEE=500
L2_RPC=http://k8s-goatnetw-mainnetg-16c67e83ec-8223da9b0e2edcca.elb.us-east-2.amazonaws.com:8545
L2_CHAIN_ID=2345
L2_START_HEIGHT=0
L2_CONFIRMATIONS=3
L2_MAX_BLOCK_RANGE=1100
L2_REQUEST_INTERVAL=5s
ENABLE_WEBHOOK=true
ENABLE_RELAYER=true
LOG_LEVEL=debug
DB_DIR=/app/db
GOATCHAIN_RPC_URI=tcp://k8s-goatnetw-mainnetg-16c67e83ec-8223da9b0e2edcca.elb.us-east-2.amazonaws.com:26657
GOATCHAIN_GRPC_URI=k8s-goatnetw-mainnetg-16c67e83ec-8223da9b0e2edcca.elb.us-east-2.amazonaws.com:9090
GOATCHAIN_ID=goat-mainnet
GOATCHAIN_ACCOUNT_PREFIX=goat
GOATCHAIN_DENOM=ugoat
BLS_SIG_TIMEOUT=30s
GIN_MODE=release
CONTRACT_TASK_MANAGER=0xfB25A9E65b01593aB5b232D3b744588c7c39Eba5
TSS_ENDPOINT=http://12.0.143.57:8180
FIREBLOCKS_CALLBACK_PUBLIC=
FIREBLOCKS_CALLBACK_PRIVATE=
FIREBLOCKS_SECRET=
FIREBLOCKS_API_KEY=
RELAYER_PRIVATE_KEY=
RELAYER_BLS_SK=
```

---

## 九、启动并查看日志

```bash
docker compose up -d
docker compose logs -f goat-relayer
```

