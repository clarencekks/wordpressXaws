# WordPress × AWS AI Agent：小白一步一步建置指南（v3）

> 📌 **本 README 即為最新版（v3，2026-10-28 更新）的完整建置指南。** 各版本檔案保存在 [`docs/`](docs/)：
>
> | 版本 | 檔案 | 說明 |
> |---|---|---|
> | **v3（最新）** | [docs/WordPress_AWS_小白建置指南_v3.md](docs/WordPress_AWS_小白建置指南_v3.md) | 審核 v2 補上遺漏步驟，並依 2026-10-26、10-27 實作經驗補充 Phase 1 與 Phase 2（2.1–2.3）細節、驗證輸出範例、一鍵檢查與學習經驗（見文末兩頁實作紀錄） |
> | v2 | [docs/WordPress_AWS_小白建置指南_v2.md](docs/WordPress_AWS_小白建置指南_v2.md) | 修正 v1 實作遇到的五個問題（私有子網路、IAM 權限、VPC 端點等） |
> | v1 | [docs/WordPress_AWS_小白建置指南_v1.md](docs/WordPress_AWS_小白建置指南_v1.md) | 初版 |

> 依據簡報《WordPress × AWS AI Agent 架構設計》（CoralEnergy 版）撰寫。
> 目標：從一個全新的 AWS 帳號開始，把 WordPress 架到雲端，最後加上簡報中的「AI 維運大腦」。
> 撰寫日期：2026-10。AWS 主控台的按鈕名稱偶爾會改版，若找不到某個按鈕，用頁面上方的搜尋框搜尋文中的關鍵字即可。

> **v3 審核版（2026-10-24）**：以全新帳號、從頭照做的角度重新檢查 v2，補上遺漏的步驟與會失敗的指令（清單見文末「[v3 修正紀錄](#v3-修正紀錄)」）。舊版 v1、v2 保存在 [`docs/`](docs/)。
>
> **v2 修訂（2026-10-24）**：依實際操作遇到的五個問題修訂。
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

### 準備 1：註冊與保護 Root 帳號

1. 到 <https://aws.amazon.com/> 註冊帳號（需要信用卡）。
2. 用 Root（註冊用的 Email）登入後，右上角點帳號名稱 → **Security credentials** → **Multi-factor authentication (MFA)** → **Assign MFA device**，用手機上的 Google Authenticator 或 Authy 綁定。
3. **之後日常操作不要用 Root。**

### 準備 2：建立日常使用的管理員帳號

1. 上方搜尋列輸入 **IAM** → 左側 **Users** → **Create user**。
2. User name：`admin-你的名字`；勾選 **Provide user access to the AWS Management Console** → 選 **I want to create an IAM user** → 設定密碼。
3. **Set permissions** → **Attach policies directly** → 勾選 `AdministratorAccess` → **Create user**。
4. 記下頁面上的 **Console sign-in URL**，登出 Root，改用這個網址和新帳號登入，並同樣為它綁定 MFA。

> 進階做法是用 IAM Identity Center，但對初學者來說，一個有 MFA 的 IAM 使用者就足夠安全。

### 準備 3：設定預算警報（最重要的一步）

1. 搜尋 **Billing and Cost Management** → 左側 **Budgets** → **Create budget**。
2. 選 **Use a template** → **Monthly cost budget**。
3. Budgeted amount 輸入 `50`（美金，可自行調整）；Email 填你的信箱 → **Create budget**。
4. 超過 85% 及 100% 時你會收到信。

✅ **驗證**：Budgets 頁面看到剛建立的預算，狀態為 OK。

### 準備 4：切換區域與打開 CloudShell

1. 主控台右上角區域選單選 **Asia Pacific (Tokyo) ap-northeast-1**。之後每次操作前都確認右上角是東京。
2. 主控台最下方（或右上角）點 **CloudShell** 圖示，會打開一個終端機視窗。
3. 在 CloudShell 輸入：

```bash
aws sts get-caller-identity
echo $AWS_REGION
```

✅ **驗證**：第一段會顯示你的 12 位數帳號 ID（Account），第二行顯示 `ap-northeast-1`。如果不是東京，表示主控台右上角的區域選錯了，切換後重新打開 CloudShell。

### 準備 5：準備一張「數值記錄表」

後面的步驟會一直用到前面產生的值。CloudShell 閒置約 20 分鐘就會斷線，**斷線後 `$變數` 會全部消失**，所以請把下面這張表複製到記事本，每拿到一個值就填上：

| 名稱 | 在哪一步產生 | 你的值 |
|---|---|---|
| `ACCOUNT_ID`（帳號 ID） | 準備 4 | |
| public1 子網路 ID | 1.1 | |
| `<RDS_ENDPOINT>` | 1.3 | |
| `<EIP>` | 1.8 | |
| `fs-xxxxxxxx`（EFS ID） | 2.2 | |
| ALB 的 DNS name | 2.3 | |
| `<ORIGIN_SECRET>` | 2.4 | |
| `<CF_DOMAIN>` | 2.4 | |
| `<MODEL_ID>` | 3.1 | |

> 文中出現 `<像這樣>` 的文字，都要換成表中你自己的值，**連同 `<` `>` 一起換掉**。指令貼上後如果出現 `<`、`>` 相關的錯誤，通常就是忘了替換。

> **新帳號的「免費方案（Free plan）」限制**：2025 年 7 月之後建立的帳號，若選擇 Free plan，部分服務（例如透過 AWS Marketplace 提供的 Bedrock 第三方模型）可能無法使用。在 Phase 3 遇到「帳號方案不支援」之類的訊息時，到 **Billing and Cost Management → Upgrade plan** 升級成付費方案即可，尚未用完的抵用金會保留。

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

要建立兩個安全群組：`wp-web-sg`（WordPress 主機用）和 `wp-db-sg`（資料庫用），**兩個都必須建在 `wp-vpc` 裡面**。

> ⚠️ **最容易漏掉的地方**：建立安全群組的頁面，**VPC 欄位預設會帶入 AWS 自動建立的「預設 VPC（default VPC）」，不是 `wp-vpc`**。如果沒有手動改，安全群組就會建在錯的 VPC，後面會發生這些狀況：
> - 1.3 建立 RDS 和 1.7 啟動主機時，安全群組清單裡找不到 `wp-db-sg`、`wp-web-sg`。
> - 建立 `wp-db-sg` 的規則時，Source 搜尋不到 `wp-web-sg`。
>
> 安全群組建立後**不能更換 VPC**，建錯了只能刪掉重建。

**① 建立 `wp-web-sg`（WordPress 主機）**

1. 搜尋 **VPC** → 左側 **Security groups** → 右上 **Create security group**。
2. **Basic details**：
   - Security group name：`wp-web-sg`
   - Description：`WordPress web server`（必填，只能用英文）
   - **VPC：點開下拉選單，把預設值改成 `wp-vpc`**（顯示為 `vpc-xxxxxxxx (wp-vpc)`）。
3. **Inbound rules** → **Add rule**：Type `HTTP`、Source `Anywhere-IPv4`（`0.0.0.0/0`）。
   - **不要加 SSH（22）**。這就是簡報第 7 頁的「關閉 Port 22」。
   - Phase 1 只開 HTTP 80 就好，**不需要開 443**：此階段還沒有憑證，HTTPS 會在 Phase 2 由 CloudFront 負責。
4. **Outbound rules**：保持預設（全部允許）。主機要靠對外連線去下載 WordPress、連 SSM 和 Secrets Manager，**不要刪掉 outbound 規則**。
5. 按 **Create security group**。

**② 建立 `wp-db-sg`（資料庫）**

1. 回到 **Security groups** → **Create security group**。
2. **Basic details**：
   - Security group name：`wp-db-sg`
   - Description：`WordPress database`
   - **VPC：同樣要改成 `wp-vpc`**（一定要先改 VPC，下一步才搜尋得到 `wp-web-sg`）。
3. **Inbound rules** → **Add rule**：Type `MYSQL/Aurora`（Port 3306）、Source 選 **Custom**，在旁邊的搜尋框輸入 `wp-web` → 選 `sg-xxxxxxxx (wp-web-sg)`。
   - 意思是：只有掛著 `wp-web-sg` 的主機可以連資料庫，網路上其他人都連不到。
4. Outbound rules 保持預設 → **Create security group**。

✅ **驗證（主控台）**：在 **Security groups** 列表中，`wp-web-sg` 與 `wp-db-sg` 的 **VPC ID** 欄位都和 `wp-vpc` 的 VPC ID 相同；`wp-web-sg` 只有一條 Port 80 規則；`wp-db-sg` 只有一條 Port 3306、來源為 `sg-xxxx (wp-web-sg)` 的規則。

✅ **驗證（CloudShell）**：

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=wp-vpc --query "Vpcs[0].VpcId" --output text)
echo "wp-vpc 的 ID：$VPC_ID"
aws ec2 describe-security-groups --filters Name=group-name,Values=wp-web-sg,wp-db-sg \
  --query "SecurityGroups[].[GroupName, GroupId, VpcId]" --output table
```

表格中兩個安全群組的 `VpcId` 都要等於上面印出的 wp-vpc ID。

**建錯 VPC 了怎麼辦？**
1. **Security groups** → 勾選建錯的那個 → **Actions → Delete security groups**。（若刪除失敗並提示有其他安全群組參照它，先刪 `wp-db-sg`，再刪 `wp-web-sg`。）
2. 照上面的步驟重新建立，這次記得把 VPC 改成 `wp-vpc`。

> 之後建立的 `wp-efs-sg`（2.2）、`wp-alb-sg`（2.3）也一樣，**VPC 欄位都要手動改成 `wp-vpc`**。

### 1.3 建立 RDS 資料庫

1. 搜尋 **RDS** → **Create database**。
2. 選 **Standard create**（新版主控台可能叫 **Full configuration**）。
3. Engine：**MySQL**；Engine version：選 **8.4 系列**（LTS 長期支援版，通常就是預設值）。
   ⚠️ **不要選 8.0**：MySQL 8.0 在 RDS 的標準支援已於 2026 年 7 月結束，繼續使用會被收取「延伸支援（Extended Support）」額外費用。
4. Templates：**Free tier**（若沒有此選項，選 **Dev/Test** 後手動照下面設定）。
5. Settings：
   - DB instance identifier：`wp-db`
   - Master username：`admin`
   - （若有 **Enable deletion protection** 選項，練習用請**不要勾**，否則清除時刪不掉）
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

✅ **驗證**：密碼箱有建立，而且內容包含正確的 username 與密碼（CloudShell）：

```bash
# 1) 確認 secret 存在
aws secretsmanager describe-secret --secret-id wp/db-app \
  --query "{Name:Name, ARN:ARN, Created:CreatedDate}" --output table

# 2) 讀出內容，確認 username 與密碼（密碼只顯示長度，不印在畫面上）
SECRET=$(aws secretsmanager get-secret-value --secret-id wp/db-app --query SecretString --output text)
echo "username：$(echo "$SECRET" | jq -r .username)"
echo "password 長度：$(echo "$SECRET" | jq -r .password | tr -d '\n' | wc -c)"
unset SECRET
```

正確結果：
- 第一段的表格中 `Name` 為 `wp/db-app`。
- `username：wpuser`
- `password 長度：32`

**常見狀況：**
- `ResourceNotFoundException` → secret 沒建立成功，或區域不是東京。確認 `echo $AWS_REGION` 是 `ap-northeast-1` 後，重跑上面的建立指令。
- `password 長度：0`，或 username 顯示 `null` → 建立時 `$WP_DB_PASS` 是空的（例如 CloudShell 斷線後只執行了第二段）。用下面的指令重新產生並覆寫內容，然後再驗證一次：

```bash
WP_DB_PASS=$(aws secretsmanager get-random-password --password-length 32 --exclude-punctuation \
  --query RandomPassword --output text)
aws secretsmanager put-secret-value --secret-id wp/db-app \
  --secret-string "{\"username\":\"wpuser\",\"password\":\"$WP_DB_PASS\"}"
```

- `ResourceExistsException`（重複建立）→ 代表已經建立過了，不需要重建，直接做上面的驗證即可。

> 主控台也可以確認：**Secrets Manager → Secrets → wp/db-app → Overview** 頁面下方的 **Retrieve secret value**，會以表格顯示 `username` 與 `password` 兩個欄位。

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
2. **Name and tags（名稱與標籤）**：這一區要設定兩個標籤。

   **什麼是標籤（Tag）？** 標籤就像貼在資源上的便利貼，由「鍵（Key）」和「值（Value）」組成，例如 `Role = wordpress`。標籤不會改變主機的功能，只是讓人和程式可以用它來**分類、搜尋和限制權限**。

   | Key | Value | 用途 |
   |---|---|---|
   | `Name` | `wp-web-1` | 主機在 EC2 列表上顯示的名稱。1.7 的網路檢查指令就是用這個名稱找主機。 |
   | `Role` | `wordpress` | 標示「這是 WordPress 主機」。Phase 3 的 AI Agent 會用它找出要檢查的主機（3.4 程式中的 `_find_instance_id`）；3.3 的 IAM 權限也規定 Agent **只能**對貼有這個標籤的主機下指令。少了這個標籤，Agent 會找不到主機，或出現 `AccessDenied`。 |

   操作步驟：
   1. 在 **Name** 欄位輸入 `wp-web-1`。（這一步其實就是自動建立 `Name` 標籤。）
   2. 點 Name 欄位右邊的 **Add additional tags**，這一區會展開成標籤表格，第一列已經是剛才的 `Name = wp-web-1`。
   3. 點 **Add new tag**（或 **Add tag**），新增一列：
      - **Key**：`Role`
      - **Value**：`wordpress`
      - **Resource types**：點開下拉選單，確認勾選 **Instances**。預設可能同時勾了 Volumes（硬碟），一起勾著沒關係，但 **Instances 一定要勾**，因為 Agent 是用「主機」上的標籤來找主機。
   4. ⚠️ 標籤**有分大小寫**，請完全照抄：`Role`（大寫 R）、`wordpress`（全小寫）。打成 `role` 或 `WordPress` 時，Agent 一樣找不到主機。

   > 忘了加、或主機已經建好了？可以事後補：**EC2 → Instances** → 勾選主機 → 下方 **Tags** 分頁 → **Manage tags** → **Add new tag** → Key `Role`、Value `wordpress` → **Save**。不需要重開機。

   ✅ **驗證**（主機啟動後，在 CloudShell 執行）：

   ```bash
   aws ec2 describe-instances --filters "Name=tag:Role,Values=wordpress" \
     --query "Reservations[].Instances[].[InstanceId, Tags[?Key=='Name']|[0].Value, State.Name]" --output table
   ```

   表格中出現 `wp-web-1` 就代表標籤正確。如果是空的，表示標籤沒加上，或大小寫打錯。
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
            try_files $uri =404;
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

#### 開始前先看：等待時間與注意事項

**⏱ 等待時間（從按下 Launch instance 開始算）**

| 階段 | 大約時間 | 怎麼知道完成了 |
|---|---|---|
| 主機開機 | 1–2 分鐘 | EC2 列表的 Instance state 變成 **Running** |
| 主機通過狀態檢查 | 2–3 分鐘 | Status check 顯示 **3/3 checks passed** |
| Session Manager 可以連線 | 開機後 2–5 分鐘 | Connect → Session Manager 分頁的 **Connect** 按鈕可以按 |
| User data 安裝完成 | 開機後 3–5 分鐘 | 下面指令 ② 的最後一行出現 `USER DATA DONE` |
| 手動重跑 User data（需要時） | 2–5 分鐘 | 出現 `結束代碼：0` |

**建議：按下 Launch instance 後，先等 5 分鐘再連線檢查**，比較不會看到「還沒完成」的結果。

**⚠️ 操作注意事項**

1. **一次只貼一行指令，按一次 Enter，等它跑完、出現新的 `#` 提示字元，再貼下一行。** 一次貼好幾行容易漏行或順序錯亂。
2. **安裝腳本執行時畫面會空白好幾分鐘，這是正常的。** 1.7 的腳本把所有輸出都寫進 `/var/log/user-data.log`，畫面上不會顯示任何進度。這時**不要按 Ctrl+C**，按了會中斷安裝，日誌也永遠不會出現 DONE。
3. **想看安裝進度**：另外開一個 Session Manager 分頁，執行 `sudo tail -f /var/log/user-data.log` 就能即時看到。這個分頁按 Ctrl+C 只會停止查看，不會中斷安裝。
4. **畫面停住、按 Enter 沒反應時**，先看最後一行：
   - 出現 `>` → 指令還在等你輸入剩下的內容（例如引號或 `<<'EOF'` 沒有結尾）。按 **Ctrl+C** 取消，回到 `#` 後重新貼一次完整的指令。
   - 最後一行是 `#` 提示字元 → 終端機是正常的，可以直接輸入。
   - 打字完全不會出現在畫面上 → 網頁終端機斷線了。關掉分頁，重新 **Connect**。重新連線不會影響主機，但要重新執行 `sudo -i`。
5. **Session Manager 閒置約 20 分鐘會自動斷線**，重新連線即可，主機上的檔案和服務不受影響。
6. **只有在指南叫你貼的時候，才把 User data 腳本貼進終端機。** 1.7 的腳本是給主機開機時自動執行的，直接貼進終端機會把你的畫面輸出導到日誌檔，看起來就像當機。

---

1. 照上面的建議，按下 Launch instance 後等大約 5 分鐘。
2. **EC2 → Instances** → 勾選 `wp-web-1` → 右上 **Connect** → 分頁 **Session Manager** → **Connect**。
3. 瀏覽器會打開一個黑色終端機，游標前面會顯示 `sh-5.2$`。這代表你已經用 Session Manager 的預設帳號 `ssm-user` 登入主機了。

接下來**一次輸入一行、按 Enter**，逐步確認 1.7 的 User data（自動安裝腳本）已經順利跑完。

#### 指令 ① 切換成管理員（root）

```bash
sudo -i
```

- **作用**：從一般帳號 `ssm-user` 切換成最高權限的 `root`。後面讀取系統日誌、管理服務都需要 root。
- **正確畫面**：提示字元從 `sh-5.2$` 變成類似 `[root@ip-10-0-0-25 ~]#`。**結尾從 `$` 變成 `#`** 就代表已經是 root。這個指令本身不會輸出其他文字。

#### 指令 ② 查看安裝腳本的最後 3 行紀錄

```bash
tail -n 3 /var/log/user-data.log
```

- **作用**：User data 腳本把執行過程都寫進了 `/var/log/user-data.log`。`tail -n 3` 只顯示檔案的最後 3 行。腳本最後一行是 `echo "USER DATA DONE"`，所以只要看到這段文字，就代表**整份腳本從頭跑到尾都沒有出錯**（腳本開頭設定了 `set -e`，任何一步失敗都會立刻停下，不會印出 DONE）。
- **正確畫面**（前面幾行的內容可能略有不同，重點是**最後一行**）：

```text
[root@ip-10-0-0-25 ~]# tail -n 3 /var/log/user-data.log
Successfully fetched the config and saved in /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.d/file_config.json.tmp
+ echo 'USER DATA DONE'
USER DATA DONE
```

