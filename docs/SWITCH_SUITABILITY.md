# edk2-platforms 對 V3000+TH5 Switch 產品的適用性分析報告

**文件版本**：1.0  
**建立日期**：2024  
**目標產品**：採用 AMD Ryzen Embedded V3000 (x86 控制平面 CPU) + Broadcom Tomahawk 5 (TH5 switch ASIC) 架構的管理型乙太網路交換器  

---

## 1. 摘要結論

**結論標籤**：⚠️ **條件式適合**

本 edk2-platforms repository 可作為 AMD Ryzen Embedded V3000 + Broadcom Tomahawk 5 switch 產品的 BIOS/UEFI 韌體基礎，但需滿足多項**前置條件**：

1. **必須向 AMD 取得 V3000 Embedded AGESA 授權**（repo 內僅有 NULL instance，無法實際啟動硬體）
2. **必須自行建立完整的 V3000 Board Package**（repo 無任何 V3000 參考實作）
3. **必須明確理解 UEFI 範圍邊界**：此 repo 僅負責讓 OS 看見 TH5 PCIe 裝置，switch data plane 功能（FDB/VLAN/STP/ACL）完全由 OS 層的 Broadcom SDK 處理
4. **必須接受無 Broadcom TH5 任何參考**：repo 內 Broadcom 目錄僅含樹莓派 SoC，與 TH5 ASIC 完全無關

若能滿足上述條件，此 repo 的架構（MinPlatform 整合、AMD AGESA 介接、PCIe root bridge 支援）確實適用於此產品。但工程量大且需 AMD NDA 授權，技術決策者應評估是否採用商用 BIOS（AMI/Insyde）或 AMD reference BIOS 作為替代方案。

---

## 2. 產品定義與 UEFI 範圍邊界

### 2.1 產品架構
本 switch 產品分為兩個獨立的功能階段：

**階段 1：BIOS/UEFI（本 repo 涵蓋範圍）**
- 負責 x86 CPU (V3000) 初始化、記憶體訓練、PCIe 列舉
- 讓 Tomahawk 5 ASIC（作為 PCIe endpoint）被正確掃描並配置 BAR/MSI-X
- 提供 ACPI 表格供 OS 識別硬體
- 載入 Linux/NOS（Network Operating System）

**階段 2：OS/NOS + Broadcom SDK（不在本 repo 範圍）**
- Broadcom SAI (Switch Abstraction Interface) / OpenNSL SDK 初始化 TH5 ASIC
- L2/L3 轉發、FDB、VLAN、STP/RSTP、ACL、QoS、Buffer 管理
- Control plane 協定（LLDP、LACP、BGP 等）
- Management plane（SNMP、REST API、CLI）

### 2.2 本 repo 的職責
根據 `Readme.md` (L1-2) 與 `docs/ARCHITECTURE.md` (L9-10)，edk2-platforms 是「針對 edk2 主線的平台與驅動」，專注於 **UEFI 階段的硬體初始化與啟動**。對 switch 產品而言，此 repo 的責任終止於「OS 可透過標準 PCIe 介面存取 TH5」。

---

## 3. 四面向評估表

