# 專案架構

> 目標產品：以 **AMD Ryzen Embedded V3000**（x86 控制 CPU）+ **Broadcom Tomahawk 5 (TH5)** switch ASIC（掛在 PCIe 下）組成的管理型乙太網路交換器。本文件（1）判斷 edk2-platforms 是否適用、（2）給出要在本 repo 建立的**新程式架構**、（3）列出**所有要新增/修改的功能工單**供 GitHub 審查。

---

## 1. 概觀

### 1.1 這個 repo 是什麼
TianoCore **`edk2-platforms`**（見根目錄 `Readme.md`）：UEFI/EDK II 開源韌體原始碼樹，須搭配 `edk2` 與（可選）`edk2-non-osi`。它只處理 **BIOS/UEFI 階段**，不含任何 switch data/control plane；`grep -ri tomahawk|th5|trident` 在整個 repo **零命中**（Broadcom 目錄下只有樹莓派 Bcm27xx/283x SoC）。

### 1.2 對本產品的適用性結論

| 問題 | 結論 | 依據 |
|---|---|---|
| 支援「CPU 經 PCIe 控管 switch ASIC」嗎？ | **UEFI 層這樣做就對，且 repo 已足夠** | TH5 在 UEFI 只是 PCIe endpoint，由上游 `edk2/MdeModulePkg/Bus/Pci/PciBusDxe` 自動列舉；平台提供 `PciHostBridgeLib` 宣告 root bridge 即可。 |
| 有 V3000 現成 Board Pkg 嗎？ | **沒有，需自建** | `grep -ri 'v3000\|mendocino\|phoenix\|rembrandt'` 於 `Platform/AMD` 與 `Silicon/AMD` 皆零命中。現有僅 EPYC Server（Genoa SP5、Turin SP5，見 `Platform/AMD/ServerBoard.md`）與 `XilinxBoard/ZynqMp`。 |
| AGESA 支援？ | **upstream 是 NULL instance，正式 PI 需向 AMD 取得** | `Platform/AMD/ServerBoard.md` 明載。V3000 需 Embedded/Client PI，不同於 SP5 的 EPYC PI。 |
| switch 管理（FDB/VLAN/STP…）要在哪？ | **不在 UEFI、不在此 repo** | 由 OS/NOS + Broadcom SDK 執行；UEFI 只負責讓 OS 能看見 TH5。 |

### 1.3 建置 / 測試 / Lint / 型別檢查指令（確認可跑的）
```bash
mkdir -p $WORKSPACE && cd $WORKSPACE
git clone https://github.com/tianocore/edk2.git
git -C edk2 submodule update --init
git clone https://github.com/tianocore/edk2-platforms.git
git clone https://github.com/tianocore/edk2-non-osi.git
export PACKAGES_PATH=$PWD/edk2:$PWD/edk2-platforms:$PWD/edk2-non-osi:\
$PWD/edk2-platforms/Platform/AMD:\
$PWD/edk2-platforms/Platform/AMD/EmbeddedV3000Board
. edk2/edksetup.sh
make -C edk2/BaseTools
# 新板建置（本專案要新增）
build -a X64 -t GCC -p Platform/AMD/EmbeddedV3000Board/Th5SwitchBoardPkg/Project.dsc
```
無 repo 層測試框架。靜態檢查靠外部 `edk2`：`PatchCheck.py`、`uncrustify`、ECC（皆**待設定**於本 repo CI）。