- **如果最後一行不是 `USER DATA DONE`**：
  - 主機剛開機不到 5 分鐘 → 腳本可能還在安裝套件。等 1–2 分鐘再執行一次同一個指令。
  - 等了很久還是沒有 → 腳本中途出錯了。執行 `tail -n 30 /var/log/user-data.log` 看最後 30 行，通常最後幾行就是錯誤原因（例如 `Could not resolve host` 代表主機連不上網際網路，請看「[問題 3](#問題-3無法下載-wordpressnetwork-is-unreachable)」）。
  - 顯示 `No such file or directory` → User data 沒有執行。通常是 1.7 漏貼了 User data，或第一行不是 `#!/bin/bash`。
  - 先用 `cloud-init status --long` 看整體狀態：`running` 表示還在跑（再等幾分鐘）；`error` 表示中途失敗；`done` 但沒有 DONE 字樣，通常是 User data 沒貼上。
  - **修好原因後不用重建主機**，直接重跑同一份腳本（重複執行不會出問題）：

    ```bash
    bash /var/lib/cloud/instance/scripts/part-001
    tail -n 3 /var/log/user-data.log
    ```

    若 `part-001` 不存在，代表 User data 根本沒有貼上，請 Terminate 主機後重做 1.7。

    ⚠️ 腳本會把所有輸出寫進日誌檔，所以**執行期間畫面會空白 2–5 分鐘，看起來像卡住，這是正常的，千萬不要按 Ctrl+C**（按了會中斷安裝，日誌也不會出現 DONE）。想看進度，可以另開一個 Session Manager 分頁執行 `sudo tail -f /var/log/user-data.log`，那個分頁按 Ctrl+C 只會停止查看，不會中斷安裝。

  - **完整的診斷流程**（照順序做，找到原因就停）：

    | 步驟 | 指令 | 看什麼 |
    |---|---|---|
    | 1 | `cloud-init status --long` | `running`：還在跑，等 3 分鐘；`error`：中途失敗，做第 3 步；`done`：做第 2 步 |
    | 2 | `ls -l /var/lib/cloud/instance/scripts/` 與 `head -n 1 /var/lib/cloud/instance/scripts/part-001` | 檔案存在、大小約 2.8 KB、第一行是 `#!/bin/bash` → 腳本有收到，做第 3 步；檔案不存在或第一行不對 → User data 沒貼好，Terminate 主機後重做 1.7 |
    | 3 | `tail -n 30 /var/log/user-data.log` | 找錯誤訊息。若最後幾行是空白、看不到錯誤，而日誌時間比腳本時間晚很多，通常代表之前手動重跑時按了 Ctrl+C 中斷，直接做第 4 步 |
    | 4 | `bash /var/lib/cloud/instance/scripts/part-001; echo "結束代碼：$?"` | 等 2–5 分鐘，畫面空白屬正常；出現 `結束代碼：0` 後，用 `tail -n 5 /var/log/user-data.log` 確認最後一行是 `USER DATA DONE` |

#### 指令 ③ 確認三個服務都在執行

```bash
systemctl is-active nginx php-fpm amazon-cloudwatch-agent
```

- **作用**：一次查詢三個服務的狀態，**依序**各回答一行：
  - `nginx`：網頁伺服器，負責接收瀏覽器的請求。
  - `php-fpm`：執行 WordPress 的 PHP 程式。
  - `amazon-cloudwatch-agent`：把日誌和指標送到 CloudWatch（1.13 與 Phase 3 會用到）。
- **正確畫面**：

```text
[root@ip-10-0-0-25 ~]# systemctl is-active nginx php-fpm amazon-cloudwatch-agent
active
active
active
```

- **如果某一行不是 `active`**（第幾行就對應第幾個服務）：

| 顯示 | 意思 | 處理方式 |
|---|---|---|
| `activating` | 正在啟動中 | 等 10 秒再查一次 |
| `inactive` | 已安裝但沒有啟動 | `systemctl start 服務名稱`，例如 `systemctl start nginx` |
| `failed` | 啟動失敗 | `systemctl status 服務名稱 --no-pager` 查看原因；Nginx 可再用 `nginx -t` 檢查設定檔 |
| `unknown` | 這個服務沒有安裝 | 回到指令 ②，確認 User data 有沒有跑完 |

#### 指令 ④ 從主機內部打開網站的健康檢查頁

```bash
curl -s localhost/healthz; echo
```

- **作用**：`curl` 是命令列版的瀏覽器，`localhost` 代表「這台主機自己」。`/healthz` 是我們在 Nginx 設定中加的健康檢查網址，固定回傳 `ok`（Phase 2 的 ALB 也會用它判斷主機是否正常）。最後的 `; echo` 只是補一個換行，讓 `ok` 單獨一行，不會黏在下一個提示字元前面。
- **正確畫面**：

```text
[root@ip-10-0-0-25 ~]# curl -s localhost/healthz; echo
ok
```

- **如果不是 `ok`**：
  - 什麼都沒顯示（只有空行）→ Nginx 沒在執行，回到指令 ③。
  - 顯示一段 HTML（例如 Nginx 歡迎頁或 `404 Not Found`）→ Nginx 沒有讀到我們的設定。執行 `nginx -t && systemctl restart nginx` 後再試一次。

#### 一次檢查全部（選用）

也可以把下面整段一起貼上，它會把上面三項檢查整理成 ✅／❌ 的結果：

```bash
echo "=== 安裝腳本 ==="
grep -q "USER DATA DONE" /var/log/user-data.log 2>/dev/null && echo "✅ User data 已完成" || echo "❌ User data 未完成（查看：tail -n 30 /var/log/user-data.log）"
echo "=== 服務狀態 ==="
for s in nginx php-fpm amazon-cloudwatch-agent; do
  st=$(systemctl is-active $s)
  [ "$st" = "active" ] && echo "✅ $s：$st" || echo "❌ $s：$st"
done
echo "=== 網站健康檢查 ==="
[ "$(curl -s localhost/healthz)" = "ok" ] && echo "✅ /healthz 回傳 ok" || echo "❌ /healthz 沒有回傳 ok"
```

正確畫面：

```text
=== 安裝腳本 ===
✅ User data 已完成
=== 服務狀態 ===
✅ nginx：active
✅ php-fpm：active
✅ amazon-cloudwatch-agent：active
=== 網站健康檢查 ===
✅ /healthz 回傳 ok
```

✅ **驗證**：看到 `USER DATA DONE`、三個 `active`、以及 `ok`（或上面五個 ✅），就可以繼續 1.10。

> 如果 Connect 按鈕是灰的或顯示 SSM Agent 未連線：通常是主機放在私有子網路、沒有公有 IP，或沒掛 `wp-ec2-role`。請看「[問題 1](#問題-1ssh-連不上session-manager-也無法連線)」。
>
> 為什麼不用 SSH？本指南的主機刻意沒有開 Port 22，也沒有金鑰，所以 **SSH 本來就連不上，這是正常的**。Session Manager 走的是 AWS 的加密通道（HTTPS 443 對外連線），不需要開任何對內 Port。

### 1.10 建立資料庫使用者並產生 wp-config.php

#### 步驟 0：先確認資料庫已經可以使用（Available）

RDS 從按下 Create database 起，通常要 **5–15 分鐘**才會建好，有時會更久。狀態還沒變成 **Available** 時連線一定會失敗，所以請先確認。下面兩種方法擇一即可。

**方法 A：主控台**
1. 搜尋 **RDS** → 左側 **Databases**。
2. 找到 `wp-db` 這一列，看 **Status** 欄位：

| Status 顯示 | 意思 | 怎麼做 |
|---|---|---|
| Creating | 正在建立 | 等待，每隔幾分鐘按右上角的重新整理（⟳） |
| Backing-up | 建好後的第一次自動備份 | 等待，通常幾分鐘 |
| Modifying／Configuring-enhanced-monitoring 等 | 正在套用設定 | 等待 |
| **Available** | ✅ 可以使用 | 繼續下面的步驟 |
| Failed／Incompatible-network 等 | 建立失敗 | 刪除後重做 1.3，並檢查子網路群組與安全群組的設定 |

3. 點進 `wp-db` → **Connectivity & security** 分頁，確認 **Endpoint** 有值（長得像 `wp-db.xxxxxxxx.ap-northeast-1.rds.amazonaws.com`）。

**方法 B：指令（在 Session Manager 的 root 或 CloudShell 都可以）**

```bash
export AWS_DEFAULT_REGION=ap-northeast-1
aws rds describe-db-instances --db-instance-identifier wp-db \
  --query "DBInstances[0].[DBInstanceStatus, Endpoint.Address]" --output table
```

正確結果：第一欄是 `available`，第二欄是 Endpoint 網址。第一欄若是 `creating` 或 `backing-up`，就再等幾分鐘重跑一次。

也可以用這一行讓它自己等，狀態變成 Available 時才會回到提示字元，最多等 30 分鐘。等待期間畫面不會有任何文字，屬正常現象：

```bash
aws rds wait db-instance-available --db-instance-identifier wp-db && echo "✅ wp-db 已經是 Available"
```

> 出現 `DBInstanceNotFound`：資料庫名稱不是 `wp-db`，或建在其他區域。到 RDS 主控台確認右上角區域是東京、DB identifier 是 `wp-db`。
> 在主機上出現 `AccessDenied ... rds:DescribeDBInstances`：角色權限不完整，請重跑 1.5。

#### 步驟 1：建立使用者並產生設定檔

確認 Available 之後，繼續在 Session Manager（root）中執行。v3 改成**自動取得 RDS Endpoint**，不需要手動替換。

> ⚠️ **這一步不能在 CloudShell 執行，一定要在 Session Manager 的主機上（root）執行**：
> - CloudShell 不在 `wp-vpc` 裡，而 RDS 設定為不公開、`wp-db-sg` 只允許 `wp-web-sg` 的主機連線，所以從 CloudShell 執行 `mysql` 會連線逾時（這是刻意的安全設計）。
> - `wp config create` 會把 `wp-config.php` 寫進主機的 `/var/www/html`，CloudShell 裡沒有這個資料夾。
>
> 分工原則：**查詢或管理 AWS 資源**（RDS 狀態、Secrets Manager）用 CloudShell；**連資料庫或操作 WordPress**（`mysql`、`wp` 指令）用 Session Manager。

**Session Manager 在哪裡？**

就是 1.9 用來跑檢查指令的那個黑色終端機。如果分頁還開著，直接在裡面輸入即可；已經關掉的話，照下面步驟重新打開：

1. AWS 主控台上方搜尋 **EC2** → 左側 **Instances**。
2. 勾選 `wp-web-1` 前面的方框。
3. 按右上角 **Connect**。
4. 上方分頁點 **Session Manager**。
5. 按右下角橘色的 **Connect**。
6. 瀏覽器會開一個新分頁，顯示 `sh-5.2$`。先輸入 `sudo -i`，提示字元變成 `[root@ip-... ~]#` 後才開始貼下面的指令。

**怎麼分辨自己在 CloudShell 還是 Session Manager？看提示字元：**

| 提示字元 | 所在位置 |
|---|---|
| `[cloudshell-user@ip-... ~]$` | CloudShell（主控台下方或右上角的 `>_` 圖示，**在你的主機外面**） |
| `sh-5.2$` | Session Manager，但還沒切換成 root，先輸入 `sudo -i` |
| `[root@ip-10-0-x-x ~]#` | ✅ Session Manager 的 root，**在你的 EC2 主機裡面**，這一步要在這裡執行 |

```bash
export AWS_DEFAULT_REGION=ap-northeast-1
RDS_HOST=$(aws rds describe-db-instances --db-instance-identifier wp-db \
  --query "DBInstances[0].Endpoint.Address" --output text)
echo "RDS Endpoint：$RDS_HOST"

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
> - 出現 `Authentication plugin 'caching_sha2_password' cannot be loaded` → MySQL 8.4 預設使用新的密碼驗證方式，主機上的 MariaDB 用戶端缺少對應外掛。執行 `dnf install -y mariadb-connector-c` 後重跑同一段指令。（此訊息只與命令列工具有關；WordPress 使用的 PHP 連線模組本身支援這種驗證方式。）

### 1.11 完成 WordPress 安裝

到這一步，主機上已經有 WordPress 程式（1.7），資料庫連線也設定好了（1.10 產生的 `wp-config.php`）。最後要用瀏覽器跑一次 WordPress 的安裝畫面，建立**網站名稱**和**管理員帳號**。

#### 步驟 1：找到網站的 IP（`<EIP>`）

如果 1.8 已經記在「數值記錄表」裡，可以跳過這步。

- **主控台**：EC2 → 左側 **Elastic IPs** → `Allocated IPv4 address` 欄位的那組數字（例如 `13.192.60.150`），而且 **Associated instance ID** 要是 `wp-web-1` 的 ID。
- **CloudShell**：

```bash
aws ec2 describe-addresses --query "Addresses[].[PublicIp, InstanceId]" --output table
```

#### 步驟 2：安裝前先從 CloudShell 確認網站連得到

把 `<EIP>` 換成上一步的 IP（`<` `>` 一起刪掉），在 **CloudShell** 執行：

```bash
curl -s -m 5 http://<EIP>/healthz; echo
curl -s -m 5 -o /dev/null -w "HTTP 狀態碼：%{http_code}\n" http://<EIP>/
```

| 結果 | 意思 | 怎麼做 |
|---|---|---|
| `ok`，以及 `HTTP 狀態碼：302` | ✅ 網站正常，WordPress 正要帶你去安裝頁 | 繼續步驟 3 |
| `ok`，以及 `HTTP 狀態碼：200` | ✅ 網站正常（可能已經安裝過） | 繼續步驟 3 |
| 什麼都沒有，等 5 秒後結束 | ❌ 連線逾時，外面連不進主機 | 看「[問題 5](#問題-5網站從網際網路連不上連線逾時)」，通常是子網路路由、安全群組 80 Port，或 Elastic IP 沒綁到主機 |
| `HTTP 狀態碼：500` | ❌ PHP 執行錯誤 | Session Manager 執行 `tail -n 20 /var/log/php-fpm/www-error.log` 查看原因 |

#### 步驟 3：用瀏覽器打開網站

1. 在**自己電腦**的瀏覽器（Chrome、Edge、Safari 都可以）網址列輸入 `http://<EIP>`，例如 `http://13.192.60.150`。
   - ⚠️ 要打 **`http://`**，不是 `https://`。Phase 1 還沒有 HTTPS 憑證，用 https 會一直轉圈或顯示無法連線。
   - 有些瀏覽器會自動把網址改成 https。如果出現「此網站不支援安全連線」之類的警告頁，點 **繼續前往網站**（Continue to site）。
   - 網址列旁邊顯示「**不安全**」是正常的，Phase 2 加上 CloudFront 後就會變成鎖頭。
2. 畫面會自動跳到 `http://<EIP>/wp-admin/install.php`，這就是 WordPress 的安裝頁。

> 如果出現的是「**建立設定檔**」，或要求你填「資料庫名稱、使用者名稱、密碼、資料庫主機」的畫面：代表主機上沒有 `wp-config.php`，1.10 還沒完成。**不要在網頁上填**，回到 1.10 用指令產生。

#### 步驟 4：選擇語言

1. 第一個畫面是語言清單，往下捲或直接按鍵盤找到 **繁體中文**。
2. 按右下角的 **Continue**。

（WordPress 會從 wordpress.org 下載中文語言包，需要幾秒鐘。主機要能連上網際網路，這也是 Web 主機要放在公有子網路的原因之一。）

#### 步驟 5：填寫網站資訊

這個畫面標題是「**歡迎使用**」，各欄位這樣填：

| 欄位 | 填什麼 | 說明 |
|---|---|---|
| **網站標題** | 例如 `我的 WordPress 網站` | 之後可以在後台改 |
| **使用者名稱** | 例如 `clarence_admin` | 管理員的登入帳號。⚠️ **不要用 `admin`、`administrator`、`root`**，這些是駭客最常猜的帳號。建立後**不能修改** |
| **密碼** | 建議直接用 WordPress 自動產生的強密碼 | 畫面上已經預先產生一組，按「**隱藏**」旁邊的眼睛可以顯示。**先複製存進密碼管理工具或安全的地方**再繼續 |
| **確認密碼** | 只有在你改用較弱的密碼時才會出現 | 勾選「確認使用弱密碼」。練習用可以，但正式網站不建議 |
| **你的電子郵件地址** | 你常用的 Email | 忘記密碼時用來重設。（Phase 1 的主機沒有設定寄信功能，暫時收不到信，所以**密碼一定要先存好**） |
| **搜尋引擎可見度** | ✅ **建議勾選**「阻擋搜尋引擎索引這個網站」 | 練習中的網站不需要被 Google 收錄。正式上線時到 **設定 → 閱讀** 取消勾選 |

填完按左下角的 **安裝 WordPress**。

#### 步驟 6：登入後台

1. 出現「**成功！**」畫面，表示安裝完成，畫面上會再顯示一次你的使用者名稱。
2. 按 **登入**，或直接打開 `http://<EIP>/wp-admin`。
3. 在登入頁輸入剛才設定的使用者名稱和密碼 → **登入**。
4. 看到左側黑色選單和「**控制台**」標題，就代表你已經在 WordPress 後台了。

#### 步驟 7：建議馬上做的兩個基本設定（選用）

1. **時區**：左側 **設定 → 一般** → **時區** 選 `台北`（或 `UTC+8`）→ 最下方 **儲存設定**。這樣文章的發佈時間才會正確。
2. **網址格式（永久連結）**：左側 **設定 → 永久連結** → 選 **文章名稱** → **儲存設定**。網址會從 `/?p=123` 變成好讀的 `/hello-world/`（1.7 的 Nginx 設定已經支援這種格式）。

✅ **驗證**：

1. **前台**：瀏覽器打開 `http://<EIP>`，看到預設主題的首頁，以及一篇「Hello world!」（或中文「你好，世界！」）文章。
2. **後台**：`http://<EIP>/wp-admin` 可以登入。
3. **主機上**（Session Manager，root）確認安裝狀態：

```bash
sudo -u apache wp --path=/var/www/html core is-installed && echo "✅ WordPress 已安裝"
sudo -u apache wp --path=/var/www/html core version
sudo -u apache wp --path=/var/www/html user list --fields=user_login,roles
sudo -u apache wp --path=/var/www/html option get home
```

正確結果：
- `✅ WordPress 已安裝`
- 版本號，例如 `6.x.x`
- 一個表格列出你的管理員帳號，roles 為 `administrator`
- `http://<你的EIP>`

> 第一次執行 `wp` 指令時，畫面可能出現 `Warning: ... cache directory` 之類的警告，不影響結果，可以忽略。

**常見問題：**

| 狀況 | 原因 | 解法 |
|---|---|---|
| 瀏覽器一直轉圈、最後顯示連線逾時 | 外面連不進主機 | 回到步驟 2 的表格；確認網址是 `http://` 不是 `https://` |
| 顯示「建立資料庫連線時發生錯誤」 | `wp-config.php` 的資料庫資訊不對，或 RDS 連不上 | 回到 1.10 的步驟 0 確認 RDS 是 Available，並重做 1.10 |
| 出現要填資料庫資訊的設定頁 | 沒有 `wp-config.php` | 回到 1.10 產生，不要在網頁上填 |
| 選了繁體中文但畫面還是英文 | 語言包下載失敗（主機連不上 wordpress.org） | 先完成安裝，之後確認網路正常，再到 **Settings → General → Site Language** 改成繁體中文 |
| 忘記管理員密碼 | Phase 1 主機無法寄重設信 | Session Manager 執行 `sudo -u apache wp --path=/var/www/html user update 你的帳號 --user_pass='新密碼'` |
| 安裝時填錯網站標題 | | 後台 **設定 → 一般** 修改即可 |

> ⚠️ 此階段網站是 **HTTP（未加密）**，登入密碼會以明碼在網路上傳輸。Phase 1 僅供練習，不要在這個網站上使用你在其他地方也用的密碼；正式上線請完成 Phase 2 的 CloudFront（免費 HTTPS）。

### 1.12 設定告警通知（CPU 過高寄信）

這一步會做兩件事：先建立一個「通知管道」（SNS 主題 `wp-alerts`，綁定你的 Email），再建立一個「告警」（CloudWatch Alarm `wp-cpu-high`），規定主機 CPU 太高時透過這個管道寄信給你。

```
主機 CPU 持續 > 80% ──► CloudWatch 告警 wp-cpu-high 變成「In alarm」──► SNS 主題 wp-alerts ──► 寄信到你的信箱
```

> 名詞：**SNS 主題（Topic）**就像一個廣播頻道，**訂閱（Subscription）**就是「誰要收這個頻道的通知」。Phase 3 也會讓 AI Agent 訂閱同一個頻道。

#### Part A：建立通知管道（SNS）

1. 主控台上方搜尋 **SNS**，進入 **Simple Notification Service**（確認右上角區域是東京）。
2. 左側選 **Topics** → 右上 **Create topic**。
3. **Type**：選 **Standard**（不要選 FIFO）。
4. **Name**：`wp-alerts`。其他欄位都保持預設 → 拉到最下面按 **Create topic**。
5. 建立後會自動進入 `wp-alerts` 的頁面。在下方 **Subscriptions** 分頁按 **Create subscription**：
   - **Topic ARN**：已自動帶入 `wp-alerts`，不用改。
   - **Protocol**：選 **Email**。
   - **Endpoint**：填你的 Email。
   - 按 **Create subscription**。
6. 到你的信箱找一封寄件者為 **AWS Notifications**、主旨為 *AWS Notification - Subscription Confirmation* 的信（找不到的話看看垃圾郵件匣），點信中的 **Confirm subscription** 連結。

✅ **驗證 Part A**：回到 SNS → Topics → `wp-alerts` → **Subscriptions** 分頁，你的 Email 那一列 **Status** 顯示 **Confirmed**。如果還是 *Pending confirmation*，代表信中的連結還沒點。

#### Part B：建立告警（CloudWatch Alarm）

建立告警的精靈共有 4 個畫面，右上角會顯示 Step 1～Step 4。

**Step 1：Specify metric and conditions（選擇要監控什麼）**

1. 主控台上方搜尋 **CloudWatch** → 左側 **Alarms → All alarms** → 右上 **Create alarm**。
2. 按 **Select metric**，會跳出一個選擇視窗：
   - 點 **EC2** → 再點 **Per-Instance Metrics**。
   - 在搜尋框輸入 `CPUUtilization`，找到 **Instance name** 是 `wp-web-1` 的那一列，勾選它。
   - 按右下角 **Select metric**。
   - （找不到 `wp-web-1`：主機剛建立時，指標可能要等 5–10 分鐘才會出現。）
3. 回到 Step 1 的頁面，**Metric** 區塊設定：
   - **Statistic**：`Average`
   - **Period**：`5 minutes`
4. **Conditions** 區塊設定：
   - **Threshold type**：**Static**
   - **Whenever CPUUtilization is...**：選 **Greater**（>）
   - **than...**：輸入 `80`
   - 展開 **Additional configuration** → **Datapoints to alarm**：`2` out of `2`（連續 2 個 5 分鐘都超過 80% 才發出告警，避免短暫的尖峰造成誤報）
5. 按右下 **Next**。

**Step 2：Configure actions（告警發生時要做什麼）**——**就是原本指南「Notification：In alarm → 選 wp-alerts」那一步**

這一頁最上面是 **Notification** 區塊：

1. **Alarm state trigger**（在什麼狀態下通知）：選 **In alarm**。
   - *In alarm* ＝ CPU 超過門檻、出問題的時候 ← **選這個**
   - *OK* ＝ 恢復正常的時候
   - *Insufficient data* ＝ 資料不足的時候
2. **Send a notification to the following SNS topic**（要通知到哪個 SNS 主題）：選 **Select an existing SNS topic**（選擇現有的主題）。
   - 不要選 *Create new topic*，我們在 Part A 已經建好了。
3. 下面的 **Send a notification to…** 欄位：點一下，會出現主題清單，選 **`wp-alerts`**（也可以直接輸入 `wp-alerts` 搜尋）。
4. 選好後，欄位下方會顯示這個主題的 **Email (endpoints)**，確認裡面有你的 Email。
5. （選用）想在 CPU 恢復正常時也收到信：按 **Add notification**，Alarm state trigger 選 **OK**，主題一樣選 `wp-alerts`。
6. 下面的 *Lambda action*、*Auto Scaling action*、*EC2 action*、*Systems Manager action* 等區塊**都不用設定**，直接按最下方 **Next**。

**Step 3：Add name and description（命名）**

1. **Alarm name**：`wp-cpu-high`
2. **Alarm description**（選填）：`WordPress 主機 CPU 連續 10 分鐘超過 80%`
3. 按 **Next**。

**Step 4：Preview and create（確認並建立）**

1. 檢查摘要：Metric 是 `wp-web-1` 的 `CPUUtilization`、條件是 `> 80`、`2 out of 2`、通知送到 `wp-alerts`。
2. 拉到最下面按 **Create alarm**。

✅ **驗證 Part B**：

- **CloudWatch → Alarms → All alarms** 出現 `wp-cpu-high`。剛建立時 **State** 會顯示 *Insufficient data*，等 5–10 分鐘收集到資料後，會變成 **OK**（綠色）。
- 點進 `wp-cpu-high` → **Actions** 分頁，看到一筆 *When in alarm, send message to topic "wp-alerts"*。

（選用）**測試通知能不能收到信**：在 **CloudShell** 執行下面的指令，把告警「假裝」切換成 In alarm 狀態。幾分鐘內你會收到一封主旨含 `ALARM: "wp-cpu-high"` 的信；下一次 CloudWatch 評估時，狀態會自動恢復成 OK。

```bash
aws cloudwatch set-alarm-state --alarm-name wp-cpu-high \
  --state-value ALARM --state-reason "手動測試告警通知"
```

> 沒收到信？先檢查 Part A 的訂閱狀態是不是 **Confirmed**，再檢查垃圾郵件匣。

### 1.13 確認日誌有送到 CloudWatch

**目的**：確認主機上的 CloudWatch Agent（1.7 安裝）有把 Nginx 的日誌送到 CloudWatch。Phase 3 的 AI Agent 要從 CloudWatch 讀日誌做診斷，所以這條管道一定要通。

```
瀏覽器／curl 的請求 ──► Nginx 寫進主機上的 /var/log/nginx/access.log
                     ──► CloudWatch Agent 每隔幾秒讀取新的內容
                     ──► 送到 CloudWatch 的日誌群組 /wordpress/nginx-access
```

> 名詞：**Log group（日誌群組）**是一種日誌的資料夾，例如 `/wordpress/nginx-access`。**Log stream（日誌串流）**是資料夾裡的檔案，本指南用**主機 ID**（`i-` 開頭）命名，所以一台主機一個串流。

#### 步驟 1：在主機上製造一筆好認的紀錄

在 **Session Manager（root）** 執行。我們請求一個不存在、名稱很特別的網址，等一下在 CloudWatch 用這個名稱搜尋：

```bash
curl -s -o /dev/null -w "HTTP 狀態碼：%{http_code}\n" localhost/cloudwatch-test-page
tail -n 1 /var/log/nginx/access.log
```

正確結果：
- `HTTP 狀態碼：404`（找不到頁面，這正是我們要的）
- 最後一行是主機上的日誌，內容包含 `GET /cloudwatch-test-page` 和 `404`，例如：

```text
127.0.0.1 - - [26/Oct/2026:16:30:12 +0000] "GET /cloudwatch-test-page HTTP/1.1" 404 ...
```

> 為什麼看 `nginx-access`、不是 `nginx-error`？v3 的 Nginx 設定會直接對不存在的網址回應 404，這是正常行為，**不會寫進錯誤日誌**。`nginx-error` 和 `php-fpm-error` 只有真的發生錯誤時才會有內容，**現在是空的很正常**。Phase 3 的 3.8 會故意製造 PHP 錯誤，到時候就會看到。

#### 步驟 2：在 CloudWatch 主控台找到這筆紀錄

等大約 **1 分鐘**（Agent 傳送需要一點時間），然後：

1. 主控台上方搜尋 **CloudWatch**（確認右上角區域是東京）。
2. 左側選單 **Logs → Log groups**。
3. 在搜尋框輸入 `/wordpress`，會看到 1.6 建立的三個群組：
   - `/wordpress/nginx-access`：所有網頁請求 ← **點這個**
   - `/wordpress/nginx-error`：Nginx 錯誤
   - `/wordpress/php-fpm-error`：PHP 錯誤
4. 進入 `/wordpress/nginx-access` 後，下方 **Log streams** 分頁會列出一個名稱像 `i-0abc123def4567890` 的串流（就是 `wp-web-1` 的主機 ID）。**點這個名稱**。
   - 想確認是不是這台主機：EC2 → Instances，`wp-web-1` 的 **Instance ID** 要和串流名稱相同。
5. 進入後會看到一行一行的日誌。在上方的 **Filter events** 搜尋框輸入 `cloudwatch-test-page`，按 Enter。
6. 看到一筆包含 `GET /cloudwatch-test-page ... 404` 的紀錄，就成功了。

> 時間顯示：CloudWatch 預設用 **UTC** 時間，比台灣時間慢 8 小時。想改成台灣時間，點日誌列表右上角的齒輪（⚙）→ Timezone 選 **Local timezone**。

#### 步驟 3（選用）：用 CloudShell 一行確認

不想點主控台的話，可以在 **CloudShell** 執行：

```bash
aws logs filter-log-events --log-group-name /wordpress/nginx-access \
  --filter-pattern '"cloudwatch-test-page"' \
  --start-time $(( ($(date +%s) - 3600) * 1000 )) \
  --query "events[].[logStreamName, message]" --output text
```

正確結果：印出一行或多行，開頭是 `i-` 開頭的主機 ID，後面是 `GET /cloudwatch-test-page ... 404` 的紀錄。

✅ **驗證**：CloudWatch 的 `/wordpress/nginx-access` 中，可以找到剛剛 `cloudwatch-test-page` 的那筆紀錄。

**找不到紀錄？依序檢查：**

| 狀況 | 原因 | 解法（在 Session Manager，root） |
|---|---|---|
| 主機上的 `tail` 也沒有這筆紀錄 | Nginx 沒有寫日誌，或 curl 沒打到 Nginx | 執行 `systemctl is-active nginx`，必須是 `active` |
| Log groups 裡沒有 `/wordpress/...` | 1.6 漏做，或區域不是東京 | 回到 1.6 建立日誌群組 |
| 群組存在，但 **Log streams 是空的** | CloudWatch Agent 沒在執行，或沒有權限 | 執行 `systemctl is-active amazon-cloudwatch-agent`；若不是 `active`，執行 `systemctl restart amazon-cloudwatch-agent` |
| Agent 是 active，但還是沒有串流 | Agent 遇到權限或設定錯誤 | 執行 `tail -n 20 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log`。出現 `AccessDenied` 代表 `wp-ec2-role` 少了 `CloudWatchAgentServerPolicy`（回到 1.5）；出現 `ResourceNotFoundException` 代表日誌群組名稱打錯 |
| 有串流，但搜尋不到 | 還沒傳上去，或搜尋時間範圍太短 | 再等 1–2 分鐘；日誌畫面上方的時間範圍改成 **1h** 或更長 |

### ✅ Phase 1 完成檢查表

做完 1.1～1.13 後，用這一節確認每個環節都正確。**最快的方法是 A 的一鍵檢查**；想了解每一項怎麼看，或某一項出現 ❌，再看 B 的逐項說明。

#### A. 一鍵檢查（CloudShell）

在 **CloudShell** 貼上整段（不需要修改任何值，會自動抓取），大約 10 秒跑完：

```bash
REGION=ap-northeast-1
ok() { echo "✅ $1"; }
ng() { echo "❌ $1"; }

echo "=== 1. 網站 ==="
EIP=$(aws ec2 describe-addresses --region $REGION --query "Addresses[0].PublicIp" --output text)
CODE=$(curl -s -m 5 -o /dev/null -w "%{http_code}" "http://$EIP/")
[ "$CODE" = "200" ] && ok "http://$EIP 回應 200（WordPress 正常）" || ng "http://$EIP 回應 $CODE（000 代表連不到）"

echo "=== 2. 沒有開 Port 22 ==="
N22=$(aws ec2 describe-security-groups --region $REGION --filters Name=group-name,Values=wp-web-sg \
  --query 'length(SecurityGroups[0].IpPermissions[?(FromPort<=`22` && ToPort>=`22`) || IpProtocol==`"-1"`])' --output text)
[ "$N22" = "0" ] && ok "wp-web-sg 沒有開放 Port 22" || ng "wp-web-sg 有 $N22 條規則開放了 Port 22（或全部 Port）"

echo "=== 3. 資料庫不公開 ==="
PUB=$(aws rds describe-db-instances --region $REGION --db-instance-identifier wp-db \
  --query "DBInstances[0].PubliclyAccessible" --output text)
[ "$PUB" = "False" ] && ok "wp-db 的 Public access 為 No" || ng "wp-db 的 Public access 為 $PUB"

echo "=== 4. 密碼在 Secrets Manager ==="
aws secretsmanager describe-secret --region $REGION --secret-id wp/db-app >/dev/null 2>&1 \
  && ok "wp/db-app 存在" || ng "找不到 wp/db-app"
MASTER_ARN=$(aws rds describe-db-instances --region $REGION --db-instance-identifier wp-db \
  --query "DBInstances[0].MasterUserSecret.SecretArn" --output text)
aws secretsmanager describe-secret --region $REGION --secret-id "$MASTER_ARN" >/dev/null 2>&1 \
  && ok "RDS 管理員密碼（rds!db-...）存在" || ng "找不到 RDS 管理員密碼（1.3 是否選了 Managed in AWS Secrets Manager？）"

echo "=== 5. Session Manager 可連線 ==="
IID=$(aws ec2 describe-instances --region $REGION \
  --filters "Name=tag:Name,Values=wp-web-1" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)
PING=$(aws ssm describe-instance-information --region $REGION \
  --filters "Key=InstanceIds,Values=$IID" --query "InstanceInformationList[0].PingStatus" --output text)
[ "$PING" = "Online" ] && ok "$IID 的 SSM 狀態為 Online" || ng "$IID 的 SSM 狀態為 $PING"

echo "=== 6. CloudWatch 日誌與告警 ==="
NLOG=$(aws logs filter-log-events --region $REGION --log-group-name /wordpress/nginx-access \
  --start-time $(( ($(date +%s) - 86400) * 1000 )) --limit 5 --query "length(events)" --output text)
[ "${NLOG:-0}" -gt 0 ] 2>/dev/null && ok "最近 24 小時 /wordpress/nginx-access 有日誌" || ng "最近 24 小時 /wordpress/nginx-access 沒有日誌"
STATE=$(aws cloudwatch describe-alarms --region $REGION --alarm-names wp-cpu-high \
  --query "MetricAlarms[0].StateValue" --output text)
[ "$STATE" = "OK" ] && ok "wp-cpu-high 告警為 OK" || ng "wp-cpu-high 告警為 $STATE（剛建立的話 INSUFFICIENT_DATA 屬正常，等 10 分鐘再看）"

echo "=== 7.（建議）收回 RDS 管理員密碼的讀取權限 ==="
aws iam get-role-policy --role-name wp-ec2-role --policy-name wp-ec2-least-privilege \
  --query "PolicyDocument" --output json | grep -q 'rds!db' \
  && echo "⚠️  wp-ec2-role 仍可讀取 RDS 管理員密碼（建議照下方 B-7 收回）" \
  || ok "wp-ec2-role 已不能讀取 RDS 管理員密碼"
```

**正確結果**：1～6 全部是 ✅。第 7 項是建議，顯示 ⚠️ 時照 B-7 處理即可。

#### B. 逐項說明（出現 ❌ 時看這裡）

| # | 檢查項目 | 主控台怎麼看 | 正確的樣子 | 出現 ❌ 怎麼辦 |
|---|---|---|---|---|
| 1 | `http://<EIP>` 打得開 WordPress | 自己電腦的瀏覽器打開 `http://<EIP>` | 看到 WordPress 首頁 | 回應 `000`：看「[問題 5](#問題-5網站從網際網路連不上連線逾時)」；回應 `302`：WordPress 還沒安裝，做 1.11；回應 `500`：看 1.11 步驟 2 的表格 |
| 2 | `wp-web-sg` **沒有** Port 22 | VPC → Security groups → `wp-web-sg` → **Inbound rules** 分頁 | 只有一條：Type `HTTP`、Port `80`、Source `0.0.0.0/0` | 選取 SSH（22）或 All traffic 那條 → **Edit inbound rules** → **Delete** → **Save rules** |
| 3 | RDS 的 Public access 是 **No** | RDS → Databases → `wp-db` → **Connectivity & security** 分頁 → **Publicly accessible** | `No` | `wp-db` → **Modify** → Connectivity → Additional configuration → 選 **Not publicly accessible** → Continue → **Apply immediately** |
| 4 | 密碼都在 Secrets Manager | Secrets Manager → **Secrets** | 列表中有 `wp/db-app`，以及一個 `rds!db-` 開頭的密碼 | 缺 `wp/db-app`：重做 1.4；缺 `rds!db-...`：1.3 沒選「Managed in AWS Secrets Manager」，到 `wp-db` → **Modify** → Credentials management 改選 **Managed in AWS Secrets Manager** |
| 5 | 可以用 Session Manager 登入主機 | EC2 → Instances → 勾選 `wp-web-1` → **Connect** → **Session Manager** | **Connect** 按鈕可以按，按下後出現 `sh-5.2$` | 看「[問題 1](#問題-1ssh-連不上session-manager-也無法連線)」 |
| 6a | CloudWatch 收得到 Nginx 日誌 | CloudWatch → Logs → Log groups → `/wordpress/nginx-access` → `i-` 開頭的串流 | 有一行一行的請求紀錄 | 照 1.13 最後的排錯表檢查 |
| 6b | `wp-cpu-high` 告警為 OK | CloudWatch → Alarms → All alarms | `wp-cpu-high` 的 State 是綠色的 **OK** | *Insufficient data*：剛建立，等 10 分鐘；**In alarm**：主機 CPU 真的很高，或是剛才手動測試的告警還沒恢復，等 5–10 分鐘 |
| 7 | （建議）收回 RDS 管理員密碼權限 | IAM → Roles → `wp-ec2-role` → `wp-ec2-least-privilege` | 政策中沒有 `rds!db-*` | 照下面的 B-7 操作 |

#### B-7. 收回主機讀取 RDS 管理員密碼的權限（建議）

**為什麼要做**：主機只有在 1.10 建立 `wpuser` 時才需要 RDS 管理員密碼，之後 WordPress 都是用 `wpuser` 連線。把這個權限收回，萬一主機被入侵，攻擊者也拿不到資料庫的最高權限（簡報第 11 頁「IAM 最小權限」）。

⚠️ **請先確認 1.10 已經完成**（1.11 能正常安裝 WordPress 就代表完成了）。

在 **CloudShell** 執行：

```bash
# 1) 取出目前的政策
aws iam get-role-policy --role-name wp-ec2-role --policy-name wp-ec2-least-privilege \
  --query "PolicyDocument" --output json > /tmp/policy.json

# 2) 移除 rds!db-* 那一行，其他權限保持不變
jq '.Statement |= map(if .Sid == "ReadWordPressSecrets"
      then .Resource |= map(select(contains("rds!db") | not)) else . end)' \
  /tmp/policy.json > /tmp/policy-new.json

# 3) 先看一下修改後的內容：ReadWordPressSecrets 底下應該只剩 wp/* 那一行
jq '.Statement[] | select(.Sid == "ReadWordPressSecrets")' /tmp/policy-new.json

# 4) 套用
aws iam put-role-policy --role-name wp-ec2-role --policy-name wp-ec2-least-privilege \
  --policy-document file:///tmp/policy-new.json && echo "✅ 已收回"
```

**驗證**：再跑一次 A 的一鍵檢查，第 7 項變成 ✅。

> 主控台做法：IAM → Roles → `wp-ec2-role` → **Permissions** 分頁 → 點 `wp-ec2-least-privilege` → **Edit** → 切到 **JSON** → 刪除含 `rds!db-*` 的那一行（並刪掉上一行結尾的逗號）→ **Next** → **Save changes**。
>
> 之後如果需要重做 1.10，重跑一次 1.5 的指令就會把權限加回來。

**想先停在這裡？** 可以。但請記得：主機與資料庫即使沒人用也會持續計費，見「[清除所有資源](#清除所有資源避免持續扣款)」或「暫停省錢」小節。

---

## Phase 2：ALB + Auto Scaling + CloudFront + S3（高可用）

完成後：網站透過 `https://xxxx.cloudfront.net` 存取；主機壞掉會自動補一台；流量變大會自動加到 2 台；資料庫每天備份到 S3。

> 為什麼需要 EFS？多台主機時，使用者上傳的圖片只會存到其中一台。把 WordPress 的整個網站目錄放到 EFS（共用硬碟），所有主機看到的檔案就一模一樣。

### 2.1 建立 S3 備份桶並做第一次資料庫備份

**目的**：建立一個 S3「備份桶（bucket）」，每天把 WordPress 資料庫匯出成一個檔案存進去，舊備份 30 天後自動刪除。Phase 3 的 Lambda 程式也會放在同一個桶裡。

```
主機上的 wp db export（匯出資料庫）──► 壓縮成 .sql.gz ──► 上傳到 S3：wp-backup-帳號ID/db/
                                                    └─ 30 天後自動刪除（生命週期規則）
```

> 名詞：**Bucket（桶）**是 S3 存放檔案的最上層容器，名稱在全世界所有 AWS 帳號中必須是唯一的。**Prefix（前綴）**就像桶裡的資料夾，例如 `db/`。

> ⚠️ **桶的名稱必須是 `wp-backup-你的12位數帳號ID`**，例如 `wp-backup-123456789012`。1.5 給主機的 S3 權限只允許這個名稱，取別的名字，上傳時會出現 `403 Forbidden`。

**開始前確認**：
- Phase 1 完成檢查表的 1～6 項都是 ✅。
- 知道你的帳號 ID：主控台右上角點帳號名稱，**Account ID** 那一串 12 位數字（複製時去掉中間的 `-`）；或在 CloudShell 執行 `aws sts get-caller-identity --query Account --output text`。

#### 步驟 1：建立備份桶

下面兩種方法**擇一**即可。方法 B 比較快，也不會打錯名稱。

**方法 A：主控台**

1. 主控台上方搜尋 **S3**，確認右上角區域是 **東京（ap-northeast-1）**。
2. 左側 **General purpose buckets** → 右上 **Create bucket**。
3. **General configuration**：
   - **AWS Region**：Asia Pacific (Tokyo) ap-northeast-1
   - **Bucket type**：**General purpose**
   - **Bucket namespace**：選 **Global namespace**（⚠️ 不要選畫面標示 recommended 的 *Account Regional namespace*，原因見下方說明）
   - **Bucket name**：`wp-backup-` 加上你的 12 位數帳號 ID，例如 `wp-backup-123456789012`（全部小寫、不要有空格）

   > **為什麼這裡選 Global namespace？** 選 *Account Regional namespace* 的話，主控台會自動在名稱後面加上 `-帳號ID-ap-northeast-1-an`，實際名稱會變成類似 `wp-backup-123456789012-ap-northeast-1-an`。這樣就和 1.5 IAM 政策裡寫的 `wp-backup-帳號ID` 對不上，主機上傳備份時會出現 `403 Forbidden`，本指南後面的指令也都會找不到這個桶。Account Regional namespace 的好處是名稱不會被別的帳號搶走，但我們的名稱已經包含帳號 ID，本來就不容易撞名，所以選 Global 比較單純。
   >
   > 已經用 Account Regional namespace 建好了？桶還是空的，直接刪除重建最簡單：S3 → 勾選那個桶 → **Delete** → 輸入桶名稱確認 → 再照這一步用 Global namespace 重新建立。
4. **Object Ownership**：保持 **ACLs disabled (recommended)**。
5. **Block Public Access settings for this bucket**：保持 **Block all public access** 勾選（⚠️ 不要取消，備份檔絕對不能公開）。
6. **Bucket Versioning**：保持 **Disable**（開啟會保留每個舊版本，多佔空間）。
7. **Default encryption**：保持 **Server-side encryption with Amazon S3 managed keys (SSE-S3)**（免費的自動加密）。
8. 拉到最下面按 **Create bucket**。出現綠色的 *Successfully created bucket* 就完成了。

**方法 B：CloudShell 指令**

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET=wp-backup-$ACCOUNT_ID
aws s3api create-bucket --bucket $BUCKET --region ap-northeast-1 \
  --create-bucket-configuration LocationConstraint=ap-northeast-1
aws s3api put-public-access-block --bucket $BUCKET --public-access-block-configuration \
  BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
echo "✅ 已建立：$BUCKET"
```

- 出現 `BucketAlreadyOwnedByYou`：這個桶你之前已經建立過了（例如做附錄 A 時），可以直接往下做。
- 出現 `BucketAlreadyExists`：名稱被別人用走了，通常是帳號 ID 打錯，請確認名稱。

#### 步驟 2：設定「30 天自動刪除舊備份」（生命週期規則）

這條規則**只套用在 `db/` 資料夾**，桶裡的其他檔案（例如 Phase 3 的 Lambda 程式）不受影響。

**方法 A：主控台**

1. S3 → 點進 `wp-backup-帳號ID` → 上方分頁 **Management** → **Lifecycle rules** 區塊按 **Create lifecycle rule**。
2. **Lifecycle rule name**：`expire-db-30d`
3. **Choose a rule scope**：選 **Limit the scope of this rule using one or more filters**。
4. **Filter type → Prefix**：輸入 `db/`（⚠️ 結尾的斜線要打）。
5. **Lifecycle rule actions**：只勾 **Expire current versions of objects**。
6. 下方出現 **Days after object creation**：輸入 `30`。
7. 按右下 **Create rule**。

**方法 B：CloudShell 指令**

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws s3api put-bucket-lifecycle-configuration --bucket wp-backup-$ACCOUNT_ID \
  --lifecycle-configuration '{"Rules":[{"ID":"expire-db-30d","Status":"Enabled","Filter":{"Prefix":"db/"},"Expiration":{"Days":30}}]}'
echo "✅ 已設定生命週期規則"
```

#### 步驟 3：確認主機有寫入權限

主機的權限在 1.5 已經設定好（`wp-ec2-least-privilege` 中的 `BackupBucketObjects`、`BackupBucketList`），**不需要再做任何設定**。這一步只是先確認一下，避免下一步上傳失敗。

在 **Session Manager（root）** 執行：

```bash
export AWS_DEFAULT_REGION=ap-northeast-1
BUCKET=wp-backup-$(aws sts get-caller-identity --query Account --output text)
echo "test" | aws s3 cp - s3://$BUCKET/db/permission-test.txt && echo "✅ 主機可以寫入 $BUCKET"
```

- 出現 `✅ 主機可以寫入 wp-backup-...`：權限正確（這個 4 bytes 的測試檔 30 天後會自動刪除，也可以手動刪掉）。
- 出現 `AccessDenied` 或 `403 Forbidden`：桶名稱不對，或 1.5 的政策沒有加上。用 `aws s3 ls | grep wp-backup` 確認桶名稱，再重跑 1.5 的指令。
- 出現 `NoSuchBucket`：步驟 1 沒有建立成功，或建在其他區域。

#### 步驟 4：做第一次手動備份

繼續在 **Session Manager（root）**，一行一行執行：

```bash
export AWS_DEFAULT_REGION=ap-northeast-1
BUCKET=wp-backup-$(aws sts get-caller-identity --query Account --output text)
FILE=/tmp/wp-$(date +%F-%H%M).sql
sudo -u apache wp --path=/var/www/html db export "$FILE"
gzip "$FILE" && aws s3 cp "$FILE.gz" "s3://$BUCKET/db/" && rm -f "$FILE.gz"
```

正確畫面類似：

```text
Success: Exported to '/tmp/wp-2026-10-27-1503.sql'.
upload: ../../tmp/wp-2026-10-27-1503.sql.gz to s3://wp-backup-123456789012/db/wp-2026-10-27-1503.sql.gz
```

> 如果 `wp db export` 出現 `caching_sha2_password` 相關錯誤，請看 1.10 的說明，安裝 `mariadb-connector-c` 後再試一次。

#### 步驟 5：設定每天自動備份

繼續在 **Session Manager（root）**。這一段會做四件事：記下桶名稱、建立備份腳本、安裝排程工具（cron）、設定每天**台灣時間凌晨 3 點**執行：

```bash
# 1) 把桶名稱存成設定檔（自動帶入帳號 ID）
echo "wp-backup-$(aws sts get-caller-identity --query Account --output text)" > /etc/wp-backup-bucket
cat /etc/wp-backup-bucket

# 2) 建立備份腳本
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

# 3) 安裝並啟動 cron（Amazon Linux 2023 預設沒有安裝）
dnf install -y cronie && systemctl enable --now crond

# 4) 每天台灣時間 03:00 執行（主機時鐘是 UTC，用 CRON_TZ 指定台北時區）
printf 'CRON_TZ=Asia/Taipei\n0 3 * * * root /usr/local/bin/wp-backup.sh\n' > /etc/cron.d/wp-backup

# 5) 立刻手動跑一次，確認腳本可用
/usr/local/bin/wp-backup.sh && echo "BACKUP OK"
```

- 最後一行出現 `BACKUP OK` 就代表腳本可以正常備份。
- `dnf install` 會跑幾十秒，期間畫面有安裝訊息是正常的。

> 之後 Phase 2 做成 AMI 時，這個排程會跟著複製到每台主機。若 ASG 擴充到 2 台，兩台都會備份，檔名含主機名稱所以不會互相覆蓋。

✅ **驗證**：

**1. S3 裡有備份檔**（CloudShell）

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
aws s3 ls s3://wp-backup-$ACCOUNT_ID/db/ --human-readable
```

看到至少一個 `.sql.gz` 檔案，大小通常是幾十 KB 到幾 MB（剛安裝的 WordPress 大約 100–300 KB）。主控台做法：S3 → `wp-backup-帳號ID` → 點 `db/` 資料夾。

**2. 備份檔內容是完整的 SQL**（CloudShell）

**為什麼要檢查**：驗證 1 只證明 S3 裡「有檔案」。檔案也可能是空的或只匯出到一半，例如資料庫連線失敗時，就可能只產生一個很小的檔案。真正需要還原時才發現備份是壞的就太晚了，所以要打開檔案看看內容。

**備份檔是什麼**：`wp db export` 會把整個 WordPress 資料庫轉成一份 **SQL 文字檔**，裡面是一行行「建立資料表（`CREATE TABLE`）」和「寫入資料（`INSERT INTO`）」的指令。只要把這份檔案匯回資料庫，就能完整重建網站內容。為了節省空間，我們用 `gzip` 把它壓縮成 `.sql.gz`。

**檢查方式**：不需要下載到自己電腦。下面的指令會直接從 S3 讀取檔案、在記憶體中解壓縮，只把需要看的部分印在畫面上。

在 **CloudShell** 依序執行。

**步驟 2-1：自動找出最新的一個備份檔**

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET=wp-backup-$ACCOUNT_ID
LATEST=$(aws s3 ls s3://$BUCKET/db/ | grep '\.sql\.gz$' | sort | tail -n 1 | awk '{print $4}')
echo "最新的備份檔：$LATEST"
```

| 指令片段 | 意思 |
|---|---|
| `aws s3 ls s3://$BUCKET/db/` | 列出 `db/` 資料夾裡的所有檔案（每行是：日期、時間、大小、檔名） |
| `grep '\.sql\.gz$'` | 只留下 `.sql.gz` 結尾的備份檔，排除權限測試用的 `permission-test.txt` |
| `sort \| tail -n 1` | 依日期排序後取最後一個，也就是最新的 |
| `awk '{print $4}'` | 只取第 4 欄的檔名 |

正確結果：印出一個檔名，例如 `最新的備份檔：wp-2026-10-27-1503.sql.gz`。如果冒號後面是空的，代表 `db/` 裡還沒有備份檔，請回到步驟 4 或 5 重新備份。

**步驟 2-2：看檔案開頭，確認是資料庫匯出檔**

```bash
aws s3 cp s3://$BUCKET/db/$LATEST - | gunzip | head -n 5
```

| 指令片段 | 意思 |
|---|---|
| `aws s3 cp s3://.../$LATEST -` | 從 S3 讀取檔案。最後的 `-` 代表「不存成檔案，直接輸出到畫面」 |
| `\| gunzip` | 把 gzip 壓縮的內容解壓縮回文字 |
| `\| head -n 5` | 只顯示前 5 行 |

正確結果類似（版本號碼可能不同）：

```text
/*M!999999\- enable the sandbox mode */
-- MariaDB dump 10.19  Distrib 10.5.x-MariaDB, for Linux (x86_64)
--
-- Host: wp-db.xxxxxxxx.ap-northeast-1.rds.amazonaws.com    Database: wordpress
-- ------------------------------------------------------
```

重點是看到 **`-- MariaDB dump`**（或 `-- MySQL dump`），以及 **`Database: wordpress`**。

**步驟 2-3：數一數有幾個資料表**

```bash
aws s3 cp s3://$BUCKET/db/$LATEST - | gunzip | grep -c "CREATE TABLE"
```

- `grep -c "CREATE TABLE"`：計算檔案中出現幾次「CREATE TABLE」，也就是備份了幾個資料表。
- 正確結果：**`12`** 左右。剛安裝好的 WordPress 預設有 12 個資料表，裝了外掛之後會更多。
- 如果是 `0`，代表檔案裡沒有任何資料表，備份是空的。

想看是哪些資料表，可以再執行：

```bash
aws s3 cp s3://$BUCKET/db/$LATEST - | gunzip | grep "CREATE TABLE"
```

會列出 `wp_posts`（文章）、`wp_users`（使用者）、`wp_options`（網站設定）等資料表。

**步驟 2-4：看檔案結尾，確認有完整匯出**

```bash
aws s3 cp s3://$BUCKET/db/$LATEST - | gunzip | tail -n 1
```

- `tail -n 1`：只顯示最後一行。
- 正確結果類似：`-- Dump completed on 2026-10-27 15:03:12`
- **`Dump completed`** 代表匯出有完整跑完。如果最後一行不是這句，表示匯出中途被中斷，檔案不完整。

**判斷結果**

| 檢查 | 正確 | 有問題時 |
|---|---|---|
| 2-1 找到檔案 | 印出 `.sql.gz` 檔名 | 回到步驟 4 或 5 重新備份 |
| 2-2 開頭 | `-- MariaDB dump`、`Database: wordpress` | 出現 `gzip: stdin: not in gzip format`：檔案壞了，重新備份 |
| 2-3 資料表數量 | 12 個以上 | 是 `0`：資料庫匯出失敗，在 Session Manager 執行 `/usr/local/bin/wp-backup.sh` 看錯誤訊息 |
| 2-4 結尾 | `-- Dump completed on ...` | 沒有這行：匯出被中斷，重新備份 |

四項都正確，這份備份就可以用來還原。

**3. 桶的設定正確**（CloudShell）

這段可以**單獨執行**，不需要先跑前面的指令：它會自己取得帳號 ID、確認桶存在，再逐項檢查。整段貼上即可：

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET=wp-backup-$ACCOUNT_ID
echo "檢查的桶：$BUCKET"

# ① 桶存在，而且在東京區域
REGION=$(aws s3api get-bucket-location --bucket $BUCKET --query LocationConstraint --output text 2>/dev/null)
[ "$REGION" = "ap-northeast-1" ] && echo "✅ 桶存在，位於 ap-northeast-1" \
  || echo "❌ 找不到桶或區域不對（結果：${REGION:-不存在}）。用 aws s3 ls | grep wp-backup 查看實際名稱"

# ② 完全不公開（四項封鎖都要是 True）
aws s3api get-public-access-block --bucket $BUCKET \
  --query "PublicAccessBlockConfiguration" --output table
PAB=$(aws s3api get-public-access-block --bucket $BUCKET --query \
  "PublicAccessBlockConfiguration.[BlockPublicAcls,IgnorePublicAcls,BlockPublicPolicy,RestrictPublicBuckets]" --output text 2>/dev/null)
[ "$(echo $PAB)" = "True True True True" ] && echo "✅ 已封鎖所有公開存取" || echo "❌ 公開存取封鎖不完整：$PAB"

# ③ 30 天自動刪除規則只套用在 db/
aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET \
  --query "Rules[].[ID, Filter.Prefix, Expiration.Days, Status]" --output table
RULE=$(aws s3api get-bucket-lifecycle-configuration --bucket $BUCKET \
  --query "Rules[?Filter.Prefix=='db/' && Status=='Enabled'].Expiration.Days | [0]" --output text 2>/dev/null)
[ "$RULE" = "30" ] && echo "✅ db/ 的檔案 30 天後自動刪除" || echo "❌ 找不到 db/ 的 30 天規則（結果：${RULE:-無}），請重做步驟 2"

# ④ 預設加密
ENC=$(aws s3api get-bucket-encryption --bucket $BUCKET \
  --query "ServerSideEncryptionConfiguration.Rules[0].ApplyServerSideEncryptionByDefault.SSEAlgorithm" --output text 2>/dev/null)
[ -n "$ENC" ] && [ "$ENC" != "None" ] && echo "✅ 預設加密：$ENC" || echo "❌ 沒有預設加密"
```

**每一項在檢查什麼：**

| 項目 | 檢查內容 | 為什麼重要 |
|---|---|---|
| ① | 桶存在，而且在東京區域 | 名稱或區域不對，後面所有指令都會出現 `NoSuchBucket` |
| ② | 「封鎖公開存取」的四個開關都是 `True` | 備份檔包含整個資料庫，包括使用者帳號和密碼雜湊，絕對不能公開 |
| ③ | `db/` 資料夾有 30 天自動刪除規則，而且是啟用（Enabled）狀態 | 避免備份無限累積、持續增加儲存費用 |
| ④ | 有預設加密（`AES256` 就是 SSE-S3） | 檔案在 AWS 硬碟上是加密儲存的 |

**正確結果**（中間會印出兩個表格，最後四行如下）：

```text
檢查的桶：wp-backup-123456789012
✅ 桶存在，位於 ap-northeast-1
✅ 已封鎖所有公開存取
✅ db/ 的檔案 30 天後自動刪除
✅ 預設加密：AES256
```

中間的兩個表格分別長這樣：

```text
-----------------------------------
|      GetPublicAccessBlock       |
+-------------------------+-------+
|  BlockPublicAcls        |  True |
|  BlockPublicPolicy      |  True |
|  IgnorePublicAcls       |  True |
|  RestrictPublicBuckets  |  True |
+-------------------------+-------+
----------------------------------------
|    GetBucketLifecycleConfiguration   |
+----------------+------+-----+--------+
|  expire-db-30d |  db/ |  30 | Enabled|
+----------------+------+-----+--------+
```

**出現 ❌ 時：**
- ①：用 `aws s3 ls | grep wp-backup` 列出實際的桶名稱。名稱後面多了 `-ap-northeast-1-an`，代表建立時選了 Account Regional namespace，請照步驟 1 的說明刪除後重建；什麼都沒列出，代表桶沒有建立成功，請重做步驟 1。
- ②：S3 → 點進桶 → **Permissions** 分頁 → **Block public access (bucket settings)** → **Edit** → 勾選 **Block all public access** → **Save changes**（或執行步驟 1 方法 B 的 `put-public-access-block` 那一行）。
- ③：重做步驟 2。注意 Prefix 要是 `db/`，結尾有斜線。
- ④：S3 → 點進桶 → **Properties** 分頁 → **Default encryption** → **Edit** → 選 **SSE-S3** → **Save changes**。

**4. 自動排程已設定**（Session Manager，root）

```bash
systemctl is-active crond
cat /etc/cron.d/wp-backup
```

顯示 `active`，以及 `CRON_TZ=Asia/Taipei` 和 `0 3 * * * root /usr/local/bin/wp-backup.sh` 兩行。隔天早上可以再執行一次驗證 1，應該會多一個以當天日期命名的檔案。

> RDS 本身也會每天自動快照（1.3 設了保留 1 天）；S3 上的 SQL 備份是第二道保險，也方便搬家。

> **想從備份還原？**（練習時不需要做，記得有這個方法就好）在 Session Manager（root）執行：
> `aws s3 cp s3://$BUCKET/db/<檔名>.sql.gz - | gunzip | sudo -u apache wp --path=/var/www/html db import -`
> ⚠️ 還原會**覆蓋**目前資料庫的所有內容。

### 2.2 建立 EFS 共用硬碟並搬移網站檔案

**目的**：Phase 2 之後，網站會由**多台主機**一起服務（2.7 的 Auto Scaling）。如果網站檔案還放在各台主機自己的硬碟上，使用者在 A 主機上傳的圖片，B 主機就看不到。所以要把整個網站目錄 `/var/www/html` 搬到 **EFS（共用網路硬碟）**，讓每台主機看到的檔案都一模一樣。

```
搬移前：EC2 ── 自己的 EBS 硬碟：/var/www/html
搬移後：EC2-1 ─┐
              ├──► EFS（共用）：/var/www/html   ← 所有主機都看到同一份檔案
       EC2-2 ─┘
       資料庫仍在 RDS，不受影響
```

> 名詞：
> - **EFS（Elastic File System）**：可同時被多台 EC2 掛載的網路硬碟，容量自動增減，用多少付多少。
> - **NFS**：EFS 使用的網路檔案傳輸方式，固定走 **TCP 2049** 這個 Port。
> - **Mount target（掛載點）**：EFS 在每個可用區（AZ）的「插座」。主機要透過同一個 AZ 的掛載點連到 EFS，所以兩個 AZ 都要建。
> - **掛載（mount）**：把網路硬碟接到主機上的某個資料夾，之後讀寫那個資料夾，就是在讀寫 EFS。

> 費用：EFS 依實際用量計費，東京區域約每 GB 每月 $0.36。WordPress 剛安裝大約 100 MB，每月不到 $0.1。

整個 2.2 分成 4 個步驟：① 建立 EFS 專用的安全群組 → ② 建立 EFS → ③ 把網站檔案搬到 EFS → ④ 驗證。

---

#### 步驟 1：建立 EFS 專用的安全群組 `wp-efs-sg`

**為什麼需要**：EFS 也有自己的防火牆。我們要規定**只有 WordPress 主機（掛著 `wp-web-sg` 的主機）**可以用 NFS（Port 2049）連進來，其他任何人都連不到。這和 1.2 的 `wp-db-sg` 是同樣的做法。

```
wp-web-sg 的主機 ──NFS（TCP 2049）──► wp-efs-sg 的 EFS   ✅ 允許
其他任何來源      ──────────────────► wp-efs-sg 的 EFS   ❌ 擋下
```

下面兩種方法**擇一**即可。

**方法 A：主控台**

1. 主控台上方搜尋 **VPC** → 左側 **Security groups** → 右上 **Create security group**。
2. **Basic details**：
   - **Security group name**：`wp-efs-sg`
   - **Description**：`WordPress EFS`（必填，只能用英文）
   - **VPC**：⚠️ **點開下拉選單，把預設的 default VPC 改成 `wp-vpc`**（顯示為 `vpc-xxxxxxxx (wp-vpc)`）。一定要先改 VPC，下一步才搜尋得到 `wp-web-sg`。
3. **Inbound rules** → **Add rule**：
   - **Type**：下拉選單選 **NFS**（Protocol 和 Port range 會自動帶入 `TCP`、`2049`）
   - **Source**：選 **Custom**，在旁邊的搜尋框輸入 `wp-web` → 選 `sg-xxxxxxxx (wp-web-sg)`
   - **Description**（選填）：`NFS from WordPress servers`
4. **Outbound rules**：保持預設，不用改。
5. 拉到最下面按 **Create security group**。

**方法 B：CloudShell 指令**

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=wp-vpc --query "Vpcs[0].VpcId" --output text)
WEB_SG=$(aws ec2 describe-security-groups --filters Name=vpc-id,Values=$VPC_ID Name=group-name,Values=wp-web-sg \
  --query "SecurityGroups[0].GroupId" --output text)
EFS_SG=$(aws ec2 create-security-group --group-name wp-efs-sg --description "WordPress EFS" \
  --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $EFS_SG --protocol tcp --port 2049 --source-group $WEB_SG
echo "✅ 已建立 wp-efs-sg：$EFS_SG（允許來自 wp-web-sg $WEB_SG 的 NFS 2049）"
```

✅ **驗證步驟 1**（CloudShell）：

```bash
aws ec2 describe-security-groups --filters Name=group-name,Values=wp-efs-sg \
  --query "SecurityGroups[0].{Name:GroupName, VPC:VpcId, Port:IpPermissions[0].FromPort, Source:IpPermissions[0].UserIdGroupPairs[0].GroupId}" \
  --output table
aws ec2 describe-security-groups --filters Name=group-name,Values=wp-web-sg --query "SecurityGroups[0].GroupId" --output text
```

正確結果：
- 表格中 **Port** 是 `2049`。
- **VPC** 和 `wp-vpc` 的 ID 相同。
- **Source** 和第二行印出的 `wp-web-sg` ID 相同。

---

#### 步驟 2：建立 EFS 檔案系統 `wp-efs`

1. 主控台上方搜尋 **EFS**，確認右上角區域是東京。
2. 按 **Create file system**，在跳出的小視窗中按左下角 **Customize**（⚠️ 不要直接按 Create，那樣會建在 default VPC，安全群組也不對）。

**Step 1：File system settings（一般設定）**

| 欄位 | 設定 | 說明 |
|---|---|---|
| **Name** | `wp-efs` | |
| **File system type** | **Regional** | 資料同時存放在多個 AZ，一個機房故障也不會遺失 |
| **Automatic backups** | 取消勾選 | 練習用可省錢；網站資料庫已經有 2.1 的備份 |
| **Lifecycle management** | 保持預設 | 很久沒讀取的檔案會自動移到較便宜的儲存層 |
| **Encryption** | 保持勾選 **Enable encryption of data at rest** | 免費的靜態加密 |
| **Throughput mode** | **Elastic**（預設） | 依實際用量自動調整速度 |
| **Performance mode** | **General Purpose**（預設） | |

按 **Next**。

**Step 2：Network access（網路設定）**

1. **Virtual Private Cloud (VPC)**：⚠️ 改成 **`wp-vpc`**（預設是 default VPC）。
2. 下方 **Mount targets** 會出現兩列，每個 AZ 一列，逐列設定：

| Availability zone | Subnet ID | Security groups |
|---|---|---|
| `ap-northeast-1a` | 選 **`wp-subnet-public1-ap-northeast-1a`** | 按 **X** 移除 `default`，再選 **`wp-efs-sg`** |
| `ap-northeast-1c` | 選 **`wp-subnet-public2-ap-northeast-1c`** | 按 **X** 移除 `default`，再選 **`wp-efs-sg`** |

   - **IP address** 保持 Automatic。
   - 為什麼選 public 子網路？因為 WordPress 主機都在 public 子網路，掛載點放在相同的子網路最單純。EFS 本身**不會**因此對外公開，它沒有公有 IP，而且只有 `wp-efs-sg` 允許的主機連得到。
3. 按 **Next**。

**Step 3：File system policy（選用）**：全部保持空白，按 **Next**。

**Step 4：Review and create**：檢查 VPC 是 `wp-vpc`、兩個掛載點的安全群組都是 `wp-efs-sg` → 按 **Create**。

建立後回到 EFS 列表：
- 記下 **File system ID**（`fs-` 開頭，例如 `fs-0123456789abcdef0`），填進「準備 5：數值記錄表」。
- 點進 `wp-efs` → **Network** 分頁，等兩個掛載點的 **Mount target state** 都從 *Creating* 變成 **Available**，大約 **2–5 分鐘**。⚠️ 還沒變成 Available 之前，步驟 3 的掛載會失敗。

✅ **驗證步驟 2**（CloudShell）：

```bash
EFS_ID=$(aws efs describe-file-systems --query "FileSystems[?Name=='wp-efs'].FileSystemId | [0]" --output text)
echo "EFS ID：$EFS_ID"
for MT in $(aws efs describe-mount-targets --file-system-id $EFS_ID --query "MountTargets[].MountTargetId" --output text); do
  STATE=$(aws efs describe-mount-targets --mount-target-id $MT --query "MountTargets[0].[AvailabilityZoneName, LifeCycleState]" --output text)
  SG=$(aws efs describe-mount-target-security-groups --mount-target-id $MT --query "SecurityGroups[0]" --output text)
  SGNAME=$(aws ec2 describe-security-groups --group-ids $SG --query "SecurityGroups[0].GroupName" --output text)
  echo "掛載點 $MT：$STATE，安全群組 $SGNAME"
done
```

正確結果類似：

```text
EFS ID：fs-0123456789abcdef0
掛載點 fsmt-0aaa...：ap-northeast-1a	available，安全群組 wp-efs-sg
掛載點 fsmt-0bbb...：ap-northeast-1c	available，安全群組 wp-efs-sg
```

兩個掛載點都要是 `available`，安全群組都要是 `wp-efs-sg`。**記下 EFS ID，步驟 3 要用。**

---

#### 步驟 3：把網站檔案搬到 EFS

**這一步在 Session Manager（root）執行**（提示字元是 `[root@ip-... ~]#`）。

> ⚠️ 搬移過程中，網站會中斷約 **1 分鐘**（步驟 3-4 到 3-6 之間）。練習環境沒有影響；正式網站請選離峰時間。

**3-1：設定 EFS ID 並確認工具已安裝**

把 `fs-xxxxxxxx` 換成步驟 2 記下的 EFS ID：

```bash
EFS_ID=fs-xxxxxxxx
echo "要掛載的 EFS：$EFS_ID"
rpm -q amazon-efs-utils || dnf install -y amazon-efs-utils
dnf install -y rsync
```

- `amazon-efs-utils`：讓主機可以用 `mount -t efs` 掛載 EFS，並使用 TLS 加密傳輸。1.7 的 User data 已經安裝過，這裡只是確認。
- `rsync`：複製檔案的工具，會完整保留檔案的擁有者與權限（Amazon Linux 2023 不一定預裝）。

**3-2：先掛到暫存位置，測試連得上**

```bash
mkdir -p /mnt/efs
mount -t efs -o tls $EFS_ID:/ /mnt/efs && echo "✅ 掛載成功"
df -h /mnt/efs
```

- 正確結果：出現 `✅ 掛載成功`，`df` 的 Filesystem 欄位顯示 `127.0.0.1:/`（使用 TLS 掛載時就會是這個樣子）。
- 如果指令**卡住超過 1 分鐘**：按 Ctrl+C 取消，然後檢查三件事：
  1. 步驟 1 的 `wp-efs-sg` 有沒有允許來自 `wp-web-sg` 的 2049。
  2. 步驟 2 的掛載點是否都已經是 Available。
  3. EFS ID 有沒有打錯。

**3-3 到 3-6 在做什麼？先看整體流程**

可以把它想成「搬家」：網站檔案原本住在主機自己的硬碟（舊家），現在要搬到 EFS（新家）。做法是**先把東西複製到新家 → 暫停營業 → 把門牌換成新家的地址 → 重新營業**。舊家的東西保留不丟，萬一新家有問題，可以馬上搬回去。

```
3-3 複製（網站照常營業）
    主機硬碟 /var/www/html  ──rsync 複製──►  EFS（暫時接在 /mnt/efs）
3-4 暫停營業、舊家改名
    停止 nginx、php-fpm；/var/www/html 改名為 /var/www/html.local-backup；建立空的 /var/www/html
3-5 換門牌
    把 EFS 接到 /var/www/html，並寫進「開機自動接上」的清單
3-6 重新營業
    啟動 nginx、php-fpm，從 /var/www/html 讀到的已經是 EFS 上的檔案
```

| 子步驟 | 網站狀態 | 要多久 |
|---|---|---|
| 3-3 | ✅ 正常服務 | 10–60 秒 |
| 3-4 | ❌ 暫停（服務已停止） | 幾秒 |
| 3-5 | ❌ 暫停 | 幾秒 |
| 3-6 | ✅ 恢復服務（改用 EFS） | 幾秒 |

所以網站實際中斷的時間，只有 3-4 到 3-6 之間的不到 1 分鐘。**3-4、3-5、3-6 請連續做完，不要中途停下來。**

**3-3：把網站檔案複製到 EFS**

延續 3-2，EFS 現在暫時接在 `/mnt/efs`。一行一行執行：

```bash
rsync -a /var/www/html/ /mnt/efs/
```

- **作用**：把 `/var/www/html` 裡的所有檔案，複製到 `/mnt/efs`（也就是 EFS）。
- `rsync`：專門用來複製或同步檔案的工具，比 `cp` 更適合大量檔案。
- `-a`（archive，封存模式）：完整保留檔案的**擁有者**（`apache`）、**權限**、**修改時間**和子資料夾結構。少了它，複製過去的檔案擁有者會變成 root，WordPress 之後就無法上傳圖片或更新外掛。
- ⚠️ `/var/www/html/` **結尾的 `/` 一定要有**：
  - 有 `/`：複製「資料夾**裡面**的內容」，結果是 `/mnt/efs/index.php` ✅
  - 沒有 `/`：複製「整個資料夾」，結果會多一層，變成 `/mnt/efs/html/index.php` ❌，網站會找不到檔案。
- **正確畫面**：執行期間沒有任何輸出，10–60 秒後回到 `#` 提示字元就是完成了。沒有訊息就代表沒有錯誤。

```bash
echo "原本的檔案數：$(find /var/www/html -type f | wc -l)"
echo "EFS 上的檔案數：$(find /mnt/efs -type f | wc -l)"
```

- **作用**：分別數一數兩邊各有幾個檔案，確認有沒有全部複製過去。
- `find 資料夾 -type f`：列出資料夾裡所有的檔案（`-type f` 代表只算檔案，不算資料夾）。
- `| wc -l`：計算有幾行，也就是有幾個檔案。
- **正確畫面**：兩個數字**完全相同**，例如：

```text
原本的檔案數：3412
EFS 上的檔案數：3412
```

- 數字不同時：再執行一次 `rsync -a /var/www/html/ /mnt/efs/`，它只會補上缺少的檔案，然後重新比對。
- EFS 的數字比較大，而且多出的剛好是一整份：可能是之前漏打結尾的 `/`，多了一層 `html` 資料夾。請執行 `ls /mnt/efs`，如果看到裡面有 `html` 資料夾，用 `rm -rf /mnt/efs/html` 刪掉後再比對一次。

```bash
umount /mnt/efs
```

- **作用**：把 EFS 從暫存位置 `/mnt/efs` 卸下。EFS 上的檔案不會因此消失，只是先拔掉連線，3-5 會改接到正式位置 `/var/www/html`。
- **正確畫面**：沒有任何輸出。
- 出現 `target is busy`：代表你目前所在的資料夾就在 `/mnt/efs` 裡面。先執行 `cd ~` 離開，再重新 `umount`。

**做完 3-3 的狀態**：EFS 上已經有一份完整的網站檔案，但網站仍然在用主機自己硬碟上的那份，所以照常運作。

**3-4：停止網站服務，把舊目錄改名保留**

⚠️ 從這一步開始網站會暫停服務，請接著把 3-4、3-5、3-6 連續做完。

```bash
systemctl stop nginx php-fpm
```

- **作用**：停止網頁伺服器（nginx）和 PHP 程式（php-fpm）。
- **為什麼要停**：下一步要把網站目錄換掉。如果服務還在跑，可能有程式正在讀取舊目錄的檔案，換到一半時就會出錯。先停下來最安全。
- **正確畫面**：沒有任何輸出。此時瀏覽器打開網站會連不上，這是預期中的狀況。

```bash
mv /var/www/html /var/www/html.local-backup
```

- **作用**：把舊的網站目錄**改名**為 `html.local-backup`。
- `mv`：移動或改名。這裡是在同一個位置改名，所以只要一瞬間，檔案完全不會被刪除。
- **為什麼不直接刪掉**：保留一份舊資料當作保險。萬一 EFS 有問題，用最後的「還原方法」就能立刻切回來。
- **正確畫面**：沒有任何輸出。

```bash
mkdir -p /var/www/html
```

- **作用**：建立一個新的、**空的** `/var/www/html` 資料夾。
- **為什麼需要**：掛載（mount）需要一個「接口」，也就是一個已經存在的空資料夾。下一步會把 EFS 接到這個資料夾上。
- `-p`：資料夾已經存在時不會報錯。
- **正確畫面**：沒有任何輸出。

**做完 3-4 的狀態**：網站暫停；舊檔案安全地放在 `/var/www/html.local-backup`；`/var/www/html` 是一個空資料夾，等著接上 EFS。

**3-5：設定開機自動掛載，並立刻掛上**

```bash
echo "$EFS_ID:/ /var/www/html efs _netdev,tls,nofail 0 0" >> /etc/fstab
```

- **作用**：在 `/etc/fstab` 這個檔案的最後面加上一行設定。
- `/etc/fstab`：Linux 的「**開機自動掛載清單**」。主機每次開機都會讀這個檔案，把列在裡面的硬碟自動接上。寫進這裡，主機重開機後 EFS 也會自動接回 `/var/www/html`。之後 2.6 製作的 AMI 會帶著這個設定，所以 ASG 新開的主機也能自動接上 EFS。
- `>>`：**加在檔案最後面**。⚠️ 是兩個 `>`。只打一個 `>` 會把整個 fstab 覆蓋掉，主機可能因此無法開機。
- 這一行設定的六個欄位：

| 欄位 | 內容 | 意思 |
|---|---|---|
| 1 | `fs-xxxxxxxx:/` | 要接上的硬碟：你的 EFS 的根目錄（`$EFS_ID` 會自動換成 3-1 設定的 ID） |
| 2 | `/var/www/html` | 要接到哪個資料夾 |
| 3 | `efs` | 硬碟類型，使用 amazon-efs-utils 掛載 |
| 4 | `_netdev,tls,nofail` | 選項：`_netdev` 等網路就緒後再掛載；`tls` 加密傳輸；`nofail` 是萬一 EFS 連不上，主機仍可正常開機，方便你連進來除錯 |
| 5 | `0` | 不需要備份工具處理（固定填 0） |
| 6 | `0` | 開機時不做磁碟檢查（網路硬碟固定填 0） |

- **正確畫面**：沒有任何輸出。
- ⚠️ **這一行只能執行一次**。重複執行會在 fstab 裡加入重複的設定（步驟 4 的驗證第 2 項會檢查出來）。

```bash
tail -n 1 /etc/fstab
```

- **作用**：顯示 fstab 的最後一行，確認剛才加的內容正確。
- **正確畫面**：顯示類似下面這行，**開頭必須是 `fs-` 加上你的 EFS ID**：

```text
fs-0123456789abcdef0:/ /var/www/html efs _netdev,tls,nofail 0 0
```

- 如果開頭是 `:/`（沒有 `fs-...`）：代表 `$EFS_ID` 變數是空的，可能是 Session Manager 重新連線過。先執行 `sed -i '$d' /etc/fstab` 刪掉最後這一行錯誤的設定，再回到 3-1 重新設定 `EFS_ID`，然後重做這一步。

```bash
mount -a
```

- **作用**：依照 fstab 清單，**立刻**把所有還沒掛上的硬碟都接上。這樣不用重開機，EFS 現在就會接到 `/var/www/html`。
- **正確畫面**：沒有任何輸出就代表成功。
- 卡住超過 1 分鐘：按 Ctrl+C，回到 3-2 的檢查項目。這時網站仍在暫停中，可以用最後的「還原方法」先恢復服務。

**做完 3-5 的狀態**：`/var/www/html` 已經接上 EFS，裡面是 3-3 複製過去的網站檔案，但服務還沒啟動。

**3-6：重新啟動網站服務**

```bash
systemctl start php-fpm nginx
```

- **作用**：重新啟動 PHP（php-fpm）和網頁伺服器（nginx）。
- **為什麼先啟動 php-fpm**：nginx 收到 PHP 網頁的請求時，會轉交給 php-fpm 處理。先讓 php-fpm 就緒，nginx 一啟動就能正常服務。
- **正確畫面**：沒有任何輸出。

```bash
systemctl is-active php-fpm nginx
```

- **作用**：確認兩個服務都在執行。
- **正確畫面**：

```text
active
active
```

- 若出現 `failed`：執行 `systemctl status nginx --no-pager`（或把 `nginx` 換成 `php-fpm`）查看原因，或直接用最後的「還原方法」切回原本的目錄。

**做完 3-6 的狀態**：網站恢復服務，而且讀取的已經是 **EFS 上的檔案**。接著做步驟 4 的驗證，確認一切正確。

---

#### 步驟 4：驗證

在 **Session Manager（root）** 貼上整段：

```bash
echo "=== 1. 網站目錄掛在 EFS 上 ==="
findmnt -n -o SOURCE,FSTYPE /var/www/html | grep -q "nfs4" \
  && echo "✅ /var/www/html 已掛載 EFS（$(findmnt -n -o SOURCE,FSTYPE /var/www/html)）" \
  || echo "❌ /var/www/html 不是 EFS"

echo "=== 2. 開機會自動掛載 ==="
[ "$(grep -c ' /var/www/html efs ' /etc/fstab)" = "1" ] && echo "✅ /etc/fstab 有一行 EFS 設定" \
  || echo "❌ /etc/fstab 的 EFS 設定數量：$(grep -c ' /var/www/html efs ' /etc/fstab)（應該剛好 1 行）"

echo "=== 3. 檔案完整 ==="
A=$(find /var/www/html.local-backup -type f | wc -l); B=$(find /var/www/html -type f | wc -l)
[ "$A" = "$B" ] && echo "✅ 檔案數一致：$B" || echo "❌ 檔案數不同：原本 $A，EFS $B"
ls -l /var/www/html/wp-config.php | grep -q "apache apache" && echo "✅ wp-config.php 擁有者為 apache" || echo "❌ wp-config.php 擁有者不對"

echo "=== 4. WordPress 可以寫入 EFS ==="
sudo -u apache touch /var/www/html/wp-content/efs-write-test && rm -f /var/www/html/wp-content/efs-write-test \
  && echo "✅ apache 可以寫入" || echo "❌ apache 無法寫入"

echo "=== 5. 網站正常 ==="
[ "$(curl -s localhost/healthz)" = "ok" ] && echo "✅ /healthz 正常" || echo "❌ /healthz 失敗"
CODE=$(curl -s -o /dev/null -w "%{http_code}" localhost/wp-login.php)
[ "$CODE" = "200" ] && echo "✅ WordPress 登入頁回應 200" || echo "❌ WordPress 登入頁回應 $CODE"
```

正確結果：

```text
=== 1. 網站目錄掛在 EFS 上 ===
✅ /var/www/html 已掛載 EFS（127.0.0.1:/ nfs4）
=== 2. 開機會自動掛載 ===
✅ /etc/fstab 有一行 EFS 設定
=== 3. 檔案完整 ===
✅ 檔案數一致：3xxx
✅ wp-config.php 擁有者為 apache
=== 4. WordPress 可以寫入 EFS ===
✅ apache 可以寫入
=== 5. 網站正常 ===
✅ /healthz 正常
✅ WordPress 登入頁回應 200
```

最後在**自己電腦的瀏覽器**打開 `http://<EIP>` 和 `http://<EIP>/wp-admin`，網站和後台都應該跟搬移前一樣正常。

（選用）**確認重開機後也會自動掛上**：EC2 → Instances → 勾選 `wp-web-1` → **Instance state → Reboot instance**。等 2–3 分鐘後重新連 Session Manager，再跑一次上面的驗證。

---

#### 出問題時

| 狀況 | 原因 | 解法 |
|---|---|---|
| 3-2 的 `mount` 卡住 | 連不到 EFS | 檢查 `wp-efs-sg` 的 2049 規則來源是 `wp-web-sg`；掛載點要是 Available；EFS ID 正確 |
| `mount: unknown filesystem type 'efs'` | 沒有安裝 `amazon-efs-utils` | `dnf install -y amazon-efs-utils` |
| 網站顯示 403 或空白 | EFS 上沒有檔案，或擁有者不對 | 檢查驗證第 3 項；擁有者不對時執行 `chown -R apache:apache /var/www/html` |
| 驗證第 2 項顯示超過 1 行 | 3-5 執行了不只一次 | `nano /etc/fstab` 刪除重複的行（只保留一行），存檔後執行 `mount -a` |
| 網站變慢 | EFS 是網路硬碟，比本機硬碟慢 | 正常現象。1.7 已安裝的 PHP OPcache 會快取程式，第二次之後的瀏覽就會變快 |

**還原方法**（想放棄 EFS、回到原本的本機目錄）：

```bash
systemctl stop nginx php-fpm
umount /var/www/html
sed -i '\| /var/www/html efs |d' /etc/fstab
rmdir /var/www/html && mv /var/www/html.local-backup /var/www/html
systemctl start php-fpm nginx
```

> `/var/www/html.local-backup` 先留著，萬一 EFS 有問題可以用上面的方法還原。**2.6-1 製作 AMI 之前會刪除它**，因為它會被打包進 AMI，而且裡面的 `wp-config.php` 含有資料庫密碼。

### 2.3 建立 ALB（負載平衡器）

#### ① 建立 ALB 專用的安全群組 `wp-alb-sg`

**為什麼需要**：ALB（負載平衡器）是網站新的「大門」，所有訪客的請求都會先到 ALB，再由 ALB 轉給後面的 WordPress 主機。ALB 也有自己的防火牆，我們要規定**誰可以連到 ALB 的 Port 80（HTTP）**。

這個安全群組會分兩個階段設定：

```
現在（2.3）：  任何人 ──HTTP 80──► wp-alb-sg 的 ALB ──► WordPress 主機    （先開放，方便測試）
2.5 之後：     CloudFront ──HTTP 80──► wp-alb-sg 的 ALB                    （只允許 CloudFront）
               其他來源   ──────────► ❌ 擋下
```

> 為什麼現在先開放給所有人（`0.0.0.0/0`）？因為 CloudFront 要到 2.4 才會建立。先開放，才能在 2.3 用 ALB 的網址測試，確認 ALB 本身運作正常。2.5 會把來源改成只允許 CloudFront。

> 和 `wp-web-sg`、`wp-efs-sg` 的差別：那兩個的來源是「另一個安全群組」，代表只允許特定的 AWS 資源。`wp-alb-sg` 面對的是網際網路上的訪客，所以來源是 IP 範圍（`0.0.0.0/0` 代表所有 IPv4 位址）。

下面兩種方法**擇一**即可。

**方法 A：主控台**

1. 主控台上方搜尋 **VPC** → 左側 **Security groups** → 右上 **Create security group**。
2. **Basic details**：
   - **Security group name**：`wp-alb-sg`
   - **Description**：`WordPress ALB`（必填，只能用英文）
   - **VPC**：⚠️ **點開下拉選單，把預設的 default VPC 改成 `wp-vpc`**（顯示為 `vpc-xxxxxxxx (wp-vpc)`）。建錯 VPC，建立 ALB 時會在清單中找不到這個安全群組。
3. **Inbound rules** → **Add rule**：
   - **Type**：下拉選單選 **HTTP**（Protocol 和 Port range 會自動帶入 `TCP`、`80`）
   - **Source**：選 **Anywhere-IPv4**（會自動帶入 `0.0.0.0/0`）
   - **Description**（選填）：`Temporary - restrict to CloudFront in 2.5`
   - **不要加 HTTPS（443）**：HTTPS 由 CloudFront 處理，CloudFront 和 ALB 之間用 HTTP 80 連線（見 2.4）。
4. **Outbound rules**：保持預設（全部允許）。ALB 需要對外連線，才能把請求轉給後面的主機並執行健康檢查。
5. 拉到最下面按 **Create security group**。

**方法 B：CloudShell 指令**

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=wp-vpc --query "Vpcs[0].VpcId" --output text)
ALB_SG=$(aws ec2 create-security-group --group-name wp-alb-sg --description "WordPress ALB" \
  --vpc-id $VPC_ID --query GroupId --output text)
aws ec2 authorize-security-group-ingress --group-id $ALB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
echo "✅ 已建立 wp-alb-sg：$ALB_SG（暫時允許所有人連 HTTP 80）"
```

✅ **驗證①**（CloudShell）：

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=wp-vpc --query "Vpcs[0].VpcId" --output text)
echo "wp-vpc 的 ID：$VPC_ID"
aws ec2 describe-security-groups --filters Name=group-name,Values=wp-alb-sg \
  --query "SecurityGroups[0].{Name:GroupName, VPC:VpcId, Port:IpPermissions[0].FromPort, Source:IpPermissions[0].IpRanges[0].CidrIp, Rules:length(IpPermissions)}" \
  --output table
```

正確結果：
- **VPC** 和第一行印出的 `wp-vpc` ID 相同。
- **Port** 是 `80`，**Source** 是 `0.0.0.0/0`。
- **Rules** 是 `1`，代表只有這一條規則。

| 出現的狀況 | 解法 |
|---|---|
| 表格的值都是 `None` | 找不到 `wp-alb-sg`，可能建在其他區域，或名稱打錯 |
| VPC 和 `wp-vpc` 不同 | 建錯 VPC 了。安全群組不能更換 VPC：勾選它 → **Actions → Delete security groups**，再重新建立 |
| `InvalidGroup.Duplicate`（方法 B） | 已經建立過了，不用重建，直接做驗證 |
| Rules 大於 1 | 多加了規則。到 **Inbound rules** → **Edit inbound rules**，只保留 HTTP 80 那一條 |

> 記下 `wp-alb-sg`：③ 建立 ALB 時要選它；④ 會把 `wp-web-sg` 的來源改成它；2.5 會把它的來源改成只允許 CloudFront。

#### ② 建立目標群組 `wp-tg`

**為什麼需要**：ALB 本身只負責接收請求，還需要知道「要把請求轉給誰」。**目標群組（Target Group）**就是這份「後端主機名單」，再加上一套**健康檢查**規則。ALB 會定期檢查名單上的每台主機，**只把流量送給健康的主機**；壞掉的主機會自動被排除，修好後再自動加回來。

```
訪客 ──► ALB ──► 目標群組 wp-tg ──┬─► wp-web-1     ✅ healthy   → 會收到流量
                                  └─► （之後 ASG 新增的主機）     → 檢查通過才會收到流量
          ALB 每 15 秒對每台主機請求一次 http://主機/healthz，回應 200 才算健康
```

> 名詞：
> - **Target（目標）**：接收流量的後端，這裡是 EC2 主機。
> - **Health check（健康檢查）**：ALB 定期請求一個網址，依回應判斷主機是否正常。
> - **Healthy／Unhealthy threshold**：連續成功或失敗幾次，才把主機判定為健康或不健康，避免偶發狀況造成誤判。

> **為什麼健康檢查用 `/healthz`，不用首頁 `/`？** WordPress 首頁可能回應 301 或 302 轉址（例如轉到舊網址），ALB 會把它當成失敗。`/healthz` 是 1.7 在 Nginx 設定的固定網址，永遠回應 200 和 `ok`，而且不經過 PHP 和資料庫，檢查又快又穩定。缺點是 PHP 或資料庫壞掉時偵測不到，這部分由 2.8 的 5xx 告警和 Phase 3 的 AI Agent 負責。

下面兩種方法**擇一**即可。

**方法 A：主控台**

1. 主控台上方搜尋 **EC2** → 左側選單往下捲到 **Load Balancing → Target Groups** → 右上 **Create target group**。

**Step 1：Specify group details（基本設定）**

| 欄位 | 設定 | 說明 |
|---|---|---|
| **Choose a target type** | **Instances** | 目標是 EC2 主機 |
| **Target group name** | `wp-tg` | |
| **Protocol : Port** | **HTTP** : `80` | ALB 用 HTTP 80 連到主機（主機的 Nginx 監聽 80） |
| **IP address type** | **IPv4** | |
| **VPC** | ⚠️ 改成 **`wp-vpc`** | 預設是 default VPC。選錯的話，下一步會看不到 `wp-web-1` |
| **Protocol version** | **HTTP1**（預設） | |

**Health checks（健康檢查）** 區塊：

| 欄位 | 設定 |
|---|---|
| **Health check protocol** | **HTTP** |
| **Health check path** | **`/healthz`**（⚠️ 預設是 `/`，一定要改） |

展開 **Advanced health check settings**：

| 欄位 | 設定 | 意思 |
|---|---|---|
| **Health check port** | **Traffic port**（預設） | 用上面設定的 Port 80 檢查 |
| **Healthy threshold** | `2` | 連續成功 2 次就判定為健康（預設 5 次，改小可以更快上線） |
| **Unhealthy threshold** | `2` | 連續失敗 2 次就判定為不健康 |
| **Timeout** | `5` 秒 | 超過 5 秒沒回應就算失敗 |
| **Interval** | `15` 秒 | 每 15 秒檢查一次（預設 30 秒） |
| **Success codes** | `200` | 回應 200 才算成功 |

按右下 **Next**。

**Step 2：Register targets（加入主機）**

1. **Available instances** 清單中，勾選 **`wp-web-1`**。
2. 下方 **Ports for the selected instances** 保持 `80`。
3. 按 **Include as pending below**。下方 **Review targets** 會出現 `wp-web-1`，狀態是 *Pending*。
4. 按右下 **Create target group**。

**方法 B：CloudShell 指令**

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=wp-vpc --query "Vpcs[0].VpcId" --output text)

# 建立目標群組（含健康檢查設定）
TG_ARN=$(aws elbv2 create-target-group --name wp-tg --target-type instance \
  --protocol HTTP --port 80 --vpc-id $VPC_ID \
  --health-check-protocol HTTP --health-check-path /healthz \
  --health-check-interval-seconds 15 --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 --unhealthy-threshold-count 2 --matcher HttpCode=200 \
  --query "TargetGroups[0].TargetGroupArn" --output text)
echo "目標群組：$TG_ARN"

# 把 wp-web-1 加進目標群組
IID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=wp-web-1" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)
aws elbv2 register-targets --target-group-arn $TG_ARN --targets Id=$IID,Port=80
echo "✅ 已加入主機 $IID"
```

✅ **驗證②**（CloudShell）：

```bash
TG_ARN=$(aws elbv2 describe-target-groups --names wp-tg --query "TargetGroups[0].TargetGroupArn" --output text)
aws elbv2 describe-target-groups --names wp-tg \
  --query "TargetGroups[0].{Name:TargetGroupName, Port:Port, Path:HealthCheckPath, Interval:HealthCheckIntervalSeconds, Healthy:HealthyThresholdCount}" \
  --output table
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query "TargetHealthDescriptions[].[Target.Id, TargetHealth.State, TargetHealth.Reason]" --output table
```

正確結果：
- 第一個表格：**Path** 是 `/healthz`、**Port** 是 `80`、**Interval** 是 `15`、**Healthy** 是 `2`。
- 第二個表格：列出 `wp-web-1` 的主機 ID，**State 是 `unused`**，Reason 是 `Target.NotInUse`。

> ⚠️ **現在顯示 `unused` 是正常的。** 目標群組還沒接到任何 ALB，所以還沒有人去檢查它。做完 ③ 之後，狀態會變成 `initial`（檢查中），大約 30 秒後變成 **`healthy`**。

| 出現的狀況 | 解法 |
|---|---|
| Path 是 `/` | 主控台：Target Groups → `wp-tg` → **Health checks** 分頁 → **Edit** → Path 改成 `/healthz` → Save changes |
| 第二個表格是空的 | 沒有加入主機：Target Groups → `wp-tg` → **Targets** 分頁 → **Register targets** → 勾選 `wp-web-1` → Include as pending below → Register pending targets |
| 建立時找不到 `wp-web-1` | VPC 選錯了。刪除這個目標群組，重新建立並選 `wp-vpc` |
| `DuplicateTargetGroupName`（方法 B） | 已經建立過了，直接做驗證 |

#### ③ 建立 ALB `wp-alb`

**為什麼需要**：ALB（Application Load Balancer，應用程式負載平衡器）是網站固定的**單一入口**：

- **分散流量**：把請求平均分給目標群組中所有健康的主機。主機從 1 台變成 2 台時，訪客完全感覺不到。
- **自動避開故障**：依 ② 的健康檢查，只把流量送給健康的主機。
- **跨機房**：ALB 同時存在於兩個 AZ。一個機房故障時，另一個繼續服務。這也是 ALB **至少要選兩個子網路**的原因，1.1 才會建立兩個 public 子網路。
- **固定網址**：ALB 有一個固定的 DNS 名稱。主機換來換去，網址都不會變。2.4 的 CloudFront 會連到這個網址。

```
               ┌─────────────── wp-alb（Internet-facing）───────────────┐
訪客 ─HTTP 80─►│  public1（1a）               public2（1c）               │
               │  監聽器 HTTP:80 ──► 轉發到目標群組 wp-tg                │
               └──────────────────────────┬──────────────────────────────┘
                                          ▼
                                      wp-web-1（之後還有 ASG 的主機）
```

> 名詞：
> - **Internet-facing**：ALB 有公開的網址，網際網路上的人連得到。另一種 *Internal* 只有 VPC 內部連得到。
> - **Listener（監聽器）**：ALB 在哪個 Port 接收請求，以及收到後要做什麼。這裡是「在 HTTP 80 接收，然後轉發給 `wp-tg`」。

> 💰 **費用提醒**：ALB **建立後就開始計費**，不管有沒有流量。東京區域約每小時 $0.0243，加上用量費，每月約 $20–25。練習完請照「清除所有資源」刪除。

下面兩種方法**擇一**即可。

**方法 A：主控台**

1. 主控台上方搜尋 **EC2** → 左側 **Load Balancing → Load Balancers** → 右上 **Create load balancer**。
2. 在 **Application Load Balancer** 那張卡片上按 **Create**（⚠️ 不要選 Network Load Balancer 或 Gateway Load Balancer）。

**Basic configuration（基本設定）**

| 欄位 | 設定 |
|---|---|
| **Load balancer name** | `wp-alb` |
| **Scheme** | **Internet-facing** |
| **Load balancer IP address type** | **IPv4** |

**Network mapping（網路配置）**

1. **VPC**：⚠️ 改成 **`wp-vpc`**。
2. **Availability Zones and subnets**：勾選兩個 AZ，並各自選擇 **public** 子網路：

| 勾選 | Subnet |
|---|---|
| ✅ `ap-northeast-1a` | **`wp-subnet-public1-ap-northeast-1a`** |
| ✅ `ap-northeast-1c` | **`wp-subnet-public2-ap-northeast-1c`** |

   ⚠️ 一定要選 **public** 子網路。選到 private 子網路時，ALB 沒有通往網際網路的路由，外面連不進來（主控台通常也會顯示警告）。

**Security groups（安全群組）**

1. 按 `default` 旁邊的 **X** 移除它。
2. 從下拉選單選 **`wp-alb-sg`**（① 建立的那個）。

**Listeners and routing（監聽器與轉發）**

| 欄位 | 設定 |
|---|---|
| **Protocol** | **HTTP** |
| **Port** | `80` |
| **Default action** | **Forward to target groups** → 選 **`wp-tg`** |

**其他區塊**（Load balancer tags、Optimize with service integrations 中的 CloudFront／WAF／Global Accelerator 等）：**全部保持預設，不要勾選**。CloudFront 會在 2.4 另外建立。

3. 拉到最下面看 **Summary**，確認 VPC、兩個 public 子網路、`wp-alb-sg`、HTTP:80 → `wp-tg` 都正確 → 按 **Create load balancer**。
4. 按 **View load balancer**，`wp-alb` 的 **State** 一開始是 *Provisioning*，大約 **3–5 分鐘**後變成 **Active**。
5. 點 `wp-alb`，在 **Details** 中複製 **DNS name**（類似 `wp-alb-1234567890.ap-northeast-1.elb.amazonaws.com`），填進「準備 5：數值記錄表」。

**方法 B：CloudShell 指令**

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=wp-vpc --query "Vpcs[0].VpcId" --output text)
SUBNETS=$(aws ec2 describe-subnets --filters Name=vpc-id,Values=$VPC_ID "Name=tag:Name,Values=wp-subnet-public*" \
  --query "Subnets[].SubnetId" --output text)
ALB_SG=$(aws ec2 describe-security-groups --filters Name=vpc-id,Values=$VPC_ID Name=group-name,Values=wp-alb-sg \
  --query "SecurityGroups[0].GroupId" --output text)
TG_ARN=$(aws elbv2 describe-target-groups --names wp-tg --query "TargetGroups[0].TargetGroupArn" --output text)
echo "public 子網路：$SUBNETS"
echo "安全群組：$ALB_SG"

# 建立 ALB
ALB_ARN=$(aws elbv2 create-load-balancer --name wp-alb --type application --scheme internet-facing \
  --ip-address-type ipv4 --subnets $SUBNETS --security-groups $ALB_SG \
  --query "LoadBalancers[0].LoadBalancerArn" --output text)

# 建立監聽器：HTTP 80 → 轉發到 wp-tg
aws elbv2 create-listener --load-balancer-arn $ALB_ARN --protocol HTTP --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN --query "Listeners[0].ListenerArn" --output text

# 等 ALB 變成 Active（約 3–5 分鐘，等待期間畫面不會有輸出）
aws elbv2 wait load-balancer-available --load-balancer-arns $ALB_ARN && echo "✅ wp-alb 已是 Active"
aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].DNSName" --output text
```

- 「public 子網路」那一行必須印出**兩個** `subnet-` ID。
- 最後一行印出的就是 ALB 的 DNS name。

✅ **驗證③**（CloudShell）：等 ALB 變成 Active 後，整段貼上：

```bash
ALB_ARN=$(aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].LoadBalancerArn" --output text)
ALB_DNS=$(aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].DNSName" --output text)
TG_ARN=$(aws elbv2 describe-target-groups --names wp-tg --query "TargetGroups[0].TargetGroupArn" --output text)
echo "ALB 網址：$ALB_DNS"

STATE=$(aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].State.Code" --output text)
[ "$STATE" = "active" ] && echo "✅ ALB 狀態：active" || echo "❌ ALB 狀態：$STATE（provisioning 請再等幾分鐘）"

SCHEME=$(aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].Scheme" --output text)
[ "$SCHEME" = "internet-facing" ] && echo "✅ Scheme：internet-facing" || echo "❌ Scheme：$SCHEME（必須是 internet-facing，需刪除重建）"

for S in $(aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].AvailabilityZones[].SubnetId" --output text); do
  N=$(aws ec2 describe-subnets --subnet-ids $S --query "Subnets[0].Tags[?Key=='Name'].Value | [0]" --output text)
  case "$N" in *public*) echo "✅ 子網路：$N";; *) echo "❌ 子網路：$N（必須是 public 子網路）";; esac
done

SGN=$(aws ec2 describe-security-groups --group-ids $(aws elbv2 describe-load-balancers --names wp-alb \
  --query "LoadBalancers[0].SecurityGroups[0]" --output text) --query "SecurityGroups[0].GroupName" --output text)
[ "$SGN" = "wp-alb-sg" ] && echo "✅ 安全群組：wp-alb-sg" || echo "❌ 安全群組：$SGN"

FWD=$(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN \
  --query "Listeners[?Port==\`80\`].DefaultActions[0].TargetGroupArn | [0]" --output text)
[ "$FWD" = "$TG_ARN" ] && echo "✅ 監聽器 HTTP:80 轉發到 wp-tg" || echo "❌ 監聽器沒有轉發到 wp-tg"

aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query "TargetHealthDescriptions[].[Target.Id, TargetHealth.State, TargetHealth.Reason]" --output table

echo "--- 透過 ALB 測試 ---"
curl -s -m 10 "http://$ALB_DNS/healthz"; echo
curl -s -m 10 -o /dev/null -w "首頁 HTTP 狀態碼：%{http_code}，轉址到：%{redirect_url}\n" "http://$ALB_DNS/"
echo "=== 請把這個 ALB DNS name 記到「準備 5：數值記錄表」：$ALB_DNS ==="
```

正確結果類似：

```text
ALB 網址：wp-alb-1234567890.ap-northeast-1.elb.amazonaws.com
✅ ALB 狀態：active
✅ Scheme：internet-facing
✅ 子網路：wp-subnet-public1-ap-northeast-1a
✅ 子網路：wp-subnet-public2-ap-northeast-1c
✅ 安全群組：wp-alb-sg
✅ 監聽器 HTTP:80 轉發到 wp-tg
-----------------------------------------
|         DescribeTargetHealth          |
+----------------------+----------+-----+
|  i-0abc123def4567890 |  healthy |  None|
+----------------------+----------+-----+
--- 透過 ALB 測試 ---
ok
首頁 HTTP 狀態碼：301，轉址到：http://13.192.60.150/
=== 請把這個 ALB DNS name 記到「準備 5：數值記錄表」：wp-alb-1234567890.ap-northeast-1.elb.amazonaws.com ===
```

- 目標狀態是 **`healthy`**，`/healthz` 回應 **`ok`**，就代表 ALB 已經能把流量送到主機了。
- 首頁回應 **200**，或回應 **301／302** 並轉址到你的 EIP，**兩種都正常**。回應 200 時沒有轉址，所以「轉址到：」後面是**空白**，這是正確的。轉址是因為 WordPress 記得的網址還是 `http://<EIP>`，會把訪客導回去；2.5 換成 CloudFront 網址後就會解決。
- **ALB 網址（DNS name）印在第一行和最後一行。** 如果畫面捲走了，單獨執行 `echo $ALB_DNS` 再看一次；或到主控台 **EC2 → Load Balancers → 點 `wp-alb`**，在 **Details** 區塊右上方的 **DNS name** 旁按複製圖示。只要 `/healthz` 回應 `ok`，就代表這個變數一定有值（變數是空的話，curl 不會有任何回應）。
- 剛建立時目標可能是 `initial`（Reason：`Elb.InitialHealthChecking`），這是正在進行第一次健康檢查。等 30 秒到 1 分鐘再跑一次。
- 剛變成 Active 時，ALB 的網址可能要再過 1–2 分鐘才查得到。如果 curl 沒有反應，稍等再試。

**目標一直不是 healthy 時：**

| State／Reason | 意思 | 解法 |
|---|---|---|
| `unhealthy`／`Target.Timeout` | ALB 連不到主機的 Port 80 | 檢查 `wp-web-sg` 的 inbound：目前（④ 之前）要允許 HTTP 80；做完 ④ 之後，來源要是 `wp-alb-sg` |
| `unhealthy`／`Target.ResponseCodeMismatch` | 主機有回應，但不是 200 | 健康檢查 Path 不是 `/healthz`，回到 ② 修改；或在主機上執行 `curl -s localhost/healthz` 確認回應 `ok` |
| `unhealthy`／`Target.FailedHealthChecks` | 主機沒有正常回應 | Session Manager 執行 `systemctl is-active nginx`，必須是 `active` |
| `unused`／`Target.NotInUse` | 目標群組沒有接到 ALB，或主機所在的 AZ 不在 ALB 的子網路中 | 確認監聽器轉發到 `wp-tg`；確認 ALB 有勾選 `ap-northeast-1a` |
| `draining` | 主機正在被移出目標群組 | 如果不是你主動移除的，重新註冊主機 |

#### ④ 收緊主機防火牆：`wp-web-sg` 只允許 ALB 連進來

**為什麼需要**：目前 `wp-web-sg` 的 HTTP 80 規則來源是 `0.0.0.0/0`，任何人都可以**繞過 ALB**，直接用 `http://<EIP>` 連到主機。這樣 ALB 的分流、健康檢查，以及 2.5 之後 CloudFront 的保護都會被跳過。把來源改成 **`wp-alb-sg`** 之後，只有 ALB 能把流量送進主機。

```
改之前：  訪客 ──► ALB ──► wp-web-1          ✅
          訪客 ──────────► wp-web-1（EIP）   ✅ ← 可以繞過 ALB，不安全

改之後：  訪客 ──► ALB（wp-alb-sg）──► wp-web-1   ✅ 來源是 wp-alb-sg，允許
          訪客 ──────────────────────► wp-web-1   ❌ 被 wp-web-sg 擋下（連線逾時）
```

> - **來源設成「安全群組」的意思**：只要網路卡上掛著 `wp-alb-sg` 的資源（也就是 ALB）就允許。ALB 的 IP 會變動，用安全群組當來源就不必管 IP。這和 1.2 的 `wp-db-sg`、2.2 的 `wp-efs-sg` 是同樣的做法。
> - **Session Manager 不受影響**：Session Manager 是主機主動連出去，不需要任何 inbound 規則，所以改完之後照樣登得進去。
> - **主機連外不受影響**：安全群組只管「進來」，主機的 outbound 沒有改。

**開始前先確認**：③ 的驗證已經通過（`wp-web-1` 是 `healthy`，`/healthz` 回應 `ok`）。

下面兩種方法**擇一**即可。

**方法 A：主控台**

1. 主控台上方搜尋 **VPC** → 左側 **Security → Security groups**。
2. 勾選 **`wp-web-sg`**（如果名稱欄是空的，看 **Security group name** 欄）→ 下方點 **Inbound rules** 分頁 → 按 **Edit inbound rules**。
3. 你會看到一條規則：Type `HTTP`、Port `80`、Source `Custom`、`0.0.0.0/0`。

   ⚠️ **不能直接把這條的 Source 改成安全群組。** 主控台會顯示錯誤 *You may not specify a referenced group id for an existing IPv4 CIDR rule.*，因為「IP 範圍」的規則不能改成「安全群組」來源。正確做法是**刪掉舊的，再新增一條**：

   **3-1.** 在 `0.0.0.0/0` 那一列，按最右邊的 **Delete**。

   **3-2.** 按左下 **Add rule**，新的一列這樣填：

| 欄位 | 設定 |
|---|---|
| **Type** | **HTTP**（Port range 自動帶出 `80`） |
| **Source** | **Custom** → 在搜尋框輸入 `wp-alb` → 選 **`sg-xxxxxxxx (wp-alb-sg)`** |
| **Description** | `HTTP from ALB only`（選填，只能用英文） |

   **3-3.** 如果還有其他規則（例如 Source 是 `::/0` 的 IPv6 HTTP 規則、Port 22、All traffic），也按 **Delete** 刪掉。畫面上最後**只剩一條**：HTTP、80、`wp-alb-sg`。

   ⚠️ 搜尋 `wp-alb` 找不到時，代表 `wp-alb-sg` 不在 `wp-vpc` 裡，回到 ① 檢查。
   💡 刪除和新增會在同一次 Save 一起生效，ALB 不會中斷。如果剛才已經出現上面的錯誤，先按 **Cancel**，再重新按 **Edit inbound rules** 照 3-1 做。

4. 按右下 **Save rules**。畫面上方出現綠色的 *Inbound security group rules successfully modified*，就完成了。改完**立刻生效**，不用重開主機。

**方法 B：CloudShell 指令**

先加新規則、再刪舊規則，中間 ALB 不會中斷：

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=wp-vpc --query "Vpcs[0].VpcId" --output text)
WEB_SG=$(aws ec2 describe-security-groups --filters Name=vpc-id,Values=$VPC_ID Name=group-name,Values=wp-web-sg \
  --query "SecurityGroups[0].GroupId" --output text)
ALB_SG=$(aws ec2 describe-security-groups --filters Name=vpc-id,Values=$VPC_ID Name=group-name,Values=wp-alb-sg \
  --query "SecurityGroups[0].GroupId" --output text)
echo "wp-web-sg：$WEB_SG    wp-alb-sg：$ALB_SG"

# 1. 新增：允許來自 wp-alb-sg 的 HTTP 80
aws ec2 authorize-security-group-ingress --group-id $WEB_SG \
  --ip-permissions '[{"IpProtocol":"tcp","FromPort":80,"ToPort":80,"UserIdGroupPairs":[{"GroupId":"'"$ALB_SG"'","Description":"HTTP from ALB only"}]}]' \
  --query "Return" --output text

# 2. 刪除：所有人都能連的 HTTP 80
aws ec2 revoke-security-group-ingress --group-id $WEB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0 \
  --query "Return" --output text
```

- 兩個 `ID` 都必須是 `sg-` 開頭。如果其中一個是 `None`，代表名稱打錯或不在 `wp-vpc` 裡，先不要往下做。
- 兩個指令各印出 `True` 就成功了。
- 如果出現 `InvalidPermission.Duplicate`，代表新規則已經存在；`InvalidPermission.NotFound` 代表舊規則已經刪掉了。兩種都可以直接做驗證。

✅ **驗證④**（CloudShell）：整段貼上：

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=wp-vpc --query "Vpcs[0].VpcId" --output text)
WEB_SG=$(aws ec2 describe-security-groups --filters Name=vpc-id,Values=$VPC_ID Name=group-name,Values=wp-web-sg \
  --query "SecurityGroups[0].GroupId" --output text)
ALB_SG=$(aws ec2 describe-security-groups --filters Name=vpc-id,Values=$VPC_ID Name=group-name,Values=wp-alb-sg \
  --query "SecurityGroups[0].GroupId" --output text)
ALB_DNS=$(aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].DNSName" --output text)
TG_ARN=$(aws elbv2 describe-target-groups --names wp-tg --query "TargetGroups[0].TargetGroupArn" --output text)
EIP=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=wp-web-1" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].PublicIpAddress" --output text)

echo "--- wp-web-sg 的 inbound 規則 ---"
aws ec2 describe-security-group-rules --filters Name=group-id,Values=$WEB_SG \
  --query "SecurityGroupRules[?IsEgress==\`false\`].{Port:FromPort, FromCIDR:CidrIpv4, FromIPv6:CidrIpv6, FromSG:ReferencedGroupInfo.GroupId}" \
  --output table
N=$(aws ec2 describe-security-group-rules --filters Name=group-id,Values=$WEB_SG \
  --query "length(SecurityGroupRules[?IsEgress==\`false\`])" --output text)
SRC=$(aws ec2 describe-security-group-rules --filters Name=group-id,Values=$WEB_SG \
  --query "SecurityGroupRules[?IsEgress==\`false\`].ReferencedGroupInfo.GroupId | [0]" --output text)
[ "$N" = "1" ] && [ "$SRC" = "$ALB_SG" ] && echo "✅ 只有一條規則，來源是 wp-alb-sg（$ALB_SG）" \
  || echo "❌ 規則有 $N 條，或來源不是 wp-alb-sg（目前：$SRC）"

echo "--- 透過 ALB（應該通） ---"
H=$(curl -s -m 10 "http://$ALB_DNS/healthz")
[ "$H" = "ok" ] && echo "✅ ALB /healthz：ok" || echo "❌ ALB /healthz 沒有回應 ok（目前：$H）"
aws elbv2 describe-target-health --target-group-arn $TG_ARN \
  --query "TargetHealthDescriptions[].[Target.Id, TargetHealth.State]" --output text

echo "--- 直接連 EIP（應該不通） ---"
C=$(curl -s -m 5 -o /dev/null -w "%{http_code}" "http://$EIP/healthz")
[ "$C" = "000" ] && echo "✅ 直接連 http://$EIP 被擋下（逾時），防火牆生效" \
  || echo "❌ 直接連 http://$EIP 還能連上（HTTP $C），0.0.0.0/0 規則可能還在"
```

正確結果類似：

```text
--- wp-web-sg 的 inbound 規則 ---
--------------------------------------------------------------
|                 DescribeSecurityGroupRules                 |
+-----------+-----------+-------+------------------------+
| FromCIDR  | FromIPv6  | Port  |        FromSG          |
+-----------+-----------+-------+------------------------+
|  None     |  None     |  80   |  sg-0abc1234def567890  |
+-----------+-----------+-------+------------------------+
✅ 只有一條規則，來源是 wp-alb-sg（sg-0abc1234def567890）
--- 透過 ALB（應該通） ---
✅ ALB /healthz：ok
i-0abc123def4567890	healthy
--- 直接連 EIP（應該不通） ---
✅ 直接連 http://13.192.60.150 被擋下（逾時），防火牆生效
```

- 最後一項會**等 5 秒**才出結果，因為被防火牆擋下的連線不會有回應，要等到逾時。這是正常的。
- 目標狀態必須還是 **`healthy`**。如果變成 `unhealthy`，代表 ALB 也被擋住了，看下面的排錯表。

> 📌 **之後要用 ALB 網址瀏覽網站。** 從現在起 `http://<EIP>` 打不開了。WordPress 記得的網址還是 `http://<EIP>`，所以從 ALB 打開網站時，部分連結、圖片或登入後台時可能會被導回 EIP 而打不開。這是正常的，2.5 把網址改成 CloudFront 之後就會恢復。

| 出現的狀況 | 原因 | 解法 |
|---|---|---|
| ❌ 規則有 2 條，其中一條 FromCIDR 是 `0.0.0.0/0` | 舊規則沒刪掉 | 回到方法 A 步驟 3，刪掉 `0.0.0.0/0` 那條，或執行方法 B 的第 2 個指令 |
| ❌ 有一條 FromIPv6 是 `::/0` | 有 IPv6 的開放規則 | Edit inbound rules → 把那一條 **Delete** → Save rules |
| 目標變成 `unhealthy`（`Target.Timeout`） | 來源選錯了，ALB 也被擋住 | 確認來源是 **`wp-alb-sg`**，不是 `wp-web-sg` 或 `wp-efs-sg`。ID 要和驗證印出的 `wp-alb-sg` 相同 |
| 直接連 EIP 還是通 | `0.0.0.0/0` 規則還在 | 同第一列 |
| Source 搜尋不到 `wp-alb-sg` | `wp-alb-sg` 不在 `wp-vpc` 裡 | 回到 ① 用驗證①檢查 VPC |
| Session Manager 連不上 | 和這一步無關 | Session Manager 不需要 inbound 規則；檢查主機是否 running、IAM 角色是否還在 |

> 💡 「Phase 1 完成檢查表」第 2 項預期 `wp-web-sg` 的來源是 `0.0.0.0/0`。做完 ④ 之後，正確答案變成 **`wp-alb-sg`**，重跑那份檢查時不用改回去。

### 2.4 建立 CloudFront（全球加速 + 免費 HTTPS）

**為什麼需要**：CloudFront 是 AWS 的 CDN（內容傳遞網路），在全球有幾百個節點。

- **免費 HTTPS**：訪客用 `https://xxxx.cloudfront.net` 連線，網址列有鎖頭，不用自己申請憑證。
- **加速**：圖片、CSS、JS 這類靜態檔會快取在離訪客最近的節點，不必每次都回到東京的主機。
- **保護後端**：訪客只連得到 CloudFront，2.5 之後 ALB 只接受 CloudFront 送來的流量。

```
訪客 ──HTTPS──► CloudFront 節點 ──HTTP 80 + 暗號標頭──► wp-alb ──► wp-web-1
                 │
                 ├─ /wp-content/*、/wp-includes/*：快取（圖片、CSS、JS）
                 └─ 其他網址：不快取，每次都轉給 WordPress（登入、後台、留言）
```

> 名詞：
> - **Distribution（分發）**：一組 CloudFront 設定，建立後會得到一個 `dxxxx.cloudfront.net` 網址。
> - **Origin（來源）**：CloudFront 去哪裡拿內容，這裡是 `wp-alb`。
> - **Behavior（行為）**：依網址路徑決定要不要快取、允許哪些 HTTP 方法。

#### 2.4-1 產生「暗號」`<ORIGIN_SECRET>`

CloudFront 轉給 ALB 的每個請求，都會帶上一個秘密標頭 `X-Origin-Verify`。2.5 會設定成只有帶著正確暗號的請求才放行，別人就無法繞過 CloudFront 直接打 ALB。

在 CloudShell 執行：

```bash
aws secretsmanager get-random-password --password-length 32 --exclude-punctuation \
  --query RandomPassword --output text
```

會印出一串 32 個英數字。複製下來，記到「準備 5：數值記錄表」，以下稱 `<ORIGIN_SECRET>`。

#### 2.4-2 選擇定價方案（新版主控台）

2025 年 11 月起，CloudFront 主控台建立 distribution 時，會先要你選**月費方案（flat-rate pricing plan）**。目前列出的方案是：

| 方案 | 月費 | 每月包含 | 適合 |
|---|---|---|---|
| **Free** | **$0** | 100 萬次請求、100 GB 流量；內含 WAF（5 條規則）、免費憑證；最多 5 個 Behavior | ✅ **本指南練習用這個** |
| Pro | $15 | 1,000 萬次請求、50 TB；WAF 25 條規則、WordPress 專用防護 | 正式小網站 |
| Business | $200 | 1.25 億次請求、50 TB | 中大型網站 |
| Premium | $1,000 起 | 5 億次請求起 | 大型網站 |

> 💡 **主控台已經不一定看得到 Pay-as-you-go（依用量計費）。** 舊版指南寫「選 Pay-as-you-go」是依舊畫面寫的。如果你的畫面只有 Free／Pro／Business／Premium，就選 **Free**。想用 Pay-as-you-go，請改用下面的**方法 B（CLI）**，用指令建立的 distribution 預設就是 Pay-as-you-go。

**Free 方案要知道的四件事：**

1. **不會有超額費用**：超過 100 GB 或 100 萬次請求也不會多收錢，只會收到 Email 通知（50%、80%、100%），長期超過時 AWS 可能降低傳送速度。練習用量遠低於這個額度。
2. **WAF 一定要開**：Free 方案內含 WAF，建立時會自動套用，**不另外收費**，不能關掉。
3. **刪除前要先取消方案**：清除資源時，要先在 distribution 取消 Free 方案（立即生效），才能 Disable 和 Delete。見「清除所有資源」。
4. **每個帳號最多 3 個 Free 方案**，一個方案對應一個 distribution。

> ⚠️ 如果選 Free 時出現帳號不符合資格的訊息（例如帳號還在新版 AWS Free Tier 的免費方案期間），不要選付費方案，改用**方法 B（CLI）**建立 Pay-as-you-go 的 distribution。Pay-as-you-go 每月有 1 TB 流量、1,000 萬次請求的永久免費額度，練習一樣是 $0。

下面兩種方法**擇一**即可。

#### 方法 A：主控台

> 主控台精靈的頁面名稱可能隨 AWS 改版略有不同。遇到對不上的畫面，截圖貼到討論串，我再補上。精靈裡沒設好的項目，建立後都可以在「2.4-3 建立後檢查與補設定」改回來。

1. 主控台上方搜尋 **CloudFront** → 左側 **Distributions** → **Create distribution**。

2. **方案選擇不一定會出現在第一頁**。依帳號和主控台版本，可能出現在第一頁、最後的確認頁，或完全沒有：
   - **有看到 Free／Pro／Business／Premium 選項** → 選 **Free**。
   - **完全沒有方案選項**（實測 2026-10-28 的畫面就是這樣）→ 不用找，直接往下做。這樣建立的就是 **Pay-as-you-go**，有每月 1 TB 的永久免費額度，練習一樣是 $0。

**Get started（開始）**

3. **Distribution name**：`wp-cf`（會變成 `Name` 標籤）。
4. **Distribution type**：選 **Single website or app** → **Next**。
5. **Domain**（自訂網域）：選 **Skip the domain setup for now** / 不設定 → **Next**。先用 CloudFront 給的網址，自己的網域之後再加。

**Specify origin（指定來源）**

| 欄位 | 設定 | 說明 |
|---|---|---|
| **Origin type** | **Elastic Load Balancer** | 來源是 ALB |
| **Origin**（或 Load balancer） | 選 **`wp-alb`**（顯示為 `wp-alb-xxxx.ap-northeast-1.elb.amazonaws.com`） | 下拉選單找不到時，直接貼上 ALB 的 DNS name |
| **Settings** | ⚠️ 選 **Customize origin settings**（自訂） | **不要**用 *Use recommended origin settings*，推薦設定是 **HTTPS only**，但我們的 ALB 只有 HTTP 80，會造成 **502 錯誤** |

選 **Customize origin settings** 之後，下方會出現更多欄位：

| 欄位 | 設定 |
|---|---|
| **Protocol** | ⚠️ **HTTP only** |
| **HTTP port** | `80` |
| **Add custom header** → **Add header** | Header name：`X-Origin-Verify`；Value：貼上 `<ORIGIN_SECRET>` |
| **Origin Shield** | No（預設） |

**Cache settings（快取設定）**，如果在這一頁出現：

| 欄位 | 設定 | 說明 |
|---|---|---|
| **Viewer protocol policy** | **Redirect HTTP to HTTPS** | 訪客用 http 連會自動轉成 https |
| **Allowed HTTP methods** | **GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE** | WordPress 登入、發文、留言要用 POST |
| **Caching** / **Cache policy** | **Do not cache by default** / **CachingDisabled** | 預設不快取，避免後台頁面被快取（靜態檔的快取在 2.4-3 另外加） |
| **Origin request policy** | **AllViewer** | 把網址主機名稱、Cookie 原封不動轉給 WordPress，避免登入失敗或轉址迴圈 |

按 **Next**。

**Enable security protections（WAF）**

6. 依你的方案選擇：
   - **Pay-as-you-go（沒有選方案）**：選 **Do not enable security protections** → **Next**。WAF 另外計費，每月約 $5 起，練習先不開；正式網站建議開啟。
   - **Free 方案**：WAF 是方案內含、不另外收費，會自動啟用，保持預設 → **Next**。

**Review and create（確認並建立）**

7. 確認 Origin 是 `wp-alb`、Protocol 是 **HTTP only**、有 `X-Origin-Verify` 標頭 → **Create distribution**。
8. 畫面會進入 `wp-cf` 的詳細頁。**Details** 中的 **Distribution domain name**（類似 `d1234abcd.cloudfront.net`）記到「準備 5：數值記錄表」，以下稱 `<CF_DOMAIN>`。
9. **Last modified** 一開始是 **Deploying**，約 **5–10 分鐘**後變成日期時間，就是部署完成。

#### 方法 B：CloudShell 指令（Pay-as-you-go）

用指令建立的 distribution 不會套用月費方案，就是 Pay-as-you-go。這段會一次建好來源、暗號標頭、預設行為和兩個靜態檔快取行為（2.4-3 的步驟 1 就不用再做）。

先把 `<ORIGIN_SECRET>` 換成 2.4-1 的暗號，再整段貼上：

```bash
ORIGIN_SECRET='<ORIGIN_SECRET>'     # ← 換成 2.4-1 產生的暗號（保留單引號）
case "$ORIGIN_SECRET" in *"<"*|"") SECRET_OK=no; echo "❌ 還沒把 <ORIGIN_SECRET> 換成暗號，這次不會建立，請修改第一行後重貼";; *) SECRET_OK=yes;; esac
ALB_DNS=$(aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].DNSName" --output text)
echo "ALB：$ALB_DNS"

# AWS 內建（managed）政策的固定 ID
CACHING_DISABLED=4135ea2d-6df8-44a3-9df3-4b5a84be39ad
CACHING_OPTIMIZED=658327ea-f89d-4fab-a63d-7e88639e58f6
ALL_VIEWER=216adef6-5c7f-47e4-b989-5492eafa07d3

STATIC_BEHAVIOR() {  # 產生「靜態檔快取」行為的設定
cat <<EOF
{ "PathPattern": "$1", "TargetOriginId": "wp-alb",
  "ViewerProtocolPolicy": "redirect-to-https", "Compress": true,
  "AllowedMethods": {"Quantity": 2, "Items": ["GET","HEAD"],
    "CachedMethods": {"Quantity": 2, "Items": ["GET","HEAD"]}},
  "CachePolicyId": "$CACHING_OPTIMIZED" }
EOF
}

cat > cf-config.json <<EOF
{
  "CallerReference": "wp-cf-$(date +%s)",
  "Comment": "wp-cf",
  "Enabled": true,
  "PriceClass": "PriceClass_200",
  "HttpVersion": "http2and3",
  "Origins": {"Quantity": 1, "Items": [{
    "Id": "wp-alb", "DomainName": "$ALB_DNS",
    "CustomOriginConfig": {"HTTPPort": 80, "HTTPSPort": 443, "OriginProtocolPolicy": "http-only",
      "OriginSslProtocols": {"Quantity": 1, "Items": ["TLSv1.2"]}},
    "CustomHeaders": {"Quantity": 1, "Items": [{"HeaderName": "X-Origin-Verify", "HeaderValue": "$ORIGIN_SECRET"}]}
  }]},
  "DefaultCacheBehavior": {
    "TargetOriginId": "wp-alb", "ViewerProtocolPolicy": "redirect-to-https", "Compress": true,
    "AllowedMethods": {"Quantity": 7, "Items": ["GET","HEAD","OPTIONS","PUT","POST","PATCH","DELETE"],
      "CachedMethods": {"Quantity": 2, "Items": ["GET","HEAD"]}},
    "CachePolicyId": "$CACHING_DISABLED", "OriginRequestPolicyId": "$ALL_VIEWER"
  },
  "CacheBehaviors": {"Quantity": 2, "Items": [
    $(STATIC_BEHAVIOR "/wp-content/*"),
    $(STATIC_BEHAVIOR "/wp-includes/*")
  ]}
}
EOF
if [ "$SECRET_OK" = yes ] && python3 -m json.tool cf-config.json > /dev/null; then
  echo "✅ 設定檔格式正確，開始建立"
  aws cloudfront create-distribution-with-tags \
    --distribution-config-with-tags "{\"DistributionConfig\": $(cat cf-config.json), \"Tags\": {\"Items\": [{\"Key\": \"Name\", \"Value\": \"wp-cf\"}]}}" \
    --query "Distribution.{Id:Id, Domain:DomainName, Status:Status}" --output table
else
  echo "❌ 沒有建立：暗號沒換掉或設定檔格式錯誤"
fi
```

- 開頭的 `ALB：` 必須印出 `wp-alb-...elb.amazonaws.com`，是空的就代表 ALB 名稱不對。
- 最後的表格會列出 **Domain**（`dxxxx.cloudfront.net`）和 **Status: InProgress**。把 Domain 記到「準備 5：數值記錄表」，以下稱 `<CF_DOMAIN>`。
- 約 **5–10 分鐘**後 Status 變成 `Deployed`。
- `PriceClass_200` 代表使用北美、歐洲、亞洲、中東、非洲的節點（不含南美、澳洲），比全球節點便宜一點。
- Pay-as-you-go 沒有內含 WAF，這裡也沒有開（WAF 每月約 $5 起）。正式網站建議開啟。

#### 2.4-3 建立後檢查與補設定（方法 A 必做）

**步驟 1：新增兩個「靜態檔快取」行為**（方法 B 已經建好，可以跳過）

進入 `wp-cf` → **Behaviors** 分頁 → **Create behavior**，建立兩次：

| 欄位 | 第一個 | 第二個 |
|---|---|---|
| **Path pattern** | `/wp-content/*` | `/wp-includes/*` |
| **Origin and origin groups** | `wp-alb` 那一個 | 同左 |
| **Viewer protocol policy** | **Redirect HTTP to HTTPS** | 同左 |
| **Allowed HTTP methods** | **GET, HEAD** | 同左 |
| **Cache key and origin requests** | **Cache policy and origin request policy (recommended)** | 同左 |
| **Cache policy** | **CachingOptimized** | 同左 |
| **Origin request policy** | 不選（No origin request policy） | 同左 |

每次按 **Create behavior** 存檔。完成後 Behaviors 分頁會有 3 列：`/wp-content/*`、`/wp-includes/*`、`Default (*)`。

> 為什麼這兩個路徑可以快取：`/wp-content/` 放上傳的圖片、佈景主題、外掛的 CSS／JS；`/wp-includes/` 放 WordPress 內建的 JS／CSS。它們不會因為登入的人不同而改變，快取起來最划算。

**步驟 2：確認來源設定**（精靈裡沒看到的欄位在這裡補）

**Origins** 分頁 → 勾選 `wp-alb` 那一列 → **Edit**：

- **Protocol**：**HTTP only**，HTTP port `80`。
- **Add custom header**：有 `X-Origin-Verify`，值是 `<ORIGIN_SECRET>`。沒有的話按 **Add header** 補上。
- 有改的話按 **Save changes**。

**步驟 3：確認預設行為**

**Behaviors** 分頁 → 勾選 **Default (\*)** → **Edit**：

- **Viewer protocol policy**：**Redirect HTTP to HTTPS**
- **Allowed HTTP methods**：**GET, HEAD, OPTIONS, PUT, POST, PATCH, DELETE**
- **Cache policy**：**CachingDisabled**
- **Origin request policy**：**AllViewer**

有改的話按 **Save changes**。每次存檔後，**Last modified** 會回到 *Deploying*，等它變回日期時間。

✅ **驗證 2.4**（CloudShell）：部署完成後，整段貼上：

```bash
CF_ID=$(aws cloudfront list-distributions --query "DistributionList.Items[?contains(Origins.Items[0].DomainName, 'wp-alb')].Id | [0]" --output text)
CF_DOMAIN=$(aws cloudfront get-distribution --id $CF_ID --query "Distribution.DomainName" --output text)
echo "Distribution：$CF_ID    網址：https://$CF_DOMAIN"

ST=$(aws cloudfront get-distribution --id $CF_ID --query "Distribution.Status" --output text)
[ "$ST" = "Deployed" ] && echo "✅ 狀態：Deployed" || echo "⏳ 狀態：$ST（還在部署，等幾分鐘再跑一次）"

aws cloudfront get-distribution-config --id $CF_ID --query "DistributionConfig" > cf-check.json
python3 - <<'PY'
import json
c = json.load(open("cf-check.json"))
o = c["Origins"]["Items"][0]
d = c["DefaultCacheBehavior"]
paths = [b["PathPattern"] for b in c.get("CacheBehaviors", {}).get("Items", [])]
hdrs = [h["HeaderName"] for h in o.get("CustomHeaders", {}).get("Items", [])]
def chk(ok, good, bad): print(("✅ " + good) if ok else ("❌ " + bad))
chk(o.get("CustomOriginConfig", {}).get("OriginProtocolPolicy") == "http-only",
    "來源 Protocol：HTTP only", "來源 Protocol 不是 HTTP only（會造成 502），到 Origins → Edit 修改")
chk("X-Origin-Verify" in hdrs, "有 X-Origin-Verify 暗號標頭", "缺少 X-Origin-Verify 標頭，到 Origins → Edit → Add header")
chk(d["ViewerProtocolPolicy"] == "redirect-to-https", "預設行為：Redirect HTTP to HTTPS", "預設行為的 Viewer protocol policy 不對")
chk("POST" in d["AllowedMethods"]["Items"], "預設行為允許 POST（可以登入）", "預設行為沒有允許 POST，登入會失敗")
chk(d.get("CachePolicyId") == "4135ea2d-6df8-44a3-9df3-4b5a84be39ad", "預設行為：CachingDisabled", "預設行為不是 CachingDisabled")
chk(d.get("OriginRequestPolicyId") == "216adef6-5c7f-47e4-b989-5492eafa07d3", "預設行為：AllViewer", "預設行為的 Origin request policy 不是 AllViewer")
chk("/wp-content/*" in paths and "/wp-includes/*" in paths, "有 /wp-content/* 和 /wp-includes/* 快取行為", "缺少靜態檔快取行為（目前：%s）" % paths)
PY

echo "--- 透過 CloudFront 測試 ---"
H=$(curl -s -m 15 "https://$CF_DOMAIN/healthz")
[ "$H" = "ok" ] && echo "✅ https://$CF_DOMAIN/healthz：ok" || echo "❌ /healthz 回應：$H"
curl -s -m 15 -o /dev/null -w "http 自動轉 https：%{http_code} → %{redirect_url}\n" "http://$CF_DOMAIN/healthz"
echo "=== 請把這個網址記到「準備 5：數值記錄表」：$CF_DOMAIN ==="
```

正確結果類似：

```text
Distribution：E1ABCDEFGHIJKL    網址：https://d1234abcd.cloudfront.net
✅ 狀態：Deployed
✅ 來源 Protocol：HTTP only
✅ 有 X-Origin-Verify 暗號標頭
✅ 預設行為：Redirect HTTP to HTTPS
✅ 預設行為允許 POST（可以登入）
✅ 預設行為：CachingDisabled
✅ 預設行為：AllViewer
✅ 有 /wp-content/* 和 /wp-includes/* 快取行為
--- 透過 CloudFront 測試 ---
✅ https://d1234abcd.cloudfront.net/healthz：ok
http 自動轉 https：301 → https://d1234abcd.cloudfront.net/healthz
=== 請把這個網址記到「準備 5：數值記錄表」：d1234abcd.cloudfront.net ===
```

也可以用瀏覽器打開 `https://<CF_DOMAIN>/healthz`，畫面顯示 `ok`，網址列有鎖頭。

> 📌 現在用瀏覽器打開 `https://<CF_DOMAIN>/` 首頁，可能被導回 `http://<EIP>` 而打不開，或版面跑掉。這是因為 WordPress 記得的網址還是 EIP，**2.5 會改好**。

| 出現的狀況 | 原因 | 解法 |
|---|---|---|
| `/healthz` 回應 **502 Bad Gateway** | 來源 Protocol 是 HTTPS only，但 ALB 只有 HTTP 80 | Origins → `wp-alb` → Edit → Protocol 改 **HTTP only** → Save，等部署完成 |
| **504 Gateway Timeout** | CloudFront 連不到 ALB | 確認 `wp-alb-sg` 目前允許 HTTP 80 來自 `0.0.0.0/0`（2.5 才會收緊）；用 2.3 驗證③確認 ALB 本身正常 |
| **403** 且頁面提到 *Request blocked* | WAF 擋下了請求 | Security 分頁看 WAF 紀錄；練習時可先把被擋的規則改成 Count |
| 驗證的 `CF_ID` 是 `None` | 找不到來源是 `wp-alb` 的 distribution | 主控台 Distributions 確認有沒有建立成功；Origin 要是 ALB 的 DNS name |
| 方法 B 出現 `InvalidArgument` 或 JSON 錯誤 | `ORIGIN_SECRET` 沒換掉，或含有特殊字元 | 暗號只能用 2.4-1 產生的英數字，重新設定後再貼一次 |
| ❌ 缺少靜態檔快取行為 | 方法 A 還沒做 2.4-3 步驟 1 | 照步驟 1 新增 |

### 2.5 讓 WordPress 改用 CloudFront 網址

**為什麼需要**：WordPress 會記住自己的網址（`home`、`siteurl`），文章裡的圖片和連結也都寫著這個網址。目前記的是 `http://<EIP>`，所以從 CloudFront 打開網站時，會被導回 EIP，而 EIP 在 2.3 ④ 之後已經打不開了。這一步要把網址全部換成 `https://<CF_DOMAIN>`。

#### 2.5-1 修改 WordPress 設定（Session Manager，root）

**開始前先準備**：2.4 記下的 CloudFront 網域（`dxxxx.cloudfront.net`）。到 **CloudFront → Distributions → `wp-cf`** 的 **Distribution domain name** 也可以複製。

> ⚠️ **不要把 `<EIP>`、`<CF_DOMAIN>` 這種含 `<` `>` 的文字原樣貼進終端機。** `<` 在 bash 是「從檔案讀入」的符號，會出現 `syntax error near unexpected token 'newline'`，變數也會是空的。下面的指令已經改成自動取得 EIP，只有 CloudFront 網域要你填。

Session Manager 登入 `wp-web-1`，執行 `sudo -i`，然後**先改第一行**再整段貼上：

```bash
CF_DOMAIN=d1234abcd.cloudfront.net     # ← 換成你的 CloudFront 網域（不要加 https://、不要加 < >）

cd /var/www/html
case "$CF_DOMAIN" in
  d1234abcd.cloudfront.net) echo "❌ 還沒把第一行換成你的 CloudFront 網域，這次不會修改任何東西";;
  *.cloudfront.net)
    # 自動取得這台主機的 EIP（從 EC2 的中繼資料服務）
    TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
    EIP=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/public-ipv4)
    OLD="http://$EIP"
    NEW="https://$CF_DOMAIN"
    echo "目前 WordPress 記的網址：$(sudo -u apache wp option get home)"
    echo "要換掉的舊網址：$OLD"
    echo "要換成的新網址：$NEW"

    # 1) 告訴 WordPress：帶著暗號標頭的請求（來自 CloudFront）都是 HTTPS
    grep -q HTTP_X_ORIGIN_VERIFY wp-config.php \
      || sed -i "1a if (isset(\$_SERVER['HTTP_X_ORIGIN_VERIFY'])) { \$_SERVER['HTTPS'] = 'on'; }" wp-config.php
    head -3 wp-config.php

    # 2) 換網址
    sudo -u apache wp option update home "$NEW"
    sudo -u apache wp option update siteurl "$NEW"
    sudo -u apache wp search-replace "$OLD" "$NEW" --skip-columns=guid
    ;;
  *) echo "❌ 第一行的網域格式不對（要是 xxxx.cloudfront.net），這次不會修改任何東西";;
esac
```

**每一段在做什麼：**

| 指令 | 意思 |
|---|---|
| `TOKEN=...`、`EIP=...` | 向 EC2 中繼資料服務查詢這台主機的公開 IP，就不用手動填 EIP |
| `grep -q ... \|\| sed -i "1a ..."` | 在 `wp-config.php` 第 2 行加一行 PHP：請求帶有 `X-Origin-Verify` 標頭（代表來自 CloudFront）時，告訴 WordPress 這是 HTTPS 連線。前面的 `grep -q` 會先檢查，**已經加過就不會重複加** |
| `wp option update home／siteurl` | 把 WordPress 記的網站網址、程式網址換成 CloudFront 網址 |
| `wp search-replace` | 把資料庫中文章、設定裡所有舊網址換成新網址。`--skip-columns=guid` 表示不改文章的唯一識別碼（WordPress 官方建議） |

> 💡 為什麼需要第 1 段：CloudFront 用 HTTPS 接待訪客，但轉給 ALB 和主機時是 HTTP。如果不告訴 WordPress，它會以為訪客用的是 http，然後一直把訪客導向 https，造成**無限轉址（ERR_TOO_MANY_REDIRECTS）**。

正確結果類似：

```text
目前 WordPress 記的網址：http://13.192.60.150
要換掉的舊網址：http://13.192.60.150
要換成的新網址：https://d1234abcd.cloudfront.net
<?php
if (isset($_SERVER['HTTP_X_ORIGIN_VERIFY'])) { $_SERVER['HTTPS'] = 'on'; }
/**
Success: Updated 'home' option.
Success: Updated 'siteurl' option.
+------------------+-----------------------+--------------+------+
| Table            | Column                | Replacements | Type |
+------------------+-----------------------+--------------+------+
| wp_options       | option_value          | 1            | SQL  |
| wp_posts         | post_content          | 2            | SQL  |
...
Success: Made 3 replacements.
```

- 「要換掉的舊網址」必須是 `http://` 加上你的 EIP。
- `head -3` 的第 2 行必須是 `if (isset($_SERVER['HTTP_X_ORIGIN_VERIFY'])) ...`，而且**只出現一次**。
- `search-replace` 的取代數量可能是 0，代表文章裡本來就沒有舊網址，也是正常的。

✅ **驗證 2.5-1**（同一個 Session Manager 視窗）：

```bash
cd /var/www/html
echo "home：   $(sudo -u apache wp option get home)"
echo "siteurl：$(sudo -u apache wp option get siteurl)"
N=$(grep -c HTTP_X_ORIGIN_VERIFY wp-config.php)
[ "$N" = "1" ] && echo "✅ wp-config.php 有 HTTPS 判斷（1 行）" || echo "❌ wp-config.php 的 HTTPS 判斷有 $N 行（應該是 1 行）"
php -l wp-config.php
```

- `home` 和 `siteurl` 都必須是 `https://dxxxx.cloudfront.net`（不能是空白，結尾不要有 `/`）。
- 最後一行要出現 `No syntax errors detected in wp-config.php`。

| 出現的狀況 | 原因 | 解法 |
|---|---|---|
| `syntax error near unexpected token 'newline'` | 把含 `<` `>` 的文字原樣貼進終端機 | 用上面的新版指令，只改第一行的網域 |
| `Value passed for 'home' option is unchanged` 且 `Replacement value '' is identical to search value ''` | 變數是空的，**什麼都沒改** | 用新版指令重跑一次 |
| `home` 是空白 | 曾經把空白值寫進去 | 用新版指令重跑一次，會直接寫入正確網址 |
| HTTPS 判斷有 2 行以上 | 舊版指令重跑過，重複加入 | `sed -i '3{/HTTP_X_ORIGIN_VERIFY/d}' wp-config.php` 刪掉第 3 行重複的那行，再跑一次驗證 |
| `php -l` 顯示 syntax error | `wp-config.php` 被改壞 | 執行 `head -5 wp-config.php` 截圖貼到討論串 |

#### 2.5-2 收緊 ALB，只接受 CloudFront

**為什麼需要**：到目前為止，任何人只要知道 ALB 的網址（`wp-alb-xxxx.elb.amazonaws.com`），就能**繞過 CloudFront** 直接連進網站。這樣 HTTPS、快取、WAF 都會被跳過。這一步加上**兩道鎖**：

```
                              第 1 道鎖：安全群組 wp-alb-sg        第 2 道鎖：ALB 監聽器規則
                              （只允許 CloudFront 的 IP）          （要帶正確暗號）
訪客 ─► CloudFront ─HTTP 80 + X-Origin-Verify: 暗號─►  ✅ 通過  ─►  ✅ 暗號正確 → 轉給 wp-tg
駭客 ──────────────────────直接連 ALB──────────────►  ❌ 擋下（連線逾時）
駭客 ─► 自己的 CloudFront ─（沒有你的暗號）────────►  ✅ 通過  ─►  ❌ 回應 403 Forbidden
```

- **第 1 道鎖（安全群組）**：只允許 AWS 公布的 CloudFront IP 範圍連進來。AWS 把這些 IP 整理成一份**受管前綴清單（managed prefix list）**，名稱是 `com.amazonaws.global.cloudfront.origin-facing`，IP 有變動時 AWS 會自動更新。
- **第 2 道鎖（暗號）**：別人也可以建立自己的 CloudFront 來連你的 ALB，第 1 道鎖擋不住。所以 ALB 還要檢查 2.4 設定的 `X-Origin-Verify` 標頭，暗號正確才轉給主機，否則回應 403。

> 名詞：
> - **Listener rule（監聽器規則）**：ALB 收到請求後，依條件決定怎麼處理。規則依 **Priority（優先順序）**由小到大比對，第一個符合的就執行；都不符合時，執行 **Default（預設）規則**。
> - **Fixed response（固定回應）**：ALB 不轉給主機，直接回一個固定的狀態碼和內容。

**開始前先確認**：
- 2.4 的驗證已經通過（`https://<CF_DOMAIN>/healthz` 回應 `ok`）。
- 2.5-1 已經完成（`home` 是 CloudFront 網址）。
- 準備好 `<ORIGIN_SECRET>`。忘記了的話，到 **CloudFront → `wp-cf` → Origins → 勾選 `wp-alb` → Edit → Add custom header** 查看。方法 B 會自動讀取，不用手動找。

> ⚠️ **順序很重要：先加規則、再改預設動作。** 如果先把預設動作改成 403，在加上規則之前，所有訪客（包括 CloudFront）都會被擋。

下面兩種方法**擇一**即可。

**方法 A：主控台**

**步驟 1：新增「暗號正確才轉發」的規則**

1. 主控台搜尋 **EC2** → 左側 **Load Balancing → Load Balancers** → 點 **`wp-alb`**。
2. 點 **Listeners and rules** 分頁 → 點藍色的 **HTTP:80** 連結，進入這個監聽器的頁面。
3. 在 **Listener rules** 區塊，按 **Add rule**。
4. **Step 1 – Add rule**：**Name** 填 `cloudfront-only` → **Next**。
5. **Step 2 – Define rule conditions**：按 **Add condition**：

| 欄位 | 設定 |
|---|---|
| **Rule condition types** | **HTTP header** |
| **HTTP header name** | `X-Origin-Verify` |
| **HTTP header value** | 貼上 `<ORIGIN_SECRET>`（前後不要有空白） |

   按 **Confirm** → **Next**。

6. **Step 3 – Define rule actions**：

| 欄位 | 設定 |
|---|---|
| **Routing actions** | **Forward to target groups** |
| **Target group** | **`wp-tg`** |

   按 **Next**。

7. **Step 4 – Set rule priority**：**Priority** 填 `1` → **Next**。
8. **Step 5 – Review and create**：確認條件是 `Http header X-Origin-Verify is <暗號>`、動作是 `Forward to wp-tg` → **Create**。

完成後，**Listener rules** 會有兩列：

| Priority | 條件 | 動作 |
|---|---|---|
| `1` | Http header `X-Origin-Verify` is `<暗號>` | Forward to `wp-tg` |
| `Default` | 其他所有請求 | Forward to `wp-tg`（下一步要改成 403） |

**步驟 2：把預設規則改成回應 403**

1. 在同一個頁面的 **Listener rules** 中，勾選 **Default** 那一列 → 右上 **Actions → Edit rule**。（介面不同時，也可以回到上一頁，勾選 HTTP:80 → **Manage listener → Edit listener**，修改 **Default actions**。）
2. **Routing actions** 改成 **Return fixed response**：

| 欄位 | 設定 |
|---|---|
| **Response code** | `403` |
| **Content type** | **text/plain** |
| **Response body** | `Forbidden` |

3. 按 **Save changes**。

**步驟 3：安全群組只允許 CloudFront**

**這一步在做什麼**：`wp-alb-sg` 目前有一條「HTTP 80、來源 `0.0.0.0/0`」的規則，代表全世界任何 IP 都能連到 ALB。我們要把來源換成 **CloudFront 的 IP 清單**，從此只有 CloudFront 的伺服器連得進來。

```
改之前：wp-alb-sg  HTTP 80  來源 0.0.0.0/0                                  → 任何人都能連
改之後：wp-alb-sg  HTTP 80  來源 pl-xxxx（cloudfront.origin-facing 前綴清單） → 只有 CloudFront 能連
```

> **什麼是前綴清單（Prefix list）？** 就是「一組 IP 範圍」打包成一個名字。CloudFront 在全球有幾百個 IP 範圍，不可能一條一條加。AWS 把它們整理成 `com.amazonaws.global.cloudfront.origin-facing`（名稱中的 *origin-facing* 意思是「CloudFront 連到你的來源時用的 IP」），IP 有增減時 **AWS 會自動更新**，你不用管。

> **為什麼要「刪掉再新增」？** 主控台不允許把一條原本是 IP 範圍（`0.0.0.0/0`）的規則，直接改成前綴清單或安全群組，會出現 *You may not specify a referenced group id for an existing IPv4 CIDR rule*（2.3 ④ 遇過）。所以要先刪舊的、再加新的。兩個動作會在同一次 **Save rules** 一起生效，中間不會斷線。

**3-1. 打開 `wp-alb-sg` 的 inbound 規則**

1. 主控台上方搜尋 **VPC** → 進入 VPC 主控台。
2. 左側選單 **Security → Security groups**。
3. 在列表中找到 **Security group name** 是 **`wp-alb-sg`** 的那一列，勾選它。（找不到時，在上方搜尋框輸入 `wp-alb`。）
4. 畫面下方點 **Inbound rules** 分頁。現在應該看到**一條**規則：

| Type | Protocol | Port range | Source |
|---|---|---|---|
| HTTP | TCP | 80 | `0.0.0.0/0` |

5. 按右邊的 **Edit inbound rules**。

**3-2. 刪掉 `0.0.0.0/0` 那一條**

6. 在 `HTTP`、`80`、`0.0.0.0/0` 那一列，按最右邊的 **Delete**。那一列會消失。
   - 如果還有 Source 是 `::/0` 的 HTTP 規則（IPv6 的「所有人」），也一起 **Delete**。

**3-3. 新增「只允許 CloudFront」的規則**

7. 按左下 **Add rule**，會多出一列空白規則。依序填：

| 欄位 | 怎麼填 | 說明 |
|---|---|---|
| **Type** | 下拉選 **HTTP** | **Protocol** 自動變成 TCP，**Port range** 自動變成 `80` |
| **Source type** | **Custom** | |
| **Source**（旁邊的搜尋框） | 輸入 `cloudfront`，下拉選單會分成 *CIDR blocks*、*Security Groups*、*Prefix lists* 幾類。在 **Prefix lists** 底下點選 **`pl-xxxxxxxx`（`com.amazonaws.global.cloudfront.origin-facing`）** | 選好後搜尋框會顯示一個 `pl-` 開頭的標籤 |
| **Description** | `HTTP from CloudFront only` | 選填，只能用英文 |

   ⚠️ 名稱一定要有 **`origin-facing`**。如果下拉選單沒有出現，確認右上角區域是**東京（ap-northeast-1）**，並且 Type 已經選好 HTTP。
   ⚠️ 搜尋框裡**只能有** `pl-xxxx` 這一個標籤。如果還有 `0.0.0.0/0`，按它旁邊的 **X** 刪掉。

8. 檢查畫面：現在**只剩一條**規則：

| Type | Protocol | Port range | Source | Description |
|---|---|---|---|---|
| HTTP | TCP | 80 | Custom：`pl-xxxxxxxx` | HTTP from CloudFront only |

9. 按右下 **Save rules**。畫面上方出現綠色的 *Inbound security group rules successfully modified* 就完成了，**立刻生效**。

> 💡 **如果按 Save rules 出現 *The maximum number of rules per security group has been reached***：前綴清單裡的每個 IP 範圍都算一條規則，CloudFront 這份大約佔 55 條，而一個安全群組預設最多 60 條 inbound 規則。只要 `wp-alb-sg` 裡**只有這一條**，就放得下。出現這個錯誤通常是還有其他規則沒刪（例如 `::/0`、443），刪掉後再存一次。真的需要更多規則時，到 **Service Quotas → Amazon Virtual Private Cloud (Amazon VPC) → Inbound or outbound rules per security group** 申請提高。

✅ **檢查步驟 3**（CloudShell，可以單獨執行）：

```bash
ALB_SG=$(aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].SecurityGroups[0]" --output text)
aws ec2 describe-security-group-rules --filters Name=group-id,Values=$ALB_SG \
  --query "SecurityGroupRules[?IsEgress==\`false\`].{Port:FromPort, FromCIDR:CidrIpv4, FromIPv6:CidrIpv6, FromPrefixList:PrefixListId}" \
  --output table
PL=$(aws ec2 describe-security-group-rules --filters Name=group-id,Values=$ALB_SG \
  --query "SecurityGroupRules[?IsEgress==\`false\`].PrefixListId | [0]" --output text)
aws ec2 describe-managed-prefix-lists --prefix-list-ids $PL --query "PrefixLists[0].PrefixListName" --output text
ALB_DNS=$(aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].DNSName" --output text)
curl -s -m 5 -o /dev/null -w "直接連 ALB：HTTP %{http_code}（000 代表被擋下，正確）\n" "http://$ALB_DNS/"
```

正確結果：
- 表格只有**一列**：Port `80`、FromPrefixList 是 `pl-` 開頭，FromCIDR 和 FromIPv6 都是 `None`。
- 下一行印出 `com.amazonaws.global.cloudfront.origin-facing`。
- 最後一行是 `HTTP 000`。CloudShell 不是 CloudFront，所以被擋下（要等 5 秒）。

| 出現的狀況 | 原因 | 解法 |
|---|---|---|
| 表格有兩列，其中一列 FromCIDR 是 `0.0.0.0/0` | 舊規則沒刪掉 | 回到 3-2，刪掉那一列 → Save rules |
| 印出的名稱不是 `...cloudfront.origin-facing` | 選錯前綴清單 | Edit inbound rules → 刪掉那條 → 照 3-3 重新新增 |
| 直接連 ALB 還是 `HTTP 200` 或 `403` | `0.0.0.0/0` 還在（或剛改完，等幾秒再試） | 同第一列 |
| 改完後透過 CloudFront 變成 **504** | 來源不是 CloudFront 的清單 | 確認名稱是 `origin-facing` 那份 |

**方法 B：CloudShell 指令**

這段會**自動從 CloudFront 讀出暗號**，依照「先加規則、再改預設、先加新安全群組規則、再刪舊規則」的順序執行，不用手動填任何值：

```bash
ALB_ARN=$(aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].LoadBalancerArn" --output text)
LISTENER_ARN=$(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN --query "Listeners[?Port==\`80\`].ListenerArn | [0]" --output text)
TG_ARN=$(aws elbv2 describe-target-groups --names wp-tg --query "TargetGroups[0].TargetGroupArn" --output text)
CF_ID=$(aws cloudfront list-distributions --query "DistributionList.Items[?contains(Origins.Items[0].DomainName, 'wp-alb')].Id | [0]" --output text)
SECRET=$(aws cloudfront get-distribution-config --id $CF_ID \
  --query "DistributionConfig.Origins.Items[0].CustomHeaders.Items[?HeaderName=='X-Origin-Verify'].HeaderValue | [0]" --output text)
echo "CloudFront：$CF_ID    暗號長度：${#SECRET}（應該是 32）"

if [ ${#SECRET} -ge 16 ] && [ "$SECRET" != "None" ]; then
  # 1) 新增規則：暗號正確才轉發到 wp-tg（已有 Priority 1 的規則就跳過）
  R1=$(aws elbv2 describe-rules --listener-arn $LISTENER_ARN --query "Rules[?Priority=='1'].RuleArn | [0]" --output text)
  if [ "$R1" = "None" ]; then
    aws elbv2 create-rule --listener-arn $LISTENER_ARN --priority 1 \
      --conditions '[{"Field":"http-header","HttpHeaderConfig":{"HttpHeaderName":"X-Origin-Verify","Values":["'"$SECRET"'"]}}]' \
      --actions Type=forward,TargetGroupArn=$TG_ARN \
      --tags Key=Name,Value=cloudfront-only --query "Rules[0].Priority" --output text && echo "✅ 已新增規則（Priority 1）"
  else
    echo "已經有 Priority 1 的規則，跳過"
  fi

  # 2) 預設規則改成回應 403
  aws elbv2 modify-listener --listener-arn $LISTENER_ARN \
    --default-actions '[{"Type":"fixed-response","FixedResponseConfig":{"StatusCode":"403","ContentType":"text/plain","MessageBody":"Forbidden"}}]' \
    --query "Listeners[0].DefaultActions[0].Type" --output text && echo "✅ 預設規則已改成 403"

  # 3) 安全群組：先加 CloudFront 前綴清單，再刪 0.0.0.0/0
  ALB_SG=$(aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].SecurityGroups[0]" --output text)
  PL=$(aws ec2 describe-managed-prefix-lists --filters Name=prefix-list-name,Values=com.amazonaws.global.cloudfront.origin-facing \
    --query "PrefixLists[0].PrefixListId" --output text)
  echo "wp-alb-sg：$ALB_SG    CloudFront 前綴清單：$PL"
  aws ec2 authorize-security-group-ingress --group-id $ALB_SG \
    --ip-permissions '[{"IpProtocol":"tcp","FromPort":80,"ToPort":80,"PrefixListIds":[{"PrefixListId":"'"$PL"'","Description":"HTTP from CloudFront only"}]}]' \
    --query "Return" --output text && echo "✅ 已允許 CloudFront"
  aws ec2 revoke-security-group-ingress --group-id $ALB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0 \
    --query "Return" --output text && echo "✅ 已刪除 0.0.0.0/0"
else
  echo "❌ 讀不到暗號，這次沒有做任何修改。請確認 2.4 的 CloudFront 來源有設定 X-Origin-Verify"
fi
```

- `暗號長度` 必須是 `32`。
- 出現 `InvalidPermission.Duplicate`（規則已經存在）或 `InvalidPermission.NotFound`（舊規則已經刪掉）都沒關係，直接做驗證。
- 出現 `RulesPerSecurityGroupLimitExceeded`，看方法 A 步驟 3-3 的說明。

✅ **驗證 2.5**（CloudShell）：等 1–2 分鐘讓設定生效，整段貼上：

```bash
ALB_ARN=$(aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].LoadBalancerArn" --output text)
ALB_DNS=$(aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].DNSName" --output text)
LISTENER_ARN=$(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN --query "Listeners[?Port==\`80\`].ListenerArn | [0]" --output text)
TG_ARN=$(aws elbv2 describe-target-groups --names wp-tg --query "TargetGroups[0].TargetGroupArn" --output text)
ALB_SG=$(aws elbv2 describe-load-balancers --names wp-alb --query "LoadBalancers[0].SecurityGroups[0]" --output text)
CF_ID=$(aws cloudfront list-distributions --query "DistributionList.Items[?contains(Origins.Items[0].DomainName, 'wp-alb')].Id | [0]" --output text)
CF_DOMAIN=$(aws cloudfront get-distribution --id $CF_ID --query "Distribution.DomainName" --output text)

echo "--- 第 2 道鎖：監聽器規則 ---"
aws elbv2 describe-rules --listener-arn $LISTENER_ARN \
  --query "Rules[].{Priority:Priority, Header:Conditions[0].HttpHeaderConfig.HttpHeaderName, Action:Actions[0].Type, Code:Actions[0].FixedResponseConfig.StatusCode}" \
  --output table
R1=$(aws elbv2 describe-rules --listener-arn $LISTENER_ARN --query "Rules[?Priority=='1'].Actions[0].TargetGroupArn | [0]" --output text)
DEF=$(aws elbv2 describe-rules --listener-arn $LISTENER_ARN --query "Rules[?IsDefault].Actions[0].FixedResponseConfig.StatusCode | [0]" --output text)
[ "$R1" = "$TG_ARN" ] && echo "✅ Priority 1：暗號正確 → 轉發到 wp-tg" || echo "❌ 沒有 Priority 1 的轉發規則"
[ "$DEF" = "403" ] && echo "✅ Default：回應 403" || echo "❌ Default 規則不是 403（目前：$DEF）"

echo "--- 第 1 道鎖：wp-alb-sg ---"
aws ec2 describe-security-group-rules --filters Name=group-id,Values=$ALB_SG \
  --query "SecurityGroupRules[?IsEgress==\`false\`].{Port:FromPort, FromCIDR:CidrIpv4, FromPrefixList:PrefixListId}" --output table
PLN=$(aws ec2 describe-security-group-rules --filters Name=group-id,Values=$ALB_SG \
  --query "SecurityGroupRules[?IsEgress==\`false\`].PrefixListId | [0]" --output text)
NAME=$(aws ec2 describe-managed-prefix-lists --prefix-list-ids $PLN --query "PrefixLists[0].PrefixListName" --output text 2>/dev/null)
OPEN=$(aws ec2 describe-security-group-rules --filters Name=group-id,Values=$ALB_SG \
  --query "length(SecurityGroupRules[?IsEgress==\`false\` && CidrIpv4=='0.0.0.0/0'])" --output text)
[ "$NAME" = "com.amazonaws.global.cloudfront.origin-facing" ] && echo "✅ 來源是 CloudFront 前綴清單" || echo "❌ 來源不是 CloudFront 前綴清單（目前：$NAME）"
[ "$OPEN" = "0" ] && echo "✅ 沒有 0.0.0.0/0" || echo "❌ 還有 0.0.0.0/0 規則，請刪除"

echo "--- 實際連線測試 ---"
H=$(curl -s -m 15 "https://$CF_DOMAIN/healthz")
[ "$H" = "ok" ] && echo "✅ 透過 CloudFront：/healthz 回應 ok" || echo "❌ 透過 CloudFront：/healthz 回應「$H」"
curl -s -m 15 -o /dev/null -w "透過 CloudFront 首頁：HTTP %{http_code}\n" "https://$CF_DOMAIN/"
C=$(curl -s -m 5 -o /dev/null -w "%{http_code}" "http://$ALB_DNS/")
case "$C" in
  000) echo "✅ 直接連 ALB：被擋下（逾時），無法繞過 CloudFront";;
  403) echo "✅ 直接連 ALB：403 Forbidden（第 2 道鎖生效；第 1 道鎖可能還沒生效）";;
  *)   echo "❌ 直接連 ALB 還能連上（HTTP $C）";;
esac
aws elbv2 describe-target-health --target-group-arn $TG_ARN --query "TargetHealthDescriptions[].[Target.Id, TargetHealth.State]" --output text
```

正確結果類似：

```text
--- 第 2 道鎖：監聽器規則 ---
-----------------------------------------------------------
|                      DescribeRules                      |
+-----------------+--------+-------------------+-----------+
|     Action      | Code   |      Header       | Priority  |
+-----------------+--------+-------------------+-----------+
|  forward        |  None  |  X-Origin-Verify  |  1        |
|  fixed-response |  403   |  None             |  default  |
+-----------------+--------+-------------------+-----------+
✅ Priority 1：暗號正確 → 轉發到 wp-tg
✅ Default：回應 403
--- 第 1 道鎖：wp-alb-sg ---
--------------------------------------------
|        DescribeSecurityGroupRules        |
+-----------+------------------+-----------+
| FromCIDR  | FromPrefixList   |   Port    |
+-----------+------------------+-----------+
|  None     |  pl-58a04531     |  80       |
+-----------+------------------+-----------+
✅ 來源是 CloudFront 前綴清單
✅ 沒有 0.0.0.0/0
--- 實際連線測試 ---
✅ 透過 CloudFront：/healthz 回應 ok
透過 CloudFront 首頁：HTTP 200
✅ 直接連 ALB：被擋下（逾時），無法繞過 CloudFront
i-0abc123def4567890	healthy
```

- 「直接連 ALB」要等 5 秒才出結果，因為被安全群組擋下的連線不會有回應。
- 目標必須還是 **`healthy`**。健康檢查是 ALB 直接連主機，不受這兩道鎖影響。
- `pl-` 後面的 ID 每個區域不同，名稱對就好。

**最後用瀏覽器確認：**

1. 打開 `https://<CF_DOMAIN>`：網站正常顯示，網址列有鎖頭，網址**一直維持** `https://xxxx.cloudfront.net`，不會跳回 EIP。
2. 打開 `https://<CF_DOMAIN>/wp-admin`：用 1.11 建立的管理員帳號登入，可以進入後台。
3. 打開 `http://<ALB 的 DNS name>`：轉圈圈後顯示無法連線（逾時）。

| 出現的狀況 | 原因 | 解法 |
|---|---|---|
| 透過 CloudFront 得到 **403 Forbidden**（純文字） | ALB 規則的暗號和 CloudFront 的不一樣 | 比對 CloudFront **Origins → Edit** 的 `X-Origin-Verify` 值，和 ALB 規則 Priority 1 的值，必須完全相同（前後不能有空白）。或用方法 B 重跑，它會自動讀取 |
| 透過 CloudFront 得到 **504 Gateway Timeout** | 安全群組的來源不是 CloudFront 的前綴清單，CloudFront 也被擋了 | 回到步驟 3，確認選的是 `com.amazonaws.global.cloudfront.origin-facing` |
| 瀏覽器顯示 **ERR_TOO_MANY_REDIRECTS** | WordPress 不知道訪客用的是 HTTPS | 回到 2.5-1，確認 `wp-config.php` 有 `HTTP_X_ORIGIN_VERIFY` 那一行 |
| 首頁跳回 `http://<EIP>` | 網址沒換成功 | 回到 2.5-1 跑驗證，`home` 和 `siteurl` 必須是 CloudFront 網址 |
| 直接連 ALB 還能連上（HTTP 200） | `0.0.0.0/0` 規則還在，而且預設規則不是 403 | 重新做步驟 2、3 |
| 版面跑掉、圖片破圖 | 瀏覽器快取了舊網址的資源 | 用無痕視窗打開；或到 CloudFront **Invalidations → Create invalidation**，路徑填 `/*` |

> 有自己的網域？到 **ACM（憑證管理）切到 us-east-1 區域** 申請憑證，在 CloudFront 的 Alternate domain name 填上網域並選擇憑證，再到 DNS 設定 CNAME 指向 `<CF_DOMAIN>`，最後重做 2.5-1（第一行填你的網域）。

### 2.6 製作 AMI 與啟動範本

**為什麼需要**：2.7 的 Auto Scaling 會在需要時**自動開新主機**，例如主機故障或流量變大的時候。新主機要一開機就能服務，所以先把 `wp-web-1` 拍成一份「主機快照」（**AMI**），再寫一張「開機設定單」（**啟動範本**）告訴 Auto Scaling 要怎麼開主機。

```
wp-web-1 ──拍照──► AMI wp-web-ami-v1 ──┐
（Nginx、PHP、CloudWatch Agent、         │
  備份排程、EFS 自動掛載設定）            ▼
                     啟動範本 wp-web-lt：用哪個 AMI、多大的主機、哪個安全群組、
                                         哪個 IAM 角色、要不要公有 IP、標籤
                                          │
                                          ▼
                     2.7 Auto Scaling：照這張設定單開新主機 → 自動掛上 EFS → 加入 wp-tg
```

> 名詞：
> - **AMI（Amazon Machine Image）**：整台主機硬碟的快照，包含作業系統、安裝好的軟體和設定檔。用它開出來的主機，內容和拍照當時一模一樣。
> - **Launch template（啟動範本）**：開主機時要填的所有欄位（AMI、主機類型、網路、權限、標籤）存成一份範本，Auto Scaling 每次開主機都照它填。
>
> 💡 **網站檔案不在 AMI 裡**：`/var/www/html` 是 EFS，新主機開機時才接上，所以之後上傳的圖片、安裝的外掛，所有主機都看得到，不用重做 AMI。只有**主機本身的設定**（例如更新 PHP、改 Nginx 設定）改變時，才需要做新版 AMI，見 2.6-6。

#### 2.6-1 做 AMI 前的準備（Session Manager，root）

AMI 拍下的是主機**現在的樣子**。主機有問題，之後每一台新主機都會有同樣的問題，所以要先檢查和整理。

Session Manager 登入 `wp-web-1` → `sudo -i`。

**步驟 1：檢查主機狀態**（整段貼上）

```bash
echo "--- EFS ---"
findmnt -n -o FSTYPE /var/www/html | grep -q nfs4 && echo "✅ /var/www/html 已掛載 EFS" || echo "❌ /var/www/html 沒有掛載 EFS"
N=$(grep -c ' /var/www/html efs ' /etc/fstab)
[ "$N" = "1" ] && echo "✅ /etc/fstab 有一行 EFS 自動掛載設定" || echo "❌ /etc/fstab 的 EFS 設定有 $N 行（應該剛好 1 行，見 2.2 排錯表）"
[ -f /var/www/html/wp-config.php ] && echo "✅ EFS 上有 wp-config.php" || echo "❌ EFS 上找不到 wp-config.php"

echo "--- 開機自動啟動的服務 ---"
for s in nginx php-fpm crond amazon-cloudwatch-agent amazon-ssm-agent; do
  E=$(systemctl is-enabled $s 2>/dev/null); A=$(systemctl is-active $s 2>/dev/null)
  [ "$E" = "enabled" ] && [ "$A" = "active" ] && echo "✅ $s：開機自動啟動、執行中" || echo "❌ $s：is-enabled=$E、is-active=$A"
done

echo "--- 其他 ---"
[ -f /etc/cron.d/wp-backup ] && echo "✅ 每日備份排程存在" || echo "❌ 找不到 /etc/cron.d/wp-backup（回到 2.1 步驟 5）"
[ "$(curl -s localhost/healthz)" = "ok" ] && echo "✅ /healthz 回應 ok" || echo "❌ /healthz 沒有回應 ok"
[ -d /var/www/html.local-backup ] && echo "⚠️ /var/www/html.local-backup 還在（步驟 3 會刪除）" || echo "✅ 沒有 /var/www/html.local-backup"
```

- 每一項都要是 ✅（最後一項是 ⚠️ 也沒關係，步驟 3 會處理）。
- 某個服務顯示 `is-enabled=disabled`：執行 `systemctl enable --now 服務名稱`（例如 `systemctl enable --now crond`），再跑一次。**沒有設成 enabled 的服務，新主機開機後不會自動啟動。**

**步驟 2：讓健康檢查也檢查 EFS**

目前的 `/healthz` 是 Nginx 直接回應 `ok`，**完全不看 EFS**。如果新主機的 EFS 沒掛上（fstab 設了 `nofail`，掛載失敗也會照常開機），`/var/www/html` 會是空的，網站其實是壞的，但 ALB 還是判定 healthy，把訪客送過去。

改成：找不到 `/var/www/html/wp-config.php`（代表 EFS 沒掛上）就回應 **503**，ALB 會判定這台主機不健康，2.7 的 Auto Scaling 會自動把它換掉。

```bash
grep -q 'wp-config.php' /etc/nginx/nginx.conf \
  || sed -i 's|location = /healthz { access_log off; default_type text/plain; return 200 "ok"; }|location = /healthz { access_log off; default_type text/plain; if (!-f /var/www/html/wp-config.php) { return 503 "efs-missing"; } return 200 "ok"; }|' /etc/nginx/nginx.conf
grep -n 'location = /healthz' /etc/nginx/nginx.conf
nginx -t && systemctl reload nginx && echo "✅ Nginx 已重新載入"
curl -s localhost/healthz; echo
```

| 指令 | 意思 |
|---|---|
| `grep -q ... \|\| sed -i ...` | 還沒改過才修改，重跑不會重複改 |
| `if (!-f /var/www/html/wp-config.php) { return 503 ...; }` | 檔案不存在就回應 503 |
| `nginx -t` | 先檢查設定檔語法，正確才重新載入，避免改壞造成網站停擺 |

正確結果：

```text
16:        location = /healthz { access_log off; default_type text/plain; if (!-f /var/www/html/wp-config.php) { return 503 "efs-missing"; } return 200 "ok"; }
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
✅ Nginx 已重新載入
ok
```

> ⚠️ 只在 2.6 做這個修改，**1.7 的 User data 不要改**。Phase 1 時還沒有 `wp-config.php`（1.10 才產生），改了會讓 1.9 的健康檢查失敗。
> 如果 `grep -n` 那一行沒有出現 `if (!-f ...`，代表你的 Nginx 設定和指南不同，`sed` 找不到要取代的文字。執行 `grep -n healthz /etc/nginx/nginx.conf`，截圖貼到討論串。

**步驟 3：刪除本機的舊網站檔案**

2.2 搬到 EFS 時，原本的網站檔案保留在 `/var/www/html.local-backup`。現在 EFS 已經正常運作，這份舊檔案沒有用了。如果不刪，它會被打包進 AMI，之後每一台新主機都有一份。**裡面的 `wp-config.php` 還寫著資料庫密碼**，留著只會增加外洩的風險。

先確認 2.5 的驗證都通過（`https://<CF_DOMAIN>` 正常、後台可登入），再執行：

```bash
if findmnt -n -o FSTYPE /var/www/html | grep -q nfs4 && [ -f /var/www/html/wp-config.php ]; then
  rm -rf /var/www/html.local-backup && echo "✅ 已刪除 /var/www/html.local-backup"
else
  echo "❌ /var/www/html 目前不是 EFS，這次不刪除，請先檢查 2.2"
fi
```

這段會先確認 `/var/www/html` 確實是 EFS，而且有 `wp-config.php`，兩者都成立才刪除，避免誤刪唯一的一份網站檔案。

#### 2.6-2 建立 AMI `wp-web-ami-v1`

> ⏱ 建立 AMI 時主機會**自動重開機一次**，網站會中斷約 **1–3 分鐘**（目前只有一台主機）。重開機可以確保硬碟資料完整寫入，請**不要**取消勾選 Reboot。

**方法 A：主控台**

1. **EC2 → Instances** → 勾選 **`wp-web-1`** → 右上 **Actions → Image and templates → Create image**。
2. 填寫：

| 欄位 | 設定 |
|---|---|
| **Image name** | `wp-web-ami-v1` |
| **Image description** | `WordPress web server v1`（選填） |
| **Reboot instance** | ✅ 保持勾選 |
| **Instance volumes** | 保持預設（**20 GiB**、gp3、Delete on termination 勾選），和 1.7 建立 `wp-web-1` 時設定的大小相同 |
| **Tags** | 選 **Tag image and snapshots together** → **Add new tag**：Key `Name`、Value `wp-web-ami-v1` |

3. 按 **Create image**。
4. 左側 **Images → AMIs**，`wp-web-ami-v1` 的 **Status** 從 *Pending* 變成 **Available**，大約 **5–10 分鐘**。

**方法 B：CloudShell 指令**

```bash
IID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=wp-web-1" "Name=instance-state-name,Values=running" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)
AMI_ID=$(aws ec2 create-image --instance-id $IID --name wp-web-ami-v1 --description "WordPress web server v1" \
  --tag-specifications "ResourceType=image,Tags=[{Key=Name,Value=wp-web-ami-v1}]" "ResourceType=snapshot,Tags=[{Key=Name,Value=wp-web-ami-v1}]" \
  --query ImageId --output text)
echo "AMI：$AMI_ID（等待變成 available，約 5–10 分鐘，期間畫面不會有輸出）"
aws ec2 wait image-available --image-ids $AMI_ID && echo "✅ $AMI_ID 已是 available"
```

`create-image` 預設就會重開機（不加 `--no-reboot`）。

#### 2.6-3 重開機後檢查 `wp-web-1`

這次重開機是**第一次真正測試** 2.2 寫的 fstab：主機開機後，EFS 有沒有自動接上。用這台檢查過，之後的新主機才會一樣正常。

AMI 變成 Available 後，在 **CloudShell** 執行：

```bash
TG_ARN=$(aws elbv2 describe-target-groups --names wp-tg --query "TargetGroups[0].TargetGroupArn" --output text)
aws elbv2 describe-target-health --target-group-arn $TG_ARN --query "TargetHealthDescriptions[].[Target.Id, TargetHealth.State]" --output text
CF_ID=$(aws cloudfront list-distributions --query "DistributionList.Items[?contains(Origins.Items[0].DomainName, 'wp-alb')].Id | [0]" --output text)
CF_DOMAIN=$(aws cloudfront get-distribution --id $CF_ID --query "Distribution.DomainName" --output text)
curl -s -m 15 "https://$CF_DOMAIN/healthz"; echo
```

- 目標狀態是 **`healthy`**，最後一行是 **`ok`**：EFS 有自動接上（`/healthz` 現在會檢查 EFS）。
- 是 `unhealthy`：重新開 Session Manager，執行 `findmnt /var/www/html` 和 `tail -n 20 /var/log/amazon/efs/mount.log`。EFS 沒有接上時執行 `mount -a`，並對照 2.2 的排錯表。**修好前不要用這個 AMI**，修好後刪掉它（AMIs → Deregister，Snapshots → Delete），再做一次 2.6-2。

#### 2.6-4 建立啟動範本 `wp-web-lt`

**方法 A：主控台**

1. **EC2** → 左側 **Instances → Launch Templates** → **Create launch template**。

**Launch template name and description**

| 欄位 | 設定 |
|---|---|
| **Launch template name** | `wp-web-lt` |
| **Template version description** | `v1` |
| **Auto Scaling guidance** | ✅ 勾選 **Provide guidance to help me set up a template that I can use with EC2 Auto Scaling**（會標示 Auto Scaling 需要的欄位） |

**Application and OS Images (Amazon Machine Image)**

2. 點 **My AMIs** 分頁 → 選 **Owned by me** → 選 **`wp-web-ami-v1`**。

**Instance type**

3. 選 **`t3.micro`**。

**Key pair (login)**

4. 選 **Don't include in launch template**（我們用 Session Manager，不用 SSH 金鑰）。

**Network settings**

5. **Subnet**：**Don't include in launch template**（子網路在 2.7 建立 Auto Scaling 時選）。
6. **Firewall (security groups)**：⚠️ **保持空白，不要選任何安全群組**。安全群組要設在下一步的網路介面裡。
7. 展開 **Advanced network configuration** → 按 **Add network interface**，填：

| 欄位 | 設定 | 說明 |
|---|---|---|
| **Device index** | `0` | 主機的第一張（主要）網路卡 |
| **Network interface** | New interface（預設） | |
| **Subnet** | **Don't include in launch template** | 由 Auto Scaling 決定 |
| **Security groups** | **`wp-web-sg`** | ⚠️ 安全群組只能設在這裡。外層第 6 步也選了的話，會出現 *Network interfaces and an instance-level security groups may not be specified on the same request* |
| **Auto-assign public IP** | **Enable** | 我們沒有 NAT Gateway，主機要有公有 IP 才能連到 SSM、CloudWatch、Secrets Manager 和下載更新 |
| **Delete on termination** | **Yes** | 主機刪除時網路卡一起刪除，不會殘留 |

   其他欄位保持預設。

**Configure storage**

8. 保持預設：會自動帶入 AMI 的 **20 GiB gp3**（就是 1.7 設定的大小）。⚠️ 不要改小，硬碟不能比 AMI 小，否則會建立失敗。每台主機的硬碟每月約 $2。

**Resource tags**

9. 按 **Add new tag** 兩次：

| Key | Value | Resource types |
|---|---|---|
| `Name` | `wp-web-asg` | **Instances** |
| `Role` | `wordpress` | **Instances** |

   `Role=wordpress` 很重要：Phase 3 的 AI Agent 用這個標籤找到要檢查的主機。

**Advanced details**（展開）

10. 填：

| 欄位 | 設定 |
|---|---|
| **IAM instance profile** | **`wp-ec2-role`** |
| **Metadata accessible** | **Enabled** |
| **Metadata version** | **V2 only (token required)** |
| **User data** | 保持空白（AMI 裡已經裝好所有東西） |

    其他欄位保持預設。

11. 右側 **Summary** 確認後，按 **Create launch template**。

**方法 B：CloudShell 指令**

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=wp-vpc --query "Vpcs[0].VpcId" --output text)
WEB_SG=$(aws ec2 describe-security-groups --filters Name=vpc-id,Values=$VPC_ID Name=group-name,Values=wp-web-sg \
  --query "SecurityGroups[0].GroupId" --output text)
AMI_ID=$(aws ec2 describe-images --owners self --filters Name=name,Values=wp-web-ami-v1 --query "Images[0].ImageId" --output text)
echo "AMI：$AMI_ID    wp-web-sg：$WEB_SG"

aws ec2 create-launch-template --launch-template-name wp-web-lt --version-description v1 \
  --launch-template-data '{
    "ImageId": "'"$AMI_ID"'",
    "InstanceType": "t3.micro",
    "IamInstanceProfile": {"Name": "wp-ec2-role"},
    "NetworkInterfaces": [{"DeviceIndex": 0, "AssociatePublicIpAddress": true,
                           "Groups": ["'"$WEB_SG"'"], "DeleteOnTermination": true}],
    "MetadataOptions": {"HttpTokens": "required", "HttpEndpoint": "enabled"},
    "TagSpecifications": [{"ResourceType": "instance", "Tags": [
      {"Key": "Name", "Value": "wp-web-asg"}, {"Key": "Role", "Value": "wordpress"}]}]
  }' --query "LaunchTemplate.[LaunchTemplateName, LatestVersionNumber]" --output text