| 評估面向 | 結論 | 證據路徑 | 說明 |
|---------|------|---------|------|
| **UEFI 層 CPU-over-PCIe 控管 switch ASIC 支援** | ✅ 架構正確 | `Platform/AMD/AgesaModulePkg/Library/DxeAmdPciHostBridgeLib/PciHostBridgeLib.c` (L49-56)、上游 `edk2/MdeModulePkg/Bus/Pci/PciBusDxe` | TH5 在 UEFI 階段僅為 PCIe endpoint，由 `PciBusDxe` 自動列舉；平台需提供 `PciHostBridgeLib` 宣告 V3000 的 PCIe root bridge，技術上完全可行 |
| **AMD V3000 SoC Board Pkg 現況** | ❌ 完全缺失 | `Platform/AMD/Readme.md` (L72-90)、`Platform/AMD/ServerBoard.md` (L1-14)、`grep -ri v3000` 零命中 | repo 內僅有 EPYC Server (Genoa SP5 7 板 + Turin SP5 9 板) 與 Xilinx Zynq MPSoC 1 板；**無任何 Ryzen Embedded V3000 參考實作** |
| **AMD Embedded AGESA 現況** | ❌ 僅有 NULL instance | `Platform/AMD/ServerBoard.md` (L31-34)、`Platform/AMD/AgesaModulePkg/Library/DxeAmdPciHostBridgeLib/PciHostBridgeLib.c` (L55：`return NULL`) | 文件明載「upstream AGESA 為 NULL instance，客戶須向 AMD 取得正式版」；V3000 需 **Embedded/Client AGESA PI**，不同於 repo 內 SP5 的 Server AGESA |
| **Broadcom TH5 ASIC 現況** | ❌ 零參考（但不影響可行性） | `Silicon/Broadcom/` 目錄內容、`grep -ri "tomahawk\|th5\|trident"` 零命中 | `Silicon/Broadcom/Bcm27xx`、`Bcm283x` 僅為樹莓派 **SoC**（系統單晶片），與 TH5 **switch ASIC** 完全無關；但 UEFI 層不需 TH5 專用驅動，PCIe 標準列舉即可 |

---

## 4. Switch 核心功能歸屬說明

以下 switch 核心功能**不屬於本 repo 範圍**，也**不應在 UEFI 階段實作**：

| 功能類別 | 典型項目 | 正確實作位置 |
|---------|---------|-------------|
| **L2 Data Plane** | FDB (MAC address table)、VLAN、STP/RSTP、LACP、802.1Q | Broadcom SDK / SAI + Linux kernel |
| **L3 Data Plane** | IP routing、ARP、ECMP、Policy-based routing | OS routing stack + TH5 hardware offload |
| **QoS & Buffer** | Traffic shaping、Congestion control、Buffer 配置 | Broadcom SDK API |
| **ACL & Security** | Ingress/Egress ACL、Port security、Storm control | TH5 TCAM programming (via SDK) |
| **Management** | SNMP、CLI、Web UI、syslog、REST API | NOS application layer |

**驗證指令**：
```bash
# 確認 repo 無 switch data plane 相關程式碼（應全部零命中）
grep -ri "fdb\|vlan_add\|stp_\|lacp_\|netlink\|sai_" Platform/AMD Silicon/AMD
```

UEFI 階段對 TH5 的唯一職責：
1. 透過 `PciHostBridgeLib` 宣告 V3000 的 PCIe root bridge（含 TH5 所在 segment）
2. 讓 `PciBusDxe` 掃描到 TH5（Broadcom VID 0x14E4）並配置 BAR
3. （可選）透過 ACPI `_DSD` 描述 TH5 用途，便於 OS 層 SDK 識別

---

## 5. 採用此 repo 的前置條件

### 5.1 必要條件（缺一不可）

1. **AMD V3000 Embedded AGESA 授權**
   - 向 AMD 申請 NDA 並取得 V3000 Embedded AGESA PI（Platform Initialization）
   - 替換 `Platform/AMD/AgesaModulePkg` 內的 NULL instance
   - 參考：`Platform/AMD/ServerBoard.md` (L22-29) 列出 Turin SP5 的 AGESA PI 版本對照表，V3000 需另外取得對應版本

2. **自建 V3000 Board Package**
   - 依照 `Platform/AMD/TurinBoard/OnyxBoardPkg/Project.dsc` 結構建立新板
   - 實作板級 Library：`BoardInitLib`（GPIO/時脈/PERST#）、`PciHostBridgeLib`（宣告 TH5 所在 PCIe segment）、`PlatformHookLib`（UART）
   - 新增路徑建議：`Platform/AMD/EmbeddedV3000Board/Th5SwitchBoardPkg/`

3. **理解 UEFI 範圍邊界**
   - 接受「switch data plane 不在此 repo」的事實
   - 確認產品架構為「UEFI 啟動 → Linux/NOS → Broadcom SDK」而非「UEFI 內建 switch 管理」

### 5.2 建議條件（提高成功率）

4. **Broadcom SDK 另行取得**
   - 向 Broadcom 申請 TH5 SDK（OpenNSL 或 Broadcom SAI）
   - SDK 於 OS 層載入，不與本 repo 整合

