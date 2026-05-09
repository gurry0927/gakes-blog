# gakes.net 文章規劃

## 已發布

- **三塊硬盤的分工戰略：用老零件撐起一個 NAS**
  - SSD/HDD/備份盤分工、hardlink 快照、RAID 誤解

---

## 概念層（完全小白，建立基礎認知）

### Docker 是什麼：為什麼 NAS 上的服務都用它跑
- 怎麼扒：公司 VM 踩坑的對比（接第一篇故事）、container vs VM 的差異
- 補充：image / container / compose 三個詞的關係、為什麼「壞了重建比修復快」

### NAS 憲法：資料怎麼擺、服務怎麼設計、怎麼讓 AI 也遵守
- 怎麼扒：
  - **資料分層邏輯** — SSD 放效能敏感（DB）、HDD 放大檔案（媒體）、backup 只放備份，每一條都有原因
  - **服務擺放原則** — `/data/services/<name>/` 放 compose 設定、`/opt/<name>/` 放 DB（SSD）、`/data/<name>/` 放媒體（HDD），結構統一才能自動備份
  - **禁止事項清單** — INFRA.md 裡的「不」字規則，每一條背後都有踩過的坑（比如「不把 secrets 寫進 compose」、「不在系統層 apt install 服務」）
  - **AI 遵守的方式** — 把 INFRA.md 作為每次對話的背景知識，開口前先丟進去；AI 才知道你的設計決策，不會每次給出違反架構的建議
- 補充：「文件即基礎設施」概念、為什麼規則比記憶可靠、壞機重建時 INFRA.md 是救命稻草

### SSH Key 是什麼：為什麼有它就能來去自如
- 小白問題（直接寫進文章）：
  - 「禁用密碼登入之後，sudo 還要密碼嗎？」→ SSH 密碼和 sudo 密碼是兩件事，進門用鑰匙，進門後動系統還是要密碼
  - 「為什麼有 SSH key 就能免密碼？」→ NAS 的 authorized_keys 記錄哪些公鑰允許登入，你有對應私鑰就進得來
  - 「那不是很危險？誰都能拿 key 進來？」→ 你控制 authorized_keys 清單，只有登記在上面的才能進
- 怎麼扒：authorized_keys 的兩把 key（Mac 和 ClaudeCode）、private/public key 的關係
- 補充：在 NAS 上禁用密碼登入的步驟、為什麼 key-based 比密碼更安全

### NAS 安全加固：UFW、fail2ban、禁用密碼登入
- 小白問題（直接寫進文章）：
  - 「我以為有 Cloudflare Tunnel 就安全了」→ Tunnel 保護的是服務 port，SSH 是另一條路
  - 「UFW 設定前為什麼要先開 22？」→ 先開洞再關門，不然把自己鎖在外面
  - 「安裝 UFW 為什麼會移除 iptables-persistent？」→ 兩個都在管防火牆規則，不能並存，要手動把規則搬進 UFW
- 怎麼扒：剛剛做的三步驟（禁密碼 → UFW → fail2ban）、VM DNAT 規則搬到 before.rules 的原因
- 補充：Tailscale 的 tailscale0 為什麼要全開、fail2ban 的運作原理

### 防禦工程：當第二個 AI 跟我說我的 NAS 在裸奔
- 起點故事：別的 AI 看完我家服務描述後丟出三條警告（Docker 繞過 UFW、Samba bind 在 `0.0.0.0`/`[::]`、IPv6 直連是現代家用 NAS 的盲區）；我請 ClaudeCode 進來深掃，又挖出更多——`.env` 644 全機可讀、nas-dashboard 是 `docker.sock + network: host` 的「單點崩潰」、Immich Postgres image 卡 7 個月沒更新
- 小白問題（直接寫進文章）：
  - 「我以為 Cloudflare Tunnel + 沒做 port forwarding 就等於對外鎖死了」→ **錯**。中華電信給每戶 `/64` 全球可路由 IPv6，每台設備都拿到公網 IP，沒有 NAT 這層遮蔽——只要 router IPv6 firewall 沒擋，你 Samba/SSH 全部對全世界開
  - 「我怎麼知道 router 真的有擋 IPv6？」→ 從手機 4G 連自己的公網 IPv6 試試。我從 Mac 開熱點 `ssh -v gakes@2001:b011:...` 看到 `Operation timed out` 那刻才真的放下心。**設定看半天不如實測一次**
  - 「我以為 UFW 設好就安全」→ Docker 直接操作 iptables 繞過 UFW。Docker container 對外開的 port 跟 UFW 完全平行——你必須在 compose 裡把 ports 改成 `127.0.0.1:port` binding 才真的擋
