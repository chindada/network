# 居家網路與虛擬化網路規劃

這個儲存庫整理以 UniFi UDM Pro 為核心、搭配 UniFi 與 Netgear 交換器及 ESXi 的居家網路升級規劃，也收錄 UniFi Network API 11.0.99 的操作手冊與原始規格。

## 文件

| 文件 | 內容 |
| --- | --- |
| [網路升級規劃](UPDATE.md) | 硬體拓撲、網段與 VLAN 分配、Wi-Fi、MOD 透傳、交換器連接埠、ESXi 與 WireGuard 規劃。 |
| [UniFi Network API 操作手冊](API.md) | 73 個 API 操作的台灣繁體中文用途、請求與回應索引，以及裝置最新統計端點詳解。 |
| [OpenAPI 原始規格](integration.json) | UniFi Network API 11.0.99 的 OpenAPI 3.1.0 JSON，可用來核對操作、參數與資料模型。 |

網路升級文件記錄目標配置。實際套用前，請依主控台與交換器的當前設定確認網段、連接埠及服務相依性。