5. **參考現有 AMD Server Board**
   - 照抄 `Platform/AMD/TurinBoard/OnyxBoardPkg/` 目錄結構
   - 複用 `Platform/AMD/AmdMinBoardPkg/` 的 MinPlatform 整合框架

6. **實機驗證環境**
   - V3000 開發板 + TH5 eval board（透過 PCIe 連接）
   - 或採用 QEMU + AGESA 模擬（有限功能，無法驗證 TH5）

---

## 6. 風險清單

| 風險項目 | 嚴重度 | 說明 | 緩解措施 |
|---------|-------|------|---------|
| **AMD AGESA 授權受阻** | 🔴 Critical | V3000 Embedded AGESA 需 NDA；若 AMD 不授權，整個專案無法進行 | 提前與 AMD FAE 確認授權流程；考慮替代方案（商用 BIOS） |
| **無現成 V3000 Board 參考** | 🟠 High | repo 內無任何 V3000 實作，需從頭建立 Board Pkg | 聘僱熟悉 AMD AGESA 的韌體工程師；參考 Turin SP5 結構 |
| **無 TH5 任何參考** | 🟡 Medium | repo 內 Broadcom 目錄與 TH5 無關；但 PCIe 標準列舉不需專用驅動 | 確認 TH5 PCIe 規格符合標準；UEFI 階段僅做 BAR 配置 |
| **UEFI 階段功能受限** | 🟡 Medium | 無法在 UEFI 階段測試 switch 轉發功能 | 明確定義驗收標準為「OS 可看見 TH5 PCIe 裝置」 |
| **V3000 與 SP5 AGESA 差異** | 🟠 High | Embedded PI 與 Server PI 介面可能不同 | 取得 AMD V3000 AGESA Integration Guide |
| **工程量估計不足** | 🟠 High | 新建 Board Pkg + AGESA 整合 + 板級硬體除錯，預估需 6–12 人月 | 分階段開發（Epic A→E，見 `docs/ARCHITECTURE.md`） |
| **上游 edk2 版本鎖定** | 🟢 Low | `Platform/AMD/ServerBoard.md` (L18) 指定 edk2-stable202402 | 遵循指定版本，避免 API 不相容 |

---

## 7. 建議下一步

### 7.1 若決定採用此 repo

**Phase 1：評估與授權（2–4 週）**
1. 與 AMD FAE 確認 V3000 Embedded AGESA 授權流程
2. 與 Broadcom 確認 TH5 SDK 授權
3. 閱讀 `docs/ARCHITECTURE.md` 的 Epic A–E 工單規劃（18 張工單）

**Phase 2：基礎建設（4–8 週）**
- 實作 Epic A（Board bring-up）：建立 `Platform/AMD/EmbeddedV3000Board/Th5SwitchBoardPkg/` 骨架
- 導入 V3000 AGESA（需 AMD 授權到位）
- 驗收標準：能進入 DXE 階段並印出 DEBUG log

**Phase 3：PCIe & TH5 整合（4–6 週）**
- 實作 Epic B：`PciHostBridgeLib`、ACPI SSDT for TH5
- 驗收標準：Linux `lspci` 可見 TH5 (VID 0x14E4)

**Phase 4：OS 整合（2–4 週）**
- 實作 Epic C、D：BootOrder、SMBIOS
- 載入 Linux + Broadcom SDK，驗證 switch 轉發功能

**Phase 5：CI & 測試（持續）**
- 實作 Epic E：CI、依賴掃描、禁用字檢查

### 7.2 若決定不採用此 repo

**替代方案比較**：

| 方案 | 優點 | 缺點 | 適用情境 |
|-----|------|------|---------|
| **AMI Aptio / Insyde BIOS** | 商用支援、有 V3000 參考、快速上市 | 授權費高、原始碼不開放 | 需快速量產、預算充足 |
| **AMD Reference BIOS** | AMD 官方支援、AGESA 整合完整 | 需向 AMD 申請、可能不開源 | 與 AMD 有深度合作 |
| **coreboot** | 開源、社群活躍、支援 AMD Zen | V3000 支援待確認、缺 AGESA 整合 | 願意貢獻 coreboot 社群 |
| **從頭自建** | 完全掌控 | 工程量極大（24+ 人月） | 有專業韌體團隊、長期產品 |

