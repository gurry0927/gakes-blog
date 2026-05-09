# #5 Tailscale 入門：AI 不走公網也能進來

> 草稿狀態：大綱（段落結構 + 素材對照）
> 預定 slug：`tailscale-intro` → `content/posts/tailscale-intro/index.md`
> 接續：#4〈Ubuntu Server〉結尾「選對 OS 還只是地基；AI 跟我要怎麼走進這個地基」
> 也回扣 #2 SSH Key 結尾「外面沒這扇門」的伏筆

## Front matter（草擬）

```toml
title = '【小白篇】Tailscale 入門：AI 不走公網也能進來我家'
description = '一條只有自家設備走得進的私人巷弄——比 VPN 簡單，比 port forwarding 安全'
tags = ['NAS', 'homelab', 'Tailscale', 'VPN', 'AI協作', '小白篇', '網路']
mermaid = true
```

## 鉤子（開場第一段就要砸出來）

「上一篇講選 Ubuntu Server 是『給 AI 一個它最熟的家』。但這個家蓋好了，AI 跟我自己**從外面要怎麼走進來**？

#2 寫 SSH Key 的時候我說過一句：『SSH 從不對外開放』。可是不對外開，那 ClaudeCode 在我筆電上怎麼有辦法 ssh 到 NAS？我手機在外面怎麼用 Samba 看家裡的照片？

答案是 Tailscale——一條只有自家設備走得進的**私人巷弄**。」

→ 這個鉤子一次連回 #2、#4，把『AI 怎麼進來』這條線收緊。

---

## 段落結構（暫定七段）

### 段 1｜開場：地基蓋好了，路怎麼走？
- 連回 #4「選對地基」、#2「SSH 從不對外」
- 拋問題：那 AI 跟我自己怎麼進這台機器？
- 預告答案：Tailscale

### 段 2｜在 Tailscale 之前，傳統有兩條路（都不好走）

**路 A：把 NAS 對全世界開門（port forwarding）**
- 這是最直覺的做法：在路由器上設一條規則「外面找我家對 22 port 的請求，全部丟給 NAS」
- 你的 NAS SSH 就在公網上裸奔了
- 全世界一秒幾百次的自動掃描腳本馬上就會找上你
- 還有個額外問題：家用網路 IP 是動態的，今天連得到、明天可能就找不到（要花錢買固定 IP 或用 DDNS）

**路 B：自己架 VPN（OpenVPN 或 WireGuard）**
- 你架一個 VPN server 在 NAS 上，每個裝置裝 client
- 連線後就像「人在家裡」一樣
- 安全性比 A 高很多，但⋯⋯**你要自己處理一堆事**：
  - 在路由器開 port（又繞回 A 的問題）
  - 生憑證、發給每個裝置
  - 處理 NAT 穿透（家用網路通常在運營商的大 NAT 後面）
  - 設防火牆規則
- 對工程師可行；對小白幾乎不可能搞定

→ 結論：我們需要一個「VPN 該有的安全 + 不需要動腦的設定」。Tailscale 就是這個東西。

### 段 3｜Tailscale 是什麼：私人巷弄

用比喻起手：

- **公網**像大馬路。所有電腦都站在路邊，IP 就是門牌。誰都看得到、誰都能來敲門。對外開服務就是把家門開到馬路上
- **Tailscale** 像在現實之上多畫了一條**只有自家設備能進的小巷**
- 你每台設備（NAS、Mac、iPhone⋯⋯）裝上 Tailscale、用同一個 Google 帳號登入，就**自動加入這條小巷**
- 在這條小巷裡，每台設備拿到一個 `100.x.x.x` 開頭的「巷弄門牌」（外面的世界看不到也走不進）
- 巷弄裡的設備**直接看得到彼此**，像在同一個區域網路

具體效果：
- 我在咖啡廳用 Mac、`ssh gakes@100.104.47.27` 直接連到家裡 NAS——完全沒碰路由器、沒開 port、沒申請固定 IP
- 我在外面 iPhone 開 Samba 連 NAS 共享，速度跟在家一樣
- 家裡網路的 IP 換了？沒關係，巷弄門牌不會變

### 段 4｜它是怎麼做到的（小白版，不深入技術）

簡短的「黑盒拆開來看一眼」：