### 1.4 目錄一覽（頂層，實際存在 + 本專案將新增）
```
/Platform/
  AMD/
    AgesaModulePkg/, AgesaPkg/, AmdCbsPkg/, AmdCpmPkg/
    AmdMinBoardPkg/   ← MinPlatform 整合骨架（本專案要 inherit）
    AmdPlatformPkg/   ← AMD 板級通用（DynamicTables, Library, Universal）
    GenoaBoard/{Cinnabar,Onyx,Quartz,Ruby,Shale,Sunstone,Titanite,...}BoardPkg
    TurinBoard/{Chalupa,Galena,Onyx,Purico,Quartz,Ruby,Titanite,Volcano,Emulation}BoardPkg
    XilinxBoard/ZynqMp
    [NEW] EmbeddedV3000Board/
          └─ Th5SwitchBoardPkg/   ← 本專案主體
  其餘：96Boards, ADLINK, ARM, ASRockRack, Ampere, Bosc, Intel, Loongson,
       Marvell, MinPlatformPkg, NXP, Phytium, Qemu, Qualcomm, RISC-V,
       RaspberryPi, SiFive, SolidRun, Sophgo, StandaloneMm
/Silicon/
  AMD/{Xilinx}
  [NEW] AMD/EmbeddedV3000Pkg/   ← V3000 共用 Library/PciHostBridge（若未來多板共用）
  Broadcom/{Bcm27xx, Bcm283x, Drivers}   ← 與 TH5 無關
  其餘：ARM, Ampere, Atmel, Bosc, Intel, Marvell, Maxim, NXP, Openmoko,
       Phytium, Qemu, Qualcomm, RISC-V, SiFive, Sophgo, Synopsys
/Drivers/   ASIX, DisplayLink, OpTee, OptionRomPkg
/Features/  Ext4Pkg, Intel/{AdvancedFeaturePkg, Debugging/*, Network/*,
           OutOfBandManagement/(Asf,Ipmi,Spcr), PlatformPayloadFeaturePkg,
           PowerManagement/S3, SystemInformation/Smbios, UserInterface/*}
/CODEOWNERS, /REVIEWERS, /CONTRIBUTORS.md, /Readme.md, /License.txt
```

---

## 2. 新程式架構（V3000 + TH5 Switch 控制板）

### 2.1 分層與新增目錄

```
Platform/AMD/EmbeddedV3000Board/Th5SwitchBoardPkg/   ← 板級（本專案主要輸出）
├── Th5SwitchBoardPkg.dec        宣告 GUID / PCD
├── Project.dsc                   平台描述（照抄 TurinBoard/OnyxBoardPkg/Project.dsc 精簡）
├── Project.fdf                   Flash 映像佈局
├── Include/
│   ├── Dsc/ProjectPcd.dsc        PCD 覆寫（PCIe 範圍、TH5 bus/segment、GPIO…）
│   └── Fdf/FlashMapInclude.fdf   SPI Flash map
├── Library/
│   ├── BoardInitLib/             板級硬體初始化（GPIO、clock、reset TH5）
│   ├── PciHostBridgeLib/         V3000 PCIe root bridge 宣告（含 TH5 所在 segment）
│   ├── PlatformHookLib/          UART / 偵錯埠
│   └── BoardBdsHookLib/          BootOrder 客製（優先管理口 PXE）
├── AcpiTables/
│   ├── Dsdt/                     V3000 + TH5 的 DSDT（_DSD 描述 TH5 用途）
│   ├── Ssdt/Th5.asl              TH5 的 ACPI 節點（PNP id、BAR、GPIO 中斷）
│   └── Platform.aslc             MADT / MCFG / SPCR
├── Drivers/
│   ├── Th5PreInitDxe/            TH5 上電/重置序列（PERST#、SMBus EEPROM 讀取）
│   └── BoardConfigDxe/           HII Setup 選單（管理口 MAC、PXE 開關）
└── Binaries/                     AGESA Blob 占位（實際向 AMD 取得）

Silicon/AMD/EmbeddedV3000Pkg/                        ← SoC 共用（若只做一塊板可省）
├── EmbeddedV3000Pkg.dec
├── Include/{Library,Ppi,Protocol}/
└── Library/
    ├── V3000NbioLib/             V3000 NBIO/PCIe 控制器參數
    └── V3000FchLib/              V3000 FCH (南橋) 設定
```

### 2.2 分層依賴（單向，違反會被 CI 擋）