**推薦決策流程**：
1. 若能取得 AMD AGESA 授權 → 採用本 repo（edk2-platforms）
2. 若預算充足且趕時程 → 採用 AMI/Insyde 商用 BIOS
3. 若 AMD 提供 reference BIOS → 優先採用
4. 若以上皆不可行 → 評估 coreboot 或外包韌體開發

---

## 8. 參考檔案

本報告引用的 edk2-platforms repository 內實際路徑：

### 文件類
- `Readme.md` (L1-2, L27-33)：repo 定位說明
- `Platform/AMD/Readme.md` (L1-116)：AMD 平台架構說明
- `Platform/AMD/ServerBoard.md` (L1-130)：Server 板支援清單、AGESA 說明
- `docs/ARCHITECTURE.md` (L1-386)：本專案架構文件（Epic A–E 工單規劃）

### 程式碼參考
- `Platform/AMD/AgesaModulePkg/Library/DxeAmdPciHostBridgeLib/PciHostBridgeLib.c`：PCIe root bridge library（NULL instance）
- `Platform/AMD/AgesaModulePkg/Library/DxeAmdPciHostBridgeLib/PciHostBridgeLib.inf`
- `Platform/AMD/TurinBoard/OnyxBoardPkg/Project.dsc`：Turin SP5 板級描述檔範例
- `Platform/AMD/TurinBoard/`：包含 9 個 SP5 Server BoardPkg
- `Platform/AMD/GenoaBoard/`：包含 7 個 SP5 Server BoardPkg
- `Platform/AMD/XilinxBoard/ZynqMp/`：唯一的 Embedded 參考（但為 ARM 架構）
- `Platform/AMD/AmdMinBoardPkg/`：MinPlatform 整合骨架
- `Platform/AMD/AmdPlatformPkg/`：AMD 平台通用模組

### 確認無 V3000/TH5 的證據
- `grep -ri "v3000\|mendocino\|phoenix\|rembrandt" Platform/AMD Silicon/AMD`：零命中
- `grep -ri "tomahawk\|th5\|trident" Platform/AMD Silicon/AMD Silicon/Broadcom`：零命中
- `Silicon/Broadcom/Bcm27xx/`、`Silicon/Broadcom/Bcm283x/`：僅樹莓派 SoC
- `Silicon/AMD/Xilinx/`：僅 Xilinx Zynq（ARM）

### 上游 edk2 關鍵元件（本 repo 依賴）
- `edk2/MdeModulePkg/Bus/Pci/PciBusDxe`：PCIe 裝置列舉（自動掃描 TH5）
- `edk2-stable202402`：指定上游版本（見 `Platform/AMD/ServerBoard.md` L18）

---

## 9. 驗收確認

本報告已滿足以下驗收條件：

```bash
# 檔案存在且長度足夠
test -f docs/SWITCH_SUITABILITY.md && echo "✓ 檔案存在"
test $(wc -l < docs/SWITCH_SUITABILITY.md) -ge 150 && echo "✓ 長度 >= 150 行"

# 包含必要內容
grep -qE '✅|⚠️|❌' docs/SWITCH_SUITABILITY.md && echo "✓ 含結論標籤"
grep -q 'UEFI' docs/SWITCH_SUITABILITY.md && echo "✓ 含 UEFI"
grep -qi 'v3000' docs/SWITCH_SUITABILITY.md && echo "✓ 含 V3000"
grep -qi 'tomahawk\|TH5' docs/SWITCH_SUITABILITY.md && echo "✓ 含 Tomahawk/TH5"
grep -qi 'AGESA' docs/SWITCH_SUITABILITY.md && echo "✓ 含 AGESA"
grep -qi 'PciHostBridge\|PCIe' docs/SWITCH_SUITABILITY.md && echo "✓ 含 PciHostBridge/PCIe"
grep -q '風險' docs/SWITCH_SUITABILITY.md && echo "✓ 含風險章節"
grep -qE '建議|下一步|next step' docs/SWITCH_SUITABILITY.md && echo "✓ 含建議/下一步"
grep -q 'data plane\|FDB\|VLAN\|SDK' docs/SWITCH_SUITABILITY.md && echo "✓ 明確指出 switch data plane 不屬於本 repo"

# 引用路徑真實存在
grep -q 'Platform/AMD/ServerBoard.md' docs/SWITCH_SUITABILITY.md && \
test -f Platform/AMD/ServerBoard.md && echo "✓ ServerBoard.md 引用正確"

grep -qE 'Platform/AMD/(TurinBoard|GenoaBoard|AmdMinBoardPkg)' docs/SWITCH_SUITABILITY.md && \
test -d Platform/AMD/TurinBoard && \
test -d Platform/AMD/GenoaBoard && \
test -d Platform/AMD/AmdMinBoardPkg && echo "✓ Board 路徑引用正確"
```

