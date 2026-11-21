# UniFi Network 11.0.99 API 統整

來源為這台主控台文件頁取得的 [integration.json](integration.json)。檔案宣告 UniFi Network API `11.0.99`、OpenAPI `3.1.0`。本文件涵蓋 **44 條路徑、73 個操作及 13 個操作分類**，另整理入門、篩選與錯誤處理三個說明章節。

## 入門（Getting Started）

- 規格宣告相對伺服器基底 `/integration`。索引中的 `/v1/...` 路徑接在基底之後。主控台的完整請求網址與標頭請以 UniFi Network 的 **Integrations** 範例為準。
- 在 **Integrations** 建立用於驗證的 API 金鑰。這份規格未宣告 `securitySchemes` 或金鑰標頭名稱。
- `GET /v1/info` 取得應用程式版本。`GET /v1/sites` 回傳的 `data[].id` 是站點 UUID，供 `{siteId}` 使用。站點下的裝置、用戶端和其他資源使用各自的識別碼。
- 下方查詢參數欄以「必填」標示必要參數。路徑大括號內的值依操作定義填入。JSON 本文與回應欄的名稱對應 `components.schemas` 內的資料模型。

## 篩選（Filtering）

有 `offset` 和 `limit` 的操作可分頁。以站點與裝置清單為例，`offset` 預設 `0`，`limit` 預設 `25`、上限 `200`。熱點票券清單的 `limit` 預設 `100`、上限 `1000`。分頁回應中的 `data`、`count`、`totalCount`、`offset` 和 `limit` 可用來取得下一頁。

有 `filter` 的操作支援 `<property>.<function>(<arguments>)`，可以用 `and(...)`、`or(...)`、`not(...)` 組合。字串用單引號，若要在字串內表示單引號，連寫兩個單引號。欄位與支援的函式依各端點描述為準。

- 已採用裝置清單（`GET /v1/sites/{siteId}/devices`）可用 `name.eq('Office')` 依名稱比對。
- 同一清單可用 `and(firmwareVersion.isNotNull(),not(name.like('Guest*')))` 篩選有韌體版本、名稱不以 `Guest` 開頭的裝置。
- `like` 中 `.` 比對一個字元，`*` 比對任意數量字元。請視需要將 URL 查詢值進行百分比編碼。

## 錯誤處理（Error Handling）

通用 `Error Message` 模型包含 `statusCode`、`statusName`、`code`、`message`、`timestamp`、`requestPath`、`requestId`。發生 HTTP `500` 時，可用 `requestId` 在伺服器記錄中追查錯誤。「成功回應」欄依各操作的 `responses` 列出 `200` 或 `201` 所對應的回應模型。

## 完整操作索引

### 應用程式資訊（Application Info，1 項）

| 方法與相對路徑 | `operationId` | 查詢參數與本文 | 成功回應 | 用途 |
| --- | --- | --- | --- | --- |
| `GET /v1/info` | `getInfo` | — | `200` JSON `Application info` | 取得 Network 應用程式版本（`applicationVersion`）。 |

### 站點（Sites，1 項）

| 方法與相對路徑 | `operationId` | 查詢參數與本文 | 成功回應 | 用途 |
| --- | --- | --- | --- | --- |
| `GET /v1/sites` | `getSiteOverviewPage` | `offset`、`limit`、`filter` | `200` JSON `Site overview page` | 查詢應用程式管理的本機站點。使用清單項目的 `id` 作為 `siteId`。 |

### UniFi 裝置（UniFi Devices，8 項）