```

- AMI 和 `wp-web-sg` 都必須印出 `ami-`、`sg-` 開頭的 ID，是 `None` 就先不要往下做。
- 最後印出 `wp-web-lt  1` 就成功了。

#### 2.6-5 驗證 2.6（CloudShell）

整段貼上：

```bash
AMI_ID=$(aws ec2 describe-images --owners self --filters Name=name,Values=wp-web-ami-v1 --query "Images[0].ImageId" --output text)
AMI_STATE=$(aws ec2 describe-images --image-ids $AMI_ID --query "Images[0].State" --output text 2>/dev/null)
[ "$AMI_STATE" = "available" ] && echo "✅ AMI $AMI_ID：available" || echo "❌ AMI 狀態：$AMI_STATE"

VPC_ID=$(aws ec2 describe-vpcs --filters Name=tag:Name,Values=wp-vpc --query "Vpcs[0].VpcId" --output text)
WEB_SG=$(aws ec2 describe-security-groups --filters Name=vpc-id,Values=$VPC_ID Name=group-name,Values=wp-web-sg \
  --query "SecurityGroups[0].GroupId" --output text)
aws ec2 describe-launch-template-versions --launch-template-name wp-web-lt --versions '$Latest' \
  --query "LaunchTemplateVersions[0].LaunchTemplateData" > lt-check.json