- **底層協定是 WireGuard**——一個 2018 年才進入 Linux 核心的現代 VPN 技術，加密強、速度快、程式碼少
- **WireGuard 自己很硬**——要手動發金鑰、設規則。Tailscale 把這些**全自動**了：
  - 設備之間的金鑰交換 → Tailscale 後台幫你處理
  - 找彼此的 IP → 它的伺服器當「介紹所」
  - NAT 穿透（兩台都在私網後面） → 它的演算法搞定大部分情境
  - 真的穿不過去 → 走它的中繼伺服器（DERP），慢一點但能通
- **你只負責**：在每台設備上裝一次、用 Google 登入。做完就好了

> 順帶解釋兩個你會看到的詞：
> - **WireGuard**：可以想成「現代版 VPN 的引擎」（就像 KVM 是 VM 的引擎）
> - **Tailscale**：把這個引擎包成「不用懂的產品」的服務

### 段 5｜你已經在用：NAS 上的 Tailscale 故事

把實際情況攤出來：

> **`tailscale status` 我家現在這樣**：
>
> | 設備 | Tailscale IP | 角色 |
> |---|---|---|
> | labserver | 100.104.47.27 | NAS 本人 |
> | macbook-air | 100.120.55.119 | 我的 Mac |
> | iphone-11 | 100.119.155.21 | 我的手機 |
> | ipad-10th-gen | 100.78.89.41 | iPad |
> | ipad-mini-6th | 100.80.104.77 | 平常翻書用的 iPad mini |
> | desktop（GakePC） | 100.124.183.38 | 桌機 |

實際應用：
- 我從 Mac/iPad/手機任何裝置，都用 `100.104.47.27` 連 NAS
- 從家裡 Wi-Fi 內也可以用區網 IP `192.168.1.182` 直連（更快）
- 但離開家後，**只有 Tailscale IP 連得到**——這條巷弄就是我的遠端通道
- 第二把 SSH key（`ClaudeCode@GakePC`）走的就是這條路：我的 ClaudeCode 從筆電 ssh 到 NAS、所有命令都走 Tailscale 私網

→ 這段直接呼應 #2「給 AI 一把鑰匙」——鑰匙是 SSH key，但門在哪？答案：在 Tailscale 巷弄裡。

### 段 6｜Tailscale vs Cloudflare Tunnel：兩條路服務不同的人

這部分對小白特別容易混淆，要講清楚：

我 NAS 上**同時用兩個工具**——它們不是替代關係，是分工。

| | Tailscale（巷弄） | Cloudflare Tunnel（街道） |
|---|---|---|
| 給誰用 | 自己人（自家設備、AI 的筆電） | 全世界（任何瀏覽器） |
| 連得到嗎 | 要先連 Tailscale | 隨便瀏覽器打網址就能 |
| 用來做什麼 | SSH、Samba、看內部 log | 對外網頁服務（Immich、LINE Bot 等） |
| 你看見的位址 | `100.x.x.x` | `xxx.gakes.net` |

舉例：
- 你想在外面遠端管 NAS → 走 Tailscale（要登入 Tailscale 才走得進）
- 你的 LINE Bot 要收 LINE 平台 webhook → Cloudflare Tunnel（LINE 是「全世界」之一，它沒有 Tailscale 帳號）
- 你太太要看 Immich 照片 → Cloudflare Tunnel（不需要她裝 Tailscale）
- 你自己要看 Immich → 兩個都行，但 Cloudflare Tunnel 比較方便

→ 一句話總結：**Tailscale 是讓「我自己」進來的路，Cloudflare Tunnel 是讓「全世界」進來的路**。

→ 這段也是這個系列的一個重要 mental model：**遠端存取不是非黑即白**，是分對象決定走哪條。

### 段 7｜結尾：私人巷弄的哲學 + 三部曲收場

從技術回到主題：

- **公網是馬路，巷弄是信任的延伸**
- 馬路上你可以擺攤（對外服務），但生活還是發生在巷弄裡（管理、自己人）
- Tailscale 不是讓你「躲起來」，是讓你**清楚分開**「給世界看的」跟「給自己用的」

**串起這個小系列的四篇**（#2 → #5）：

> #2 給 AI 一把鑰匙進來
> #3 給 AI 一份家規（INFRA.md）
> #4 給 AI 一個它最熟的家（Ubuntu Server）
> #5 給 AI 一條只有自家人能走的路（Tailscale）

