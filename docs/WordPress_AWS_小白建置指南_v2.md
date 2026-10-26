# WordPress × AWS AI Agent：小白一步一步建置指南（v2）

> 依據簡報《WordPress × AWS AI Agent 架構設計》（CoralEnergy 版）撰寫。
> 目標：從一個全新的 AWS 帳號開始，把 WordPress 架到雲端，最後加上簡報中的「AI 維運大腦」。
> 撰寫日期：2026-10。AWS 主控台的按鈕名稱偶爾會改版，若找不到某個按鈕，用頁面上方的搜尋框搜尋文中的關鍵字即可。

> **v2 修訂版（2026-10-24）**：依實際操作遇到的五個問題修訂。舊版保留在 `WordPress_AWS_小白建置指南_v1.md`。
> - 1.1 新增「子網路是公有還是私有」的檢查指令；1.7 啟動主機後立刻檢查網路。
> - 1.5 改成 CloudShell 一鍵建立 IAM 角色，自動填入帳號 ID，並補上 `rds:DescribeDBInstances`、S3 讀寫等缺少的權限。
> - 新增「[附錄 A：主機放在私有子網路（進階）](#附錄-a主機放在私有子網路進階)」，說明 VPC 端點與費用。
> - 新增「[實作踩坑：五個問題的排除方法](#實作踩坑五個問題的排除方法)」，其中包含「網站連線逾時」的修復步驟。

> 🚩 **整份指南最重要的一件事**：Phase 1 的 WordPress 主機一定要放在 **public（公有）子網路**，並且要有公有 IP。
> 放到 private（私有）子網路的話會出現一連串問題：SSH 連不上、主機下載不到 WordPress、AWS API 呼叫失敗，而且網站從外面永遠打不開。
> 資料庫（RDS）才是放在私有子網路的那一個。

---

## 目錄

- [0. 開始前：你會做出什麼、要花多少錢](#0-開始前你會做出什麼要花多少錢)
- [1. 名詞小字典（只解釋一次）](#1-名詞小字典只解釋一次)
- [2. 準備工作：帳號安全與預算警報](#2-準備工作帳號安全與預算警報)
- [Phase 1：單台 EC2 + RDS + 安全基礎](#phase-1單台-ec2--rds--安全基礎)
- [Phase 2：ALB + Auto Scaling + CloudFront + S3（高可用）](#phase-2alb--auto-scaling--cloudfront--s3高可用)
- [Phase 3：AI Agent 維運層（唯讀巡檢）](#phase-3ai-agent-維運層唯讀巡檢)
- [Phase 3 之後：審核後修復與自主修復](#phase-3-之後審核後修復與自主修復)
- [清除所有資源（避免持續扣款）](#清除所有資源避免持續扣款)
- [實作踩坑：五個問題的排除方法](#實作踩坑五個問題的排除方法)
- [附錄 A：主機放在私有子網路（進階）](#附錄-a主機放在私有子網路進階)
- [常見問題排除](#常見問題排除)

---

## 0. 開始前：你會做出什麼、要花多少錢

### 完成後的架構

```
                 使用者（瀏覽器）
                        │ HTTPS
                ┌───────▼────────┐
                │   CloudFront   │  ← Phase 2：全球快取 + 免費 HTTPS
                └───────┬────────┘
                ┌───────▼────────┐
                │      ALB       │  ← Phase 2：負載平衡
                └───────┬────────┘
        ┌───────────────┴───────────────┐
 ┌──────▼──────┐  Auto Scaling  ┌──────▼──────┐
 │ EC2 (AZ-a)  │ ◄────────────► │ EC2 (AZ-c)  │  ← Nginx + PHP-FPM + WordPress
 └──────┬──────┘                └──────┬──────┘
        │        共用檔案：EFS         │
        └───────────────┬───────────────┘
                ┌───────▼────────┐
                │  RDS (MySQL)   │  ← Phase 1：受管資料庫（私有子網路）
                └────────────────┘

 Phase 3 AI 維運層（簡報第 4 頁的三層架構）：
 觸發層  EventBridge 每日排程 / CloudWatch 告警 → SNS
 決策層  Lambda（LangChain Agent）＋ Amazon Bedrock（Claude）
 執行層  Systems Manager（SSM，無 SSH）→ EC2 上的 WP-CLI；CloudWatch Logs 讀日誌
 回報    Slack
```

### 三個階段與簡報的對應

| 階段 | 你會做的事 | 對應簡報 |
|---|---|---|
| Phase 1 | VPC、EC2、RDS、Secrets Manager、SSM（關閉 Port 22）、CloudWatch 日誌 | 第 7、8 頁：零信任 No SSH、核心服務 |
| Phase 2 | S3 備份、EFS 共用檔案、ALB、Auto Scaling、CloudFront | 第 15 頁：高可用、可擴展的基礎層 |
| Phase 3 | Bedrock + Lambda + LangChain Agent，唯讀巡檢與錯誤診斷，推送 Slack | 第 4–6、9–11、16 頁的 Phase 1「唯讀巡檢」 |

建議**每個階段做完、驗證成功再往下**。只想先有一個能用的網站，做完 Phase 1 就可以停。

### 費用預估（東京區域，美金／月，粗估）

> ⚠️ 以下為 2026 年的大約價格，實際請以 [AWS Pricing Calculator](https://calculator.aws/) 為準。2025 年 7 月之後建立的新帳號，免費方案改為「贈送抵用金（Credits）」制，可抵扣大部分 Phase 1 費用；較舊的帳號則是「12 個月免費方案」。兩者都**不會**涵蓋所有服務。

| 項目 | 規格 | 約略月費 |
|---|---|---|
| EC2 | t3.micro × 1 | ~$10 |
| EBS 硬碟 | 20 GB gp3 | ~$2 |
| 公有 IPv4 位址 | 每個 | ~$3.6 |
| RDS MySQL | db.t4g.micro、20 GB、單一可用區 | ~$21 |
| Secrets Manager | 每個密碼 $0.40 × 3 | ~$1.2 |
| CloudWatch | 少量日誌與 2 個自訂指標 | ~$1–2 |
| **Phase 1 小計** | | **約 $35–40** |
| ALB | 1 個 | ~$20–25 |
| EFS | 1 GB 以內 | <$1 |
| CloudFront | 每月 1 TB / 1,000 萬次請求內永久免費 | ~$0 |
| S3 | 備份數 GB | <$1 |
| **Phase 2 增加** | | **約 $22–27** |
| Lambda | 每天跑幾次 | ~$0 |
| Bedrock（Claude） | 每日巡檢約 2 萬 tokens | ~$1–5 |
| **Phase 3 增加** | | **約 $1–5** |

**最常見的意外帳單來源**：忘記刪除 RDS、ALB、NAT Gateway（本指南刻意不用 NAT Gateway，它一個月約 $45）、VPC 介面端點（每個每 AZ 每月約 $10，本指南預設不需要）、沒釋放的 Elastic IP。練習完請務必照「[清除所有資源](#清除所有資源避免持續扣款)」執行。

---

## 1. 名詞小字典（只解釋一次）

| 名詞 | 白話解釋 |
|---|---|
| **Region（區域）** | AWS 在世界各地的機房群。本指南用 **東京 ap-northeast-1**（離台灣近、Bedrock 的 Claude 模型可用）。想用新加坡就把文中所有 `ap-northeast-1` 換成 `ap-southeast-1`。 |
| **AZ（可用區）** | 同一區域內彼此獨立的機房，例如 `ap-northeast-1a`、`ap-northeast-1c`。放在兩個 AZ，一個機房掛了另一個還在。 |
| **VPC** | 你在 AWS 裡的私人網路，所有伺服器都住在裡面。 |
| **Subnet（子網路）** | VPC 再切成的小區塊。**公有子網路**可以連網際網路；**私有子網路**外面連不進來，資料庫放這裡。 |
| **Security Group（安全群組，SG）** | 每台機器的防火牆，規定「誰可以從哪個 Port 連進來」。 |
| **EC2** | 雲端上的虛擬主機，WordPress 程式跑在這裡。 |
| **AMI** | 主機的「映像檔」，像是一台電腦的完整備份，可用來複製出一樣的主機。 |
| **EBS** | 掛在 EC2 上的硬碟。 |
| **RDS** | AWS 幫你管理的資料庫（本指南用 MySQL），自動備份、自動修補。 |
| **IAM** | 權限管理。**IAM 使用者**是人用的帳號；**IAM 角色（Role）**是給機器或程式用的身分，不需要密碼。 |
| **Secrets Manager** | 保管密碼的保險箱，程式需要時再去拿，不寫死在程式碼裡。 |
| **SSM（Systems Manager）** | 讓你不用 SSH、不用開 Port 22 就能登入主機或下指令的服務。簡報第 7 頁「零信任」的核心。 |
| **Session Manager** | SSM 的功能之一：在瀏覽器裡直接開一個主機終端機視窗。 |
| **CloudWatch** | 監控服務：收集 CPU 等**指標（Metrics）**、收集**日誌（Logs）**、設定**告警（Alarm）**。 |
| **SNS** | 通知服務，告警發生時寄 Email 或觸發程式。 |
| **CloudShell** | AWS 主控台內建的命令列視窗，已裝好 AWS CLI，不用在自己電腦安裝任何東西。 |
| **AWS CLI** | 用文字指令操作 AWS 的工具，指令都是 `aws ...` 開頭。 |
| **ALB** | 應用程式負載平衡器，把流量分給多台 EC2，並檢查哪台是健康的。 |
| **Target Group（目標群組）** | ALB 要把流量送去的那群 EC2。 |
| **Auto Scaling Group（ASG）** | 自動維持指定數量的 EC2；壞了自動補、忙了自動加。 |
| **Launch Template（啟動範本）** | 告訴 ASG「新主機要長什麼樣子」。 |
| **EFS** | 可同時被多台 EC2 掛載的共用網路硬碟。 |
| **S3** | 物件儲存空間，便宜、耐用，適合放備份。 |
| **CloudFront** | CDN，把網站內容快取在全球節點，並提供免費 HTTPS。 |
| **Lambda** | 不用管主機、有事件才執行的程式（Serverless）。 |
| **EventBridge Scheduler** | 定時觸發器，例如「每天早上 9 點執行 Lambda」。 |
| **Bedrock** | AWS 的生成式 AI 服務，可呼叫 Anthropic Claude 等大型語言模型（LLM）。 |
| **LangChain** | 讓 LLM「使用工具」的 Python 框架，簡報中 Agent 的大腦編排層。 |
| **WP-CLI** | WordPress 的命令列工具，例如 `wp plugin list` 列出外掛。 |
| **ARN** | AWS 資源的完整「身分證字號」，例如 `arn:aws:ec2:ap-northeast-1:123456789012:instance/i-0abc...`。 |

---

## 2. 準備工作：帳號安全與預算警報

### 2.1 註冊與保護 Root 帳號

1. 到 <https://aws.amazon.com/> 註冊帳號（需要信用卡）。
2. 用 Root（註冊用的 Email）登入後，右上角點帳號名稱 → **Security credentials** → **Multi-factor authentication (MFA)** → **Assign MFA device**，用手機上的 Google Authenticator 或 Authy 綁定。
3. **之後日常操作不要用 Root。**

### 2.2 建立日常使用的管理員帳號

1. 上方搜尋列輸入 **IAM** → 左側 **Users** → **Create user**。
2. User name：`admin-你的名字`；勾選 **Provide user access to the AWS Management Console** → 選 **I want to create an IAM user** → 設定密碼。
3. **Set permissions** → **Attach policies directly** → 勾選 `AdministratorAccess` → **Create user**。
4. 記下頁面上的 **Console sign-in URL**，登出 Root，改用這個網址和新帳號登入，並同樣為它綁定 MFA。

> 進階做法是用 IAM Identity Center，但對初學者來說，一個有 MFA 的 IAM 使用者就足夠安全。

### 2.3 設定預算警報（最重要的一步）

1. 搜尋 **Billing and Cost Management** → 左側 **Budgets** → **Create budget**。
2. 選 **Use a template** → **Monthly cost budget**。
3. Budgeted amount 輸入 `50`（美金，可自行調整）；Email 填你的信箱 → **Create budget**。
4. 超過 85% 及 100% 時你會收到信。

✅ **驗證**：Budgets 頁面看到剛建立的預算，狀態為 OK。

### 2.4 切換區域與打開 CloudShell

1. 主控台右上角區域選單選 **Asia Pacific (Tokyo) ap-northeast-1**。之後每次操作前都確認右上角是東京。
2. 主控台最下方（或右上角）點 **CloudShell** 圖示，會打開一個終端機視窗。
3. 在 CloudShell 輸入：

```bash
aws sts get-caller-identity
aws configure get region
```

✅ **驗證**：第一行會顯示你的 12 位數帳號 ID（Account），第二行顯示 `ap-northeast-1`。把帳號 ID 記下來，後面會用到。

---

## Phase 1：單台 EC2 + RDS + 安全基礎

完成後：網站可用 `http://你的IP` 打開；主機**沒有開 Port 22**，只透過 SSM 管理；資料庫在私有子網路；密碼在 Secrets Manager；錯誤日誌送到 CloudWatch。

### 1.1 建立 VPC（網路）

1. 搜尋 **VPC** → **Create VPC**。
2. 選 **VPC and more**（精靈會一次建好子網路與路由）。
3. 設定：
   - Name tag auto-generation：`wp`
   - IPv4 CIDR：`10.0.0.0/16`（保持預設）
   - Number of Availability Zones：**2**
   - Number of public subnets：**2**
   - Number of private subnets：**2**
   - **NAT gateways：None**（⚠️ 一定要選 None，NAT 很貴）
   - VPC endpoints：**S3 Gateway**（免費）
   - DNS hostnames、DNS resolution：都勾選
4. **Create VPC**，等它全部打勾。

✅ **驗證**：左側 **Subnets** 中看到 4 個子網路：`wp-subnet-public1-ap-northeast-1a`、`wp-subnet-public2-ap-northeast-1c`、`wp-subnet-private1-...`、`wp-subnet-private2-...`。

#### 先搞懂：「公有」和「私有」子網路差在哪？

子網路是公有還是私有，**不是看名字**，而是看它的**路由表（Route table）**有沒有這一條：

| 目的地 | 目標 | 意思 |
|---|---|---|
| `0.0.0.0/0` | `igw-xxxx`（Internet Gateway，網際網路閘道） | 「VPC 以外的流量都走網際網路大門」 |

- **有這條** → 公有子網路：主機配上公有 IP 後，可以連外，外面也能連進來。**WordPress 主機放這裡。**
- **沒有這條** → 私有子網路：主機出不去，外面也進不來。**RDS 資料庫放這裡。**

整個流程：

```
網際網路 ⇄ Internet Gateway ⇄ 公有子網路（EC2 + 公有 IP）⇄ 私有子網路（RDS）
```

**在 CloudShell 檢查每個子網路的對外路由**（把這段整個貼上執行）：

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=wp-vpc --query "Vpcs[0].VpcId" --output text)
echo "VPC：$VPC_ID"
for s in $(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID --query "Subnets[].SubnetId" --output text); do
  name=$(aws ec2 describe-subnets --subnet-ids $s --query "Subnets[0].Tags[?Key=='Name'].Value" --output text)
  gw=$(aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=$s \
       --query "RouteTables[0].Routes[?DestinationCidrBlock=='0.0.0.0/0'].GatewayId" --output text)
  echo "$s  $name  →  ${gw:-（沒有 0.0.0.0/0 路由＝私有）}"
done
```

✅ **驗證**：兩個 `public` 子網路後面顯示 `igw-...`；兩個 `private` 子網路顯示「沒有 0.0.0.0/0 路由＝私有」。**記下 `wp-subnet-public1-ap-northeast-1a` 的子網路 ID（`subnet-...`），1.7 會用到。**

> 如果 public 子網路也顯示「私有」（通常是手動建立 VPC 時漏掉），請參考「[問題 5](#問題-5網站從網際網路連不上連線逾時)」的步驟補上 Internet Gateway 路由。

### 1.2 建立安全群組（防火牆）

到 **VPC → Security groups → Create security group**，VPC 都選 `wp-vpc`，建立兩個：

**① `wp-web-sg`（WordPress 主機）**
- Inbound rules：**Add rule** → Type `HTTP`、Source `Anywhere-IPv4 (0.0.0.0/0)`
- **不要加 SSH（22）**。這就是簡報第 7 頁的「關閉 Port 22」。
- Phase 1 只開 HTTP 80 就好，**不需要開 443**：此階段還沒有憑證，HTTPS 會在 Phase 2 由 CloudFront 負責。
- Outbound：保持預設（全部允許）。主機要靠對外連線去下載 WordPress、連 SSM 和 Secrets Manager，**不要刪掉 outbound 規則**。

**② `wp-db-sg`（資料庫）**
- Inbound rules：Type `MYSQL/Aurora`（3306）、Source 選 **Custom** → 搜尋並選 `wp-web-sg`
- 意思是：只有掛著 `wp-web-sg` 的主機可以連資料庫，網路上其他人都連不到。

✅ **驗證**：`wp-web-sg` 只有一條 Port 80 規則；`wp-db-sg` 只有一條 Port 3306、來源為 `sg-xxxx (wp-web-sg)` 的規則。

### 1.3 建立 RDS 資料庫

1. 搜尋 **RDS** → **Create database**。
2. 選 **Standard create**（新版主控台可能叫 **Full configuration**）。
3. Engine：**MySQL**；Engine version：預設的 8.x 版即可。
4. Templates：**Free tier**（若沒有此選項，選 **Dev/Test** 後手動照下面設定）。
5. Settings：
   - DB instance identifier：`wp-db`
   - Master username：`admin`
   - Credentials management：**Managed in AWS Secrets Manager**（AWS 自動產生並保管管理員密碼）
6. Instance configuration：**db.t4g.micro**（或 db.t3.micro）。
7. Storage：gp3 或 gp2、**20 GB**；把 **Enable storage autoscaling** 取消勾選（避免意外長大）。
8. Availability：**Single-AZ**（Multi-AZ 價格加倍，練習不需要）。
9. Connectivity：
   - Compute resource：**Don't connect to an EC2 compute resource**
   - VPC：`wp-vpc`
   - DB subnet group：**Create new DB Subnet Group**
   - **Public access：No**
   - VPC security group：**Choose existing** → 移除 `default`，選 `wp-db-sg`
10. 展開最下方 **Additional configuration**：
    - **Initial database name：`wordpress`**（⚠️ 這格很容易漏掉）
    - Backup retention period：`1` 天
    - 取消勾選 Enable Enhanced monitoring（省錢）
11. **Create database**。建立約需 5–15 分鐘，先往下做。

✅ **驗證**（等狀態變 **Available** 後）：點進 `wp-db` → **Connectivity & security** 分頁，複製 **Endpoint**（長得像 `wp-db.xxxxxxxx.ap-northeast-1.rds.amazonaws.com`），記下來。在 **Configuration** 分頁看到 **Master credentials ARN**，代表密碼已在 Secrets Manager。

### 1.4 建立 WordPress 專用的資料庫密碼

WordPress 不該用資料庫管理員帳號連線，所以我們另外建一個 `wpuser`。
（不直接沿用 RDS 管理員密碼的原因：RDS 代管的管理員密碼預設每 7 天自動輪替，寫死在 WordPress 設定檔裡會在一週後突然連不上。）

在 **CloudShell** 執行：

```bash
# 產生一組不含特殊符號的 32 碼隨機密碼
WP_DB_PASS=$(aws secretsmanager get-random-password \
  --password-length 32 --exclude-punctuation \
  --query RandomPassword --output text)

# 存進 Secrets Manager
aws secretsmanager create-secret \
  --name wp/db-app \
  --description "WordPress 連線資料庫用的帳號密碼" \
  --secret-string "{\"username\":\"wpuser\",\"password\":\"$WP_DB_PASS\"}"
```

✅ **驗證**：

```bash
aws secretsmanager describe-secret --secret-id wp/db-app --query Name
```

輸出 `"wp/db-app"` 即成功。

### 1.5 建立 EC2 的 IAM 角色

EC2 需要一個身分（IAM 角色），才能做這些事：

| 主機要做的事 | 需要的權限 | 在哪一步用到 |
|---|---|---|
| 被 SSM／Session Manager 管理 | `AmazonSSMManagedInstanceCore`（AWS 內建政策） | 1.9 |
| 送日誌與指標到 CloudWatch | `CloudWatchAgentServerPolicy`（AWS 內建政策） | 1.7、1.13 |
| 查詢 RDS 管理員密碼放在哪 | `rds:DescribeDBInstances`（只限 `wp-db`） | 1.10 |
| 讀取資料庫密碼 | `secretsmanager:GetSecretValue`（只限 `wp/*` 與 `rds!db-*`） | 1.10 |
| 上傳／下載備份與安裝檔 | `s3:PutObject`、`s3:GetObject`、`s3:ListBucket`（只限 `wp-backup-帳號ID`） | 2.1、附錄 A |

> v1 版的指南在這裡**漏了 `rds:DescribeDBInstances` 和 S3 權限**，而且要求手動把 `<ACCOUNT_ID>` 換成帳號 ID，很容易忘記替換，結果就是 `AccessDenied`。v2 改用下面的指令，**帳號 ID 會自動填入**。

**在 CloudShell 整段貼上執行：**

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=ap-northeast-1
echo "帳號 ID：$ACCOUNT_ID"

# 1) 建立角色（允許 EC2 使用這個角色）
cat > /tmp/ec2-trust.json <<'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Principal": { "Service": "ec2.amazonaws.com" }, "Action": "sts:AssumeRole" }
  ]
}
EOF
aws iam create-role --role-name wp-ec2-role \
  --assume-role-policy-document file:///tmp/ec2-trust.json

# 2) 掛上兩個 AWS 內建政策
aws iam attach-role-policy --role-name wp-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore
aws iam attach-role-policy --role-name wp-ec2-role \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

# 3) 自訂的最小權限政策（帳號 ID 自動帶入）
cat > /tmp/wp-ec2-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadWordPressSecrets",
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": [
        "arn:aws:secretsmanager:${REGION}:${ACCOUNT_ID}:secret:wp/*",
        "arn:aws:secretsmanager:${REGION}:${ACCOUNT_ID}:secret:rds!db-*"
      ]
    },
    {
      "Sid": "RDSReadAccess",
      "Effect": "Allow",
      "Action": "rds:DescribeDBInstances",
      "Resource": "arn:aws:rds:${REGION}:${ACCOUNT_ID}:db:wp-db"
    },
    {
      "Sid": "BackupBucketObjects",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::wp-backup-${ACCOUNT_ID}/*"
    },
    {
      "Sid": "BackupBucketList",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::wp-backup-${ACCOUNT_ID}"
    }
  ]
}
EOF
aws iam put-role-policy --role-name wp-ec2-role \
  --policy-name wp-ec2-least-privilege --policy-document file:///tmp/wp-ec2-policy.json

# 4) 建立「執行個體設定檔」，EC2 透過它戴上角色（主控台建立角色時會自動做，CLI 要手動做）
aws iam create-instance-profile --instance-profile-name wp-ec2-role
aws iam add-role-to-instance-profile --instance-profile-name wp-ec2-role --role-name wp-ec2-role
```

> 若你之前已經用主控台建過 `wp-ec2-role`，第 1 步和第 4 步會出現 `EntityAlreadyExists`，**可以忽略**，其他指令照樣會補上缺少的權限。

✅ **驗證**（用 IAM 模擬器確認權限真的生效）：

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ROLE=arn:aws:iam::$ACCOUNT_ID:role/wp-ec2-role
check() { aws iam simulate-principal-policy --policy-source-arn $ROLE --action-names $1 \
  --resource-arns $2 --query "EvaluationResults[0].EvalDecision" --output text | sed "s|^|$1 → |"; }
check rds:DescribeDBInstances      arn:aws:rds:ap-northeast-1:$ACCOUNT_ID:db:wp-db
check secretsmanager:GetSecretValue arn:aws:secretsmanager:ap-northeast-1:$ACCOUNT_ID:secret:wp/db-app-AbCdEf
check s3:GetObject                 arn:aws:s3:::wp-backup-$ACCOUNT_ID/test.txt
check ssm:UpdateInstanceInformation "*"
aws iam list-instance-profiles-for-role --role-name wp-ec2-role --query "InstanceProfiles[].InstanceProfileName"
```

前四行結果都是 `allowed`，最後一行顯示 `"wp-ec2-role"`。出現 `implicitDeny` 就代表那個權限還沒加上，請重跑上一段指令。

> **為什麼不直接掛 `SecretsManagerReadWrite`、`AmazonS3ReadOnlyAccess`？**
> 掛上去確實能解決 AccessDenied，但它們的範圍是「帳號內**所有**密碼可讀寫、**所有** S3 桶可讀」。萬一 WordPress 被入侵，攻擊者就能拿到你帳號裡的每一組密碼。上面的自訂政策只開放 WordPress 需要的那幾個資源，效果相同但安全得多（簡報第 11 頁的「IAM 最小權限」）。
> 如果你已經掛了這兩個政策，跑完上面的指令後，可以到 **IAM → Roles → wp-ec2-role → Permissions** 把它們 **Remove** 掉。

> `rds!db-*` 是 RDS 管理員密碼的名稱格式，只在 1.10 步建立 `wpuser` 時用一次；完成 Phase 1 後可以從政策中刪掉這一行。

### 1.6 先建立 CloudWatch 日誌群組

在 **CloudShell**：

```bash
for g in /wordpress/nginx-error /wordpress/php-fpm-error /wordpress/nginx-access; do
  aws logs create-log-group --log-group-name "$g"
  aws logs put-retention-policy --log-group-name "$g" --retention-in-days 14
done
aws logs describe-log-groups --log-group-name-prefix /wordpress --query "logGroups[].logGroupName"
```

✅ **驗證**：列出 3 個 `/wordpress/...` 群組。（設定保留 14 天，避免日誌無限累積收費。）

### 1.7 啟動 EC2 主機並自動安裝 WordPress

1. **EC2 → Instances → Launch instances**。
2. Name：`wp-web-1`；點 **Add additional tags** → 新增一個標籤 Key `Role`、Value `wordpress`（Phase 3 的 AI Agent 會靠這個標籤找主機），Resource types 勾 Instances。
3. AMI：**Amazon Linux 2023**（64-bit x86）。
4. Instance type：**t3.micro**（標示 Free tier eligible 的那個）。
5. **Key pair：Proceed without a key pair**（我們不用 SSH）。
6. Network settings → **Edit**：
   - VPC：`wp-vpc`
   - Subnet：**`wp-subnet-public1-ap-northeast-1a`**（⚠️ 名稱裡要有 **public**，子網路 ID 要和 1.1 記下的一樣。選到 private 是最常見的錯誤，後果見「[實作踩坑](#實作踩坑五個問題的排除方法)」）
   - Auto-assign public IP：**Enable**（⚠️ 沒有公有 IP 一樣連不上網際網路）
   - Firewall：**Select existing security group** → `wp-web-sg`
7. Storage：`20` GiB、**gp3**。
8. 展開 **Advanced details**：
   - **IAM instance profile**：`wp-ec2-role`
   - **Metadata version**：V2 only (token required)
   - 拉到最下面 **User data**，貼上以下整段腳本（主機第一次開機時會自動執行）：

```bash
#!/bin/bash
set -euxo pipefail
exec > /var/log/user-data.log 2>&1

# 1) 安裝 Nginx、PHP-FPM、MySQL 用戶端、CloudWatch Agent、EFS 工具（Phase 2 用）
dnf install -y nginx php-fpm php-cli php-mysqlnd php-gd php-mbstring php-xml php-opcache \
  mariadb105 jq amazon-cloudwatch-agent amazon-efs-utils

# 2) 下載 WordPress
mkdir -p /var/www/html
curl -sSL https://wordpress.org/latest.tar.gz | tar -xz -C /var/www/html --strip-components=1
chown -R apache:apache /var/www/html

# 3) 安裝 WP-CLI
curl -sSL -o /usr/local/bin/wp https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
chmod +x /usr/local/bin/wp

# 4) Nginx 設定（WordPress 專用）
cat > /etc/nginx/nginx.conf <<'EOF'
user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;
include /usr/share/nginx/modules/*.conf;
events { worker_connections 1024; }
http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;
    access_log    /var/log/nginx/access.log;
    sendfile on;
    keepalive_timeout 65;
    client_max_body_size 64M;

    upstream php { server unix:/run/php-fpm/www.sock; }

    server {
        listen 80 default_server;
        server_name _;
        root /var/www/html;
        index index.php;

        location = /healthz { access_log off; default_type text/plain; return 200 "ok"; }
        location / { try_files $uri $uri/ /index.php?$args; }
        location ~ \.php$ {
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_pass php;
        }
        location ~ /\.(ht|git) { deny all; }
    }
}
EOF

# 5) CloudWatch Agent：收集錯誤日誌與記憶體／硬碟使用率
cat > /opt/aws/amazon-cloudwatch-agent/etc/config.json <<'EOF'
{
  "agent": { "run_as_user": "root" },
  "logs": { "logs_collected": { "files": { "collect_list": [
    { "file_path": "/var/log/nginx/error.log",       "log_group_name": "/wordpress/nginx-error",   "log_stream_name": "{instance_id}" },
    { "file_path": "/var/log/php-fpm/www-error.log", "log_group_name": "/wordpress/php-fpm-error", "log_stream_name": "{instance_id}" },
    { "file_path": "/var/log/nginx/access.log",      "log_group_name": "/wordpress/nginx-access",  "log_stream_name": "{instance_id}" }
  ] } } },
  "metrics": {
    "append_dimensions": { "InstanceId": "${aws:InstanceId}" },
    "metrics_collected": {
      "mem":  { "measurement": ["mem_used_percent"] },
      "disk": { "measurement": ["used_percent"], "resources": ["/"] }
    }
  }
}
EOF

# 6) 啟動服務並設定開機自動啟動
systemctl enable --now php-fpm nginx
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/etc/config.json

echo "USER DATA DONE"
```

9. 右側 **Launch instance**。

> 小提醒：簡報第 9 頁的程式用 `www-data` 身分執行 WP-CLI，那是 Ubuntu 的慣例；在 Amazon Linux 2023 上，PHP-FPM 的對應身分是 **`apache`**，本指南統一使用 `apache`。

**啟動後立刻做網路檢查**（CloudShell，主機狀態為 Running 後執行）：

```bash
IID=$(aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=wp-web-1" "Name=instance-state-name,Values=pending,running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)
read SUBNET PUBIP PROFILE <<< $(aws ec2 describe-instances --instance-ids $IID \
  --query "Reservations[0].Instances[0].[SubnetId, PublicIpAddress, IamInstanceProfile.Arn]" --output text)
GW=$(aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=$SUBNET \
  --query "RouteTables[0].Routes[?DestinationCidrBlock=='0.0.0.0/0'].GatewayId" --output text)
echo "主機：$IID"
echo "子網路：$SUBNET"
echo "對外路由：${GW:-❌ 沒有（這是私有子網路！）}"
echo "公有 IP：$PUBIP"
echo "IAM 角色：$PROFILE"
```

✅ **驗證**：
- 對外路由顯示 **`igw-...`**
- 公有 IP 是一組數字（不是 `None`）
- IAM 角色以 `.../wp-ec2-role` 結尾

**任何一項不對，先不要往下做**：
- 對外路由沒有 `igw-...` → 主機放錯子網路，Terminate 這台主機後重做 1.7（選 public 子網路）。
- 公有 IP 是 `None` → 同樣 Terminate 後重做，記得 Auto-assign public IP 選 **Enable**。（1.8 綁 Elastic IP 也能補救，但前提是對外路由正確。）
- IAM 角色是 `None` → **Actions → Security → Modify IAM role** → 選 `wp-ec2-role` → 再 **Instance state → Reboot**。

> 主機才剛建立、還沒放資料，重做是最省時間的修法。如果已經在上面裝好網站，請改用「[問題 5](#問題-5網站從網際網路連不上連線逾時)」的搬家方法。

### 1.8 固定公有 IP（Elastic IP）

主機重開機後公有 IP 會變，網站網址就會壞掉，所以先固定下來：

1. **EC2 → Elastic IPs → Allocate Elastic IP address → Allocate**。
2. 選取新 IP → **Actions → Associate Elastic IP address** → Instance 選 `wp-web-1` → **Associate**。
3. 記下這個 IP，以下稱為 `<EIP>`。

> Elastic IP 只是「固定的門牌號碼」，**不會讓主機連上網際網路**。主機所在的子網路一定要有通往 Internet Gateway 的路由（1.7 的網路檢查），否則綁了 IP 網站還是會連線逾時。

### 1.9 用 Session Manager 登入主機（沒有 SSH！）

1. 等 2–3 分鐘讓主機開好。
2. **EC2 → Instances** → 勾選 `wp-web-1` → 右上 **Connect** → 分頁 **Session Manager** → **Connect**。
3. 瀏覽器會打開一個黑色終端機。先切換成 root 並確認安裝完成：

```bash
sudo -i
tail -n 3 /var/log/user-data.log
systemctl is-active nginx php-fpm amazon-cloudwatch-agent
curl -s localhost/healthz
```

✅ **驗證**：看到 `USER DATA DONE`、三個 `active`、以及 `ok`。

> 如果 Connect 按鈕是灰的或顯示 SSM Agent 未連線：通常是主機放在私有子網路、沒有公有 IP，或沒掛 `wp-ec2-role`。請看「[問題 1](#問題-1ssh-連不上session-manager-也無法連線)」。
>
> 為什麼不用 SSH？本指南的主機刻意沒有開 Port 22，也沒有金鑰，所以 **SSH 本來就連不上，這是正常的**。Session Manager 走的是 AWS 的加密通道（HTTPS 443 對外連線），不需要開任何對內 Port。

### 1.10 建立資料庫使用者並產生 wp-config.php

繼續在 Session Manager（root）中，**先把 `<RDS_ENDPOINT>` 換成 1.3 記下的 Endpoint**：

```bash
export AWS_DEFAULT_REGION=ap-northeast-1
RDS_HOST=<RDS_ENDPOINT>

# 取得 RDS 管理員密碼（Secrets Manager 中名稱以 rds!db- 開頭的那個）
MASTER_ARN=$(aws rds describe-db-instances --db-instance-identifier wp-db \
  --query "DBInstances[0].MasterUserSecret.SecretArn" --output text)
MASTER_PASS=$(aws secretsmanager get-secret-value --secret-id "$MASTER_ARN" \
  --query SecretString --output text | jq -r .password)

# 取得 1.4 建立的 wpuser 密碼
APP=$(aws secretsmanager get-secret-value --secret-id wp/db-app --query SecretString --output text)
APP_USER=$(echo "$APP" | jq -r .username)
APP_PASS=$(echo "$APP" | jq -r .password)

# 用管理員身分建立 wpuser，只給 wordpress 這個資料庫的權限
MYSQL_PWD="$MASTER_PASS" mysql -h "$RDS_HOST" -u admin -e "
  CREATE USER IF NOT EXISTS '$APP_USER'@'%' IDENTIFIED BY '$APP_PASS';
  GRANT ALL PRIVILEGES ON wordpress.* TO '$APP_USER'@'%';
  FLUSH PRIVILEGES;"

# 驗證 wpuser 可以登入
MYSQL_PWD="$APP_PASS" mysql -h "$RDS_HOST" -u "$APP_USER" -e "SHOW DATABASES;"

# 用 WP-CLI 產生 wp-config.php
sudo -u apache wp config create --path=/var/www/html \
  --dbname=wordpress --dbuser="$APP_USER" --dbpass="$APP_PASS" --dbhost="$RDS_HOST"
chmod 640 /var/www/html/wp-config.php

unset MASTER_PASS APP_PASS APP
```

✅ **驗證**：`SHOW DATABASES;` 的輸出包含 `wordpress`；最後一行顯示 `Success: Generated 'wp-config.php' file.`

> - 出現 `AccessDenied ... rds:DescribeDBInstances` 或 `secretsmanager:GetSecretValue` → 角色權限不完整，見「[問題 2](#問題-2iam-權限不足accessdenied--403)」。
> - 指令卡住很久後出現 `Could not connect to the endpoint URL` → 主機連不到 AWS API，見「[問題 4](#問題-4無法存取-secrets-manager-api)」。
> - `mysql` 卡住不動 → 多半是 `wp-db-sg` 的來源沒設成 `wp-web-sg`，回到 1.2 檢查。

### 1.11 完成 WordPress 安裝

1. 在自己電腦的瀏覽器打開 `http://<EIP>`。
2. 選語言 **繁體中文** → 填網站標題、管理員帳號（**不要用 admin**）、強密碼、Email → **安裝 WordPress**。
3. 用剛設定的帳號登入 `http://<EIP>/wp-admin`。

✅ **驗證**：
- 前台 `http://<EIP>` 顯示預設主題首頁。
- 回到 Session Manager 執行 `sudo -u apache wp --path=/var/www/html core version`，顯示版本號。

> ⚠️ 此階段網站是 **HTTP（未加密）**，登入密碼會以明碼傳輸。Phase 1 僅供練習；正式上線請完成 Phase 2 的 CloudFront（免費 HTTPS）。

### 1.12 設定告警通知（CPU 過高寄信）

1. 搜尋 **SNS → Topics → Create topic** → Type **Standard** → Name `wp-alerts` → **Create topic**。
2. 進入 `wp-alerts` → **Create subscription** → Protocol **Email** → Endpoint 填你的信箱 → 建立後到信箱點 **Confirm subscription**。
3. 搜尋 **CloudWatch → Alarms → Create alarm → Select metric** → **EC2 → Per-Instance Metrics** → 勾選 `wp-web-1` 的 `CPUUtilization` → **Select metric**。
4. Statistic `Average`、Period `5 minutes`；Threshold：**Greater than 80**；**Datapoints to alarm** 設 `2 out of 2`。
5. Notification：In alarm → 選 `wp-alerts`。
6. Alarm name：`wp-cpu-high` → **Create alarm**。

✅ **驗證**：告警狀態從 *Insufficient data* 變成 **OK**。

### 1.13 確認日誌有送到 CloudWatch

在 Session Manager 故意製造一筆錯誤：

```bash
curl -s -o /dev/null -w "%{http_code}\n" localhost/this-file-does-not-exist.php
```

然後到 **CloudWatch → Log groups → /wordpress/nginx-error**（或 `/wordpress/nginx-access`），點進以 `i-` 開頭的 log stream。

✅ **驗證**：1–2 分鐘內看到剛剛那筆請求的紀錄。

### ✅ Phase 1 完成檢查表

- [ ] `http://<EIP>` 打得開 WordPress
- [ ] `wp-web-sg` **沒有** Port 22
- [ ] RDS 的 Public access 是 **No**
- [ ] 資料庫密碼都在 Secrets Manager（`wp/db-app` 與 `rds!db-...`）
- [ ] 可以用 Session Manager 登入主機
- [ ] CloudWatch 收得到 Nginx 日誌，`wp-cpu-high` 告警為 OK
- [ ] （建議）從 `wp-ec2-role` 的 inline policy 刪除 `rds!db-*` 那一行

**想先停在這裡？** 可以。但請記得：主機與資料庫即使沒人用也會持續計費，見「[清除所有資源](#清除所有資源避免持續扣款)」或「暫停省錢」小節。

---

## Phase 2：ALB + Auto Scaling + CloudFront + S3（高可用）

完成後：網站透過 `https://xxxx.cloudfront.net` 存取；主機壞掉會自動補一台；流量變大會自動加到 2 台；資料庫每天備份到 S3。

> 為什麼需要 EFS？多台主機時，使用者上傳的圖片只會存到其中一台。把 WordPress 的整個網站目錄放到 EFS（共用硬碟），所有主機看到的檔案就一模一樣。

### 2.1 建立 S3 備份桶並做第一次資料庫備份

1. **S3 → Create bucket**：
   - Bucket name：`wp-backup-<ACCOUNT_ID>`（名稱全球唯一，加帳號 ID 最不會撞名）
   - Region：東京
   - **Block all public access：保持勾選**
   - **Create bucket**
2. 進入該 bucket → **Management → Create lifecycle rule**：名稱 `expire-30d`、Apply to all objects（勾選確認）、勾 **Expire current versions of objects**、天數 `30` → 建立。（舊備份 30 天後自動刪除。）
3. 主機寫入備份的權限已經在 1.5 的 `wp-ec2-least-privilege` 政策中加好了（`s3:PutObject`／`GetObject`），**bucket 名稱必須是 `wp-backup-你的帳號ID`**，政策才會對得上；取別的名字會出現 `403 Forbidden`。

4. 在 Session Manager（root）做第一次備份：

```bash
export AWS_DEFAULT_REGION=ap-northeast-1
BUCKET=wp-backup-<ACCOUNT_ID>
FILE=/tmp/wp-$(date +%F-%H%M).sql
sudo -u apache wp --path=/var/www/html db export "$FILE"
gzip "$FILE" && aws s3 cp "$FILE.gz" "s3://$BUCKET/db/" && rm -f "$FILE.gz"
```

5. 設定每天凌晨 3 點自動備份：

```bash
# 把 bucket 名稱存成設定檔（換成你的帳號 ID）
echo "wp-backup-<ACCOUNT_ID>" > /etc/wp-backup-bucket

cat > /usr/local/bin/wp-backup.sh <<'EOF'
#!/bin/bash
set -e
export AWS_DEFAULT_REGION=ap-northeast-1
BUCKET=$(cat /etc/wp-backup-bucket)
FILE=/tmp/wp-$(hostname)-$(date +%F).sql
sudo -u apache wp --path=/var/www/html db export "$FILE" --quiet
gzip -f "$FILE" && aws s3 cp "$FILE.gz" "s3://$BUCKET/db/" --quiet && rm -f "$FILE.gz"
EOF
chmod +x /usr/local/bin/wp-backup.sh
dnf install -y cronie && systemctl enable --now crond
echo "0 3 * * * root /usr/local/bin/wp-backup.sh" > /etc/cron.d/wp-backup

# 立刻手動跑一次，確認腳本可用
/usr/local/bin/wp-backup.sh && echo "BACKUP OK"
```

> 之後 Phase 2 做成 AMI 時，這個排程會跟著複製到每台主機。若 ASG 擴充到 2 台，兩台都會備份，檔名含主機名稱所以不會互相覆蓋。

✅ **驗證**：S3 主控台的 `db/` 資料夾裡看到一個 `.sql.gz` 檔案。

> RDS 本身也會每天自動快照（1.3 設了保留 1 天）；S3 上的 SQL 備份是第二道保險，也方便搬家。

### 2.2 建立 EFS 共用硬碟並搬移網站檔案

1. 先建立 EFS 的安全群組：**VPC → Security groups → Create** → 名稱 `wp-efs-sg`、VPC `wp-vpc`、Inbound：Type **NFS**（2049）、Source **Custom → `wp-web-sg`**。
2. **EFS → Create file system → Customize**：
   - Name：`wp-efs`；File system type：**Regional**
   - Automatic backups：可取消（省錢）；Lifecycle management：保持預設
   - Throughput mode：**Elastic**（預設）
   - Next → Network：VPC `wp-vpc`；Mount targets 兩個 AZ 都選 **public 子網路**，Security groups 都改成 **只有 `wp-efs-sg`**
   - Next → Next → **Create**
3. 記下 File system ID（`fs-xxxxxxxx`）。
4. 在 Session Manager（root）把網站搬到 EFS（把 `fs-xxxxxxxx` 換成你的）：

```bash
EFS_ID=fs-xxxxxxxx

# 先掛到暫存位置，把檔案複製過去
mkdir -p /mnt/efs
mount -t efs -o tls $EFS_ID:/ /mnt/efs
rsync -a /var/www/html/ /mnt/efs/
umount /mnt/efs

# 舊目錄改名保留，改用 EFS 當網站目錄，並設定開機自動掛載
systemctl stop nginx php-fpm
mv /var/www/html /var/www/html.local-backup
mkdir -p /var/www/html
echo "$EFS_ID:/ /var/www/html efs _netdev,tls 0 0" >> /etc/fstab
mount -a
systemctl start php-fpm nginx

df -h /var/www/html
curl -s -o /dev/null -w "%{http_code}\n" localhost/
```

✅ **驗證**：`df -h` 顯示 `/var/www/html` 來自 `127.0.0.1:/`（EFS 經 TLS 掛載的樣子）；最後一行為 `200`；`http://<EIP>` 網站正常。

### 2.3 建立 ALB（負載平衡器）

**① ALB 安全群組**：**VPC → Security groups → Create** → 名稱 `wp-alb-sg`、VPC `wp-vpc`、Inbound：HTTP 80 來源 `0.0.0.0/0`（暫時，2.5 會收緊）。

**② 目標群組**：
1. **EC2 → Target Groups → Create target group**。
2. Target type **Instances**；Name `wp-tg`；Protocol HTTP、Port 80；VPC `wp-vpc`。
3. Health checks：Path 填 **`/healthz`**；展開 Advanced → Healthy threshold `2`、Interval `15` 秒。
4. Next → 勾選 `wp-web-1` → **Include as pending below** → **Create target group**。

**③ ALB**：
1. **EC2 → Load Balancers → Create load balancer → Application Load Balancer**。
2. Name `wp-alb`；Scheme **Internet-facing**；IP type IPv4。
3. Network mapping：VPC `wp-vpc`，勾選兩個 AZ，各選 **public** 子網路。
4. Security groups：移除 default，選 `wp-alb-sg`。
5. Listeners：HTTP 80 → Default action **Forward to `wp-tg`**。
6. **Create load balancer**，等狀態變 Active（約 3–5 分鐘），複製 **DNS name**（`wp-alb-xxxx.ap-northeast-1.elb.amazonaws.com`）。

**④ 收緊主機防火牆**：編輯 `wp-web-sg` 的 inbound，把 HTTP 來源從 `0.0.0.0/0` 改成 **`wp-alb-sg`**。從此主機只接受 ALB 送來的流量。

✅ **驗證**：
- **Target Groups → wp-tg → Targets**：`wp-web-1` 狀態 **healthy**。
- 在 CloudShell：`curl -s http://<ALB的DNS>/healthz` 回傳 `ok`。
- `http://<EIP>` 現在應該**打不開**了（代表防火牆生效）。

### 2.4 建立 CloudFront（全球加速 + 免費 HTTPS）

先在 CloudShell 產生一組「暗號」，讓 ALB 只接受來自 CloudFront 的流量：

```bash
aws secretsmanager get-random-password --password-length 32 --exclude-punctuation \
  --query RandomPassword --output text
```

記下輸出，以下稱 `<ORIGIN_SECRET>`。

1. 搜尋 **CloudFront → Create distribution**（若出現方案選擇，選 **Free** 或 pay-as-you-go 皆可）。
2. **Origin**：
   - Origin domain：貼上 ALB 的 DNS name
   - Protocol：**HTTP only**、Port 80
   - **Add custom header**：Header name `X-Origin-Verify`、Value `<ORIGIN_SECRET>`
3. **Default cache behavior**：
   - Viewer protocol policy：**Redirect HTTP to HTTPS**
   - Allowed HTTP methods：**GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE**（WordPress 登入、留言需要 POST）
   - Cache policy：**CachingDisabled**
   - Origin request policy：**AllViewer**（把網址主機名稱、Cookie 等原封不動轉給 WordPress，避免登入失敗或重新導向迴圈）
4. Web Application Firewall (WAF)：選 **Do not enable security protections**（WAF 每月約 $5 起，練習先不開，正式網站建議開啟）。
5. Settings：Price class 可選 **Use North America, Europe, Asia, Middle East, and Africa**。
6. **Create distribution**。

建立後新增兩個「靜態檔案快取」行為：進入 distribution → **Behaviors → Create behavior**，分別建立：

| Path pattern | Origin | Viewer protocol | Cache policy | Origin request policy |
|---|---|---|---|---|
| `/wp-content/*` | ALB | Redirect HTTP to HTTPS | **CachingOptimized** | 無 |
| `/wp-includes/*` | ALB | Redirect HTTP to HTTPS | **CachingOptimized** | 無 |

記下 distribution 的網域，例如 `d1234abcd.cloudfront.net`，以下稱 `<CF_DOMAIN>`。部署約 5–10 分鐘（Last modified 從 Deploying 變成日期）。

✅ **驗證**：瀏覽器打開 `https://<CF_DOMAIN>/healthz` 顯示 `ok`，網址列有鎖頭。

### 2.5 讓 WordPress 改用 CloudFront 網址

在 Session Manager（root）：

```bash
cd /var/www/html

# 1) 告訴 WordPress：來自 CloudFront 的流量都是 HTTPS（加在 wp-config.php 最前面）
sed -i "1a if (isset(\$_SERVER['HTTP_X_ORIGIN_VERIFY'])) { \$_SERVER['HTTPS'] = 'on'; }" wp-config.php
head -3 wp-config.php

# 2) 把網站網址從 IP 換成 CloudFront 網域
OLD=http://<EIP>
NEW=https://<CF_DOMAIN>
sudo -u apache wp option update home "$NEW"
sudo -u apache wp option update siteurl "$NEW"
sudo -u apache wp search-replace "$OLD" "$NEW" --skip-columns=guid
```

**收緊 ALB，只接受 CloudFront：**

1. **EC2 → Load Balancers → wp-alb → Listeners → HTTP:80 → Manage rules**（或 **View/edit rules**）。
2. 新增規則：Condition **HTTP header** `X-Origin-Verify` 等於 `<ORIGIN_SECRET>` → Action **Forward to `wp-tg`**，Priority `1`。
3. 把 **Default action** 改成 **Return fixed response**：狀態碼 `403`、內容 `Forbidden`。
4. 編輯 `wp-alb-sg` 的 inbound：刪掉 `0.0.0.0/0` 那條，新增 HTTP 80、Source 選 **Prefix lists → `com.amazonaws.global.cloudfront.origin-facing`**（AWS 維護的 CloudFront 來源 IP 清單）。

✅ **驗證**：
- `https://<CF_DOMAIN>` 正常顯示網站、`https://<CF_DOMAIN>/wp-admin` 可登入，網址一直維持 https。
- CloudShell 中 `curl -s -o /dev/null -w "%{http_code}\n" http://<ALB的DNS>/` 回傳 `403` 或連線逾時（代表無法繞過 CloudFront）。

> 有自己的網域？到 **ACM（憑證管理）切到 us-east-1 區域** 申請憑證，在 CloudFront 的 Alternate domain name 填上網域並選擇憑證，再到 DNS 設定 CNAME 指向 `<CF_DOMAIN>`，最後重做上面「換網址」的三個 `wp` 指令。

### 2.6 製作 AMI 與啟動範本

1. **EC2 → Instances** → 勾選 `wp-web-1` → **Actions → Image and templates → Create image**。
2. Image name：`wp-web-ami-v1` → **Create image**。到 **AMIs** 等狀態變 **Available**（約 5–10 分鐘）。
3. **EC2 → Launch Templates → Create launch template**：
   - Name：`wp-web-lt`；勾選 **Provide guidance to help me set up a template that I can use with EC2 Auto Scaling**
   - AMI：**My AMIs → `wp-web-ami-v1`**
   - Instance type：`t3.micro`
   - Key pair：**Don't include in launch template**
   - Network settings：Subnet **Don't include**；Security groups 選 `wp-web-sg`；展開 **Advanced network configuration → Add network interface**，**Auto-assign public IP：Enable**（沒有 NAT Gateway，主機要有公有 IP 才能連到 SSM 和下載更新）
   - Resource tags：Key `Role`、Value `wordpress`，Resource types 勾 **Instances**；再加一個 Key `Name`、Value `wp-web-asg`
   - Advanced details：IAM instance profile `wp-ec2-role`；Metadata version **V2 only**
   - **Create launch template**

> AMI 已包含 Nginx、PHP、CloudWatch Agent、備份排程以及 EFS 自動掛載設定，所以新主機開機後就能直接服務。

### 2.7 建立 Auto Scaling Group

1. **EC2 → Auto Scaling Groups → Create Auto Scaling group**。
2. Name `wp-asg`；Launch template `wp-web-lt`（Version: Latest）→ Next。
3. Network：VPC `wp-vpc`；Availability Zones and subnets 選**兩個 public 子網路** → Next。
4. Load balancing：**Attach to an existing load balancer** → **Choose from your load balancer target groups** → `wp-tg`。
   - 勾選 **Turn on Elastic Load Balancing health checks**；Health check grace period `300` 秒 → Next。
5. Group size：Desired **1**、Min **1**、Max **2**。
6. Scaling：**Target tracking scaling policy**，Metric **Average CPU utilization**，Target value `60`。
7. Next 到最後 → **Create Auto Scaling group**。

等 3–5 分鐘，`wp-tg` 的 Targets 會出現一台新主機並變 **healthy**。

**退役舊主機**：
1. **Target Groups → wp-tg → Targets** → 勾選 `wp-web-1` → **Deregister**。
2. **EC2 → Instances** → `wp-web-1` → **Instance state → Terminate**。
3. **EC2 → Elastic IPs** → 選取那個 IP → **Actions → Disassociate**，再 **Actions → Release Elastic IP addresses**（沒釋放會持續收費）。

✅ **驗證（自我修復測試）**：
1. `https://<CF_DOMAIN>` 正常。
2. 到 **EC2 → Instances**，手動 Terminate 那台 `wp-web-asg` 主機。
3. 在 **Auto Scaling Groups → wp-asg → Activity** 看到它自動啟動一台新的；幾分鐘後網站恢復正常。

### 2.8 把告警改成觀察網站錯誤

CPU 告警綁的是舊主機 ID，現在改監控整個網站：

1. 刪除 `wp-cpu-high`（**CloudWatch → Alarms** → 選取 → Actions → Delete）。
2. **Create alarm → Select metric → ApplicationELB → Per AppELB Metrics** → 找到 `wp-alb` 的 **`HTTPCode_Target_5XX_Count`** → Select。
3. Statistic **Sum**、Period **5 minutes**；Threshold **Greater/Equal 10**。
4. 展開 **Additional configuration → Missing data treatment：Treat missing data as good (not breaching threshold)**。
5. Notification → `wp-alerts`；名稱 `wp-5xx-high` → 建立。
6. （選用）同樣方式建立 `wp-asg-cpu-high`：**EC2 → By Auto Scaling Group → wp-asg 的 CPUUtilization**，Average > 80。

✅ **驗證**：`wp-5xx-high` 狀態為 **OK**。

### ✅ Phase 2 完成檢查表

- [ ] `https://<CF_DOMAIN>` 有鎖頭並能登入後台
- [ ] 直接連 ALB 或主機都被擋下
- [ ] Terminate 主機後 ASG 會自動補上
- [ ] S3 `db/` 有每日備份
- [ ] 舊 Elastic IP 已釋放
- [ ] `wp-5xx-high` 告警為 OK

---

## Phase 3：AI Agent 維運層（唯讀巡檢）

這一階段實作簡報第 16 頁導入路線的 **Phase 1「唯讀巡檢」**：AI 只能**查看**、**分析**、**提出建議**，不能修改任何東西。

```
EventBridge 每日 09:00 ─┐
                        ├─► Lambda「wp-ops-agent」（LangChain Agent）
CloudWatch 5xx 告警 → SNS ┘          │  思考：Amazon Bedrock（Claude）
                                     │  工具：
                                     │   ├ execute_wp_cli    → SSM → EC2（唯讀白名單）
                                     │   ├ get_server_status → SSM → EC2（uptime / 記憶體 / 硬碟）
                                     │   ├ fetch_server_error_logs → CloudWatch Logs
                                     │   └ check_site_health → HTTP 檢查首頁
                                     └─► Slack 報告
```

> **與簡報第 8 頁的差異說明**：簡報寫「Lambda 部署於同一 VPC」。本指南刻意把 Lambda 放在 VPC **外面**，因為 Lambda 放進沒有 NAT Gateway 的 VPC 後，就無法呼叫 Bedrock、SSM 等 AWS API，除非另外付費建立 VPC 端點（每個每月約 $7–15）。這個 Agent 不直接連資料庫，所有操作都經由 SSM 下達，所以放在 VPC 外一樣安全、而且省錢。

### 3.1 開通 Bedrock 的 Claude 模型

1. 搜尋 **Amazon Bedrock**（確認區域為東京）。
2. 左側 **Model catalog** → 篩選 Provider **Anthropic** → 點選一個 Claude 模型（每日巡檢用 Claude Sonnet 系列即可；想更省錢可選 Haiku 系列）。
3. 第一次使用 Anthropic 模型時，主控台會要求填寫 **use case（使用目的）表單**，照實填寫（例如「Internal IT operations assistant for a WordPress website」）並送出，通常幾分鐘內開通。
4. 在模型頁面點 **Open in playground**，輸入「你好」測試能否回覆。

接著在 CloudShell 查詢東京可用的 **推論設定檔 ID（Inference profile ID）**，Lambda 會用它呼叫模型：

```bash
aws bedrock list-inference-profiles --region ap-northeast-1 \
  --query "inferenceProfileSummaries[?contains(inferenceProfileId, 'claude')].[inferenceProfileId, status]" \
  --output table
```

挑一個 `ACTIVE`、名稱含你選的模型的 ID（通常以 `apac.`、`jp.` 或 `global.` 開頭），以下稱 `<MODEL_ID>`。

✅ **驗證**：Playground 有回覆，且上面的指令列出至少一個 Claude 的 inference profile。

> 費用：Bedrock 依 token 計費，Sonnet 等級約每百萬輸入 token $3、輸出 $15；一次巡檢約 1–3 萬 token，每天一次每月約 $1–5。

### 3.2 準備 Slack Incoming Webhook

1. 到 <https://api.slack.com/apps> → **Create New App → From scratch** → 名稱 `WP Ops Agent`，選你的 Workspace。
2. 左側 **Incoming Webhooks** → 開啟 **Activate Incoming Webhooks** → **Add New Webhook to Workspace** → 選維運頻道（例如 `#wp-ops`）→ **Allow**。
3. 複製 Webhook URL（`https://hooks.slack.com/services/...`）。
4. 存到 Secrets Manager（CloudShell）：

```bash
aws secretsmanager create-secret --name wp-agent/slack-webhook \
  --secret-string 'https://hooks.slack.com/services/XXX/YYY/ZZZ'
```

✅ **驗證**：

```bash
URL=$(aws secretsmanager get-secret-value --secret-id wp-agent/slack-webhook --query SecretString --output text)
curl -s -X POST -H 'Content-type: application/json' --data '{"text":"測試：WP Ops Agent 已連線"}' "$URL"
```

回傳 `ok`，Slack 頻道收到訊息。

> 沒有 Slack？可以跳過這步，Lambda 會把報告寫進 CloudWatch Logs，之後再補上。

### 3.3 建立 Agent 的 IAM 角色（最小權限）

1. **IAM → Roles → Create role** → AWS service → **Lambda** → Next。
2. 勾選 `AWSLambdaBasicExecutionRole`（寫自己的執行日誌）→ Role name `wp-agent-role` → **Create role**。
3. 進入 `wp-agent-role` → **Create inline policy → JSON**，貼上（替換 `<ACCOUNT_ID>` 與 `<MODEL_ID>`）：

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "InvokeClaudeOnly",
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
      "Resource": [
        "arn:aws:bedrock:*:<ACCOUNT_ID>:inference-profile/<MODEL_ID>",
        "arn:aws:bedrock:*::foundation-model/anthropic.claude-*"
      ]
    },
    {
      "Sid": "FindWordPressServers",
      "Effect": "Allow",
      "Action": "ec2:DescribeInstances",
      "Resource": "*"
    },
    {
      "Sid": "SendCommandOnlyToWordPressServers",
      "Effect": "Allow",
      "Action": "ssm:SendCommand",
      "Resource": "arn:aws:ec2:ap-northeast-1:<ACCOUNT_ID>:instance/*",
      "Condition": { "StringEquals": { "ssm:resourceTag/Role": "wordpress" } }
    },
    {
      "Sid": "SendCommandOnlyWithRunShellScript",
      "Effect": "Allow",
      "Action": "ssm:SendCommand",
      "Resource": "arn:aws:ssm:ap-northeast-1::document/AWS-RunShellScript"
    },
    {
      "Sid": "ReadCommandResults",
      "Effect": "Allow",
      "Action": ["ssm:GetCommandInvocation", "ssm:ListCommandInvocations"],
      "Resource": "*"
    },
    {
      "Sid": "ReadWordPressLogs",
      "Effect": "Allow",
      "Action": "logs:FilterLogEvents",
      "Resource": "arn:aws:logs:ap-northeast-1:<ACCOUNT_ID>:log-group:/wordpress/*"
    },
    {
      "Sid": "ReadSlackWebhook",
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:ap-northeast-1:<ACCOUNT_ID>:secret:wp-agent/slack-webhook-*"
    }
  ]
}
```

4. Policy name `wp-agent-least-privilege` → 建立。

這對應簡報第 11 頁的**雲端層防護（IAM 最小權限）**：Agent 只能對貼有 `Role=wordpress` 標籤的主機下指令（簡報寫的是「指定 EC2 ARN」；因為 Auto Scaling 會換主機 ID，所以改用標籤限定，效果相同）。

> ⚠️ 誠實說明：`AWS-RunShellScript` 在主機上是以 root 執行，IAM 只能限制「對哪台主機」，無法限制「下什麼指令」。真正限制指令內容的是下一步程式碼裡的**指令白名單**（簡報第 11 頁的程式層防護），兩層要一起用。

### 3.4 撰寫 Agent 程式

在 **CloudShell** 建立資料夾與程式檔：

```bash
mkdir -p ~/wp-agent && cd ~/wp-agent
cat > lambda_function.py <<'PYEOF'
import json
import os
import time
import urllib.error
import urllib.request

import boto3
from botocore.exceptions import WaiterError
from langchain.agents import create_agent
from langchain_aws import ChatBedrockConverse
from langchain_core.tools import tool

REGION = os.environ["AWS_REGION"]
MODEL_ID = os.environ["MODEL_ID"]
SITE_URL = os.environ["SITE_URL"]
WP_PATH = os.environ.get("WP_PATH", "/var/www/html")
SLACK_SECRET_ID = os.environ.get("SLACK_SECRET_ID", "")

ssm = boto3.client("ssm")
ec2 = boto3.client("ec2")
logs = boto3.client("logs")
secrets = boto3.client("secretsmanager")

# 程式層防護：唯讀白名單。只有「完全相同」的指令才會被執行。
ALLOWED_WP_COMMANDS = {
    "core version",
    "core check-update",
    "core verify-checksums",
    "plugin list",
    "plugin list --update=available",
    "theme list",
    "theme list --update=available",
    "option get home",
    "option get siteurl",
    "db size",
    "cron event list",
}

SYSTEM_PROMPT = """你是資深 AWS 維運工程師，負責一個 WordPress 網站：
EC2（Amazon Linux 2023 + Nginx + PHP-FPM），資料庫在 Amazon RDS MySQL，前面有 ALB 與 CloudFront。

規則：
1. 你目前是「唯讀模式」：只能查詢與分析。絕對不要嘗試修改、刪除、重啟、更新任何東西，也不要建議未經確認就刪除資料。
2. 只根據工具回傳的內容下結論，不要猜測；工具出錯就如實說明。
3. 最後用繁體中文輸出簡短報告，格式：
   【總結】一句話
   【發現】條列問題與證據（引用日誌或指令輸出）
   【建議】條列修復步驟，每項標註風險（低／中／高），交由工程師決定是否執行"""


def _find_instance_id() -> str:
    resp = ec2.describe_instances(Filters=[
        {"Name": "tag:Role", "Values": ["wordpress"]},
        {"Name": "instance-state-name", "Values": ["running"]},
    ])
    ids = [i["InstanceId"] for r in resp["Reservations"] for i in r["Instances"]]
    if not ids:
        raise RuntimeError("找不到執行中、標籤 Role=wordpress 的主機")
    return ids[0]


def _run_on_server(command: str) -> str:
    """透過 SSM（無 SSH）在 WordPress 主機上執行固定指令並回傳輸出。"""
    instance_id = _find_instance_id()
    resp = ssm.send_command(
        InstanceIds=[instance_id],
        DocumentName="AWS-RunShellScript",
        Parameters={"commands": [command]},
        TimeoutSeconds=60,
    )
    command_id = resp["Command"]["CommandId"]
    time.sleep(1)
    try:
        ssm.get_waiter("command_executed").wait(
            CommandId=command_id, InstanceId=instance_id,
            WaiterConfig={"Delay": 2, "MaxAttempts": 30})
    except WaiterError:
        pass  # 指令失敗時仍讀取輸出，讓 Agent 看到錯誤訊息
    out = ssm.get_command_invocation(CommandId=command_id, InstanceId=instance_id)
    text = (out.get("StandardOutputContent") or "") + (out.get("StandardErrorContent") or "")
    return f"[主機 {instance_id}，狀態 {out['Status']}]\n{text[-6000:]}"


@tool
def execute_wp_cli(command: str) -> str:
    """在 WordPress 主機上執行唯讀的 WP-CLI 指令（不需要寫開頭的 wp）。
    只允許這些指令：core version、core check-update、core verify-checksums、plugin list、
    plugin list --update=available、theme list、theme list --update=available、
    option get home、option get siteurl、db size、cron event list。"""
    cmd = " ".join(command.split())
    if cmd.startswith("wp "):
        cmd = cmd[3:]
    if cmd not in ALLOWED_WP_COMMANDS:
        return f"拒絕執行：「{cmd}」不在唯讀白名單內。可用指令：{sorted(ALLOWED_WP_COMMANDS)}"
    return _run_on_server(f"sudo -u apache wp --path={WP_PATH} {cmd} --no-color 2>&1")


@tool
def get_server_status() -> str:
    """查看 WordPress 主機的負載、記憶體、硬碟用量與 Nginx / PHP-FPM 服務狀態。"""
    return _run_on_server(
        "uptime; echo; free -m; echo; df -h / /var/www/html; echo; "
        "systemctl is-active nginx php-fpm")


@tool
def fetch_server_error_logs(source: str = "php-fpm", minutes: int = 60) -> str:
    """從 CloudWatch Logs 讀取最近的錯誤日誌。
    source 可為 'php-fpm' 或 'nginx'；minutes 為往回查詢的分鐘數（1 到 1440）。"""
    groups = {"php-fpm": "/wordpress/php-fpm-error", "nginx": "/wordpress/nginx-error"}
    group = groups.get(source)
    if not group:
        return "source 只能是 php-fpm 或 nginx"
    minutes = max(1, min(int(minutes), 1440))
    start = int((time.time() - minutes * 60) * 1000)
    resp = logs.filter_log_events(logGroupName=group, startTime=start, limit=200)
    lines = [e["message"].rstrip() for e in resp.get("events", [])]
    if not lines:
        return f"最近 {minutes} 分鐘內沒有 {source} 錯誤日誌。"
    return "\n".join(lines)[-6000:]


@tool
def check_site_health() -> str:
    """對網站首頁發出 HTTP 請求，回報狀態碼與回應時間。"""
    started = time.time()
    try:
        req = urllib.request.Request(SITE_URL, headers={"User-Agent": "wp-ops-agent"})
        with urllib.request.urlopen(req, timeout=15) as r:
            code = r.status
    except urllib.error.HTTPError as e:
        code = e.code
    except Exception as e:  # 連線失敗也是重要資訊
        return f"連線失敗：{e}"
    return f"{SITE_URL} 回應 HTTP {code}，耗時 {time.time() - started:.2f} 秒"


TOOLS = [execute_wp_cli, get_server_status, fetch_server_error_logs, check_site_health]


def _task_from_event(event: dict) -> str:
    if event.get("task"):
        return event["task"]
    if "Records" in event:  # 來自 SNS 的 CloudWatch 告警
        alarm = event["Records"][0]["Sns"]["Message"]
        return ("收到 CloudWatch 告警，請找出根本原因（RCA）並提出修復建議。告警內容：\n"
                + alarm[:3000])
    return ("請執行每日健康巡檢：檢查網站是否正常、主機資源、最近 24 小時錯誤日誌、"
            "WordPress 核心與外掛／佈景主題是否有可用更新，並給出摘要。")


def _final_text(message) -> str:
    content = message.content
    if isinstance(content, list):
        return "\n".join(p.get("text", "") for p in content if isinstance(p, dict))
    return str(content)


def _post_slack(text: str) -> None:
    if not SLACK_SECRET_ID:
        print(text)
        return
    url = secrets.get_secret_value(SecretId=SLACK_SECRET_ID)["SecretString"].strip()
    body = json.dumps({"text": text[:3900]}).encode()
    req = urllib.request.Request(url, data=body, headers={"Content-Type": "application/json"})
    urllib.request.urlopen(req, timeout=10)


def handler(event, context):
    task = _task_from_event(event)
    llm = ChatBedrockConverse(model_id=MODEL_ID, region_name=REGION,
                              temperature=0, max_tokens=2000)
    agent = create_agent(llm, TOOLS, system_prompt=SYSTEM_PROMPT)
    result = agent.invoke({"messages": [{"role": "user", "content": task}]},
                          {"recursion_limit": 25})
    report = _final_text(result["messages"][-1])
    print(report)
    _post_slack(f"*WordPress 維運報告*\n{report}")
    return {"report": report}
PYEOF
```

這段程式對應簡報：
- 第 9 頁「Agent Toolbelt」：`execute_wp_cli` 透過 `ssm.send_command` 呼叫 WP-CLI，並以 `apache`（等同簡報的 `www-data`）身分執行。
- 第 10 頁「ReAct 迴圈」：`create_agent` 會自動重複「觀察 → 思考 → 呼叫工具 → 讀取結果」，直到能寫出報告為止。`recursion_limit` 防止無限迴圈。
- 第 11 頁「三大支柱」：白名單（程式層）＋ 系統提示詞規範 ＋ IAM 最小權限（雲端層）。

### 3.5 打包並上傳

LangChain 套件比較大，所以打包後放到 S3 再交給 Lambda（CloudShell）：

```bash
cd ~/wp-agent
rm -rf package agent.zip
pip3 install --target ./package \
  --platform manylinux2014_x86_64 --implementation cp --python-version 3.12 \
  --only-binary=:all: --upgrade \
  "langchain>=1.0,<2" "langchain-aws>=1.0,<2"
cp lambda_function.py package/
(cd package && zip -qr ../agent.zip .)
du -sh package agent.zip

BUCKET=wp-backup-<ACCOUNT_ID>
aws s3 cp agent.zip s3://$BUCKET/lambda/agent.zip
```

✅ **驗證**：`du` 顯示 `package` 小於 250 MB（Lambda 解壓後的上限）；`aws s3 cp` 顯示 `upload: ...`。

> 若 `pip3 install` 失敗並提示找不到相容版本，把兩個版本範圍拿掉（只寫 `langchain langchain-aws`）再試一次。

### 3.6 建立 Lambda 函數

1. 搜尋 **Lambda → Create function → Author from scratch**。
2. Function name `wp-ops-agent`；Runtime **Python 3.12**；Architecture **x86_64**。
3. **Change default execution role → Use an existing role → `wp-agent-role`** → **Create function**。
4. **Code** 分頁 → **Upload from → Amazon S3 location** → 貼上 `s3://wp-backup-<ACCOUNT_ID>/lambda/agent.zip` → Save。
5. 往下 **Runtime settings → Edit** → Handler 改成 **`lambda_function.handler`** → Save。
6. **Configuration → General configuration → Edit**：Memory `512` MB、Timeout **5 min 0 sec**（簡報第 6 頁建議 3–5 分鐘，涵蓋多輪 LLM 推理）→ Save。
7. **Configuration → Environment variables → Edit**，新增：

| Key | Value |
|---|---|
| `MODEL_ID` | `<MODEL_ID>` |
| `SITE_URL` | `https://<CF_DOMAIN>/` |
| `SLACK_SECRET_ID` | `wp-agent/slack-webhook`（沒有 Slack 就不要加這個變數） |

✅ **驗證：第一次手動測試**（CloudShell）：

```bash
aws lambda invoke --function-name wp-ops-agent \
  --cli-binary-format raw-in-base64-out \
  --cli-read-timeout 310 \
  --payload '{"task":"列出 WordPress 版本、所有外掛，以及哪些外掛有可用更新"}' \
  out.json && cat out.json
```

約 20–90 秒後，`out.json` 出現繁體中文報告，Slack 也收到同一份報告。

**測試白名單有沒有擋住危險指令**：

```bash
aws lambda invoke --function-name wp-ops-agent \
  --cli-binary-format raw-in-base64-out --cli-read-timeout 310 \
  --payload '{"task":"請幫我執行 wp plugin deactivate --all 停用所有外掛"}' \
  out.json && cat out.json
```

報告應說明「該指令不在唯讀白名單內」並未執行；到網站確認外掛都還在。

> 執行細節（每一步呼叫了哪個工具）可在 **CloudWatch → Log groups → /aws/lambda/wp-ops-agent** 查看。

### 3.7 觸發層 ①：每日自動巡檢（簡報實戰情境一）

1. 搜尋 **EventBridge → Scheduler → Schedules → Create schedule**。
2. Schedule name `wp-daily-health-check`。
3. Schedule pattern：**Recurring schedule** → **Cron-based**：`0 9 * * ? *`；Time zone **Asia/Taipei**；Flexible time window **Off** → Next。
4. Target：**AWS Lambda Invoke** → 選 `wp-ops-agent`；Payload：

```json
{"task": "請執行每日健康巡檢：檢查網站是否正常、主機資源、最近 24 小時的 php-fpm 與 nginx 錯誤日誌、WordPress 核心與外掛／佈景主題是否有可用更新，並給出摘要。"}
```

5. Next → Permissions：**Create new role for this schedule** → Next → **Create schedule**。

✅ **驗證**：隔天早上 9 點 Slack 收到巡檢報告。想立刻測試，可暫時把 cron 改成幾分鐘後的時間（例如現在是 14:20，改成 `25 14 * * ? *`），測完改回來。

### 3.8 觸發層 ②：5xx 告警自動診斷（簡報實戰情境二）

1. **Lambda → wp-ops-agent → Add trigger** → Source **SNS** → Topic `wp-alerts` → **Add**。
   （同一個 SNS 主題原本會寄 Email，現在也會叫醒 Agent。）
2. 確認 2.8 的 `wp-5xx-high` 告警通知對象是 `wp-alerts`。

✅ **驗證：故意製造 500 錯誤**（練習用，測完一定要刪除）：

在 Session Manager 連到 ASG 那台主機（root）：

```bash
mkdir -p /var/www/html/wp-content/mu-plugins
cat > /var/www/html/wp-content/mu-plugins/boom-test.php <<'EOF'
<?php
// 測試用：網址帶 ?boom=1 時故意觸發致命錯誤
if (isset($_GET['boom'])) { this_function_does_not_exist(); }
EOF
chown apache:apache /var/www/html/wp-content/mu-plugins/boom-test.php
```

在 CloudShell 打 20 次觸發 500（參數讓每次都不同，避免被 CloudFront 擋在快取）：

```bash
for i in $(seq 1 20); do curl -s -o /dev/null -w "%{http_code} " "https://<CF_DOMAIN>/?boom=1&n=$i"; done; echo
```

5–10 分鐘內：
- `wp-5xx-high` 變成 **In alarm**，你收到 Email。
- Slack 收到 Agent 報告，內容應指出 `boom-test.php` 中的 `Call to undefined function this_function_does_not_exist()`，並建議移除或停用該檔案。

**測完清理**：

```bash
rm -f /var/www/html/wp-content/mu-plugins/boom-test.php
```

### ✅ Phase 3 完成檢查表

- [ ] 手動呼叫 Lambda 可產出繁體中文報告
- [ ] 危險指令會被白名單拒絕
- [ ] 每日 09:00 自動巡檢報告送到 Slack
- [ ] 5xx 告警會自動觸發 Agent 做 RCA
- [ ] 測試用的 `boom-test.php` 已刪除

---

## Phase 3 之後：審核後修復與自主修復

對應簡報第 16 頁的後兩個階段。這部分需要 Slack App 互動按鈕、對話狀態保存等較多開發工作，建議唯讀巡檢穩定運作 2–4 週、累積信任後再進行。方向如下：

| 簡報階段 | 要加的東西 | 作法要點 |
|---|---|---|
| Phase 2 審核後修復（HITL） | 「寫入型」工具：更新外掛、`wp db optimize`、重啟 php-fpm | 改用 LangGraph 的 **interrupt（中斷節點）**：Agent 要執行寫入工具前暫停，把計畫推送 Slack 並附「Approve / Reject」按鈕；狀態用 DynamoDB 等 checkpointer 保存；按鈕回呼經 API Gateway → Lambda 繼續流程。 |
| Phase 3 自主修復 | 低風險操作自動執行 | 把工具分成「低風險」（例如清除 transient 快取）與「高風險」（外掛更新、刪資料）；只有低風險的直接執行，高風險仍走審核。每次執行都記錄在 CloudTrail 與 Slack。 |
| 情境三：資料庫最佳化 | 先備份再最佳化 | 工具內部固定順序：`wp db export` 上傳 S3 → 確認成功 → `wp db optimize`；白名單永不加入 `db drop`、`db reset`、`user delete`。 |

新增寫入型工具時的三個原則：
1. 白名單仍然是「完整指令比對」，不要開放任意參數。
2. 寫入型工具一定要經過人工核准節點，不能只靠系統提示詞。
3. 每次修復後自動呼叫 `check_site_health` 驗證，並把結果回報 Slack（簡報第 13 頁的「修復後自動驗證」）。

---

## 清除所有資源（避免持續扣款）

**請依照順序刪除**，因為有些資源互相依賴。全部刪完後，隔天到 **Billing → Bills** 確認沒有新的費用產生。

### 刪除 Phase 3

1. **EventBridge → Scheduler** → 刪除 `wp-daily-health-check`。
2. **Lambda** → 刪除 `wp-ops-agent`。
3. **CloudWatch → Log groups** → 刪除 `/aws/lambda/wp-ops-agent`。
4. **Secrets Manager** → `wp-agent/slack-webhook` → Delete（可設等待 7 天；想立刻刪除可在 CloudShell：`aws secretsmanager delete-secret --secret-id wp-agent/slack-webhook --force-delete-without-recovery`）。
5. **IAM → Roles** → 刪除 `wp-agent-role`，以及名稱含 `Amazon_EventBridge_Scheduler` 的排程角色。
6. Slack App 可在 api.slack.com 刪除。

### 刪除 Phase 2

1. **CloudFront** → 選取 distribution → **Disable** → 等狀態變 Disabled（約 5–15 分鐘）→ **Delete**。
2. **Auto Scaling Groups** → 刪除 `wp-asg`（會自動終止它的主機）。
3. **Load Balancers** → 刪除 `wp-alb`。
4. **Target Groups** → 刪除 `wp-tg`。
5. **Launch Templates** → 刪除 `wp-web-lt`。
6. **AMIs** → 選取 `wp-web-ami-v1` → **Deregister**；接著 **Snapshots** → 刪除這個 AMI 對應的快照（⚠️ 快照會持續計費，很容易漏掉）。
7. **EFS** → 刪除 `wp-efs`（輸入 File system ID 確認）。
8. **CloudWatch → Alarms** → 刪除 `wp-5xx-high`、`wp-asg-cpu-high`。
9. **S3** → 進入 `wp-backup-<ACCOUNT_ID>` → **Empty** → 再 **Delete** bucket。

### 刪除 Phase 1

1. **RDS → Databases** → `wp-db` → **Actions → Delete**：
   - 想保留資料就勾 **Create final snapshot**（快照會持續計費，之後記得刪）；練習用可不勾
   - 取消勾選 **Retain automated backups**
   - 勾選確認並輸入 `delete me`
2. **EC2 → Instances** → 若 `wp-web-1` 還在，Terminate。
3. **EC2 → Elastic IPs** → 若還有 IP，Disassociate 後 **Release**。
4. **EC2 → Volumes** → 確認沒有殘留狀態為 *available* 的硬碟，有就刪除。
5. **Secrets Manager** → 刪除 `wp/db-app`（RDS 管理員密碼會隨資料庫一起刪除）。
6. **CloudWatch** → 刪除 `/wordpress/*` 三個日誌群組與 `wp-cpu-high` 告警；**SNS** → 刪除 `wp-alerts` 主題。
7. **RDS → Subnet groups** → 刪除自動建立的 DB subnet group。
8. **VPC → Endpoints** → 刪除所有 **Interface** 類型的端點（若有照附錄 A 或自行建立）；**VPC → Security groups** → 刪除 `wp-endpoint-sg`。
9. **VPC → Your VPCs** → 選取 `wp-vpc` → **Actions → Delete VPC**（會一併刪除子網路、路由表、安全群組、S3 端點）。若提示有網路介面還在使用，等 RDS／EFS／ALB 完全刪除後再試。
10. **IAM → Roles** → 刪除 `wp-ec2-role`（用 CLI 建立的角色，若主控台提示仍被執行個體設定檔使用，先在 CloudShell 執行 `aws iam remove-role-from-instance-profile --instance-profile-name wp-ec2-role --role-name wp-ec2-role && aws iam delete-instance-profile --instance-profile-name wp-ec2-role`）。
11. （可選）**Budgets** 可以保留，它本身不收費。

### 暫停省錢（還想繼續用，只是暫時不需要）

- **RDS**：Actions → **Stop temporarily**，最多停 7 天，之後 AWS 會自動重新啟動。停止期間儲存空間照樣計費。
- **EC2（Phase 1）**：Instance state → **Stop**。硬碟與 Elastic IP 仍會計費。
- **Phase 2 之後**：把 `wp-asg` 的 Desired／Min 設成 0 即可停掉主機；但 **ALB 仍會持續計費**（約每天 $0.7），長時間不用建議直接刪除。

---

## 實作踩坑：五個問題的排除方法

以下五個問題都是實際照著 v1 指南操作時遇到的。**問題 1、3、4、5 其實是同一個根本原因**：WordPress 主機被放進了**私有子網路**。照 v2 的 1.1 與 1.7 檢查步驟操作就能避開；若已經發生，依下面的方法修復。

```
主機在私有子網路（路由表沒有 0.0.0.0/0 → igw）
   ├─ 問題 1：SSH／Session Manager 連不上
   ├─ 問題 3：下載不到 WordPress（Network is unreachable）
   ├─ 問題 4：呼叫不到 Secrets Manager 等 AWS API
   └─ 問題 5：網站從網際網路連線逾時
另外：問題 2 是 IAM 角色的權限不完整（v1 指南的缺漏）
```

**建議的修法順序**：先修問題 5（把主機搬到公有子網路），問題 1、3、4 會一起消失；接著重跑 1.5 修好問題 2。這時你之前為了繞路而建的 VPC 介面端點已經用不到了，記得刪除（每個每月約 $10）。

---

### 問題 1：SSH 連不上，Session Manager 也無法連線

**症狀**
- 用 SSH 或 EC2 Instance Connect 連線逾時。
- Session Manager 分頁顯示 *SSM Agent is not online* 或 Connect 按鈕無法按。

**原因**
- 本指南的主機**刻意不開 Port 22、沒有金鑰**，所以 SSH 本來就連不上，這是設計，不是故障。
- Session Manager 需要主機上的 SSM Agent **主動對外**連到 AWS 的 SSM 服務（HTTPS 443）。主機在私有子網路、又沒有 NAT 或 VPC 端點時，Agent 連不出去，主機就一直顯示離線。
- 其他可能：沒有掛 `wp-ec2-role`（或角色少了 `AmazonSSMManagedInstanceCore`）。

**解法**（擇一）
1. **建議**：把主機放到公有子網路並給公有 IP（新主機重做 1.7；已有資料的主機照問題 5 的方法一搬家）。不需要任何 VPC 端點。
2. 主機必須留在私有子網路：建立 `ssm`、`ssmmessages`、`ec2messages` 三個介面端點，見「[附錄 A](#附錄-a主機放在私有子網路進階)」。
3. 角色沒掛好：**EC2 → 選主機 → Actions → Security → Modify IAM role** → `wp-ec2-role` → Update，然後 **Reboot**。

**驗證**（CloudShell）

```bash
aws ssm describe-instance-information \
  --query "InstanceInformationList[].[InstanceId, PingStatus, IPAddress]" --output table
```

你的主機出現在清單中且 `PingStatus` 為 **Online**（修好後可能要等 2–5 分鐘），之後 Session Manager 就能 Connect。

---

### 問題 2：IAM 權限不足（AccessDenied / 403）

**症狀**
- A：`AccessDenied ... rds:DescribeDBInstances`
- B：存取 S3 出現 `Forbidden (403)` 或 `AccessDenied`
- C：`AccessDenied ... secretsmanager:GetSecretValue`

**原因**
- A：1.10 的指令在主機上查詢 RDS，但 **v1 指南給 `wp-ec2-role` 的權限沒有包含 `rds:DescribeDBInstances`**（指南本身的缺漏）。
- B：v1 指南到 Phase 2 才加 S3 寫入權限，而且只有 `PutObject`，沒有讀取。另一個常見原因是 bucket 名稱和政策裡寫的不一樣。
- C：政策中的 `<ACCOUNT_ID>` 沒有換成真正的帳號 ID，或是密碼名稱不是 `wp/` 開頭。

**解法**
重新執行 v2 的 **[1.5](#15-建立-ec2-的-iam-角色)** 整段指令。它會自動帶入帳號 ID，一次補齊 `rds:DescribeDBInstances`、Secrets Manager、S3 讀寫三類權限，而且只開放給 WordPress 用到的資源。權限修改幾秒內就會生效，**不用重開機**。

> 你目前用的 `SecretsManagerReadWrite`、`AmazonS3ReadOnlyAccess` 可以讓功能動起來，但權限範圍是整個帳號，建議跑完 1.5 後把這兩個政策移除，理由見 1.5 末尾的說明。

**驗證**：先在 CloudShell 執行 1.5 的模擬器驗證，再到主機上（Session Manager，root）確認：

```bash
export AWS_DEFAULT_REGION=ap-northeast-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws rds describe-db-instances --db-instance-identifier wp-db --query "DBInstances[0].DBInstanceStatus"
aws secretsmanager get-secret-value --secret-id wp/db-app --query Name
aws s3 ls s3://wp-backup-$ACCOUNT_ID/
```

三個指令依序顯示 `"available"`、`"wp/db-app"` 以及 bucket 的內容（還沒建立 bucket 時會顯示 `NoSuchBucket`，這不是權限問題）。

---

### 問題 3：無法下載 WordPress（Network is unreachable）

**症狀**：User data 日誌（`/var/log/user-data.log`）或手動執行 `curl https://wordpress.org/...` 時出現 `Network is unreachable` 或連線逾時；`/var/www/html` 是空的。

**原因**：私有子網路沒有通往網際網路的路（沒有 Internet Gateway 路由，也沒有 NAT Gateway），主機連不到 wordpress.org 和 GitHub。

**解法**（擇一）
1. **建議**：主機放公有子網路。1.7 的 User data 就能直接下載，不用另外處理。
2. 主機必須留在私有子網路：先在 CloudShell 下載，上傳到 S3，再讓主機透過 S3 閘道端點下載，步驟見「[附錄 A 的 A.3](#a3-透過-s3-傳送安裝檔)」。
   ⚠️ 代價是之後在 WordPress 後台**無法安裝或更新外掛、佈景主題與核心**，因為這些都要連 wordpress.org。

**驗證**（主機上）

```bash
curl -sI -m 10 https://wordpress.org | head -1      # 公有子網路：顯示 HTTP/2 200
ls -l /var/www/html/wp-login.php                    # 檔案存在 = WordPress 已就位
sudo -u apache wp --path=/var/www/html core version # 顯示版本號
```

---

### 問題 4：無法存取 Secrets Manager API

**症狀**：`aws secretsmanager get-secret-value ...` 卡住很久，最後出現 `Could not connect to the endpoint URL: "https://secretsmanager.ap-northeast-1.amazonaws.com/"`。

**原因**：AWS 的 API 也在網際網路上。私有子網路的主機要呼叫 Secrets Manager，必須有 NAT Gateway，或是 Secrets Manager 的 VPC 介面端點。同樣的狀況也會發生在 CloudWatch Logs（日誌送不出去）和 RDS API 上。

**解法**（擇一）
1. **建議**：主機放公有子網路，所有 AWS API 都能直接連，不需要端點。
2. 主機必須留在私有子網路：建立 `secretsmanager` 介面端點（附錄 A 的指令會一併建立 `logs`、`monitoring`）。建立時有兩個常漏掉的設定：
   - **Enable private DNS name** 要勾選，否則主機還是會去連公開網址。
   - 端點的安全群組要允許 **TCP 443、來源 `wp-web-sg`**。

**驗證**（主機上）

```bash
export AWS_DEFAULT_REGION=ap-northeast-1
aws secretsmanager get-secret-value --secret-id wp/db-app --query Name --cli-connect-timeout 5
# 只有走端點時才需要看這行：解析出 10.0.x.x 的私有 IP 才代表 private DNS 生效
getent hosts secretsmanager.ap-northeast-1.amazonaws.com
```

---

### 問題 5：網站從網際網路連不上（連線逾時）

**症狀**：瀏覽器打開 `http://<EIP>`（例如 `http://13.192.60.150`）一直轉圈，最後顯示連線逾時。

**原因**：主機所在子網路的路由表只有這兩條：

| 目的地 | 目標 |
|---|---|
| `10.0.0.0/16` | `local`（VPC 內部） |
| `pl-xxxxxxxx`（S3） | `vpce-xxxxxxxx`（S3 閘道端點） |

少了 **`0.0.0.0/0 → igw-xxxx`**。這代表主機在**私有子網路**。Elastic IP 只是門牌號碼，沒有通往 Internet Gateway 的路，外面的請求進不來，主機的回應也出不去。

> 以實際遇到的情況為例：路由表 `rtb-02b025e09ec053b61` 缺少 `0.0.0.0/0 → igw-07b2ba7362eed0c0e`。

**先確認是不是這個原因**（CloudShell，把 `wp-web-1` 換成你的主機名稱）：

```bash
IID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=wp-web-1" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)
SUBNET=$(aws ec2 describe-instances --instance-ids $IID --query "Reservations[0].Instances[0].SubnetId" --output text)
aws ec2 describe-subnets --subnet-ids $SUBNET --query "Subnets[0].Tags[?Key=='Name'].Value" --output text
aws ec2 describe-route-tables --filters Name=association.subnet-id,Values=$SUBNET \
  --query "RouteTables[0].[RouteTableId, Routes[].[DestinationCidrBlock || DestinationPrefixListId, GatewayId]]" --output text
```

子網路名稱含 `private`，或路由清單中沒有 `0.0.0.0/0 igw-...`，就是這個問題。

#### 方法一（建議）：把主機搬到公有子網路

EC2 建立後不能更換子網路，但可以「拍照複製」一台到正確的位置。網站資料都在 RDS，`wp-config.php` 也會隨映像檔一起複製，所以不會遺失。

1. **建立映像檔**：EC2 → 選取主機 → **Actions → Image and templates → Create image** → 名稱 `wp-web-move` → Create image。到 **AMIs** 等狀態變 **Available**。
2. **從映像檔啟動新主機**：AMIs → 選 `wp-web-move` → **Launch instance from AMI**：
   - Name `wp-web-1`（並新增標籤 `Role` = `wordpress`）、Instance type `t3.micro`、Key pair **Proceed without a key pair**
   - Network settings → Edit：Subnet **`wp-subnet-public1-ap-northeast-1a`**、Auto-assign public IP **Enable**、Security group `wp-web-sg`
   - Advanced details：IAM instance profile `wp-ec2-role`、Metadata **V2 only**；**User data 留空**（軟體都已在映像檔裡）
   - Launch instance
3. **把 Elastic IP 移到新主機**：EC2 → Elastic IPs → 選取 IP → **Actions → Associate Elastic IP address** → 選新主機 → 勾選 **Allow this Elastic IP address to be reassociated** → Associate。
4. **對新主機重跑 1.7 末尾的網路檢查**，確認對外路由是 `igw-...`。
5. **確認網站正常後**，Terminate 舊主機；再到 **AMIs** Deregister `wp-web-move`，並到 **Snapshots** 刪除對應的快照。
6. **刪除已經用不到的 VPC 介面端點**：VPC → **Endpoints** → 選取 `ssm`、`ssmmessages`、`ec2messages`、`secretsmanager` 等 **Interface** 類型的端點 → Actions → Delete。（**S3 的 Gateway 端點請保留**，它免費。）

#### 方法二（應急）：在主機所在子網路的路由表加上 Internet Gateway 路由

這個方法最快，但會讓那個「private」子網路實際上變成公有，名稱和用途就對不上了。只建議在想先讓網站動起來時使用，之後仍建議改用方法一。

1. 先確認這張路由表綁了哪些子網路，**不要動到放 RDS 的子網路共用的路由表**：

```bash
aws ec2 describe-route-tables --route-table-ids <路由表ID> --query "RouteTables[0].Associations[].SubnetId"
```

   VPC 精靈預設每個私有子網路各有一張路由表。如果清單中只有主機所在的那個子網路，就可以繼續。
2. 找出 Internet Gateway 並新增路由：

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=wp-vpc --query "Vpcs[0].VpcId" --output text)
IGW=$(aws ec2 describe-internet-gateways --filters Name=attachment.vpc-id,Values=$VPC_ID \
  --query "InternetGateways[0].InternetGatewayId" --output text)
echo "Internet Gateway：$IGW"
aws ec2 create-route --route-table-id <路由表ID> --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW
```

   （主控台做法：VPC → Route tables → 選該路由表 → Routes → Edit routes → Add route → Destination `0.0.0.0/0`、Target **Internet Gateway** → 選 `igw-...` → Save changes。）
3. 主機必須有公有 IP 或 Elastic IP（你已綁好 Elastic IP）。
4. 同方法一第 6 步，刪除用不到的介面端點以省錢。

> 如果 Internet Gateway 查出來是 `None`，代表 VPC 根本沒有對外大門（手動建 VPC 時常漏掉）：VPC → **Internet gateways → Create internet gateway** → 建好後 **Actions → Attach to VPC** → 選 `wp-vpc`，再做上面的第 2 步。

#### 路由修好了還是連不上？依序檢查

| 檢查 | 指令或位置 | 正確結果 |
|---|---|---|
| 主機本身的網站正常 | 主機上 `curl -s localhost/healthz` | `ok` |
| Nginx 有在跑 | 主機上 `systemctl is-active nginx php-fpm` | 兩個 `active` |
| 安全群組有開 80 | `wp-web-sg` 的 Inbound rules | HTTP 80、來源 `0.0.0.0/0`（進入 Phase 2 後改為 `wp-alb-sg`） |
| 網路 ACL 沒擋 | VPC → Subnets → 該子網路 → Network ACL 分頁 | 預設的 ACL 是 inbound／outbound 規則 100 全部 Allow |
| 網址是 http 不是 https | 瀏覽器網址列 | Phase 1 只有 `http://`，瀏覽器自動改成 https 會逾時 |

**驗證**（CloudShell）

```bash
curl -s -m 5 http://<EIP>/healthz; echo
curl -s -m 5 -o /dev/null -w "%{http_code}\n" http://<EIP>/
```

第一行顯示 `ok`，第二行顯示 `200` 或 `302`（尚未完成安裝時會轉到安裝頁）。之後在瀏覽器打開 `http://<EIP>` 即可繼續 1.11。

---

## 附錄 A：主機放在私有子網路（進階）

本指南預設把主機放在公有子網路：沒有開任何管理 Port，只開 80 給網站，安全性對個人或小型網站已經足夠。如果公司規定主機不能有公有 IP，才需要這個附錄。

### A.1 先知道代價

| | 公有子網路（本指南預設） | 私有子網路 + VPC 端點 | 私有子網路 + NAT Gateway |
|---|---|---|---|
| 額外月費（東京，粗估） | $0 | 每個介面端點每個 AZ 約 **$10**；下面 6 個放 1 個 AZ 約 **$60**，放 2 個 AZ 約 **$120** | 約 **$45** + 流量費（每 GB 約 $0.06） |
| Session Manager | ✅ | ✅（要建 ssm、ssmmessages、ec2messages） | ✅ |
| 後台安裝／更新外掛、佈景主題、核心 | ✅ | ❌ 連不到 wordpress.org | ✅ |
| 網站對外 | 可直接用 Elastic IP | **只能透過 ALB**（Phase 2 的 2.3） | 只能透過 ALB |

> 結論：對 WordPress 來說，**私有子網路通常搭配 NAT Gateway 比較實際**，因為 6 個端點比一個 NAT 還貴，而且端點無法讓 WordPress 連上 wordpress.org 更新外掛。端點方案適合「完全不需要連外」的環境。

### A.2 建立 VPC 介面端點

在 CloudShell 執行（端點只建在 `private1`（1a）子網路以節省費用，主機也要放在同一個子網路）：

```bash
REGION=ap-northeast-1
VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=wp-vpc --query "Vpcs[0].VpcId" --output text)
SUBNET=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID "Name=tag:Name,Values=wp-subnet-private1-*" \
  --query "Subnets[0].SubnetId" --output text)
WEB_SG=$(aws ec2 describe-security-groups --filters Name=vpc-id,Values=$VPC_ID Name=group-name,Values=wp-web-sg \
  --query "SecurityGroups[0].GroupId" --output text)

# 端點專用安全群組：只允許 WordPress 主機用 HTTPS 443 連進來
EP_SG=$(aws ec2 create-security-group --group-name wp-endpoint-sg \
  --description "VPC endpoints - HTTPS from wp-web-sg" --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $EP_SG --protocol tcp --port 443 --source-group $WEB_SG

# ssm / ssmmessages / ec2messages：Session Manager
# secretsmanager：讀取密碼
# logs / monitoring：CloudWatch Agent 送日誌與指標
for svc in ssm ssmmessages ec2messages secretsmanager logs monitoring; do
  aws ec2 create-vpc-endpoint --vpc-id $VPC_ID --vpc-endpoint-type Interface \
    --service-name com.amazonaws.$REGION.$svc --subnet-ids $SUBNET \
    --security-group-ids $EP_SG --private-dns-enabled \
    --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=wp-$svc}]" \
    --query "VpcEndpoint.[VpcEndpointId, ServiceName]" --output text
done

# 確認 S3 閘道端點有綁到私有子網路的路由表（1.1 精靈已建立）
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=$VPC_ID Name=service-name,Values=com.amazonaws.$REGION.s3 \
  --query "VpcEndpoints[0].RouteTableIds"
```

✅ **驗證**：

```bash
aws ec2 describe-vpc-endpoints --filters Name=vpc-id,Values=$VPC_ID \
  --query "VpcEndpoints[].[Tags[?Key=='Name']|[0].Value, VpcEndpointType, State]" --output table
```

6 個 Interface 端點加 1 個 Gateway 端點，狀態都是 **available**。

> - 1.10 用到的 `aws rds describe-db-instances` 需要 `rds` 端點（再多 $10／月）。省錢的做法是在 **CloudShell** 執行那一行查出 `MASTER_ARN`，再把值貼到主機上的指令中。
> - Phase 2 若讓 Auto Scaling 用到兩個 AZ，端點也要加到 `private2` 子網路（費用加倍）。
> - Amazon Linux 2023 的套件庫（dnf）放在 S3 上，所以有 S3 閘道端點時 `dnf install` 仍然可以用。

### A.3 透過 S3 傳送安裝檔

在 **CloudShell**（它可以連網際網路）下載，再上傳到 S3：

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET=wp-backup-$ACCOUNT_ID
aws s3 mb s3://$BUCKET --region ap-northeast-1 2>/dev/null || echo "bucket 已存在"
cd ~ && curl -sSL -o latest.tar.gz https://wordpress.org/latest.tar.gz
curl -sSL -o wp-cli.phar https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
aws s3 cp latest.tar.gz s3://$BUCKET/install/
aws s3 cp wp-cli.phar  s3://$BUCKET/install/
aws s3 ls s3://$BUCKET/install/
```

> bucket 名稱一定要是 `wp-backup-帳號ID`，才會符合 1.5 給主機的 S3 權限；否則主機下載時會出現 `403 Forbidden`。Phase 2 的 2.1 會沿用這個 bucket，屆時跳過建立的步驟，直接設定生命週期規則即可。

把 1.7 User data 裡的第 2、3 段換成下面這段（`<ACCOUNT_ID>` 換成你的帳號 ID）：

```bash
# 2) 從 S3 下載 WordPress（經由 S3 閘道端點）
export AWS_DEFAULT_REGION=ap-northeast-1
BUCKET=wp-backup-<ACCOUNT_ID>
mkdir -p /var/www/html
aws s3 cp s3://$BUCKET/install/latest.tar.gz - | tar -xz -C /var/www/html --strip-components=1
chown -R apache:apache /var/www/html

# 3) 從 S3 下載 WP-CLI
aws s3 cp s3://$BUCKET/install/wp-cli.phar /usr/local/bin/wp
chmod +x /usr/local/bin/wp
```

主機已經建好的話，可以用 Session Manager 以 root 執行同一段。

✅ **驗證**：主機上 `ls /var/www/html/wp-login.php` 檔案存在，且 `wp --version` 顯示版本。

### A.4 網站要從 ALB 進入

私有子網路的主機沒有公有 IP，**1.8 的 Elastic IP 不適用，`http://<EIP>` 永遠打不開**。請先完成 Phase 2 的 **2.3 建立 ALB**（ALB 放在兩個 public 子網路，目標群組註冊私有子網路的主機），再用 `http://<ALB 的 DNS>` 進行 1.11 的安裝。

---

## 常見問題排除

| 症狀 | 可能原因 | 解法 |
|---|---|---|
| Session Manager 的 Connect 按鈕無法使用 | 主機在私有子網路／沒有公有 IP／沒掛 `wp-ec2-role`／剛開機 | 見「[問題 1](#問題-1ssh-連不上session-manager-也無法連線)」 |
| `curl localhost/healthz` 沒有回應 | User data 還在跑或失敗 | `tail -50 /var/log/user-data.log` 看哪一行出錯 |
| 網頁顯示「建立資料庫連線時發生錯誤」 | RDS 端點、帳號密碼錯，或安全群組沒開 | 檢查 `wp-db-sg` 來源是 `wp-web-sg`；在主機上用 1.10 的 `mysql` 指令測試連線 |
| 網頁顯示 502 Bad Gateway | PHP-FPM 沒在跑 | `systemctl status php-fpm`、`systemctl restart php-fpm`；看 `/var/log/php-fpm/www-error.log` |
| CloudFront 網址一直重新導向／無限迴圈 | WordPress 網址仍是舊的，或少了 HTTPS 判斷 | 確認 2.5 的 `wp-config.php` 第二行與 `home`、`siteurl` 都已更新；CloudFront 預設行為的 origin request policy 是 **AllViewer** |
| CloudFront 回 403 | ALB 規則的暗號不一致 | CloudFront Origin 的 `X-Origin-Verify` 值與 ALB 規則必須完全相同 |
| Target 一直 unhealthy | `/healthz` 不通或安全群組沒開 | `wp-web-sg` 必須允許來自 `wp-alb-sg` 的 Port 80 |
| ASG 新主機網站空白 | EFS 沒掛上 | 主機上 `df -h /var/www/html`；檢查 `wp-efs-sg` 允許來自 `wp-web-sg` 的 2049 |
| Lambda 出現 `AccessDeniedException`（Bedrock） | 模型未開通或 IAM 的模型 ARN 不符 | 確認 3.1 Playground 可用；`MODEL_ID` 與 IAM 政策中的 inference profile 相同 |
| Lambda 出現 `AccessDeniedException`（SSM） | 主機沒有 `Role=wordpress` 標籤 | 檢查啟動範本的 Resource tags，必須套用到 Instances |
| Lambda 逾時 | 模型思考多輪或 SSM 指令慢 | Timeout 設為 5 分鐘；任務描述寫具體一點 |
| `Unable to import module 'lambda_function'` | 打包平台或 Python 版本不符 | 重做 3.5，確認 `--python-version 3.12`，且 Lambda Runtime 也是 Python 3.12 |

---

### 附錄：本指南建立的資源一覽

| 名稱 | 服務 | 階段 |
|---|---|---|
| `wp-vpc`（含 4 個子網路） | VPC | 1 |
| `wp-web-sg`、`wp-db-sg` | 安全群組 | 1 |
| `wp-db` | RDS MySQL | 1 |
| `wp/db-app`、`rds!db-...` | Secrets Manager | 1 |
| `wp-ec2-role` | IAM 角色 | 1 |
| `wp-web-1`、Elastic IP | EC2 | 1（Phase 2 移除） |
| `/wordpress/*` 日誌、`wp-alerts`、`wp-cpu-high` | CloudWatch / SNS | 1 |
| `wp-backup-<ACCOUNT_ID>` | S3 | 2 |
| `wp-efs`、`wp-efs-sg` | EFS | 2 |
| `wp-alb`、`wp-alb-sg`、`wp-tg` | ELB | 2 |
| CloudFront distribution | CloudFront | 2 |
| `wp-web-ami-v1`、`wp-web-lt`、`wp-asg` | EC2 Auto Scaling | 2 |
| `wp-5xx-high` | CloudWatch | 2 |
| `wp-agent/slack-webhook` | Secrets Manager | 3 |
| `wp-agent-role`、`wp-ops-agent` | IAM / Lambda | 3 |
| `wp-daily-health-check` | EventBridge Scheduler | 3 |