AMI_ID=$AMI_ID WEB_SG=$WEB_SG python3 - <<'PY'
import json, os
d = json.load(open("lt-check.json"))
ni = (d.get("NetworkInterfaces") or [{}])[0]
tags = {t["Key"]: t["Value"] for s in d.get("TagSpecifications", []) if s["ResourceType"] == "instance" for t in s["Tags"]}
def chk(ok, good, bad): print(("✅ " + good) if ok else ("❌ " + bad))
chk(d.get("ImageId") == os.environ["AMI_ID"], "AMI 是 wp-web-ami-v1", "AMI 不是 wp-web-ami-v1（目前：%s）" % d.get("ImageId"))
chk(d.get("InstanceType") == "t3.micro", "主機類型 t3.micro", "主機類型：%s" % d.get("InstanceType"))
chk(not d.get("SecurityGroupIds") and not d.get("SecurityGroups"), "外層沒有設定安全群組", "外層也設了安全群組，建立主機會失敗")
chk(ni.get("Groups") == [os.environ["WEB_SG"]], "網路介面的安全群組是 wp-web-sg", "網路介面的安全群組不是 wp-web-sg（目前：%s）" % ni.get("Groups"))
chk(ni.get("AssociatePublicIpAddress") is True, "自動指派公有 IP：Enable", "沒有自動指派公有 IP，新主機會連不到 SSM")
chk(ni.get("DeviceIndex") == 0, "Device index 0", "Device index 不是 0")
chk("SubnetId" not in ni, "沒有指定子網路", "網路介面指定了子網路，請改成 Don't include")
chk("wp-ec2-role" in json.dumps(d.get("IamInstanceProfile", {})), "IAM 角色 wp-ec2-role", "沒有設定 IAM 角色 wp-ec2-role")
chk(d.get("MetadataOptions", {}).get("HttpTokens") == "required", "Metadata：V2 only", "Metadata 不是 V2 only")
chk(tags.get("Role") == "wordpress" and tags.get("Name") == "wp-web-asg", "標籤 Name=wp-web-asg、Role=wordpress", "標籤不正確（目前：%s）" % tags)
PY
```

正確結果：

```text
✅ AMI ami-0abc1234def567890：available
✅ AMI 是 wp-web-ami-v1
✅ 主機類型 t3.micro
✅ 外層沒有設定安全群組
✅ 網路介面的安全群組是 wp-web-sg
✅ 自動指派公有 IP：Enable
✅ Device index 0
✅ 沒有指定子網路
✅ IAM 角色 wp-ec2-role
✅ Metadata：V2 only
✅ 標籤 Name=wp-web-asg、Role=wordpress
```

**有 ❌ 時怎麼改**：啟動範本**不能直接修改**，只能建立新版本。到 **Launch Templates** → 勾選 `wp-web-lt` → **Actions → Modify template (Create new version)** → 改好有問題的欄位 → **Create template version**。驗證會自動檢查最新版本，2.7 也會選 *Latest*。

| 出現的狀況 | 原因 | 解法 |
|---|---|---|
| ❌ 外層也設了安全群組 | 第 6 步選了安全群組 | 建立新版本，Firewall 改成 *Don't include in launch template* |
| ❌ 沒有自動指派公有 IP | 網路介面的 Auto-assign public IP 沒選 Enable | 建立新版本修改。沒有公有 IP 的新主機連不到 SSM，也無法通過健康檢查 |
| AMI 一直是 *Pending* | 硬碟比較大或 AWS 正忙 | 再等 10 分鐘；超過 30 分鐘仍是 Pending，可以刪掉重做 |
| AMI 是 *Failed* | 建立失敗 | AMIs → Deregister，再做一次 2.6-2 |
| 找不到 `wp-web-ami-v1` | 不在 My AMIs → Owned by me，或區域不是東京 | 確認右上角區域是 ap-northeast-1 |

#### 2.6-6 注意事項

> 💰 **公有 IP 費用**：AWS 對每個公有 IPv4 位址收費，約每小時 $0.005（每台主機每月約 $3.6）。Auto Scaling 擴充到 2 台時就是 2 個。這是不用 NAT Gateway（每月約 $45）的代價，整體還是比較便宜。

> 📦 **AMI 也要錢**：AMI 背後是 EBS 快照。快照只依**實際用到的空間**計費（20 GiB 的硬碟大約用了 3–5 GB），每月約 $0.2–0.3。做了新版本之後，舊版 AMI 和快照記得刪除（AMIs → Deregister，Snapshots → Delete）。

> 🔄 **之後要更新主機時**（例如更新套件、改 Nginx 設定），不要只改正在執行的主機，因為 Auto Scaling 開的新主機還是舊的。正確做法：
> 1. 在其中一台主機做好修改並測試。
> 2. 用它做新的 AMI：`wp-web-ami-v2`。
> 3. 啟動範本 **Modify template (Create new version)**，只把 AMI 換成 v2。
> 4. **Auto Scaling Groups → wp-asg → Instance refresh → Start instance refresh**，Auto Scaling 會逐台換成新主機，網站不中斷。
> 5. 確認正常後，刪除舊版 AMI 和快照。

> ℹ️ 已經處理過、不用擔心的兩件事：
> - **每台主機都會跑每日備份**：備份檔名包含主機名稱，不會互相覆蓋（2.1 步驟 5）。
> - **`wp-cpu-high` 告警綁在 `wp-web-1`**：2.7 退役 `wp-web-1` 後它會變成 *Insufficient data*，2.8 會改成觀察整個 Auto Scaling 群組。

### 2.7 建立 Auto Scaling Group

1. **EC2 → Auto Scaling Groups → Create Auto Scaling group**。
2. Name `wp-asg`；Launch template `wp-web-lt`（Version: Latest）→ Next。
3. **Network**：

| 欄位 | 設定 | 說明 |
|---|---|---|
| **VPC** | **`wp-vpc`** | ⚠️ 預設可能是 default VPC，要改 |
| **Availability Zones and subnets** | 選 **`wp-subnet-public1-ap-northeast-1a`** 和 **`wp-subnet-public2-ap-northeast-1c`** | 兩個 public 子網路；不要選 private 子網路（沒有對外路由，新主機連不到 SSM） |
| **Availability Zone distribution**（標示 *new*） | **Balanced best effort**（預設） | 盡量讓主機平均分在兩個 AZ；某個 AZ 開不出主機時，改到另一個 AZ 開。另一個選項 *Balanced only* 會在失敗的 AZ 一直重試，主機可能遲遲補不上，不適合我們 |
| **Additional capacity settings**（展開後的 **Capacity Reservation preference**） | **Default**（保持預設，不用展開修改） | 這是給「預先買好的保留容量」用的，我們沒有購買，保持預設就是一般的隨需（On-Demand）主機 |

   按 **Next**。
4. **Configure advanced options**（這一頁由上到下）：

| 區塊／欄位 | 設定 | 說明 |
|---|---|---|
| **Load balancing** | **Attach to an existing load balancer** → **Choose from your load balancer target groups** → **`wp-tg`** | 新主機開好後自動加入 `wp-tg`，ALB 就會把流量送給它 |
| **VPC Lattice integration options** | **No VPC Lattice service**（預設） | 我們用不到 |
| **Application Recovery Controller (ARC) zonal shift** | 不勾選（預設） | 進階的 AZ 故障切換功能，練習用不到 |
| **Health checks** | ✅ 勾選 **Turn on Elastic Load Balancing health checks** | 除了 EC2 本身的狀態，也採用 ALB 的 `/healthz` 結果。`/healthz` 失敗（例如 EFS 沒掛上）的主機會被自動換掉 |
| **Health check grace period** | `300` 秒 | 新主機開機後 5 分鐘內不判定健康狀態，給它時間開機、掛上 EFS |

   按 **Next**。

5. **Configure group size and scaling**（這一頁由上到下）：

| 區塊／欄位 | 設定 | 說明 |
|---|---|---|
| **Group size → Desired capacity** | `1` | 平常維持 1 台 |
| **Scaling → Min desired capacity** | `1` | 最少 1 台，主機壞了會自動補 |
| **Scaling → Max desired capacity** | `2` | 最多 2 台，限制費用上限 |
| **Automatic scaling** | **Target tracking scaling policy** | 依指標自動增減主機 |
| └ **Metric type** | **Average CPU utilization** | |
| └ **Target value** | `60` | 平均 CPU 超過 60% 就加開主機，降下來後再減少 |
| **Instance maintenance policy** | 選 **Prioritize availability**（Launch before terminating） | 要換主機時，**先開好新主機、確認健康，再關掉舊的**。我們平常只有 1 台，選這個才不會在換主機時讓網站中斷 |
| └ **Min healthy percentage** | `100` | 換主機期間，健康的主機數量不能少於目前數量 |
| └ **Max healthy percentage** | ⚠️ 改成 `200` | 換主機期間最多可以暫時變成 2 倍（1 台 → 2 台）。如果用預設的 110%，1 台的群組沒有空間先開新主機 |
| **Additional settings** | 保持預設（Instance scale-in protection、Monitoring、Default instance warmup 都不勾） | |

   按 **Next**。

   > 💡 **Instance maintenance policy 什麼時候會用到？** 主機健康檢查失敗被替換、2.6-6 的 Instance refresh 換新版 AMI 時。另外兩個選項：*Control costs*（先關舊的再開新的，會中斷）、*Flexible*（自訂百分比）。換主機期間短暫多一台，費用只多幾分錢。

6. 後面的頁面（Add notifications、Add tags）都保持預設，按 **Next** 到 **Review** → **Create Auto Scaling group**。

等 3–5 分鐘，`wp-tg` 的 Targets 會出現一台新主機並變 **healthy**。

#### 退役舊主機 `wp-web-1`

**為什麼需要**：`wp-web-1` 是 Phase 1 手動建立的主機，**不屬於 Auto Scaling 管理**。它壞了沒有人會補，而且它和 ASG 的主機同時在 `wp-tg` 裡，等於多付一台主機和一個 Elastic IP 的錢。ASG 的主機正常服務後，就可以把它退役。

```
退役前：ALB ─► wp-tg ─┬─► wp-web-1（手動建立，壞了不會自動補）＋ Elastic IP
                      └─► wp-web-asg（ASG 管理，壞了自動補）
