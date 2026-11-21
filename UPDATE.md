# 居家與虛擬化網路升級完整規劃（UniFi + Netgear + ESXi 架構）

## 一、 Wi-Fi SSID 命名規範與選定方案

1. **命名設計原則**
   - **格式規範**：純小寫英文單一單詞（a-z），無空格、無底線、無特殊符號，確保 100% 設備（含老舊 IoT 家電）相容性與最高輸入便利性。
   - **頻段分離策略**：2.4GHz 與 5GHz 採用完全不同的 SSID，避免智慧家電配對失敗，並讓手機與電腦穩定鎖定高速 5GHz 頻段。
   - **安全性**：不暴露真實姓名、門牌或路由器品牌型號，且不依賴隱藏 SSID，統一使用 WPA2/WPA3 加密。

2. **推薦 SSID 配對清單（主題：科技、夢想、財富、AI）**

| 頻段                     | 推薦 SSID（任選一組搭配）              | 意涵與用途                                                 |
| :----------------------- | :------------------------------------- | :--------------------------------------------------------- |
| **5GHz（主力高速網）**   | `matrix` / `nova` / `oracle` / `nexus` | 象徵 AI 算力、新星與連結中樞，供手機、筆電等高速設備連線。 |
| **2.4GHz（智慧家電網）** | `vault` / `aurum` / `mint` / `utopia`  | 象徵寶庫、黃金與理想國度，供 VIZO 閘道與各類智慧家電連線。 |

---

## 二、 硬體拓撲與節點角色

- **核心路由與防火牆**：Ubiquiti Dream Machine Pro（UDM Pro），負責對外 WAN 撥接、全部網段 DHCP 派發、VLAN 路由與 WireGuard VPN 伺服器。
- **10G 核心交換器**：Ubiquiti Switch Pro XG 8 PoE，作為全屋網路中樞（10G SFP+ 上聯 UDM Pro），直接連接並供電（PoE++）給 U7 Pro XGS 無線基地台與客廳 USW-Flex-2.5G-5，並直連 Work MAC 與 VIZO Gateway。
- **客廳影音邊緣交換器**：**Ubiquiti UniFi Flex 2.5G 5（USW-Flex-2.5G-5）**（純有線配置），透過 2.5GbE 單線複用與 PoE 受電，同時承載一般網路流量（Sony PS5、Apple TV、Sony TV）與中華電信 MOD 訊號，並納入 UniFi 單一介面統一納管。
- **伺服器虛擬化交換器**：Netgear GS510TLP，連接 ESXi 主機，承載內部多台虛擬機（Center / Trader / Blog）。

---

## 三、 網段與 VLAN 規劃總表（172.28.x.0/24）

採用 `172.28.[VLAN].0/24` 網段，徹底避開企業常用的 `10.x.x.x`、家用預設的 `192.168.x.x`，以及手機熱點預設網段（iPhone 熱點 `172.20.10.x`、Android 熱點 `192.168.43.x`），確保在外連回家用 VPN 時零路由衝突。

| VLAN ID         | 網段名稱      | IP 範圍（/24）                   | 閘道 IP       | 歸屬設備與連線方式                                                                        | 防火牆與隔離策略                                               |
| :-------------- | :------------ | :------------------------------- | :------------ | :---------------------------------------------------------------------------------------- | :------------------------------------------------------------- |
| **1 (Default)** | Management    | `172.28.1.0/24`                  | `172.28.1.1`  | UDM Pro、Pro XG 8 PoE、U7 Pro XGS、**USW-Flex-2.5G-5**、GS510TLP（`.251`）、ESXi（`.10`） | 最高管理網段，僅允許 Trusted 與 VPN 連入管理。                 |
| **20**          | Trusted       | `172.28.20.0/24`                 | `172.28.20.1` | 有線：Work MAC、Apple TV、Sony PS5、Sony TV；無線：手機、筆電（綁定 5GHz SSID）           | 主力信任網段，可主動存取所有其他內網網段。                     |
| **30**          | IoT           | `172.28.30.0/24`                 | `172.28.30.1` | 有線：VIZO Gateway；無線：所有智慧家電（綁定 2.4GHz SSID）                                | **完全隔離**：禁止主動連線至其他內網（僅允許回應已建立連線）。 |
| **40**          | VMLAN         | `172.28.40.0/24`                 | `172.28.40.1` | ESXi 內部的 VM(s)（Center / Trader / Blog）                                               | 依服務需求僅開放特定 Port 給 Trusted 與 VPN。                  |
| **50**          | MOD_Bypass    | 不配發 IP（Third-Party Gateway） | 無            | 中華電信 MOD 訊號單線透傳（純 Layer 2 通道）                                              | 完全獨立於 IP 路由之外，避免 IGMP 群播干擾內網。               |
| **90 (VPN)**    | WireGuard VPN | `172.28.90.0/24`                 | `172.28.90.1` | 外部手機透過熱點或公網連回家的 VPN 客戶端                                                 | 連入後具備與 Trusted 同等權限，可存取家中所有資源。            |