- 三層防禦的故事：
  - **邊緣（router 層）** — IPv6 防火牆（很多人沒驗證過、預設可能沒開）
  - **Host（NAS 自己）** — `chmod 600` 收回 `.env`、Docker compose 改 `127.0.0.1` binding、Samba `bind interfaces only` 只掛 LAN + localhost
  - **App** — hosts allow 白名單兜底（前兩層都破還能擋）
- 沒解的小謎（值得寫進文章當意外故事）：Samba 4.19 對 POINTOPOINT 介面（Tailscale）silently skip——`testparm` 顯示設定正確、log 沒任何錯誤、但 `ss -tln` 就是 bind 不到 `100.104.47.27`。最後在「LAN-only 但 Tailscale 沒 SMB」跟「bind 0.0.0.0 + hosts allow 兜底」之間做 trade-off
- 跟 AI 怎麼說 →：「請從**外面（公網）**的視角盤點我這台 NAS 有哪些 port 對外可達、用什麼程式在 listen、各服務的 `.env` 權限對不對。按嚴重度排序給我一份清單，並對每一項提具體修法。」
- 補充：
  - **縱深防禦不是「重複」**——是「假設前一層會失守」的心理準備；router 哪天 reset、host 哪天裝新服務暴露 port、container 哪天有 RCE 漏洞，下一層永遠還在
  - 接續 #2「信任 = 可撤回的權限清單」——這篇是「**防禦 = 可以分層的清單**」，同個思維的另一面
  - 「我怎麼跟 AI 一起做 audit」這個過程本身值得記下來，是 2026 之後的小白標配能力——不需要變專家，但要會請 AI 站在攻擊者位置看自己

---

## 入門層（小白能看懂，不需要動手）

### 2026 還敢組電腦：14400F + DDR4 32G 的選擇邏輯
- 怎麼扒：硬體選擇的當下考量，AI/記憶體漲價背景，為什麼不選 DDR5
- 補充：14400F 無核顯、NAS 不需要顯卡的邏輯

### NAS 的作業系統：為什麼我選 Ubuntu Server 不選 TrueNAS / Unraid
- 怎麼扒：自己熟悉 Linux、全用 Docker 所以不需要 NAS OS 的特殊功能
- 比較：TrueNAS 的優缺點、Unraid 授權費問題

### 在 NAS 上跑 VM：不需要顯卡的伺服器虛擬化
- 怎麼扒：KVM 設定記憶、安裝 Windows 11 過程、VM 用途（跑 Windows 專屬軟體）
- 補充：VM 通常不需要獨立顯卡，遠端 RDP 就夠

---

## 設定層（中白，需要動手）

### NAS 前置設定：BIOS Wake on LAN 與復電自動重啟
- 怎麼扒：BIOS 設定截圖或步驟記憶、B250M-K 的設定位置
- 補充：WoL 指令、iptables DNAT、Tailscale 跨網段喚醒

### UPS：市電中斷自動切換，電池快沒時指令關機
- 怎麼扒：NUT（Network UPS Tools）設定、關機觸發條件
- 補充：保護硬碟比保護服務更重要，突然斷電的壞處

### 三顆硬碟怎麼掛上去：分割、格式化、fstab 掛載
- 怎麼扒：當初的 lsblk、mkfs、fstab 設定
- 補充：UUID 掛載才不會因順序變動出錯、spindown 設定（hdparm）