退役後：ALB ─► wp-tg ───► wp-web-asg（只剩 ASG 管理的主機）
```

退役分三步，**順序不要顛倒**：

| 步驟 | 做什麼 | 為什麼 |
|---|---|---|
| 1. Deregister | 把 `wp-web-1` 從 `wp-tg` 移出 | ALB 不再送**新**請求給它，正在處理中的請求可以做完（優雅下線），訪客不會看到錯誤 |
| 2. Terminate | 刪除 `wp-web-1` 主機 | 停止主機和硬碟的費用 |
| 3. Release EIP | 釋放 Elastic IP | 沒有綁在主機上的 Elastic IP **每小時照樣收費**，一定要釋放 |

> 💡 **不會遺失資料**：網站檔案在 EFS、資料庫在 RDS、備份在 S3，`wp-web-1` 的硬碟上已經沒有需要保留的東西。主機的設定也都已經拍進 2.6 的 AMI。

**開始前先確認**（CloudShell，整段貼上）：

```bash
TG_ARN=$(aws elbv2 describe-target-groups --names wp-tg --query "TargetGroups[0].TargetGroupArn" --output text)
echo "--- wp-tg 的主機 ---"
for T in $(aws elbv2 describe-target-health --target-group-arn $TG_ARN --query "TargetHealthDescriptions[].Target.Id" --output text); do
  N=$(aws ec2 describe-instances --instance-ids $T --query "Reservations[0].Instances[0].Tags[?Key=='Name'].Value | [0]" --output text)
  S=$(aws elbv2 describe-target-health --target-group-arn $TG_ARN --targets Id=$T --query "TargetHealthDescriptions[0].TargetHealth.State" --output text)
  echo "$T  $N  $S"