```
Platform/AMD/EmbeddedV3000Board/Th5SwitchBoardPkg
   │ 依賴
   ▼
Silicon/AMD/EmbeddedV3000Pkg   Platform/AMD/AmdMinBoardPkg   Platform/AMD/AmdPlatformPkg
   │                               │                            │
   └──────────────┬────────────────┴────────────┬───────────────┘
                  ▼                             ▼
        Platform/AMD/AgesaModulePkg      Features/Intel/*FeaturePkg
                  │                             │
                  └─────────────┬───────────────┘
                                ▼
              edk2 (MdePkg, MdeModulePkg, NetworkPkg, SecurityPkg)
```
禁止方向：Silicon → Platform、Features → Platform、同層 Board 互相依賴。

### 2.3 執行期資料流（UEFI 階段對 TH5 的處理）

```
SEC → PEI
  ├─ AGESA PEI (V3000): CPU/DF/UMC/PSP/NBIO init  [需 AMD Embedded AGESA]
  └─ BoardInitPeiLib: 板級 GPIO、PERST# 拉高、TH5 參考時脈 enable
DXE
  ├─ PciHostBridgeLib: 宣告 V3000 的 PCIe root bridges (含 TH5 segment)
  ├─ edk2 PciBusDxe: 掃到 TH5 → 配 BAR / MSI-X
  ├─ Th5PreInitDxe: 讀 SMBus EEPROM（SKU、MAC base）→ 塞進 HOB / PCD
  ├─ (可選) UNDI/SNP: PCIe NIC for 管理口 PXE
  └─ ACPI 安裝: DSDT/SSDT 描述 TH5 (_HID, _CRS, _DSD)
BDS
  └─ 依 BootOrder 載 Linux/NOS → OS 的 Broadcom SDK 接手 TH5
```

### 2.4 要照抄的現成檔案

| 要新增什麼 | 照抄哪個檔案 | 注意事項 |
|---|---|---|
| 板級骨架 | `Platform/AMD/TurinBoard/OnyxBoardPkg/{Project.dsc,Project.fdf,Include/}` | 改 PLATFORM_GUID、PACKAGE_GUID、OUTPUT_DIRECTORY；AGESA include 從 SP5 換成 V3000（需 AMD 提供） |
| MinPlatform 整合 | `Platform/AMD/AmdMinBoardPkg/AmdMinBoardPkg.inc.dsc`、`AmdMinBoardPkg.ci.yaml` | 新板的 CI yaml 直接複製調整 |
| PCIe root bridge | `Platform/AMD/AgesaModulePkg/Library/DxeAmdPciHostBridgeLib/PciHostBridgeLib.c/.inf` | 告訴 PciBusDxe 要掃哪些 segment / bus；TH5 所在的 root port 必列入 |
| ACPI DSDT 樣式 | `Platform/Ampere/JadePkg/Drivers/AcpiPlatformDxe/*.c` + `Features/Intel/OutOfBandManagement/IpmiFeaturePkg/BmcAcpi/BmcSsdt/BmcSsdt.asl` | TH5 以 `_DSD` 附加 vendor properties，便於 OS 層 SDK 識別 |
| 板級 BDS 客製 | `Platform/AMD/AmdMinBoardPkg/Library/`（找 BoardBdsHookLib） | BootOrder、HII |
| UEFI PXE 驅動範例 | `Drivers/OptionRomPkg/UndiRuntimeDxe/`、`Drivers/ASIX/Bus/Usb/UsbNetworking/Ax88179/` | 僅若要 UEFI 階段 PXE；多數 switch 專案不需要 |
| SMBIOS Type | `Features/Intel/SystemInformation/SmbiosFeaturePkg/SmbiosBasicDxe/Type*Function.c` | 新增 Type 9（PCIe slot：TH5）、Type 41（On-board device） |
| HII Setup 選單 | `Features/Intel/UserInterface/UserAuthFeaturePkg/UserAuthenticationDxeSmm/UserAuthenticationDxeVfr.vfr` | 參考 VFR / Formset 寫 BoardConfigDxe |

---

## 3. 功能工單清單（GitHub Issues）

> 下列 18 張工單對應「要新增/修改的所有功能」。每張皆附標題、Labels、描述、驗收條件、依賴；可直接複製貼到 GitHub Issue。Milestones 以 Epic 分組。