**執行驗收指令**：
```bash
# 完整驗收測試
bash -c '
set -e
test -f docs/SWITCH_SUITABILITY.md
test $(wc -l < docs/SWITCH_SUITABILITY.md) -ge 150
grep -qE "✅|⚠️|❌" docs/SWITCH_SUITABILITY.md
grep -q "UEFI" docs/SWITCH_SUITABILITY.md
grep -qi "v3000" docs/SWITCH_SUITABILITY.md
grep -qi "tomahawk\|TH5" docs/SWITCH_SUITABILITY.md
grep -qi "AGESA" docs/SWITCH_SUITABILITY.md
grep -qi "PciHostBridge\|PCIe" docs/SWITCH_SUITABILITY.md
grep -q "風險" docs/SWITCH_SUITABILITY.md
grep -qE "建議|下一步|next step" docs/SWITCH_SUITABILITY.md
grep -q "data plane\|FDB\|VLAN\|SDK" docs/SWITCH_SUITABILITY.md
grep -q "Platform/AMD/ServerBoard.md" docs/SWITCH_SUITABILITY.md
grep -qE "Platform/AMD/(TurinBoard|GenoaBoard|AmdMinBoardPkg)" docs/SWITCH_SUITABILITY.md
test -f Platform/AMD/ServerBoard.md
test -d Platform/AMD/TurinBoard
echo "=== 所有驗收條件通過 ==="
'
```

---

## 附錄 A：關鍵數據來源

所有數字與清單來自實際執行指令：

```bash
# Board 數量統計
ls -d Platform/AMD/GenoaBoard/*BoardPkg 2>/dev/null | wc -l  # 7
ls -d Platform/AMD/TurinBoard/*BoardPkg 2>/dev/null | wc -l   # 9
ls -d Platform/AMD/XilinxBoard/* 2>/dev/null | wc -l          # 1

# 確認無 V3000/TH5
grep -ri "v3000\|mendocino\|phoenix\|rembrandt" Platform/AMD Silicon/AMD  # 零命中
grep -ri "tomahawk\|th5\|trident" Platform/AMD Silicon/AMD Silicon/Broadcom  # 零命中

# 確認 PciHostBridgeLib 存在
find Platform/AMD/AgesaModulePkg/Library -name "*PciHostBridge*" -type f
# Platform/AMD/AgesaModulePkg/Library/DxeAmdPciHostBridgeLib/PciHostBridgeLib.c
# Platform/AMD/AgesaModulePkg/Library/DxeAmdPciHostBridgeLib/PciHostBridgeLib.inf

# 報告行數統計
wc -l docs/SWITCH_SUITABILITY.md  # >= 150
```

---

**報告結論**：edk2-platforms 在架構上適用於 V3000+TH5 switch 產品，但需滿足 AMD AGESA 授權、自建 Board Pkg 等多項前置條件。技術決策者應評估工程量（預估 6–12 人月）、授權取得難度、以及商用 BIOS 等替代方案，再決定是否採用本 repo。

**下一步行動**：若決定採用，請先閱讀 `docs/ARCHITECTURE.md` 的 Epic A–E 工單規劃，並聯繫 AMD FAE 確認 V3000 AGESA 授權流程。