done
ASG_OK=$(aws autoscaling describe-auto-scaling-groups --auto-scaling-group-names wp-asg \
  --query "length(AutoScalingGroups[0].Instances[?HealthStatus=='Healthy' && LifecycleState=='InService'])" --output text)
[ "$ASG_OK" -ge 1 ] 2>/dev/null && echo "✅ wp-asg 有 $ASG_OK 台健康的主機" || echo "❌ wp-asg 沒有健康的主機，先不要退役 wp-web-1"
CF_ID=$(aws cloudfront list-distributions --query "DistributionList.Items[?contains(Origins.Items[0].DomainName, 'wp-alb')].Id | [0]" --output text)
CF_DOMAIN=$(aws cloudfront get-distribution --id $CF_ID --query "Distribution.DomainName" --output text)
[ "$(curl -s -m 15 https://$CF_DOMAIN/healthz)" = "ok" ] && echo "✅ 網站正常（https://$CF_DOMAIN）" || echo "❌ 網站不正常，先不要退役"
```

正確結果類似：

```text
--- wp-tg 的主機 ---
i-0aaa1111bbbb22222  wp-web-1    healthy
i-0ccc3333dddd44444  wp-web-asg  healthy
✅ wp-asg 有 1 台健康的主機
✅ 網站正常（https://d1234abcd.cloudfront.net）
```

**`wp-web-asg` 必須是 `healthy`**，三項都是 ✅ 才往下做。如果 `wp-web-asg` 不是 healthy，先看 2.7 的排錯，否則退役 `wp-web-1` 後網站會停擺。

**方法 A：主控台**

**步驟 1：從目標群組移出（Deregister）**

1. **EC2** → 左側 **Load Balancing → Target Groups** → 點 **`wp-tg`**。
2. 點 **Targets** 分頁。清單中會有兩台主機，看 **Name** 欄找到 **`wp-web-1`**。（沒有 Name 欄時，對照上面確認步驟印出的主機 ID。）
3. 只勾選 **`wp-web-1`** 那一列。⚠️ 不要勾到 `wp-web-asg`。
4. 按右上 **Deregister** → 確認。
5. `wp-web-1` 的 **Health status** 會變成 **Draining**。這是 ALB 在等它把正在處理的請求做完，**最多 300 秒（5 分鐘）**，之後就會從清單消失。
   - 不想等的話，可以直接做步驟 2，對練習網站沒有影響。

**步驟 2：刪除主機（Terminate）**

1. **EC2 → Instances**。
2. 只勾選 **Name 是 `wp-web-1`** 的那一台。⚠️ **再確認一次名稱**，不要勾到 `wp-web-asg`（刪掉的話 ASG 會再開一台，但網站會中斷幾分鐘）。
3. 右上 **Instance state → Terminate (delete) instance** → 確認視窗按 **Terminate (delete)**。
4. **Instance state** 會從 *Shutting-down* 變成 **Terminated**（約 1–2 分鐘）。Terminated 的主機會在清單上顯示約一小時後自動消失，不會收費。
   - 主機的 20 GiB 硬碟會**一起刪除**（1.7 建立時預設 Delete on termination）。
   - 如果出現 *termination protection* 的錯誤：**Actions → Instance settings → Change termination protection** → 取消勾選 **Enable** → Save，再 Terminate 一次。

**步驟 3：釋放 Elastic IP**

1. **EC2** → 左側 **Network & Security → Elastic IPs**。
2. 找到 1.8 建立的那個 IP。主機刪除後，它會**自動解除關聯**，**Associated instance ID** 欄位會是空的（`-`）。
   - 如果還顯示 `wp-web-1` 的主機 ID：先勾選它 → **Actions → Disassociate Elastic IP address** → **Disassociate**。
3. 勾選這個 IP → **Actions → Release Elastic IP addresses** → **Release**。
4. 這個 IP 會從清單消失。⚠️ 釋放之後**無法取回**同一個 IP，不過我們已經改用 CloudFront 網址，不再需要它。

> ⚠️ **為什麼一定要釋放**：AWS 對每個公有 IPv4 位址收費（約每月 $3.6），不管有沒有綁在主機上。主機刪掉了、IP 還留著，就是白付錢，而且很容易忘記。

**方法 B：CloudShell 指令**

```bash
TG_ARN=$(aws elbv2 describe-target-groups --names wp-tg --query "TargetGroups[0].TargetGroupArn" --output text)
IID=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=wp-web-1" "Name=instance-state-name,Values=running,stopped" \
  --query "Reservations[0].Instances[0].InstanceId" --output text)