| 方法與相對路徑 | `operationId` | 查詢參數與本文 | 成功回應 | 用途 |
| --- | --- | --- | --- | --- |
| `GET /v1/pending-devices` | `getPendingDevicePage` | `offset`、`limit`、`filter` | `200` JSON `Device pending adoption page` | 分頁查詢等待採用的裝置與基本資訊。 |
| `GET /v1/sites/{siteId}/devices` | `getAdoptedDeviceOverviewPage` | `offset`、`limit`、`filter` | `200` JSON `Adopted device overview page` | 分頁查詢站點已採用的裝置。使用清單項目的 `id` 作為 `deviceId`。 |
| `POST /v1/sites/{siteId}/devices` | `adoptDevice` | JSON 本文（必填） `IntegrationDeviceAdoptionRequestDto` | `200` JSON `Adopted device details` | 依 `macAddress` 將裝置採用到指定站點。本文須提供 `ignoreDeviceLimit`。 |
| `GET /v1/sites/{siteId}/devices/{deviceId}` | `getAdoptedDeviceDetails` | — | `200` JSON `Adopted device details` | 查詢單台裝置的韌體、上行連線、功能、連接埠與無線電等資訊。 |
| `GET /v1/sites/{siteId}/devices/{deviceId}/statistics/latest` | `getAdoptedDeviceLatestStatistics` | — | `200` JSON `Latest statistics for a device` | 讀取單台裝置的最新統計，例如運作時間、傳輸速率、CPU 與記憶體使用率。見[統計端點詳解](#統計端點詳解)。 |
| `POST /v1/sites/{siteId}/devices/{deviceId}/actions` | `executeAdoptedDeviceAction` | JSON 本文（必填） `Device action request` | `200`，規格未定義本文 | 對指定裝置執行 `RESTART`。 |
| `POST /v1/sites/{siteId}/devices/{deviceId}/interfaces/ports/{portIdx}/actions` | `executePortAction` | JSON 本文（必填） `Port action request` | `200`，規格未定義本文 | 對指定連接埠執行 PoE 電源重啟（`POWER_CYCLE`）。 |
| `DELETE /v1/sites/{siteId}/devices/{deviceId}` | `removeDevice` | — | `200`，規格未定義本文 | 取消採用並從站點移除裝置。裝置若處於連線狀態，會還原為原廠設定。 |

### 連線用戶端（Clients，3 項）

| 方法與相對路徑 | `operationId` | 查詢參數與本文 | 成功回應 | 用途 |
| --- | --- | --- | --- | --- |
| `GET /v1/sites/{siteId}/clients` | `getConnectedClientOverviewPage` | `offset`、`limit`、`filter` | `200` JSON `Client overview page` | 分頁列出站點上的有線、無線與 VPN 等已連線用戶端。 |
| `GET /v1/sites/{siteId}/clients/{clientId}` | `getConnectedClientDetails` | — | `200` JSON `Client details` | 取得單一已連線用戶端的名稱、連線類型、可用的 IP 位址與存取資訊。有線與無線用戶端的模型另有 `macAddress`。 |
| `POST /v1/sites/{siteId}/clients/{clientId}/actions` | `executeConnectedClientAction` | JSON 本文（必填） `Client action request` | `200` JSON `Client action response` | 以 `AUTHORIZE_GUEST_ACCESS` 授權訪客。授權會取代既有的有效授權並重設訪客流量計數。`UNAUTHORIZE_GUEST_ACCESS` 撤銷授權並中斷訪客連線。 |

### 網路（Networks，6 項）

| 方法與相對路徑 | `operationId` | 查詢參數與本文 | 成功回應 | 用途 |
| --- | --- | --- | --- | --- |
| `GET /v1/sites/{siteId}/networks` | `getNetworksOverviewPage` | `offset`、`limit`、`filter` | `200` JSON `Network overview page` | 分頁取得站點的網路清單。 |
| `POST /v1/sites/{siteId}/networks` | `createNetwork` | JSON 本文（必填） `Create or update Network` | `201` JSON `Network details` | 在指定站點新增網路。 |
| `GET /v1/sites/{siteId}/networks/{networkId}` | `getNetworkDetails` | — | `200` JSON `Network details` | 查詢指定網路的詳細設定。 |
| `PUT /v1/sites/{siteId}/networks/{networkId}` | `updateNetwork` | JSON 本文（必填） `Create or update Network` | `200` JSON `Network details` | 更新指定網路的設定。 |
| `DELETE /v1/sites/{siteId}/networks/{networkId}` | `deleteNetwork` | `force` | `200`，規格未定義本文 | 刪除指定網路，提供選用的 `force` 參數。 |
| `GET /v1/sites/{siteId}/networks/{networkId}/references` | `getNetworkReferences` | — | `200` JSON `Network references` | 查詢哪些資源參照指定網路。 |

### Wi-Fi 廣播（WiFi Broadcasts，5 項）

| 方法與相對路徑 | `operationId` | 查詢參數與本文 | 成功回應 | 用途 |
| --- | --- | --- | --- | --- |
| `GET /v1/sites/{siteId}/wifi/broadcasts` | `getWifiBroadcastPage` | `offset`、`limit`、`filter` | `200` JSON `IntegrationWifiBroadcastPageDto` | 分頁查詢站點的 Wi-Fi 廣播清單。 |
| `POST /v1/sites/{siteId}/wifi/broadcasts` | `createWifiBroadcast` | JSON 本文（必填） `Wifi broadcast create or update` | `201` JSON `Wifi broadcast details` | 在站點新增 Wi-Fi 廣播設定。 |
| `GET /v1/sites/{siteId}/wifi/broadcasts/{wifiBroadcastId}` | `getWifiBroadcastDetails` | — | `200` JSON `Wifi broadcast details` | 取得指定 Wi-Fi 廣播的詳細設定。 |
| `PUT /v1/sites/{siteId}/wifi/broadcasts/{wifiBroadcastId}` | `updateWifiBroadcast` | JSON 本文（必填） `Wifi broadcast create or update` | `200` JSON `Wifi broadcast details` | 更新指定 Wi-Fi 廣播的設定。 |
| `DELETE /v1/sites/{siteId}/wifi/broadcasts/{wifiBroadcastId}` | `deleteWifiBroadcast` | `force` | `200`，規格未定義本文 | 刪除指定 Wi-Fi 廣播，提供選用的 `force` 參數。 |

### 熱點（Hotspot，5 項）

| 方法與相對路徑 | `operationId` | 查詢參數與本文 | 成功回應 | 用途 |
| --- | --- | --- | --- | --- |
| `GET /v1/sites/{siteId}/hotspot/vouchers` | `getVouchers` | `offset`、`limit`、`filter` | `200` JSON `Hotspot voucher detail page` | 分頁查詢熱點的訪客票券。 |
| `POST /v1/sites/{siteId}/hotspot/vouchers` | `createVouchers` | JSON 本文（必填） `Hotspot voucher creation request` | `201` JSON `IntegrationVoucherCreationResultDto` | 為站點產生一張或多張熱點票券。 |
| `DELETE /v1/sites/{siteId}/hotspot/vouchers` | `deleteVouchers` | `filter`（必填） | `200` JSON `Voucher deletion results` | 依必填的 `filter` 條件批次刪除熱點票券，回應可包含刪除筆數。 |
| `GET /v1/sites/{siteId}/hotspot/vouchers/{voucherId}` | `getVoucher` | — | `200` JSON `Hotspot voucher details` | 取得單張熱點票券的詳細資料。 |
| `DELETE /v1/sites/{siteId}/hotspot/vouchers/{voucherId}` | `deleteVoucher` | — | `200` JSON `Voucher deletion results` | 刪除指定的單張熱點票券。 |

### 防火牆（Firewall，13 項）

| 方法與相對路徑 | `operationId` | 查詢參數與本文 | 成功回應 | 用途 |
| --- | --- | --- | --- | --- |
| `GET /v1/sites/{siteId}/firewall/policies` | `getFirewallPolicies` | `offset`、`limit`、`filter` | `200` JSON `Firewall policy page` | 查詢站點的防火牆政策清單。 |
| `POST /v1/sites/{siteId}/firewall/policies` | `createFirewallPolicy` | JSON 本文（必填） `Create or update firewall policy` | `201` JSON `Firewall policy` | 為站點建立防火牆政策。 |
| `GET /v1/sites/{siteId}/firewall/policies/ordering` | `getFirewallPolicyOrdering` | `sourceFirewallZoneId`（必填）、`destinationFirewallZoneId`（必填） | `200` JSON `IntegrationFirewallPolicyOrderingDto` | 取得指定來源與目的區域之間的自訂防火牆政策排序。 |
| `PUT /v1/sites/{siteId}/firewall/policies/ordering` | `updateFirewallPolicyOrdering` | `sourceFirewallZoneId`（必填）、`destinationFirewallZoneId`（必填）<br>JSON 本文（必填） `IntegrationFirewallPolicyOrderingDto` | `200` JSON `IntegrationFirewallPolicyOrderingDto` | 變更指定來源與目的區域之間的自訂政策排序。 |
| `GET /v1/sites/{siteId}/firewall/policies/{firewallPolicyId}` | `getFirewallPolicy` | — | `200` JSON `Firewall policy` | 查詢單一防火牆政策。 |
| `PUT /v1/sites/{siteId}/firewall/policies/{firewallPolicyId}` | `updateFirewallPolicy` | JSON 本文（必填） `Create or update firewall policy` | `200` JSON `Firewall policy` | 更新指定防火牆政策。 |
| `PATCH /v1/sites/{siteId}/firewall/policies/{firewallPolicyId}` | `patchFirewallPolicy` | JSON 本文（必填） `Patch firewall policy` | `200` JSON `Firewall policy` | 部分更新指定防火牆政策。 |
| `DELETE /v1/sites/{siteId}/firewall/policies/{firewallPolicyId}` | `deleteFirewallPolicy` | — | `200`，規格未定義本文 | 刪除指定防火牆政策。 |
| `GET /v1/sites/{siteId}/firewall/zones` | `getFirewallZones` | `offset`、`limit`、`filter` | `200` JSON `Firewall zones page` | 查詢站點的防火牆區域清單。 |
| `POST /v1/sites/{siteId}/firewall/zones` | `createFirewallZone` | JSON 本文（必填） `Create or update firewall zone` | `201` JSON `Firewall zone` | 建立自訂防火牆區域。 |
| `GET /v1/sites/{siteId}/firewall/zones/{firewallZoneId}` | `getFirewallZone` | — | `200` JSON `Firewall zone` | 取得指定的防火牆區域。 |
| `PUT /v1/sites/{siteId}/firewall/zones/{firewallZoneId}` | `updateFirewallZone` | JSON 本文（必填） `Create or update firewall zone` | `200` JSON `Firewall zone` | 更新指定防火牆區域。 |
| `DELETE /v1/sites/{siteId}/firewall/zones/{firewallZoneId}` | `deleteFirewallZone` | — | `200`，規格未定義本文 | 刪除指定的自訂防火牆區域。 |

### 存取控制（Access Control (ACL Rules)，7 項）

| 方法與相對路徑 | `operationId` | 查詢參數與本文 | 成功回應 | 用途 |
| --- | --- | --- | --- | --- |
| `GET /v1/sites/{siteId}/acl-rules` | `getAclRulePage` | `offset`、`limit`、`filter` | `200` JSON `IntegrationAclRulePageDto` | 分頁查詢站點的存取控制規則。 |
| `POST /v1/sites/{siteId}/acl-rules` | `createAclRule` | JSON 本文（必填） `ACL rule update` | `201` JSON `ACL rule` | 為站點建立自訂 ACL 規則。 |
| `GET /v1/sites/{siteId}/acl-rules/ordering` | `getAclRuleOrdering` | — | `200` JSON `ACL rule ordering` | 取得站點自訂 ACL 規則的排序。 |
| `PUT /v1/sites/{siteId}/acl-rules/ordering` | `updateAclRuleOrdering` | JSON 本文（必填） `ACL rule ordering` | `200` JSON `ACL rule ordering` | 變更站點自訂 ACL 規則的排序。 |
| `GET /v1/sites/{siteId}/acl-rules/{aclRuleId}` | `getAclRule` | — | `200` JSON `ACL rule` | 取得單一 ACL 規則。 |
| `PUT /v1/sites/{siteId}/acl-rules/{aclRuleId}` | `updateAclRule` | JSON 本文（必填） `ACL rule update` | `200` JSON `ACL rule` | 更新指定的自訂 ACL 規則。 |
| `DELETE /v1/sites/{siteId}/acl-rules/{aclRuleId}` | `deleteAclRule` | — | `200`，規格未定義本文 | 刪除指定的自訂 ACL 規則。 |

### 交換功能（Switching，6 項）

| 方法與相對路徑 | `operationId` | 查詢參數與本文 | 成功回應 | 用途 |
| --- | --- | --- | --- | --- |
| `GET /v1/sites/{siteId}/switching/lags` | `getLagPage` | `offset`、`limit`、`filter` | `200` JSON `IntegrationLagPageDto` | 分頁查詢站點的鏈路聚合群組（LAG）。 |
| `GET /v1/sites/{siteId}/switching/lags/{lagId}` | `getLag` | — | `200` JSON `LAG details` | 取得指定鏈路聚合群組的詳細資訊。 |
| `GET /v1/sites/{siteId}/switching/mc-lag-domains` | `getMcLagDomainPage` | `offset`、`limit`、`filter` | `200` JSON `IntegrationMcLagDomainDtoPageDto` | 分頁查詢站點的跨機箱鏈路聚合網域（MC-LAG）。 |
| `GET /v1/sites/{siteId}/switching/mc-lag-domains/{mcLagDomainId}` | `getMcLagDomain` | — | `200` JSON `IntegrationMcLagDomainDto` | 取得指定 MC-LAG 網域的詳細資訊。 |
| `GET /v1/sites/{siteId}/switching/switch-stacks` | `getSwitchStackPage` | `offset`、`limit`、`filter` | `200` JSON `IntegrationSwitchStackDtoPageDto` | 分頁查詢站點的交換器堆疊。 |
| `GET /v1/sites/{siteId}/switching/switch-stacks/{switchStackId}` | `getSwitchStack` | — | `200` JSON `IntegrationSwitchStackDto` | 取得指定交換器堆疊的詳細資訊。 |

### DNS 政策（DNS Policies，5 項）

| 方法與相對路徑 | `operationId` | 查詢參數與本文 | 成功回應 | 用途 |
| --- | --- | --- | --- | --- |
| `GET /v1/sites/{siteId}/dns/policies` | `getDnsPolicyPage` | `offset`、`limit`、`filter` | `200` JSON `IntegrationDnsPolicyPageDto` | 分頁查詢站點的 DNS 政策。 |
| `POST /v1/sites/{siteId}/dns/policies` | `createDnsPolicy` | JSON 本文（必填） `Create or update DNS policy` | `201` JSON `DNS policy` | 為站點建立 DNS 政策。 |
| `GET /v1/sites/{siteId}/dns/policies/{dnsPolicyId}` | `getDnsPolicy` | — | `200` JSON `DNS policy` | 取得指定 DNS 政策。 |
| `PUT /v1/sites/{siteId}/dns/policies/{dnsPolicyId}` | `updateDnsPolicy` | JSON 本文（必填） `Create or update DNS policy` | `200` JSON `DNS policy` | 更新指定 DNS 政策。 |
| `DELETE /v1/sites/{siteId}/dns/policies/{dnsPolicyId}` | `deleteDnsPolicy` | — | `200`，規格未定義本文 | 刪除指定 DNS 政策。 |

### 流量比對清單（Traffic Matching Lists，5 項）

| 方法與相對路徑 | `operationId` | 查詢參數與本文 | 成功回應 | 用途 |
| --- | --- | --- | --- | --- |
| `GET /v1/sites/{siteId}/traffic-matching-lists` | `getTrafficMatchingLists` | `offset`、`limit`、`filter` | `200` JSON `Traffic matching lists page` | 取得站點的流量比對清單。 |
| `POST /v1/sites/{siteId}/traffic-matching-lists` | `createTrafficMatchingList` | JSON 本文（必填） `Create or update traffic matching list` | `201` JSON `Traffic matching list` | 建立流量比對清單。 |
| `GET /v1/sites/{siteId}/traffic-matching-lists/{trafficMatchingListId}` | `getTrafficMatchingList` | — | `200` JSON `Traffic matching list` | 取得指定的流量比對清單。 |
| `PUT /v1/sites/{siteId}/traffic-matching-lists/{trafficMatchingListId}` | `updateTrafficMatchingList` | JSON 本文（必填） `Create or update traffic matching list` | `200` JSON `Traffic matching list` | 更新指定的流量比對清單。 |
| `DELETE /v1/sites/{siteId}/traffic-matching-lists/{trafficMatchingListId}` | `deleteTrafficMatchingList` | — | `200`，規格未定義本文 | 刪除指定的流量比對清單。 |

### 輔助資源（Supporting Resources，8 項）

| 方法與相對路徑 | `operationId` | 查詢參數與本文 | 成功回應 | 用途 |
| --- | --- | --- | --- | --- |
| `GET /v1/sites/{siteId}/wans` | `getWansOverviewPage` | `offset`、`limit` | `200` JSON `WAN overview page` | 查詢站點可用的 WAN 介面定義。 |
| `GET /v1/sites/{siteId}/vpn/site-to-site-tunnels` | `getSiteToSiteVpnTunnelPage` | `offset`、`limit`、`filter` | `200` JSON `IntegrationSiteToSiteVpnTunnelOverviewPageDto` | 分頁查詢站點對站點 VPN 通道。 |
| `GET /v1/sites/{siteId}/vpn/servers` | `getVpnServerPage` | `offset`、`limit`、`filter` | `200` JSON `IntegrationVpnServerOverviewPageDto` | 分頁查詢站點的 VPN 伺服器。 |
| `GET /v1/sites/{siteId}/radius/profiles` | `getRadiusProfileOverviewPage` | `offset`、`limit`、`filter` | `200` JSON `Radius Profile Overview Page` | 查詢可用的 RADIUS 驗證設定檔及其設定來源。 |
| `GET /v1/sites/{siteId}/device-tags` | `getDeviceTagPage` | `offset`、`limit`、`filter` | `200` JSON `IntegrationDeviceTagPageDto` | 列出站點的裝置標籤，可用於指定 Wi-Fi 廣播的套用範圍。 |
| `GET /v1/dpi/categories` | `getDpiApplicationCategories` | `offset`、`limit`、`filter` | `200` JSON `DPI category page` | 查詢深度封包檢測（DPI）預先定義的應用程式分類。 |
| `GET /v1/dpi/applications` | `getDpiApplications` | `offset`、`limit`、`filter` | `200` JSON `DPI application page` | 查詢深度封包檢測（DPI）可辨識的應用程式及所屬分類。 |
| `GET /v1/countries` | `getCountries` | `offset`、`limit`、`filter` | `200` JSON `Country definition page` | 查詢 ISO 國家代碼與名稱。 |

## 統計端點詳解

`getAdoptedDeviceLatestStatistics` 使用 `GET /v1/sites/{siteId}/devices/{deviceId}/statistics/latest`。兩個路徑參數都是必填 UUID。`siteId` 取自站點清單的 `data[].id`，`deviceId` 取自站點已採用裝置清單的 `data[].id`。此操作無查詢參數或請求本文。

`200 OK` 回傳 `application/json`，模型為 `Latest statistics for a device`。下表的解讀依欄位名稱，百分比的數值範圍、速率 `Bps` 的計量單位和取樣時點沒有明文定義。

| 欄位 | 型別 | 必填條件 | 用途 |
| --- | --- | --- | --- |
| `uptimeSec` | `integer`（`int64`） | 選填 | 裝置運作秒數。 |
| `lastHeartbeatAt`、`nextHeartbeatAt` | `string`（`date-time`） | 選填 | 上次與下次心跳時間，規格未說明確切計算方式。 |
| `loadAverage1Min`、`loadAverage5Min`、`loadAverage15Min` | `number`（`double`） | 選填 | 1、5、15 分鐘的平均負載。 |
| `cpuUtilizationPct`、`memoryUtilizationPct` | `number`（`double`） | 選填 | CPU 與記憶體使用率。 |
| `uplink` | 物件 | 選填 | 裝置的上行介面統計。 |
| `uplink.txRateBps`、`uplink.rxRateBps` | `integer`（`int64`） | 選填 | 上行介面的傳送與接收速率。 |
| `interfaces` | 物件 | 必填 | 介面統計。空物件符合規格。 |
| `interfaces.radios` | 物件陣列 | 選填 | 無線電統計。 |
| `interfaces.radios[].frequencyGHz` | `number` | 每筆必填 | 頻段，列舉 `0.9`、`2.4`、`5`、`6`、`60`（GHz）。 |
| `interfaces.radios[].txRetriesPct` | `number`（`double`） | 每筆選填 | 無線傳送重送率。 |

## 覆蓋範圍

本索引按 `integration.json` 的 `paths` 逐筆列出。可用來核對由此檔產生的 API 文件頁；UniFi 管理介面各個功能是否有官方 API 對應，需逐項對照該介面。