### Epic 分組
- **Epic A**：Board bring-up（5 張）— 讓板子能進 DXE
- **Epic B**：PCIe & TH5 整合（4 張）— OS 能看到並使用 TH5
- **Epic C**：Boot / 網路（3 張）— BootOrder、管理口 PXE、更新流程
- **Epic D**：可觀測性 & 管理（3 張）— SMBIOS、ACPI、日誌
- **Epic E**：品質 / CI（3 張）— 建置、靜態檢查、禁用字

### 工單樣板
```
Title: [領域] 一句話說明
Labels: epic:A|B|C|D|E, area:platform|silicon|features|ci, type:feat|chore|fix, priority:P0|P1|P2
Milestone: V3000-TH5 bring-up M1 | M2 | M3
Assignees: TBD
Linked PRs: —

## 背景
（為什麼要做；對 V3000+TH5 專案的意義）

## 範圍
- 新增 / 修改的路徑
- 不處理什麼

## 設計要點
- 介面 / 依賴 / 配置

## 驗收條件 (Definition of Done)
- [ ] ...

## 依賴
- #xx, #yy

## 風險 / 備註
```

---

### 🅰️ Epic A：Board bring-up

#### #A1 建立 `Platform/AMD/EmbeddedV3000Board/Th5SwitchBoardPkg` 骨架
**Labels**：`epic:A, area:platform, type:feat, priority:P0`
**背景**：repo 無 V3000 Board Pkg，需從頭建立。
**範圍**：新增 `Th5SwitchBoardPkg.dec/.dsc/.fdf`、`Include/Dsc/ProjectPcd.dsc`、`Include/Fdf/FlashMapInclude.fdf`。照抄 `Platform/AMD/TurinBoard/OnyxBoardPkg/`。
**驗收**：`build -a X64 -t GCC -p .../Project.dsc` 僅以 NULL AGESA 也能跑到 link 階段。CI 加入該 .dsc。
**依賴**：—

#### #A2 導入 V3000 Embedded AGESA（Blob 介接）
**Labels**：`epic:A, area:silicon, type:feat, priority:P0`
**背景**：upstream 只有 NULL instance（見 `Platform/AMD/ServerBoard.md`）；V3000 需 Embedded PI，須向 AMD 取得。
**範圍**：`Silicon/AMD/EmbeddedV3000Pkg/`、`Th5SwitchBoardPkg/Binaries/Agesa/`。
**驗收**：AGESA PEI/DXE 模組被 FV 包入；PEI phase 能印出 CPU/記憶體 init 訊息。
**依賴**：#A1、外部 AMD NDA
**風險**：若 AMD 不授權，整個專案阻擋。

#### #A3 BoardInitLib：GPIO / 時脈 / PERST#
**Labels**：`epic:A, area:platform, type:feat, priority:P0`
**範圍**：`Th5SwitchBoardPkg/Library/BoardInitLib/`。
**驗收**：TH5 在 DXE 開始前已被 release from reset，PCIe 參考時脈 enable；GPIO log 顯示狀態。
**依賴**：#A1、#A2

#### #A4 PlatformHookLib：UART / 偵錯埠
**Labels**：`epic:A, area:platform, type:feat, priority:P0`
**範圍**：`Th5SwitchBoardPkg/Library/PlatformHookLib/`。
**驗收**：115200 8N1 可見 `DEBUG` 輸出；SPCR ACPI 指向同一埠。
**依賴**：#A1

#### #A5 Flash Layout 與 FV 分配（FDF）
**Labels**：`epic:A, area:platform, type:feat, priority:P1`
**範圍**：`Project.fdf`、`Include/Fdf/FlashMapInclude.fdf`；劃分 `FV_BB`、`FV_MAIN`、`FV_FTW`、NVRAM。
**驗收**：映像大小符合 SPI Flash；fault-tolerant write 可用；`FvMainCompact` 可壓縮。
**依賴**：#A1

### 🅱️ Epic B：PCIe & TH5 整合