ALLOC=$(aws ec2 describe-addresses --filters Name=instance-id,Values=$IID --query "Addresses[0].AllocationId" --output text)
echo "wp-web-1：$IID    Elastic IP：$ALLOC"

if [ "$IID" != "None" ]; then
  # 1) 從目標群組移出，等它 draining 完成（最多 5 分鐘）
  aws elbv2 deregister-targets --target-group-arn $TG_ARN --targets Id=$IID && echo "✅ 已移出 wp-tg，等待 draining…"
  aws elbv2 wait target-deregistered --target-group-arn $TG_ARN --targets Id=$IID && echo "✅ draining 完成"
  # 2) 刪除主機
  aws ec2 terminate-instances --instance-ids $IID --query "TerminatingInstances[0].CurrentState.Name" --output text
  aws ec2 wait instance-terminated --instance-ids $IID && echo "✅ wp-web-1 已刪除"
  # 3) 釋放 Elastic IP
  [ "$ALLOC" != "None" ] && aws ec2 release-address --allocation-id $ALLOC && echo "✅ 已釋放 Elastic IP"
else
  echo "找不到執行中的 wp-web-1（可能已經刪除），直接做驗證"
fi
```

- 第一行必須印出 `i-` 開頭的主機 ID，以及 `eipalloc-` 開頭的 Elastic IP ID。
- 整段大約跑 **5–7 分鐘**（draining 最多 5 分鐘、刪除主機 1–2 分鐘），期間畫面會停住，是正常的。

✅ **驗證退役**（CloudShell）：

```bash
TG_ARN=$(aws elbv2 describe-target-groups --names wp-tg --query "TargetGroups[0].TargetGroupArn" --output text)
echo "--- wp-tg 的主機（應該只剩 wp-web-asg）---"
for T in $(aws elbv2 describe-target-health --target-group-arn $TG_ARN --query "TargetHealthDescriptions[].Target.Id" --output text); do
  N=$(aws ec2 describe-instances --instance-ids $T --query "Reservations[0].Instances[0].Tags[?Key=='Name'].Value | [0]" --output text)
  S=$(aws elbv2 describe-target-health --target-group-arn $TG_ARN --targets Id=$T --query "TargetHealthDescriptions[0].TargetHealth.State" --output text)
  echo "$T  $N  $S"
done
L=$(aws ec2 describe-instances --filters "Name=tag:Name,Values=wp-web-1" "Name=instance-state-name,Values=pending,running,stopping,stopped" \
  --query "length(Reservations[].Instances[])" --output text)
[ "$L" = "0" ] && echo "✅ wp-web-1 已刪除" || echo "❌ wp-web-1 還在"
E=$(aws ec2 describe-addresses --query "length(Addresses)" --output text)
[ "$E" = "0" ] && echo "✅ 沒有殘留的 Elastic IP" || echo "❌ 還有 $E 個 Elastic IP，請到 Elastic IPs 釋放"
CF_ID=$(aws cloudfront list-distributions --query "DistributionList.Items[?contains(Origins.Items[0].DomainName, 'wp-alb')].Id | [0]" --output text)
CF_DOMAIN=$(aws cloudfront get-distribution --id $CF_ID --query "Distribution.DomainName" --output text)
[ "$(curl -s -m 15 https://$CF_DOMAIN/healthz)" = "ok" ] && echo "✅ 網站正常（https://$CF_DOMAIN）" || echo "❌ 網站不正常"
```

正確結果類似：

```text
--- wp-tg 的主機（應該只剩 wp-web-asg）---
i-0ccc3333dddd44444  wp-web-asg  healthy
✅ wp-web-1 已刪除
✅ 沒有殘留的 Elastic IP
✅ 網站正常（https://d1234abcd.cloudfront.net）
```

也用瀏覽器打開 `https://<CF_DOMAIN>` 和 `https://<CF_DOMAIN>/wp-admin`，確認網站和後台都正常。

> 📌 **退役之後要注意：**
> - **Session Manager 改連 `wp-web-asg`**：之後要登入主機，在 EC2 → Instances 選 `wp-web-asg` → Connect → Session Manager。ASG 的主機可能被替換，主機 ID 會變，**每次都用名稱找**。
> - **前面指南裡寫 `wp-web-1` 的檢查指令**（例如 1.9、2.3 ④）已經不適用，不用再跑。
> - **`wp-cpu-high` 告警**會變成 *Insufficient data*，因為它觀察的主機不在了。2.8 會改成觀察整個 Auto Scaling 群組。
> - **每日備份不受影響**：ASG 的主機是用 2.6 的 AMI 開的，已經有備份排程，檔名會變成新主機的名稱。

| 出現的狀況 | 原因 | 解法 |
|---|---|---|
| Deregister 後一直是 *Draining* | 正在等請求做完，最多 300 秒 | 等 5 分鐘；或直接 Terminate，不影響 |
| 退役後網站顯示 **503** | `wp-tg` 沒有健康的主機（`wp-web-asg` 不是 healthy） | 看 `wp-web-asg` 的健康狀態和原因（2.3 ③ 排錯表）；ASG 會自動補主機，等 5 分鐘再看 |
| 誤刪了 `wp-web-asg` | 勾錯主機 | 不用處理，ASG 會在幾分鐘內自動開一台新的（這也是 2.7 驗證要測的自我修復） |
| Release 時出現 *address is in use* | IP 還綁在主機上 | 先 **Disassociate**，再 Release |
| 驗證顯示還有 Elastic IP | 沒有釋放，或建立過不只一個 | Elastic IPs 頁面逐一 Release（確認沒有綁在需要的主機上） |

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

> **開始前確認**：Phase 3 會用到 Phase 2 建立的 S3 bucket（放 Lambda 程式）、CloudFront 網址（健康檢查）和 ALB 的 5xx 告警，請先完成 Phase 2。只做了 Phase 1 的話，3.5 要先照 2.1 第 1 步建立 bucket，`SITE_URL` 改填 `http://<EIP>/`，並略過 3.8。

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
4. 在模型頁面點 **Open in playground**，輸入「你好」測試能否回覆。**這一步不能省略**：第一次用管理員帳號呼叫模型時，帳號才會完成該模型的訂閱；Lambda 的角色沒有訂閱權限，若跳過這步，Lambda 第一次呼叫可能會被拒絕。

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

和 1.5 一樣，改用 CloudShell 指令自動帶入帳號 ID，避免手動替換漏掉造成 `AccessDenied`。**先把第一行的 `<MODEL_ID>` 換成 3.1 查到的 ID**，再整段貼上：

```bash
MODEL_ID=<MODEL_ID>          # 例如 3.1 查到的 apac.anthropic.claude-... 或 global.anthropic.claude-...
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=ap-northeast-1

# 1) 建立 Lambda 用的角色，並掛上寫執行日誌的內建政策
cat > /tmp/lambda-trust.json <<'JSON'
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Principal": { "Service": "lambda.amazonaws.com" }, "Action": "sts:AssumeRole" }
  ]
}
JSON
aws iam create-role --role-name wp-agent-role --assume-role-policy-document file:///tmp/lambda-trust.json
aws iam attach-role-policy --role-name wp-agent-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# 2) 最小權限政策
cat > /tmp/wp-agent-policy.json <<JSON
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "InvokeClaudeOnly",
      "Effect": "Allow",
      "Action": ["bedrock:InvokeModel", "bedrock:InvokeModelWithResponseStream"],
      "Resource": [
        "arn:aws:bedrock:*:${ACCOUNT_ID}:inference-profile/${MODEL_ID}",
        "arn:aws:bedrock:*::foundation-model/anthropic.claude-*"
      ]
    },
    { "Sid": "FindWordPressServers", "Effect": "Allow", "Action": "ec2:DescribeInstances", "Resource": "*" },
    {
      "Sid": "SendCommandOnlyToWordPressServers",
      "Effect": "Allow",
      "Action": "ssm:SendCommand",
      "Resource": "arn:aws:ec2:${REGION}:${ACCOUNT_ID}:instance/*",
      "Condition": { "StringEquals": { "ssm:resourceTag/Role": "wordpress" } }
    },
    {
      "Sid": "SendCommandOnlyWithRunShellScript",
      "Effect": "Allow",
      "Action": "ssm:SendCommand",
      "Resource": "arn:aws:ssm:${REGION}::document/AWS-RunShellScript"
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
      "Resource": "arn:aws:logs:${REGION}:${ACCOUNT_ID}:log-group:/wordpress/*"
    },
    {
      "Sid": "ReadSlackWebhook",
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:${REGION}:${ACCOUNT_ID}:secret:wp-agent/slack-webhook-*"
    }
  ]
}
JSON
aws iam put-role-policy --role-name wp-agent-role \
  --policy-name wp-agent-least-privilege --policy-document file:///tmp/wp-agent-policy.json
```