從 #1 的硬體開始一路到這裡，整套 NAS 不是「裝了一堆軟體」，是一個**有設計的小生態**——對自己負責、對 AI 開放、對世界謹慎。

---

## 實際素材清單（對照環境探勘）

| 段落 | 素材 |
|---|---|
| 段 5 | 真實 `tailscale status` 7 台設備清單（建議只露暱稱跟 100.x IP，不露 user email） |
| 段 5 | INFRA.md 內網存取那段（labserver IP 100.104.47.27） |
| 段 6 | 對比表的左欄都是 Tailscale 用法（SSH/Samba），右欄是 Cloudflare Tunnel 5 個 hostname（→ 跟 INFRA.md 對齊） |

## 視覺素材建議

**封面**：抽象一點。「巷弄」「私人通道」「同心圓」的視覺。

**內文 mermaid 候選**（段 6 的「兩條路」視覺化）：

```mermaid
graph TB
    WORLD[🌍 全世界<br/>瀏覽器、LINE Webhook、家人]
    ME[👤 我自己<br/>Mac、iPad、iPhone]
    AI[🤖 ClaudeCode<br/>跑在筆電上]

    WORLD ==>|Cloudflare Tunnel<br/>xxx.gakes.net| FE[對外服務<br/>nas-dashboard / Immich / LINE Bot / rmbg]
    ME -.Tailscale 私人巷弄<br/>100.104.47.27.-> CORE[NAS 內部<br/>SSH / Samba / log]
    AI -.同一條巷弄.-> CORE

    FE -.同一台機器.-> CORE
```

> 這張圖把「兩條路+各自服務的人」一次說清楚。

---

## 給你的概念補課（也會同步更新到 _concepts.md）

我會把這篇文章用到的兩個你不太熟的詞補進 [drafts/_concepts.md](_concepts.md)：

- **WireGuard**：Tailscale 底下的 VPN 引擎，已在段 4 簡略講
- **VPN 是什麼**：簡略講「在公網上開一條加密隧道」的概念，配合段 2 路 B
- **NAT 穿透 / port forwarding**：段 2 提到，會專門開一段給你

寫完這篇 prose 之前我會去把這幾個補進去。

---

## 不寫進這篇的（避免失焦）

- ❌ Tailscale 註冊／安裝步驟（這是 tutorial 文，不適合小白思考類）
- ❌ Tailscale ACL / Funnel / Subnet routes 進階功能（單獨一篇進階篇可以）
- ❌ Cloudflare Tunnel 怎麼設定（→ 未來 Cloudflare Tunnel 專文）
- ❌ DNAT / iptables 細節（→ 安全加固篇）
- ❌ 所有設備離線只剩 NAS 怎麼辦（邊緣情境，會破壞行文節奏）

---

## 已定案（mentor 模式：我直接決定）

1. ✅ 標題：「【小白篇】Tailscale 入門：AI 不走公網也能進來我家」
2. ✅ 段 5 公開 7 台設備清單但隱去 email（用「我的 Mac」「我的手機」這類暱稱對應 100.x IP）
3. ✅ 段 4 不深入技術（WireGuard 內部、NAT 穿透演算法、DERP 細節都不展開）
4. ✅ 段 6 完整把分工表寫出來——這是這個系列幫小白建立 mental model 最有價值的一段
5. ✅ 段 7 用「四篇小系列收場」做結，不預告下一篇（讓系列有自然的段落感；之後要寫安全加固或 Docker 篇都接得住）

---

## 與 #4 / 整體系列的串連檢查

**從 #4 接過來**：#4 結尾「選對 OS 還只是地基；AI 跟我要怎麼走進這個地基」→ #5 開場直接答「走 Tailscale」。順。

**回扣 #2**：#2 講「給 AI 鑰匙」沒講路；#5 講「路」。讀者會有「對！#2 那時就在埋伏筆」的感覺。

**回扣 #1**：#1 的「跟 AI 協作」在這四篇被具體化成「鑰匙 + 家規 + 家 + 路」整套。系列收尾自然。

**留給後續文章的鉤子**（不在 #5 寫）：
- UFW / fail2ban：「巷弄安全」可以再深一層
- Cloudflare Tunnel 專文：「街道」那條路的細節
- Docker：第一篇開場那個「公司 VM 踩坑」一直沒被回答的那篇