#### #B1 V3000 PciHostBridgeLib：宣告 TH5 所在 root bridge
**Labels**：`epic:B, area:platform, type:feat, priority:P0`
**背景**：`edk2/MdeModulePkg/.../PciBusDxe` 靠此 Library 列舉 TH5。
**範圍**：`Th5SwitchBoardPkg/Library/PciHostBridgeLib/`，照抄 `Platform/AMD/AgesaModulePkg/Library/DxeAmdPciHostBridgeLib/`。
**驗收**：`pci` UEFI shell 指令可看到 TH5 (Broadcom VID 0x14E4) 與其 BAR 分配完整。
**依賴**：#A2

#### #B2 Th5PreInitDxe：讀取 SKU / MAC base EEPROM
**Labels**：`epic:B, area:platform, type:feat, priority:P1`
**範圍**：`Th5SwitchBoardPkg/Drivers/Th5PreInitDxe/`。SMBus 讀 EEPROM → 存成 HOB / PCD / SetVariable。
**驗收**：OS 可從 ACPI `_DSD` 或 UEFI variable 讀到 MAC base。
**依賴**：#B1
**不處理**：不做 TH5 switching 初始化；僅讀資料與驗證存在。

#### #B3 ACPI SSDT：TH5 節點（`_HID`/`_CRS`/`_DSD`）
**Labels**：`epic:B, area:platform, type:feat, priority:P1`
**範圍**：`Th5SwitchBoardPkg/AcpiTables/Ssdt/Th5.asl`。
**驗收**：Linux `lspci -vv` 顯示裝置；OS 的 Broadcom SDK/驅動能依 ACPI 綁定。
**依賴**：#B1

#### #B4 MCFG / MADT / SPCR 更新
**Labels**：`epic:B, area:platform, type:feat, priority:P1`
**範圍**：`Th5SwitchBoardPkg/AcpiTables/Platform.aslc` 或動態表（`Platform/AMD/AmdPlatformPkg/DynamicTables`）。
**驗收**：OS 看到正確 PCIe segment、APIC、serial。
**依賴**：#B1、#A4

### 🅲 Epic C：Boot / 網路

#### #C1 BoardBdsHookLib：BootOrder 客製
**Labels**：`epic:C, area:platform, type:feat, priority:P1`
**範圍**：`Th5SwitchBoardPkg/Library/BoardBdsHookLib/`。預設優先 NVMe → 管理口 PXE → USB。
**驗收**：第一次開機 BootOrder 符合規格；`efibootmgr` 一致。
**依賴**：#A1

#### #C2 管理口 UEFI PXE / HTTP Boot
**Labels**：`epic:C, area:features, type:feat, priority:P2`
**範圍**：啟用 `NetworkPkg`（來自上游 edk2）、`Features/Intel/Network/NetworkFeaturePkg`，驗證 NIC UNDI 可用。
**驗收**：PXE 可開 Linux installer / NOS image。
**依賴**：#B1
**備註**：若產品不走 UEFI PXE，可降為 P3 或關閉。

#### #C3 Capsule Update 支援（BIOS 更新）
**Labels**：`epic:C, area:features, type:feat, priority:P2`
**範圍**：引入 `FmpDevicePkg`、`SignedCapsulePkg`（上游 edk2），於 `Project.dsc` 配置。
**驗收**：`fwupdmgr` 或 UEFI capsule 工具可成功更新 BIOS。
**依賴**：#A5

### 🅳 Epic D：可觀測性 & 管理

#### #D1 SMBIOS：Type 9/41 for TH5
**Labels**：`epic:D, area:features, type:feat, priority:P2`
**範圍**：fork `Features/Intel/SystemInformation/SmbiosFeaturePkg/SmbiosBasicDxe/` 相關 Type 函式。
**驗收**：`dmidecode` 可見 TH5 條目（slot、on-board device）。
**依賴**：#B1

#### #D2 (可選) IPMI / OOB 整合
**Labels**：`epic:D, area:features, type:feat, priority:P3`
**範圍**：若產品含 BMC，啟用 `Features/Intel/OutOfBandManagement/IpmiFeaturePkg`。
**驗收**：BMC 可讀 POST code 與 SEL。
**依賴**：—

