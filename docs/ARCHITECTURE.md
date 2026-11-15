# 專案架構

> 評估情境：使用 **AMD Ryzen Embedded V3000** 當控制 CPU，**Broadcom Tomahawk 5 (TH5)** switch ASIC 掛在 PCIe 下；由 CPU 透過 PCIe 管理 switch，而非 CPU 內建 switch。本文件判斷 edk2-platforms 是否支援這種架構，並描述 repo 的實際組織，供後續 AI 規劃/開發/審查依循。

## 1. 概觀

### 這是什麼
TianoCore **`edk2-platforms`**（見根目錄 `Readme.md`），屬 UEFI/EDK II 開源韌體專案，內容是 **各家 SoC/主機板的 UEFI (BIOS) 韌體原始碼**，必須搭配 [`edk2`](https://github.com/tianocore/edk2) 核心 repo 與（可選）`edk2-non-osi` 建置。

**它是**：UEFI PEI/DXE/SMM 模組、Library、Protocol、ACPI、FV 映像描述。

**它不是**：
- 不是網路交換器的 data-plane / control-plane 程式（沒有 FDB、VLAN、STP、LACP、ACL、QoS、switch SDK bindings）。
- 不是 Linux-based NOS（SONiC/FBOSS）、也不是 switch ASIC SDK。
- 不是 Broadcom Tomahawk 系列的任何驅動；`grep -ri tomahawk|th5|bcm5|trident` 在整個 repo **零結果**（只對應到 Broadcom Bcm27xx/283x 樹莓派 SoC，不是 switch ASIC）。

### 對「V3000 + TH5 (PCIe)」的直接適用性判斷

| 問題 | 結論 | 依據 |
|---|---|---|
| 這份 code 支援 CPU 經 PCIe 控制 switch ASIC（TH5）嗎？ | **不直接支援，也不需要在 UEFI 支援** | UEFI 階段只需把 TH5 當普通 PCIe endpoint 列舉（由 `edk2/MdeModulePkg/Bus/Pci/PciBusDxe` 完成），實際 switch 管理交給 OS 的 SDK/驅動。此 repo 內**沒有** TH5 或任何 managed-switch 驅動。 |
| 有 AMD V3000（Ryzen Embedded, Mendocino/Phoenix 系）的現成平台包嗎？ | **沒有** | `grep -ri 'v3000\|v2000\|v1000\|mendocino\|phoenix\|rembrandt'` 在 `Platform/AMD` 與 `Silicon/AMD` 皆無命中。`Platform/AMD` 現有板只有 EPYC Server（Genoa SP5、Turin SP5）與 `XilinxBoard/ZynqMp`（見 `Platform/AMD/ServerBoard.md`）。 |
| AGESA / PSP / Nbio 支援呢？ | 有 Server 版骨架 | `Platform/AMD/AgesaModulePkg/{Nbio,Psp,Fch,…}` 但目前 upstream 放的是 **NULL instance**；實際可開機的 AGESA 要向 AMD 取得（見 `Platform/AMD/ServerBoard.md`「Server boards edk2 build」一節）。V3000 需要的是 Client/Embedded AGESA，上游沒有。 |
| UEFI 期要透過 TH5 的管理口做 PXE/HTTP boot 嗎？ | 可選；若要，走 PCIe NIC 標準路徑 | 參考 `Drivers/OptionRomPkg/UndiRuntimeDxe/`（UNDI 範例）與 `Drivers/ASIX/.../Ax88179/SimpleNetwork.c`。Switch ASIC 本身的管理介面通常是 OOB，不走 UEFI。 |
| 控制面（FDB、VLAN、STP、SNMP、CLI）放哪？ | **不在 UEFI、不在此 repo** | 這些由 Linux + NOS（SONiC/ONL/自研）在 OS 層做，透過 Broadcom SDK（非開源，需向 Broadcom 取得）操作 TH5。 |

**結論**：edk2-platforms **可以作為 V3000 控制板的 UEFI 層**，但需要「自行新增一個 Ryzen Embedded 的 Board Pkg（copy 自最接近的 AMD 參考板 + 向 AMD 索取對應 Embedded AGESA）」。對 TH5 本身**不需也不應**在此 repo 動手；UEFI 只需做好 PCIe 列舉與 MMIO/中斷資源分配即可。

### 技術棧 / 語言
- 主要語言：C（UEFI Driver Model）、ACPI ASL、少量組語、Python（build 工具在外部 `edk2/BaseTools`）。
- 建置系統：EDK II `build`，透過 `.dec`（宣告）、`.dsc`（平台描述）、`.fdf`（Flash 映像）、`.inf`（模組）組織。
- 目標架構：AARCH64、X64、IA32、RISCV64、LOONGARCH64（`Readme.md`）；V3000 走 **X64**。

### 目錄一覽（頂層，實際存在）
```
/Platform/   各家主機板平台包
  AMD/      AgesaModulePkg, AgesaPkg, AmdCbsPkg, AmdCpmPkg,
            AmdMinBoardPkg, AmdPlatformPkg,
            GenoaBoard/{Cinnabar,Onyx,Quartz,Ruby,Shale,Sunstone,Titanite,…}BoardPkg,
            TurinBoard/{Chalupa,Galena,Onyx,Purico,Quartz,Ruby,Titanite,Volcano,Emulation}BoardPkg,
            XilinxBoard/ZynqMp
  其他：96Boards, ADLINK, ARM, ASRockRack, Ampere, Bosc, Intel, Loongson,
       Marvell, MinPlatformPkg, NXP, Phytium, Qemu, Qualcomm, RISC-V,
       RaspberryPi, SiFive, SolidRun, Sophgo, StandaloneMm
/Silicon/    SoC/晶片共用：AMD(Xilinx), ARM, Ampere, Atmel, Bosc, Broadcom
             (Bcm27xx/Bcm283x — 樹莓派, 非 switch ASIC), Intel, Marvell,
             Maxim, NXP, Openmoko, Phytium, Qemu, Qualcomm, RISC-V,
             SiFive, Sophgo, Synopsys
/Drivers/    通用 UEFI 驅動：ASIX(USB NIC), DisplayLink(GOP), OpTee,
             OptionRomPkg(ATAPI, FTDI, UNDI, CirrusLogic, Renesas FW…)
/Features/   選用功能包：Ext4Pkg,
             Intel/{AdvancedFeaturePkg, Debugging/*, Network/NetworkFeaturePkg,
                    OutOfBandManagement/(Asf,Ipmi,Spcr),
                    PlatformPayloadFeaturePkg, PowerManagement/S3,
                    SystemInformation/SmbiosFeaturePkg,
                    UserInterface/(Logo,UserAuth,VirtualKeyboard)}
/CODEOWNERS, /REVIEWERS, /CONTRIBUTORS.md
/License.txt, /License-History.txt   BSD-2-Clause-Patent
/Readme.md                           建置與貢獻說明
```
無頂層 `tests/`、`ci/`、`package.json`：**本 repo 無獨立測試或 CI**，驗證走外部 `edk2` 工具。

### 建置 / 測試 / Lint / 型別檢查指令

> 以下路徑與指令與 `Readme.md`、`Platform/AMD/ServerBoard.md` 實際對齊。

```bash
# 1) 準備 workspace
mkdir -p $WORKSPACE && cd $WORKSPACE
git clone https://github.com/tianocore/edk2.git
git -C edk2 submodule update --init
git clone https://github.com/tianocore/edk2-platforms.git
git clone https://github.com/tianocore/edk2-non-osi.git

# 2) PACKAGES_PATH（Linux 範例，AMD 板子還要加 Platform/AMD 子樹）
export PACKAGES_PATH=$PWD/edk2:$PWD/edk2-platforms:$PWD/edk2-non-osi

# 3) 建置環境與 BaseTools
. edk2/edksetup.sh
make -C edk2/BaseTools

# 4) 建置某平台（例：ARM Juno / AMD Turin Onyx）
build -a AARCH64 -t GCC -p Platform/ARM/JunoPkg/ArmJuno.dsc
# AMD 範例請看 Platform/AMD/ServerBoard.md 的 PACKAGES_PATH 與 .dsc 指令
```
輸出在 `Build/{PlatformName}/{TARGET}_{TOOLCHAIN}/FV`。交叉編譯設 `{TOOLCHAIN}_{ARCH}_PREFIX`，例如 `GCC_AARCH64_PREFIX=aarch64-linux-gnu-`。

**測試 / Lint / 型別檢查**：repo 內無；通用做法（來自上游 edk2，皆**待設定**於本 repo 的 CI）：
- `python edk2/BaseTools/Scripts/PatchCheck.py` — 提交格式檢查。
- `uncrustify` + EDK II 設定 — 程式碼格式。
- ECC（`edk2/BaseTools/Source/Python/Ecc/`）— 命名/風格靜態檢查。
- C/UEFI 無獨立型別檢查，倚賴 toolchain warning。

## 2. 分層與依賴方向

EDK II 透過 `.inf`（模組）→ `.dsc`（平台）→ `.fdf`（映像）描述，用 `LibraryClass` + `Protocol/Ppi/Guid` 解耦。各目錄職責與允許的依賴方向：

| 目錄 | 職責 | 可以依賴 | 不可以依賴 |
|---|---|---|---|
| `Silicon/{Vendor}/...` | SoC 等級通用模組（Pkg、Library、Driver、Include） | 外部 `edk2/MdePkg`、`edk2/MdeModulePkg`；同 Vendor 其他 Silicon Pkg | `Platform/*`；其他 Vendor Silicon |
| `Platform/{Vendor}/{Board}Pkg` | 具體主機板 `.dsc/.fdf`、板級 Library/Driver、ACPI | `Silicon/{Vendor}/*`、`Features/*`、`Drivers/*`、外部 edk2 | 其他 Vendor Platform；避免深度耦合特定晶片 |
| `Platform/AMD/AgesaModulePkg` | AMD AGESA（記憶體/CCX/DF/FCH/NBIO/PSP…）封裝 | AgesaPkg、AmdCpmPkg | Board Pkg（反向禁止） |
| `Platform/AMD/AmdMinBoardPkg`, `AmdPlatformPkg` | AMD 板級共用、MinPlatform 整合 | AgesaModulePkg、AgesaPkg | 具體 Board |
| `Drivers/{Vendor 或 Generic}` | 與平台無關的可重用 UEFI 驅動 | 外部 edk2 標準協定 | `Platform/*`、`Silicon/*` |
| `Features/{Category}/*FeaturePkg` | 可選功能（除錯、OOB 管理、SMBIOS、使用者認證…） | 外部 edk2；少數 Silicon 介面 | `Platform/*` |

**核心規則**：Platform 依賴 Silicon/Drivers/Features；**反向嚴禁**。

## 3. 資料流

### 3.1 典型 UEFI 開機路徑（任一平台）
1. `build -p Platform/{Vendor}/{Board}/XXX.dsc` 讀 `.dsc` 決定要編哪些模組。
2. `.fdf` 把 SEC/PEI/DXE/BDS 模組與 FV 塞進 Flash 映像。
3. 執行期：
   - PEI：早期記憶體/CPU 初始化（AMD 走 AGESA PEI 模組，見 `Platform/AMD/AgesaModulePkg/`）。
   - DXE：載入 `PciBusDxe`（來自上游 edk2）自動列舉 PCIe 樹；平台提供 `PciHostBridgeLib`（如 `Platform/AMD/AgesaModulePkg/Library/DxeAmdPciHostBridgeLib/PciHostBridgeLib.c`）告知 root bridge 範圍。
   - BDS：UEFI Boot Manager 依 BootOrder 開機。

### 3.2 本專案情境（V3000 + TH5）在 UEFI 的實際資料流

```
[開機]
  └─ PEI: AGESA 初始化 V3000 的 DF/NBIO/UMC/PSP
        （需要 AMD 提供 Embedded V3000 AGESA；upstream 是 NULL instance）
  └─ DXE: PciHostBridgeLib 宣告 V3000 的 PCIe root bridge
        └─ PciBusDxe 掃描 → 發現 TH5 (Broadcom VID/DID, Class=Network/Other)
              └─ 分配 BAR/中斷 → 完成
        └─ 管理網口若需 PXE：載入對應 PCIe NIC 的 UEFI driver（UNDI/SNP）
  └─ BDS: 從 NVMe/USB/PXE 載入 Linux 或 NOS
            └─ OS 載入 Broadcom SDK → 真正的 switch 初始化、FDB/VLAN/STP 設定
```

**要點**：UEFI 期不會也不應該去 "programing TH5 switch core"；TH5 在這階段只是一個 PCIe 裝置，拿到資源就好。真正的 switch 管理在 OS 層發生。

## 4. 範本檔案（要新增某類東西時，照抄哪裡）

| 要新增什麼 | 照抄哪個檔案 / 目錄 | 注意事項 |
|---|---|---|
| **V3000 新主機板（最相關）** | 先 copy `Platform/AMD/TurinBoard/OnyxBoardPkg/` 或 `GenoaBoard/OnyxBoardPkg/` 的目錄骨架；AGESA 部分改接 AMD 提供的 V3000 (Embedded/Client) PI | 改 `PLATFORM_GUID`、`PLATFORM_NAME`、`.fdf` 的 FV 佈局、PCD；務必向 AMD 取得 Embedded AGESA 取代 upstream 的 NULL instance（見 `Platform/AMD/ServerBoard.md`）。SP5 vs. FP7/FP8 腳位、SMBUS、GPIO、記憶體拓撲全部不同，需整套重做 |
| 一般新主機板（非 AMD） | `Platform/ARM/JunoPkg/`、`Platform/Marvell/Armada70x0Db/`、`Platform/RaspberryPi/*` 擇一 | 必改 PLATFORM/PACKAGE GUID、`OUTPUT_DIRECTORY`、所有 Library/PCD override |
| 新 SoC/晶片 Silicon Pkg | `Silicon/Marvell/Armada7k8k/`、`Silicon/NXP/LX2160A/` | 僅放可被多板共用程式；板差用 PCD/Library 抽象 |
| **PCIe root bridge 宣告（V3000 必做）** | `Platform/AMD/AgesaModulePkg/Library/DxeAmdPciHostBridgeLib/PciHostBridgeLib.c/.inf` | 給 `PciBusDxe` 用；TH5 由此被列舉。設定好 bus/IO/MMIO 範圍即可 |
| 新 UEFI DXE 驅動（Driver Binding 模型） | `Drivers/ASIX/Bus/Usb/UsbNetworking/Ax88179/`（四件組：Driver entry、DriverBinding、ComponentName、Protocol 實作） | `.inf` 的 `[Protocols]` 正確列 produced/consumed；Switch ASIC 的 UEFI 驅動**通常不需要**，除非要在 UEFI 期走 switch 的管理口 PXE |
| 新 UEFI 應用程式（.efi） | `Drivers/OptionRomPkg/Application/BltLibSample/` | `MODULE_TYPE = UEFI_APPLICATION` |
| 新 Library（給其他模組 link） | `Drivers/OptionRomPkg/Library/FrameBufferBltLib/`、`Features/Intel/Debugging/BeepDebugFeaturePkg/Library/BeepLib/`（含 `BeepLibNull`） | 一併提供 `*Null` 版本以便關閉 feature |
| 新 Feature Pkg | `Features/Intel/UserInterface/LogoFeaturePkg/`、`Features/Intel/PowerManagement/S3FeaturePkg/` | 必備 `Include/{Feature}.dsc`、`Include/PreMemory.fdf`、`Include/PostMemory.fdf` |
| 新 ACPI 表 / SSDT | `Features/Intel/OutOfBandManagement/IpmiFeaturePkg/BmcAcpi/BmcSsdt/BmcSsdt.asl` + `BmcAcpi.c` | 用 `InstallAcpiTable` 載入；TH5 可考慮用 ACPI DSD 讓 OS 辨識其用途 |
| 新 SMBIOS Type | `Features/Intel/SystemInformation/SmbiosFeaturePkg/SmbiosBasicDxe/Type2BaseBoardManufacturerFunction.c` | 每個 Type 一個 `.c`，在 `SmbiosBasicEntryPoint.c` 註冊 |
| 新檔案系統驅動（參考） | `Features/Ext4Pkg/Ext4Dxe/` | 產生 `EFI_SIMPLE_FILE_SYSTEM_PROTOCOL` |

## 5. 本專案適用的開發規則

> 目標：讓未來寫出來的程式能進得了 upstream，也能被 switch 產品量產 BIOS 採用。

1. **分層依賴單向**：Platform → Silicon/Drivers/Features → edk2 核心；反向禁止。
2. **每模組必備三件組**：`Xxx.c`（或多個）、`Xxx.inf`（宣告 sources/packages/libraryclasses/protocols/guids/pcd）、必要時 `Xxx.uni`。缺 `.inf` 的 C 檔不會被編譯。
3. **命名慣例**：型別 `UPPER_CASE_WITH_UNDERSCORES`；函式/變數 PascalCase；全域 `g` 前綴；`.inf` 內 `MODULE_TYPE` 必填（`DXE_DRIVER`、`UEFI_APPLICATION`、`PEIM`、`DXE_SMM_DRIVER`、`UEFI_DRIVER`…）。
4. **Protocol/PPI/GUID** 都要有唯一 GUID 並在 `.dec` 的 `[Guids]/[Protocols]/[Ppis]` 註冊後才能被 `.inf` 引用。
5. **PCD 抽離板差**：不要在 Silicon 原始碼寫死位址/時脈/GPIO/PCIe 範圍；由 `Platform/{Board}Pkg/*.dsc` 覆寫。V3000 板的 PCIe 配置、TH5 所在 bus/segment 都應透過 PCD 或 `PciHostBridgeLib` 傳入。
6. **UEFI Driver Model**：產生/消耗 Protocol 的驅動要實作 `DriverBinding.c`（Supported/Start/Stop）與 `ComponentName.c`，範本 `Drivers/ASIX/.../Ax88179/`。
7. **Library 提供 Null 版本**（可短路 feature），例：`Features/Intel/Debugging/BeepDebugFeaturePkg/Library/BeepLib/BeepLibNull.*`。
8. **錯誤處理**：所有返回 `EFI_STATUS` 的函式一路回傳狀態；失敗路徑反向釋放資源（記憶體、Event、Protocol）。`ASSERT` 只用於內部不可能狀態，不處理外部輸入。
9. **檔案長度建議** `.c` < 1500 行（例：`Silicon/Marvell/Drivers/Net/Pp2Dxe/Mvpp2Lib.c` 已偏大），超過時拆檔。目前**未自動檢查**。
10. **授權表頭**：每個新檔案開頭 SPDX `BSD-2-Clause-Patent` 與 Copyright，與 `License.txt` 一致。
11. **沒有單元測試文化**：UEFI 模組通常無 host-side unit test；驗證走實機或 QEMU 開機。新模組應提供 QEMU 或 `Platform/Qemu/*` 建置目標以便審查。
12. **Switch 功能不得混進 UEFI**：L2/L3 轉發、VLAN、STP、ACL 等**嚴禁**塞進 DXE 驅動；UEFI 僅負責 PCIe 列舉與必要的 ACPI 描述。這條對本次 V3000+TH5 專案特別重要。
13. **AGESA 來源**：AMD 板的實際可開機 AGESA 必須向 AMD 取得；upstream 的是 NULL instance（見 `Platform/AMD/ServerBoard.md`）。V3000 需要 Embedded/Client PI，不是 EPYC SP5 的那份。

## 6. 機器檢查

### 已設定
- `build`（EDK II 編譯器前端）會驗證 `.inf/.dsc/.fdf` 一致性、Library 解析、PCD 覆寫，是最實際的結構檢查。
- `CODEOWNERS` / `REVIEWERS` 指派審查者。
- 授權檔：`License.txt`、`License-History.txt`。

### 待設定（repo 內目前無對應設定檔，建議補）

| 檢查 | 工具 / 指令 | 目的 |
|---|---|---|
| 提交/碼風格 | `python edk2/BaseTools/Scripts/PatchCheck.py HEAD~1..HEAD` | commit message 與 patch 的 EDK II 風格 |
| 程式碼格式 | `uncrustify -c edk2/.pytool/Plugin/UncrustifyCheck/uncrustify.cfg -F files.txt` | 統一 C 檔格式 |
| 靜態風格 | ECC（`edk2/BaseTools/Source/Python/Ecc/`） | 命名、include、Protocol/GUID 一致性 |
| 多平台建置矩陣 | GitHub Actions / `stuart_ci_build` | 每個 `Platform/*/*.dsc` CI 內 build（DEBUG + RELEASE） |
| 檔案大小門檻 | 自訂腳本 + `wc -l` | 警告 `.c` > 1500 行、函式 > 200 行 |
| 依賴邊界 | 自訂腳本：掃 `.inf` 的 `[Packages]` | 禁止 `Silicon/*/*.inf` 引用 `Platform/*.dec` |
| SPDX / 版權 | `reuse lint` 或自訂 grep | 保證每個新檔案含正確授權頭 |
| **Switch 專案額外**：禁用字檢查 | 自訂腳本 grep `fdb|vlan_add|stp_|lacp_` 於 `Platform/{Vendor}/{V3000Board}/` | 避免有人把 switch control plane 誤塞進 UEFI |

> 若要把本 repo 作為 V3000 + TH5 switch 產品的 UEFI 層，建議至少把「多平台建置矩陣」、「依賴邊界」、「Switch 禁用字檢查」設成必過，否則很容易在不同 board 間互污染或誤把 OS 層邏輯寫進 UEFI。