---

## 四、 客廳影音區純有線配置（USW-Flex-2.5G-5 ↔ Pro XG 8 PoE）

客廳區全部採純有線連接（無 Wi-Fi 介入）。**USW-Flex-2.5G-5 的 Port 5（PoE In 上聯埠）** 連接至 Pro XG 8 PoE 的 Port 8，建立 **2.5Gbps Trunk 通道並直接透過網路線取電**（免插變壓器）：

| USW-Flex-2.5G-5 埠號 | 連接設備                   | 歸屬網段 / VLAN          | UniFi Port Manager 設定（Native / Tagged）                          |
| :------------------- | :------------------------- | :----------------------- | :------------------------------------------------------------------ |
| **Port 5（PoE In）** | 上聯 Pro XG 8 PoE (Port 8) | Management + Trunk       | **Native VLAN 1** + **Allow All Tagged**（2.5Gbps 連線 + PoE 供電） |
| **Port 1**           | Sony TV（純有線）          | Trusted（`172.28.20.x`） | **Native VLAN 20**（Block All Tagged）                              |
| **Port 2**           | Apple TV（純有線）         | Trusted（`172.28.20.x`） | **Native VLAN 20**（Block All Tagged）                              |
| **Port 3**           | Sony PS5（純有線）         | Trusted（`172.28.20.x`） | **Native VLAN 20**（Block All Tagged）                              |
| **Port 4**           | 中華電信 MOD（純有線）     | MOD_Bypass（VLAN 50）    | **Native VLAN 50**（Block All Tagged）                              |

> **實務優勢備註**：USW-Flex-2.5G-5 的預設 PoE 輸入埠即為 **Port 5**，務必將來自 Pro XG 8 PoE 的網路線接在 Port 5，才能同時啟用 PoE 受電與全速 2.5G 上聯。

---

## 五、 全區埠口與服務設定備忘

1. **UniFi Pro XG 8 PoE 核心埠口設定**：
   - **接小烏龜 MOD 孔**：Native VLAN 50（Block All Tagged）。
   - **Port 8（接 USW-Flex-2.5G-5 Port 5）**：**Native VLAN 1 + Allow All Tagged**（並確認該埠 PoE 輸出維持開啟，供電給客廳交換器）。
   - **接 GS510TLP 埠口**：Native VLAN 1 + Allow All Tagged。
   - **接 U7 Pro XGS 埠口**：Native VLAN 1 + Allow All Tagged（同時廣播 VLAN 20 與 VLAN 30 的無線網路）。
   - **接 Work MAC 埠口**：Native VLAN 20（Block All Tagged）。
   - **接 VIZO Gateway 埠口**：Native VLAN 30（Block All Tagged）。

2. **伺服器區對接（Netgear GS510TLP ↔ ESXi）**：
   - **ESXi vSwitch**：Management Network 維持 VLAN 0（對應 VLAN 1）；新增 VMLAN Port Group 並指定 VLAN ID 40 供虛擬機使用。

3. **VPN 分流設定（WireGuard Client）**：
   - 若僅需連回家中內網設備，`AllowedIPs` 設為 `172.28.0.0/16`；若需全部上網流量走家中固網出口，則設為 `0.0.0.0/0`。