#### #D3 Status Code / DEBUG 日誌策略
**Labels**：`epic:D, area:platform, type:chore, priority:P2`
**範圍**：`Project.dsc` 的 `PCD DEBUG_PRINT_ERROR_LEVEL`、`StatusCodeHandler`。
**驗收**：RELEASE build 無冗餘日誌；DEBUG build 有完整 PEI/DXE 軌跡。
**依賴**：#A4

### 🅴 Epic E：品質 / CI

#### #E1 新增 CI：多平台 build 矩陣
**Labels**：`epic:E, area:ci, type:chore, priority:P0`
**範圍**：GitHub Actions / stuart_ci_build；至少 build `Th5SwitchBoardPkg/Project.dsc` DEBUG + RELEASE。
**驗收**：每個 PR 綠燈才可合併。
**依賴**：#A1

#### #E2 依賴邊界掃描
**Labels**：`epic:E, area:ci, type:chore, priority:P1`
**範圍**：Python 腳本掃所有 `.inf` 的 `[Packages]`；禁止 `Silicon/*/*.inf` 引用 `Platform/*.dec`。
**驗收**：違規即 CI 失敗，附明確錯誤訊息。
**依賴**：#E1

#### #E3 Switch 禁用字 / 層界違規檢查
**Labels**：`epic:E, area:ci, type:chore, priority:P1`
**範圍**：grep `fdb|vlan_add|stp_|lacp_|netlink|sai_` 於 `Platform/AMD/EmbeddedV3000Board/` 底下；命中即失敗。
**驗收**：防止 OS 層邏輯誤寫入 UEFI。
**依賴**：#E1

### 工單依賴總圖
```
#A1 ──┬─> #A3 ──┐
      ├─> #A4 ──┤
      ├─> #A5 ──┤──> #C3
      ├─> #C1    
      └─> #E1 ──> #E2, #E3
#A2 ──> #A3 ──> #B1 ──┬─> #B2 ──> #D1
                     ├─> #B3
                     ├─> #B4
                     └─> #C2
#D2, #D3 獨立
```

---

## 4. 範本檔案（速查）

| 要新增什麼 | 照抄哪個檔案 | 注意事項 |
|---|---|---|
| V3000 新主機板 | `Platform/AMD/TurinBoard/OnyxBoardPkg/` 目錄骨架 | 改所有 GUID；AGESA 改接 V3000 Embedded PI |
| PCIe root bridge | `Platform/AMD/AgesaModulePkg/Library/DxeAmdPciHostBridgeLib/PciHostBridgeLib.c/.inf` | TH5 所在 segment 必列入 |
| DXE 驅動（Driver Binding） | `Drivers/ASIX/Bus/Usb/UsbNetworking/Ax88179/`（四件組：Entry、DriverBinding、ComponentName、Protocol） | switch 專案多半不需寫 TH5 UEFI driver |
| Library + Null 版本 | `Features/Intel/Debugging/BeepDebugFeaturePkg/Library/BeepLib/BeepLibNull.*` | 讓 feature 可關閉 |
| Feature Pkg | `Features/Intel/UserInterface/LogoFeaturePkg/` | 需提供 `Include/*.dsc`、`PreMemory.fdf`、`PostMemory.fdf` |
| ACPI SSDT | `Features/Intel/OutOfBandManagement/IpmiFeaturePkg/BmcAcpi/BmcSsdt/BmcSsdt.asl` + `BmcAcpi.c` | 用 `InstallAcpiTable` 載入 |
| SMBIOS Type | `Features/Intel/SystemInformation/SmbiosFeaturePkg/SmbiosBasicDxe/Type*Function.c` | 每 Type 一 `.c`，於 EntryPoint 註冊 |
| HII Setup | `Features/Intel/UserInterface/UserAuthFeaturePkg/UserAuthenticationDxeSmm/UserAuthenticationDxeVfr.vfr` | VFR + Formset |
| 檔案系統驅動 | `Features/Ext4Pkg/Ext4Dxe/` | 產生 `EFI_SIMPLE_FILE_SYSTEM_PROTOCOL` |

---