### 請 AI 協作設定 NAS 環境：從零到所有服務跑起來
- 怎麼扒：和 AI 對話的過程、提問框架（接續第一篇的 AI 協作心得）
- 補充：Docker Compose 分資料夾管理、Cloudflare Tunnel 設定

---

## 網路層

### Tailscale 入門：比 VPN 簡單的遠端連線
- 怎麼扒：安裝步驟、手機/Mac/NAS 各端設定、MagicDNS
- 補充：不需要開放 port、不需要固定 IP

### Tailscale vs Cloudflare Tunnel：各自適合什麼
- 怎麼扒：兩個都在用、分工邏輯（內網用 Tailscale、對外服務用 CF Tunnel）
- 補充：Cloudflare 免費方案限制、Tailscale 免費方案設備數限制

---

## 服務介紹層

### NAS Dashboard：用儀表板控制 PC 與 VM
- 怎麼扒：nas-dashboard 的功能、WoL 原理、VM 開關機 API
- 補充：Supabase Auth 白名單、admin/viewer 角色設計

### Immich：NAS 上的家庭相簿，Google Photos 的替代方案
- 怎麼扒：安裝過程、external library 設定、自動掃描排程
- 補充：PostgreSQL 放 SSD 的原因（接第一篇）、人臉辨識功能

### code-server：在 NAS 上跑 VS Code，隨時遠端開發
- 怎麼扒：systemd user 服務設定、Cloudflare Tunnel 綁定
- 補充：為什麼不用 Docker（需要 home 目錄存取）

### LINE Bot：用 LINE 跟 NAS 說話
- 怎麼扒：linebot 現有功能、LINE Messaging API、webhook 架構
- 補充：reply vs push 額度差異、未來想做的功能（提醒、yt-dlp 觸發）

### Samba：讓全家人都能用 NAS 的共用磁碟
- 怎麼扒：Mac/iPhone/iPad 連線方式、photos 資料夾作為 Immich 照片入口
- 補充：只開放必要子目錄的安全設計

### 壞機不怕：從 SOP 到 5 小時內重建完整環境
- 怎麼扒：INFRA.md 的重建步驟、.env 從備份取回、哪些東西真的會消失
- 補充：文件即基礎設施的概念、為什麼 INFRA.md 比腦袋可靠

### Supabase Auth：不用自己寫 auth，一套認證管所有服務
- 怎麼扒：user_roles 白名單設計、token 快取、keepalive 防休眠
- 補充：Supabase 免費方案的限制

---

## 小工具（有空再寫）

### rmbg：在家裡跑去背 AI 模型
- 怎麼扒：Python FastAPI、Docker 打包、模型是哪一個

### yt-dlp 下載器：自架影片下載服務
- 怎麼扒：設定方式、存到 HDD 路徑

---

## TODO（系列維運）

### Cross-link pass
等 #3〈NAS 憲法〉、#4〈Ubuntu Server〉、#5〈Tailscale〉至少幾篇上線後，回頭把各篇之間的 internal link 一次補齊：

- **#2〈SSH Key〉** Q2 結尾「順手把密碼登入關掉」forward ref → 同篇段落 anchor
- **#2〈SSH Key〉** 段 6「下一篇〈NAS 憲法〉」→ #3 文章
- **#3〈NAS 憲法〉** 提到 SSH key 的部分 → 回扣 #2
- **#4〈Ubuntu Server〉** 提到 Tailscale 的部分 → #5
- **#5〈Tailscale〉** 段 1 鉤子提到 #2 SSH 從不對外 → 回扣 #2
- 任何文章提到 `Docker Compose 未來會有專文介紹` → 等 Docker 篇寫完補 link
- 任何文章提到 INFRA.md → 統一指向 #3〈NAS 憲法〉

集中做的好處：風格一致、密度可控、避免零散加 link 造成不對齊。