✅ **驗證**：

```bash
aws iam get-role-policy --role-name wp-agent-role --policy-name wp-agent-least-privilege \
  --query "PolicyDocument.Statement[0].Resource[0]" --output text
```

輸出應包含你的 12 位數帳號 ID 和完整的模型 ID，**不能出現 `<` 或 `>`**。

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

BUCKET=wp-backup-$(aws sts get-caller-identity --query Account --output text)
aws s3 cp agent.zip s3://$BUCKET/lambda/agent.zip
```

✅ **驗證**：`du` 顯示 `package` 小於 250 MB（Lambda 解壓後的上限）；`aws s3 cp` 顯示 `upload: ...`。

> 若 `pip3 install` 失敗並提示找不到相容版本，把兩個版本範圍拿掉（只寫 `langchain langchain-aws`）再試一次。

### 3.6 建立 Lambda 函數

用 CloudShell 建立，角色 ARN 與程式位置都會自動帶入。**先把前兩行換成你的值**（`<MODEL_ID>` 來自 3.1，`<CF_DOMAIN>` 來自 2.4）：

```bash
MODEL_ID=<MODEL_ID>
CF_DOMAIN=<CF_DOMAIN>        # 例如 d1234abcd.cloudfront.net（不要加 https://）
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# 沒有設定 Slack（跳過 3.2）的話，把下一行的值改成空字串：SLACK_SECRET_ID=
SLACK_SECRET_ID=wp-agent/slack-webhook

aws lambda create-function --function-name wp-ops-agent \
  --runtime python3.12 --architectures x86_64 \
  --role arn:aws:iam::$ACCOUNT_ID:role/wp-agent-role \
  --handler lambda_function.handler \
  --code S3Bucket=wp-backup-$ACCOUNT_ID,S3Key=lambda/agent.zip \
  --timeout 300 --memory-size 512 \
  --environment "Variables={MODEL_ID=$MODEL_ID,SITE_URL=https://$CF_DOMAIN/,SLACK_SECRET_ID=$SLACK_SECRET_ID}"

aws lambda wait function-active-v2 --function-name wp-ops-agent && echo "Lambda 已就緒"
```

- Timeout 300 秒＝5 分鐘（簡報第 6 頁建議 3–5 分鐘，涵蓋多輪 LLM 推理）；Memory 512 MB。
- 若出現 `The role defined for the function cannot be assumed by Lambda`，代表 3.3 剛建立的角色還沒生效，等 10 秒再執行一次 `create-function`。
- 之後修改程式要重新部署時：重做 3.5 上傳，再執行 `aws lambda update-function-code --function-name wp-ops-agent --s3-bucket wp-backup-$ACCOUNT_ID --s3-key lambda/agent.zip`。

> 2.1 設定的「30 天自動刪除」規則只套用在 `db/` 資料夾，S3 上的 `lambda/agent.zip` 不會被刪除。

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
6. **SNS → Topics → wp-alerts → Subscriptions** → 刪除 Protocol 為 **lambda** 的那筆訂閱（若 Phase 1 清除時會整個刪除 `wp-alerts`，可略過）。
7. Slack App 可在 api.slack.com 刪除。
8. CloudShell 中的 `~/wp-agent` 資料夾可用 `rm -rf ~/wp-agent ~/latest.tar.gz ~/wp-cli.phar` 刪除（CloudShell 本身不收費）。

### 刪除 Phase 2

1. **CloudFront** → 選取 distribution `wp-cf`。⚠️ 如果用的是 **Free 方案**（方法 A），先進入 distribution → 找 **Pricing plan**／**Manage plan** → **Cancel plan**（Free 方案立即取消，不取消就無法刪除）。然後 **Disable** → 等狀態變 Disabled（約 5–15 分鐘）→ **Delete**。Free 方案自動建立的 WAF web ACL，到 **WAF & Shield → Web ACLs**（Global／CloudFront）一併刪除。若有用自己的網域，也到 **ACM（us-east-1）** 刪除憑證、到 DNS 刪除 CNAME。
2. **Auto Scaling Groups** → 刪除 `wp-asg`（會自動終止它的主機）。
3. **Load Balancers** → 刪除 `wp-alb`。
4. **Target Groups** → 刪除 `wp-tg`。
5. **Launch Templates** → 刪除 `wp-web-lt`。
6. **AMIs** → 選取 `wp-web-ami-v1` → **Deregister**；接著 **Snapshots** → 刪除這個 AMI 對應的快照（⚠️ 快照會持續計費，很容易漏掉）。
7. **EFS** → 刪除 `wp-efs`（輸入 File system ID 確認）。
8. **CloudWatch → Alarms** → 刪除 `wp-5xx-high`、`wp-asg-cpu-high`。
9. **S3** → 進入 `wp-backup-<ACCOUNT_ID>` → **Empty** → 再 **Delete** bucket。

### 刪除 Phase 1

1. **RDS → Databases** → `wp-db` → **Actions → Delete**（若 Delete 是灰的或提示 deletion protection，先 **Modify** → 取消勾選 **Enable deletion protection** → Apply immediately）：
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

---

## v3 修正紀錄

以全新帳號從頭照做的角度檢查 v2，找到並修正以下問題：

| 位置 | v2 的問題 | v3 的修正 |
|---|---|---|
| 準備 4 | `aws configure get region` 在 CloudShell 常常是空的，會誤以為區域沒設好 | 改用 `echo $AWS_REGION` |
| 新增準備 5 | 步驟之間傳遞的值（RDS 端點、EFS ID、CloudFront 網域等）沒有統一記錄，CloudShell 斷線後變數會消失 | 新增「數值記錄表」與替換 `<...>` 的說明 |
| 準備 5 | 未提到新帳號 Free plan 的服務限制 | 補上限制說明與升級方式 |
| 1.3 | 未指定 MySQL 版本；8.0 已進入付費延伸支援 | 指定 8.4，並提醒不要勾刪除保護 |
| 1.7 | Nginx 的 PHP 區塊少了 `try_files $uri =404;`（安全性最佳實務） | 已補上 |
| 1.10 | MySQL 8.4 的新驗證方式可能讓 MariaDB 命令列工具連不上 | 補上錯誤訊息與 `mariadb-connector-c` 解法 |
| 2.2 | `rsync` 不一定有預裝；EFS 掛載失敗時主機可能卡在開機 | 先安裝 rsync；fstab 加上 `nofail`；補上 mount 卡住的檢查 |
| 2.3–2.5 | 沒說明中間會有一段網站打不開的時間 | 補上說明 |
| 2.4 | CloudFront 若選月費方案，刪除前要先取消訂閱 | 建議選 Pay-as-you-go |
| 2.6 | 啟動範本同時在外層與網路介面設定安全群組會建立失敗；建立 AMI 會重開機 | 改成只在網路介面設定安全群組；補上重開機提醒 |
| Phase 3 開頭 | 沒寫出對 Phase 2 的依賴 | 補上前提條件與只做 Phase 1 時的替代作法 |
| 3.1 | Playground 測試看似可有可無，其實是完成模型訂閱的必要步驟 | 標註為必做 |
| 3.3 | IAM 政策仍要手動替換 `<ACCOUNT_ID>`、`<MODEL_ID>`，和 v1 造成 AccessDenied 的原因相同 | 改成 CLI 自動帶入帳號 ID，並驗證政策中沒有殘留 `<` `>` |
| 3.6 | 主控台的「Amazon S3 location」需要 https 格式的網址，貼 `s3://` 可能失敗；手動設定項目多 | 改成一行 `aws lambda create-function`，自動帶入角色與程式位置 |
| 3.5 | bucket 名稱要手動替換 | 自動帶入帳號 ID |
| 清除 | 缺少刪除保護、SNS 的 Lambda 訂閱、ACM 憑證、CloudShell 檔案 | 已補上 |
| 2.7 退役舊主機（10/28 補充） | 只有三行，沒有說明順序、等待時間與風險 | 加上退役前確認、三個步驟的原因、主控台逐步操作（Draining、勾錯主機、termination protection、EIP 自動解除關聯）、CLI 做法、驗證、退役後注意事項與排錯表 |
| 2.7 進階選項（10/28 實作） | 沒有提到 Instance maintenance policy 等新欄位 | 改成欄位表：Instance maintenance policy 選 Prioritize availability（Min 100%、Max 200%），並說明 VPC Lattice、ARC zonal shift、健康檢查與寬限期；依實際畫面把 Instance maintenance policy 與 Additional settings 移到 Group size and scaling 頁 |
| 2.7 Network（10/28 實作） | 新版主控台多了 Availability Zone distribution 和 Additional capacity settings，指南沒有提到 | 改成欄位表：選 Balanced best effort、Capacity Reservation preference 保持 Default，並說明原因 |
| 2.6 硬碟大小（10/28） | AMI 和啟動範本寫成 8 GiB，但 1.7 建立主機時設定的是 20 GiB | 改成 20 GiB，說明啟動範本不能改小，修正快照費用說明 |
| 2.6（10/28 審核） | 做 AMI 前沒有檢查；`/healthz` 不檢查 EFS，EFS 沒掛上的主機也會被判定健康；重開機後沒有確認；沒有驗證；舊網站檔案（含資料庫密碼）會被打包進 AMI；網路介面欄位不完整 | 改寫為 2.6-1～2.6-6：做 AMI 前檢查與整理（服務開機自動啟動、`/healthz` 檢查 `wp-config.php`、刪除 local-backup）、AMI 主控台／CLI、重開機後檢查、啟動範本逐欄位設定與 CLI、一鍵驗證與排錯、公有 IP 費用與日後更新 AMI 的流程 |
| 2.5-2 步驟 3（10/28 補充） | 安全群組改成 CloudFront 前綴清單的步驟太簡略 | 拆成 3-1～3-3 逐畫面步驟，解釋前綴清單與「刪掉再新增」的原因、規則上限，加上可單獨執行的檢查與排錯表 |
| 2.5-2（10/28 補充） | 收緊 ALB 只有四行說明 | 說明兩道鎖（CloudFront 前綴清單＋暗號標頭）與「先加規則再改預設」的順序；主控台新增規則 5 個畫面、預設規則改 403、安全群組刪除後新增；CLI 自動從 CloudFront 讀出暗號；一鍵驗證、瀏覽器確認與排錯表 |
| 2.5-1（10/28 實作） | 把 `OLD=http://<EIP>` 原樣貼上，`<` 被 bash 當成重新導向，出現 `syntax error near unexpected token 'newline'`，網址變數是空的 | EIP 改從主機中繼資料自動取得，只需填 CloudFront 網域並自動檢查格式；wp-config 修改改成不會重複加入；加上逐段解釋、正確輸出、驗證與排錯表 |
| 2.4 方法 A（10/28 實作） | 實際畫面沒有「Choose a plan」這一頁 | 方案選擇改成「有看到就選 Free，沒有就直接往下做（Pay-as-you-go）」；WAF 步驟依方案分成兩種做法 |
| 2.4（10/28 實作） | 新版主控台建立 distribution 時先選月費方案，畫面看不到 Pay-as-you-go，指南寫的選項對不上 | 改寫 2.4：方案比較表（建議 Free）與四個注意事項；方法 A 依新精靈逐頁說明，強調來源要改 HTTP only（推薦設定是 HTTPS only 會 502）；方法 B 用 CLI 建立 Pay-as-you-go；建立後檢查、一鍵驗證與排錯表；清除步驟加上先取消 Free 方案 |
| 2.3 ④（10/27 補充、實作） | 收緊 `wp-web-sg` 只有一行說明；實作時直接改 Source 出現 *You may not specify a referenced group id for an existing IPv4 CIDR rule* | 改成主控台「刪掉舊規則、Add rule 新增」的步驟、先加後刪的 CLI 做法、驗證腳本（規則、ALB 通、EIP 不通）與排錯表；修正「做完 ④ 網站就無法瀏覽」的說法 |
| 2.3 ③驗證（10/27 實作） | 網址只印在輸出最上方容易被捲走；實際首頁回應 200，不是 301 | 最後一行再印一次 ALB DNS name，並說明主控台哪裡複製；說明首頁 200 或 301 都正常 |
| 2.3 ②③（10/27 補充） | 目標群組和 ALB 只有簡短說明 | 改成逐欄位設定表（含 Advanced health check）、CLI 做法、驗證腳本與 unhealthy 排錯表；說明 `unused` 狀態和首頁 301 轉址是正常的 |
| 2.3 ①（10/27 補充） | ALB 安全群組只有一行說明 | 改成和 `wp-efs-sg` 相同的格式：用途與兩階段示意圖、主控台／CLI 兩種做法、驗證與排錯表 |
| 2.2（10/27 補充） | 建立 EFS 安全群組與 EFS 的步驟太簡略，搬移指令沒有解釋（3-3～3-6 再加強：逐行解釋、正確畫面、每步完成後的狀態） | 改寫成 4 個步驟：安全群組（主控台／CLI）、EFS 逐欄位設定、逐段解釋的搬移指令、一鍵驗證；加上排錯表與還原方法 |
| 2.1 驗證 3（10/27 補充） | 依賴前面設定的變數，CloudShell 重新連線後出現 `NoSuchBucket` | 改成可單獨執行並自動判斷 ✅／❌，新增區域與加密檢查、正確輸出範例和每項 ❌ 的修正方法 |
| 2.1 驗證 2（10/27 補充） | 要手動填檔名，也沒解釋指令 | 自動抓最新備份檔；拆成 4 個小步驟（找檔案、看開頭、數資料表、看結尾），逐段解釋指令並附正確輸出與判斷表 |
| 2.1 命名空間（10/27 補充） | 新版 S3 主控台多了 Bucket namespace 選項，預設推薦的 Account Regional 會自動加上名稱後綴，造成 IAM 政策不符 | 指定選 Global namespace，說明原因與選錯時的處理 |
| 2.1（10/27 補充） | 建立步驟簡略；生命週期規則套用到整個桶，會刪掉 Lambda 程式；排程用 UTC 時間（台灣上午 11 點才備份） | 改寫成 5 個步驟（主控台與 CLI 兩種做法）；規則只套用 `db/`；排程改為台灣時間 03:00；加上權限測試、備份內容與桶設定的驗證、還原方法 |
| Phase 1 檢查表（10/26 補充） | 只有清單，沒有說明怎麼檢查 | 新增 CloudShell 一鍵檢查（自動判斷 ✅／❌）、逐項的主控台位置與修正方法，以及收回 RDS 管理員密碼權限的指令 |
| 1.13（10/26 補充） | 步驟太簡略；v3 的 Nginx 設定不會把 404 寫進錯誤日誌，原本叫你去 nginx-error 找會找不到 | 改用好認的測試網址並指向 nginx-access；逐步說明 Log group／Log stream 的位置與搜尋方法，加上 CLI 確認與排錯表 |
| 1.12（10/26 補充） | 「Notification：In alarm → 選 wp-alerts」太簡略，看不懂要在哪個畫面設定 | 改寫成 Part A（SNS 主題與 Email 訂閱）＋ Part B（告警精靈 Step 1～4 逐畫面說明），並加上確認訂閱與測試告警通知的方法 |
| 1.11（10/26 補充） | 只有三行簡述 | 改寫成 7 個步驟：找 IP、安裝前連線檢查、瀏覽器 http 注意事項、語言、各欄位填法、登入、時區與永久連結；加上 WP-CLI 驗證與常見問題表 |
| 1.10（10/26 補充） | 沒有說明如何確認 RDS 已是 Available；Endpoint 要手動替換 | 新增步驟 0（主控台狀態表、CLI 查詢、自動等待指令）；Endpoint 改為自動取得 |
| 1.9（10/26 補充） | 四個檢查指令沒有說明作用，也沒有示範正確與錯誤時的畫面；沒有提醒等待時間與執行時畫面會空白 | 逐行解釋並附上正確輸出範例、錯誤處理、一次檢查全部的腳本；新增等待時間表、操作注意事項（不要按 Ctrl+C、畫面卡住的判斷）與四步診斷流程 |
| 1.7（10/26 補充） | 加標籤的步驟太簡略，沒說明標籤用途與操作位置 | 改成逐步操作，說明 `Name`、`Role` 兩個標籤的用途、大小寫、事後補加方式與驗證指令 |
| 1.4（10/26 補充） | 驗證只確認 secret 名稱，看不到 username 是否正確 | 新增讀出 username 與密碼長度的驗證，以及空密碼、重複建立時的處理；`--query` 欄位名稱改用英文（JMESPath 不接受中文欄位名稱） |
| 1.2（10/26 補充） | 沒有提醒 VPC 欄位預設是 default VPC，要手動改成 `wp-vpc` | 改寫成逐步操作，加上醒目提醒、CLI 驗證與建錯時的處理；2.2、2.3 的安全群組也加上同樣提醒 |

**無法在本機驗證、請實際操作時留意的地方**：主控台按鈕名稱（AWS 經常改版）、Bedrock 可用的模型 ID、Free plan 的實際限制範圍、MariaDB 用戶端與 MySQL 8.4 的相容性，以及 LangChain 套件的最新版本。以上都已附上查詢方法或遇錯時的解法。

---

## 2026-10-26 實作紀錄：修改內容與學習到的經驗

這一頁記錄 2026-10-26 照著 v3 指南實際操作 Phase 1 的過程：遇到的問題、指南因此做的修改，以及學到的經驗。之後重做或教別人時，可以先看這一頁。

### 一、今天的實作進度

| 階段 | 結果 |
|---|---|
| 1.1–1.8 網路、安全群組、RDS、密碼、IAM、EC2、Elastic IP | ✅ 完成（主機 IP `10.0.5.40`，位於 public1 子網路） |
| 1.9 Session Manager 檢查 | ✅ 完成。User data 一度沒跑完，重跑後全部通過 |
| 1.10–1.13 資料庫使用者、WordPress 安裝、告警、日誌 | 進行中，指南已依操作時的疑問補充 |
| Phase 1 完成檢查表 | 已改成可執行的一鍵檢查 |

### 二、今天修改的內容

| 章節 | 修改前的問題 | 修改內容 |
|---|---|---|
| 1.2 安全群組 | 沒有提醒 VPC 欄位預設是 default VPC | 逐步操作，醒目提醒要改成 `wp-vpc`，加上 CLI 驗證與建錯時的處理；2.2、2.3 也加上同樣提醒 |
| 1.4 資料庫密碼 | 驗證只看得到 secret 名稱，看不到 username | 新增讀出 username 與密碼長度的驗證，以及空密碼、重複建立時的處理 |
| 1.4 驗證指令 | `--query` 使用中文欄位名稱，執行會出錯 | 改用英文欄位名稱 |
| 1.7 標籤 | 加標籤的步驟太簡略 | 說明 `Name`、`Role` 的用途、大小寫、操作位置、事後補加方式與驗證指令 |
| 1.9 主機檢查 | 四個指令沒有解釋，也沒有正確畫面範例 | 逐行解釋並附輸出範例、錯誤處理表、一鍵檢查腳本 |
| 1.9 開頭 | 沒有等待時間與操作注意事項 | 新增等待時間表、六點注意事項、四步診斷流程、重跑 User data 的方法 |
| 1.10 資料庫使用者 | 沒說明怎麼確認 RDS 已經 Available；Endpoint 要手動替換 | 新增步驟 0（主控台狀態表、CLI 查詢、自動等待）；Endpoint 改為自動取得 |
| 1.10 執行位置 | 沒說明為什麼不能在 CloudShell 跑；沒說 Session Manager 在哪裡 | 補上原因、CloudShell 與 Session Manager 的分工、開啟步驟與提示字元對照表 |
| 1.11 WordPress 安裝 | 只有三行 | 改寫成 7 個步驟，加上安裝前連線檢查、欄位填法、驗證指令與常見問題表 |
| 1.12 告警通知 | 「Notification：In alarm → 選 wp-alerts」看不懂 | 拆成 Part A（SNS 與 Email 訂閱）與 Part B（告警精靈 Step 1～4 逐畫面說明），加上測試通知的方法 |
| 1.13 CloudWatch 日誌 | 步驟太簡略；叫你到 `nginx-error` 找 404 紀錄會找不到 | 改用好認的測試網址並指向 `nginx-access`，加上搜尋方法、CLI 確認與排錯表 |
| Phase 1 完成檢查表 | 只有清單，沒說怎麼檢查 | 新增一鍵檢查、逐項主控台位置與修正方法、收回 RDS 管理員密碼權限的指令 |

### 三、學習到的經驗

**1. AWS 主控台的「預設值」不一定是你要的**
建立安全群組、EC2、RDS 時，VPC 欄位預設會帶入 AWS 自動建立的 default VPC。每次建立資源時，都要**主動確認 VPC 欄位是 `wp-vpc`**。安全群組建錯 VPC 之後不能修改，只能刪掉重建。

**2. 「建立成功」不等於「內容正確」**
`describe-secret` 只能證明 secret 存在，證明不了裡面的 username 和密碼是對的。驗證時要**讀出實際內容**來確認，例如 username 是 `wpuser`、密碼長度是 32。

**3. AWS CLI 的 `--query` 欄位名稱只能用英文**
`--query` 使用 JMESPath 語法，自訂的欄位名稱（例如 `{Name:Name}`）不能用中文。想看中文說明，用 `echo` 另外印出。

**4. 標籤（Tag）是給程式看的，大小寫要完全一致**
`Role=wordpress` 不只是備註。Phase 3 的 AI Agent 靠它找主機，IAM 權限也靠它限制 Agent 能操作哪台主機。打成 `role` 或 `WordPress`，程式就找不到主機。

**5. User data 是「默默」執行的**
User data 腳本把輸出都寫進 `/var/log/user-data.log`，執行時畫面一片空白，很容易誤以為當機。
- 判斷進度：用 `cloud-init status --long` 看狀態，用 `tail -f /var/log/user-data.log` 看即時日誌。
- **不要按 Ctrl+C**。今天就是因為按了 Ctrl+C 中斷了腳本，日誌最後變成空白、也沒有出現 DONE。
- 中斷了也不用重建主機，執行 `bash /var/lib/cloud/instance/scripts/part-001` 重跑即可。

**6. 看懂終端機的狀態**

| 畫面最後一行 | 意思 |
|---|---|
| `>` | 指令還沒輸入完整（例如 `<<'EOF'` 在等結尾），按 Ctrl+C 取消 |
| `sh-5.2$` | Session Manager，但還不是 root，先輸入 `sudo -i` |
| `[root@ip-10-0-x-x ~]#` | Session Manager 的 root，在主機裡面 |
| `[cloudshell-user@... ~]$` | CloudShell，在主機外面 |
| 打字不會顯示 | 網頁終端機斷線了，關掉分頁重新 Connect |

另外，指令要**一次貼一行**，等出現新的提示字元再貼下一行。

**7. 先確認「準備好了」再往下做**
很多錯誤的原因只是資源還沒準備好，例如 RDS 要 5–15 分鐘才會變成 Available、主機開機後 3–5 分鐘才裝完軟體、新建的指標要 5–10 分鐘才會出現。每一步開始前先確認狀態，可以省掉很多除錯時間。

**8. CloudShell 和 Session Manager 的分工，取決於網路位置**
- **CloudShell** 在 `wp-vpc` 外面：適合查詢和管理 AWS 資源（RDS 狀態、Secrets Manager、IAM）。
- **Session Manager** 在主機裡面：連資料庫（`mysql`）和操作 WordPress（`wp`）都只能在這裡做。
- 資料庫連不到是**刻意的安全設計**，代表防護有效，不是故障。

**9. 能自動抓的值，就不要手動替換**
`<RDS_ENDPOINT>`、`<ACCOUNT_ID>` 這類需要手動替換的佔位文字，最容易出錯（忘記換、`<` `>` 沒刪掉）。能用指令自動取得的值（帳號 ID、RDS Endpoint、Elastic IP），指南都已改成自動帶入。另外，CloudShell 斷線後變數會消失，記得把重要的值寫進「準備 5：數值記錄表」。

**10. 通知要「測試過」才算設定好**
SNS 的 Email 訂閱要在確認信裡點過連結（狀態為 Confirmed）才會生效。設定好告警後，用 `aws cloudwatch set-alarm-state` 手動觸發一次，確認真的收得到信。

**11. 看日誌要找對地方、看對時間**
- 一般請求（包括 404）記錄在 `nginx-access`；只有真的發生錯誤時，`nginx-error` 和 `php-fpm-error` 才會有內容。
- CloudWatch 預設顯示 UTC 時間，比台灣慢 8 小時，可以改成 Local timezone。
- 測試時使用好認的網址（例如 `/cloudwatch-test-page`），方便搜尋。

**12. 用完的權限要收回**
RDS 管理員密碼只在 1.10 用一次，用完就從 `wp-ec2-role` 收回（Phase 1 檢查表的 B-7）。權限給得越少，主機被入侵時的損失就越小，這就是簡報第 11 頁講的「IAM 最小權限」。

---

## 2026-10-27 實作紀錄：修改內容與學習到的經驗

這一頁記錄 2026-10-27 照著 v3 指南實際操作 Phase 2（2.1–2.3）的過程：遇到的問題、指南因此做的修改、學到的經驗，以及接下來要注意的事項。

### 一、今天的實作進度

| 階段 | 結果 |
|---|---|
| 2.1 S3 備份桶、第一次資料庫備份 | ✅ 完成。驗證 3 一度出現 `NoSuchBucket`，改成可單獨執行的腳本後通過 |
| 2.2 EFS（`wp-efs-sg`、`wp-efs`、搬移網站檔案） | ✅ 完成 |
| 2.3 ① ALB 安全群組 `wp-alb-sg` | ✅ 完成 |
| 2.3 ② 目標群組 `wp-tg`、③ ALB `wp-alb` | ✅ 完成。透過 ALB 的 `/healthz` 回應 `ok`，首頁回應 200 |
| 2.3 ④ 收緊 `wp-web-sg` | 進行中。主控台直接改 Source 會出錯，指南已改成「刪掉舊規則、再新增」 |

### 二、今天修改的內容

| 章節 | 修改前的問題 | 修改內容 |
|---|---|---|
| 2.1 建立步驟 | 步驟簡略；生命週期規則套用到整個桶，會刪掉 Lambda 程式；排程用 UTC 時間 | 改寫成 5 個步驟；規則只套用 `db/`；排程改為 `CRON_TZ=Asia/Taipei` 03:00 |
| 2.1 命名空間 | 新版 S3 主控台預設推薦 Account Regional namespace，會在桶名後面自動加後綴，IAM 政策對不上 | 指定選 **Global namespace**，並說明選錯時的處理 |
| 2.1 驗證 2、3 | 要手動填檔名；依賴前面設定的變數，CloudShell 重新連線後出現 `NoSuchBucket` | 自動抓最新備份檔；每個驗證都改成可單獨執行、自動判斷 ✅／❌ |
| 2.2 EFS | 安全群組與 EFS 的步驟太簡略，搬移指令沒有解釋 | 改寫成 4 個步驟，逐段解釋搬移指令（3-3～3-6），加上排錯表與還原方法 |
| 2.3 ① | ALB 安全群組只有一行說明 | 改成用途、主控台／CLI、驗證、排錯表的完整格式 |
| 2.3 ②③ | 目標群組和 ALB 只有簡短說明 | 逐欄位設定表（含 Advanced health check）、CLI、驗證腳本、unhealthy 排錯表 |
| 2.3 ③ 驗證 | ALB 網址只印在輸出最上方，容易被捲走；只寫首頁會回 301 | 最後一行再印一次 DNS name，說明主控台哪裡複製；首頁回 200 或 301 都正常，200 時「轉址到」是空白 |
| 2.3 ④ | 只有一行說明；照做會出現 *You may not specify a referenced group id for an existing IPv4 CIDR rule* | 改成「Delete 舊規則 → Add rule 新增」；CLI 先加後刪；驗證 ALB 通、EIP 不通 |

### 三、學習到的經驗

**1. 主控台的新選項，預設推薦不一定適合你**
S3 新增的 Bucket namespace 預設推薦 *Account Regional*，但它會在桶名後面自動加上 `-<帳號>-<區域>-an`，和 IAM 政策裡寫的 `wp-backup-<ACCOUNT_ID>` 對不上。看到不認識的新選項時，先確認它會不會改變資源名稱或行為，再決定要不要用預設值。

**2. CloudShell 的變數不會一直存在**
CloudShell 閒置或重新連線之後，前面設定的 `$ACCOUNT_ID`、`$ALB_DNS` 都會消失，指令就會變成去找一個不存在的桶（`NoSuchBucket`）。所以指南裡的驗證都改成**整段可以單獨貼上執行**，開頭自己重新取得需要的值。

**3. 生命週期規則和排程，要看清楚「範圍」和「時區」**
- 生命週期規則沒有限制前綴（Prefix）的話，會套用到整個桶，連 Lambda 程式也會被刪掉。只對 `db/` 套用 30 天刪除。
- cron 預設用 UTC，「03:00」其實是台灣上午 11 點。加上 `CRON_TZ=Asia/Taipei` 才是台灣時間。

**4. 安全群組的來源，用「安全群組」比用「IP」好**
`wp-db-sg`、`wp-efs-sg`、做完 ④ 的 `wp-web-sg`，來源都是另一個安全群組。ALB、主機的 IP 會變動，用安全群組當來源就不必管 IP，也只放行特定的 AWS 資源。只有面對網際網路訪客的 `wp-alb-sg` 才用 IP 範圍（`0.0.0.0/0`）。

**5. 「IP 範圍」的規則不能直接改成「安全群組」來源**
主控台編輯 inbound 規則時，原本是 `0.0.0.0/0` 的規則，不能直接把 Source 換成 `wp-alb-sg`，會出現 *You may not specify a referenced group id for an existing IPv4 CIDR rule*。正確做法是**刪掉舊規則、Add rule 新增一條**，同一次 Save 一起生效。用 CLI 時則是**先加新規則、再刪舊規則**，中間不會斷線。

**6. 目標群組的狀態要配合當下的進度來看**
- 還沒接到 ALB 時，主機狀態是 `unused`，這是正常的。
- 剛接上時是 `initial`，大約 30 秒到 1 分鐘後變成 `healthy`。
- 健康檢查用 `/healthz`，不用首頁 `/`，因為首頁可能回應轉址，會被當成失敗。

**7. 「沒有輸出」不一定是錯誤**
- `curl` 的「轉址到：」是空白，是因為首頁回應 200，沒有轉址。
- ALB 的網址印在輸出的第一行，貼上的結果太長被捲走而已。只要 `/healthz` 回應 `ok`，變數就一定有值。
- 直接連 EIP 要等 5 秒才出結果，因為被防火牆擋下的連線不會有回應，只能等到逾時。

**8. 實際結果和指南的「範例輸出」可以不完全一樣**
指南原本只寫首頁會回 301，實際是 200，兩種都正常。看驗證結果時，重點是**判斷標準**（✅／❌、`healthy`、`ok`），不是每個字都要一樣。

### 四、接下來要注意的事項

| 注意事項 | 說明 |
|---|---|
| 💰 **ALB 建立後就開始計費** | 每月約 $20–25，不管有沒有流量。今天不繼續做的話，可以先照「清除所有資源」刪掉 ALB，下次再建 |
| 🌐 **做完 ④ 之後，要用 ALB 網址瀏覽網站** | `http://<EIP>` 會打不開（代表防火牆生效）。WordPress 記得的網址還是 EIP，部分連結和後台登入可能被導回 EIP，2.5 改成 CloudFront 網址後就會恢復 |
| 📝 **把 ALB DNS name 記到「準備 5：數值記錄表」** | 2.4 建立 CloudFront 時的 Origin domain 要用它 |
| ✅ **重跑 Phase 1 完成檢查表時** | 第 2 項原本預期 `wp-web-sg` 的來源是 `0.0.0.0/0`；做完 ④ 之後正確答案是 `wp-alb-sg`，不用改回去 |
| 🔐 **2.5 還要再收緊一次** | `wp-alb-sg` 目前允許所有人連 HTTP 80，2.5 會改成只允許 CloudFront，並加上 `X-Origin-Verify` 秘密標頭 |
| 💾 **確認每天的備份有在跑** | 明天早上到 S3 `wp-backup-<ACCOUNT_ID>/db/` 看有沒有新的備份檔（台灣時間 03:00 產生） |
| 🔁 **Session Manager 不受防火牆影響** | 收緊 `wp-web-sg` 之後照樣登得進去，因為 Session Manager 是主機主動連出去 |