## 5. 本專案適用的開發規則

1. **分層依賴單向**：Platform → Silicon/Drivers/Features → edk2 核心；反向禁止（由 #E2 自動擋）。
2. **每模組必備三件組**：`Xxx.c`、`Xxx.inf`、必要時 `Xxx.uni`。缺 `.inf` 不會被編譯。
3. **命名**：型別 `UPPER_CASE_UNDERSCORE`；函式/變數 PascalCase；全域 `g` 前綴；`.inf` 內 `MODULE_TYPE` 必填（`DXE_DRIVER`、`PEIM`、`UEFI_APPLICATION`、`DXE_RUNTIME_DRIVER`…）。
4. **Protocol/PPI/GUID** 必須在 `.dec` 的對應區塊註冊後才能被 `.inf` 引用。
5. **PCD 抽離板差**：V3000 的 PCIe 範圍、TH5 bus/segment、GPIO 編號**全部**透過 PCD 或 Library 傳入，不寫死在 Silicon 原始碼。
6. **UEFI Driver Model**：產生/消耗 Protocol 的驅動必有 `DriverBinding.c`（Supported/Start/Stop）與 `ComponentName.c`。
7. **Library 提供 Null 版本**（可短路 feature）。
8. **錯誤處理**：所有 `EFI_STATUS` 函式一路回傳狀態；失敗反向釋放；`ASSERT` 只用於內部不可能狀態。
9. **檔案長度建議**：`.c` < 1500 行、函式 < 200 行（由 #E3 擴充檢查）。
10. **授權表頭**：每新檔含 SPDX `BSD-2-Clause-Patent` 與 Copyright。
11. **測試**：UEFI 無 host-side unit test；驗證走實機 + QEMU (`Platform/Qemu/*`)。
12. **Switch 功能不得混進 UEFI**：L2/L3 轉發、VLAN、STP、ACL、SNMP 等禁止；由 #E3 CI 擋。
13. **AGESA 來源**：V3000 Embedded PI 向 AMD 取得；upstream 的 NULL instance 僅供空編譯。

---

## 6. 機器檢查

### 已設定（repo 現狀）
- `build`（EDK II 編譯器前端）驗證 `.inf/.dsc/.fdf` 一致性與 Library 解析。
- `AmdMinBoardPkg.ci.yaml`、`AmdPlatformPkg.ci.yaml` 顯示 AMD 子樹已有 stuart CI 樣板，可沿用。
- `CODEOWNERS` / `REVIEWERS` 指派審查者。
- 授權：`License.txt`、`License-History.txt`。

### 待設定（對應工單 #E1–#E3 與一般化檢查）

| 檢查 | 工具 / 指令 | 目的 | 工單 |
|---|---|---|---|
| 提交/碼風格 | `python edk2/BaseTools/Scripts/PatchCheck.py HEAD~1..HEAD` | EDK II commit/patch 風格 | #E1 延伸 |
| 格式 | `uncrustify -c edk2/.pytool/Plugin/UncrustifyCheck/uncrustify.cfg` | 統一 C 格式 | #E1 |
| 靜態風格 | ECC（`edk2/BaseTools/Source/Python/Ecc/`） | 命名 / include / GUID 一致性 | #E1 |
| 多平台 build 矩陣 | GitHub Actions / `stuart_ci_build` | 每 PR 保證各 .dsc build 過 | **#E1** |
| 依賴邊界 | 掃 `.inf` 的 `[Packages]` | Silicon 不可依賴 Platform | **#E2** |
| Switch 禁用字 | grep `fdb|vlan_add|stp_|lacp_|netlink|sai_` | 防 OS 層邏輯寫入 UEFI | **#E3** |
| 檔案大小門檻 | `wc -l` + 腳本 | .c < 1500、函式 < 200 | #E3 延伸 |
| SPDX / 版權 | `reuse lint` 或 grep | 授權頭正確 | #E1 延伸 |

> 本架構文件成立後，先開 #A1、#A2、#E1 三張工單進 M1；AGESA（#A2）取得時程是整個專案的最大風險，應優先鎖定。
