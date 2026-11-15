# 專案架構

> 本文件用於判斷此程式碼庫是否適合作為「switch（網路交換器）產品」的基礎，並描述其實際組織方式，供後續 AI 規劃/開發/審查依循。

## 1. 概觀

### 這是什麼
這個 repo 是 **TianoCore `edk2-platforms`**（見根目錄 `Readme.md`），屬於 UEFI/EDK II 開源韌體專案，內容是 **各家 SoC/主機板的 UEFI 韌體原始碼**（BIOS 等級），搭配獨立的 [`edk2`](https://github.com/tianocore/edk2) 核心 repo 使用。

它**不是**：
- 不是網路交換器 (L2/L3 switch) 的 data-plane / control-plane 程式（沒有 FDB、VLAN、STP、LACP、ACL、QoS、SDK bindings）。
- 不是 Linux-based NOS（像 SONiC、OpenSwitch、FBOSS），也不是 switch ASIC SDK。
- 不是網路應用程式框架。

它**是**：
- UEFI 韌體：PEI/DXE/SMM 模組、Library、Protocol、ACPI 表、FV 映像描述。
- 含有少量 **UEFI 期間使用的 NIC 驅動**（Simple Network Protocol），只做 PXE/HTTP boot 等開機期間的單埠網路，不是交換器資料面。

### 技術棧 / 語言
- 主要語言：C（UEFI Driver Model）、ACPI ASL、少量組語、Python（build 工具腳本，位於外部 `edk2/BaseTools`）。
- 建置系統：EDK II `build` 工具，透過 `.dec`（宣告）、`.dsc`（平台描述）、`.fdf`（Flash 映像描述）、`.inf`（模組描述）組織。
- 目標架構：AARCH64、X64、IA32、RISCV64、LOONGARCH64（見 `Readme.md`）。

### 目錄一覽（頂層，實際存在）
```
/Platform/   各家平台（主機板）描述：96Boards, ADLINK, AMD, ARM, ASRockRack,
             Ampere, Bosc, Intel, Loongson, Marvell, MinPlatformPkg, NXP,
             Phytium, Qemu, Qualcomm, RISC-V, RaspberryPi, SiFive, SolidRun,
             Sophgo, StandaloneMm
/Silicon/    各家 SoC/晶片共用的 package：AMD, ARM, Ampere, Atmel, Bosc,
             Broadcom, Intel, Marvell, Maxim, NXP, Openmoko, Phytium, Qemu,
             Qualcomm, RISC-V, SiFive, Sophgo, Synopsys
/Drivers/    與特定平台無關、可共用的 UEFI 驅動：ASIX（USB NIC）,
             DisplayLink, OpTee, OptionRomPkg(ATAPI, FTDI, UNDI, CirrusLogic…)
/Features/   選用功能包：Ext4Pkg, Intel/AdvancedFeaturePkg,
             Intel/Debugging/*, Intel/Network/NetworkFeaturePkg,
             Intel/OutOfBandManagement/*(Asf, Ipmi, Spcr),
             Intel/PlatformPayloadFeaturePkg, Intel/PowerManagement/S3,
             Intel/SystemInformation/SmbiosFeaturePkg,
             Intel/UserInterface/*(Logo, UserAuth, VirtualKeyboard)
/CODEOWNERS, /REVIEWERS, /CONTRIBUTORS.md   維護者
/License.txt, /License-History.txt           BSD-2-Clause-Patent
/Readme.md                                   建置與貢獻說明
```
沒有頂層 `tests/`、`ci/`、`package.json` 之類；本 repo **沒有獨立測試或 CI 配置**，一切靠外部 `edk2` 的工具。

### 建置 / 測試 / Lint / 型別檢查指令
> 這些指令需要先 clone `edk2`（與可選 `edk2-non-osi`）並設定 `WORKSPACE`、`PACKAGES_PATH`，見 `Readme.md`。

- 設定環境（在 `$WORKSPACE`）：
  ```bash
  export PACKAGES_PATH=$PWD/edk2:$PWD/edk2-platforms:$PWD/edk2-non-osi
  . edk2/edksetup.sh
  make -C edk2/BaseTools
  ```
- 建置某個平台（以 ARM Juno 為例，與 `Readme.md` 一致）：
  ```bash
  build -a AARCH64 -t GCC -p Platform/ARM/JunoPkg/ArmJuno.dsc
  ```
  輸出放在 `Build/{PlatformName}/{TARGET}_{TOOLCHAIN}/FV`。
- 交叉編譯需設 `{TOOLCHAIN}_{ARCH}_PREFIX`，例如 `GCC_AARCH64_PREFIX=aarch64-linux-gnu-`。
- 批次建置（第三方腳本，非本 repo 一部分）：`uefi-tools/edk2-build.sh <platform>`。
- **測試**：本 repo 無測試框架。EDK II 社群使用的檢查都在外部 `edk2`：
  - `edk2/BaseTools/Scripts/PatchCheck.py` — 提交格式/內容檢查（待設定於 CI）。
  - `uncrustify` + EDK II 設定檔 — 程式碼格式（待設定）。
  - ECC（EDK II Code Checker, in BaseTools）— 命名/風格靜態檢查（待設定）。
- **型別檢查**：C/UEFI 無獨立型別檢查步驟；編譯 warning 由 toolchain 負責。

### 是否適合 switch 產品？（給本次分析目標的直接結論）
- 如果「switch 產品」= 管理型乙太網路交換器（有 switch ASIC、要跑 L2/L3、VLAN、STP、SNMP、CLI）：**此 repo 本身不適合當主體**。這裡沒有任何 switching data/control plane 元件。
- 如果「switch 產品」內部 CPU 控制板需要一個 **UEFI/BIOS**（例如 ARM/x86 管理 CPU 從 UEFI 開機再跑 Linux NOS）：**可以用本 repo 作為 UEFI 韌體層**，從 `Platform/{Vendor}/{Board}Pkg` 挑最接近的平台範本客製。
- 若目標是為 switch 新增 UEFI 階段的網路啟動支援（PXE/HTTP Boot 走管理口），可以參考 `Silicon/Marvell/Drivers/Net/Pp2Dxe/`、`Drivers/ASIX/Bus/Usb/UsbNetworking/*` 等 SimpleNetworkProtocol 範本；但注意這些只支援單一網口，不是 switch fabric。

## 2. 分層與依賴方向

EDK II 的分層由 `.inf`（模組）→ `.dsc`（平台）→ `.fdf`（映像）描述，並透過 `LibraryClass` 與 `Protocol/Ppi/Guid` 解耦。各頂層目錄的職責與允許的依賴方向如下：

| 目錄 | 職責 | 可以依賴 | 不可以依賴 |
|---|---|---|---|
| `Silicon/{Vendor}/...` | SoC/晶片等級的通用模組（Pkg、Library、Driver、Include） | 外部 `edk2/MdePkg`, `edk2/MdeModulePkg`；同一 Vendor 其他 Silicon Pkg | `Platform/*`（反向依賴禁止）；其他 Vendor Silicon |
| `Platform/{Vendor}/{Board}Pkg` | 具體主機板的 `.dsc/.fdf`、平台專屬 Library/Driver、ACPI | `Silicon/{Vendor}/*`、`Features/*`、`Drivers/*`、外部 edk2 | 其他 Vendor Platform；避免深度依賴特定晶片的 Platform 檔 |
| `Drivers/{Vendor 或 Generic}` | 與平台無關的可重用 UEFI 驅動（如 USB NIC、DisplayLink GOP） | 外部 edk2 標準協定 | `Platform/*`、`Silicon/*`（必須通用） |
| `Features/{Category}/*FeaturePkg` | 可選功能模組（除錯、OOB 管理、SMBIOS、使用者認證…）| 外部 edk2；少數 `Silicon/*` 介面 | `Platform/*`（反向禁止） |
| `Features/Intel/*FeaturePkg/Include/*.dsc,fdf` | 可被 Platform include 的片段 | — | — |

核心規則：**Platform 依賴 Silicon/Drivers/Features；反之不可**。這是 edk2-platforms 的隱含分層，違反會造成無法在不同板子上重用。

## 3. 資料流（以典型的開機路徑為例）

**開機期間一次「UEFI PXE 開機嘗試」在本 repo 的路徑**，以 Marvell Armada 平台為例：

1. 平台選擇：`build -p Platform/Marvell/Armada70x0Db/Armada70x0Db.dsc` 讀入 `.dsc`，決定要編哪些模組。
2. FD 映像組成：`Platform/Marvell/.../*.fdf` 把 PEI/DXE/BDS 模組與 FV 塞進 Flash 映像。
3. 執行期：
   - PEI 階段：早期記憶體/CPU 初始化（多在 `Silicon/Marvell/Armada7k8k/Library`、`Silicon/Marvell/Armada7k8k/Drivers`）。
   - DXE 階段：
     - 平台 glue：`Platform/Marvell/Armada70x0Db/NonDiscoverableInitLib/NonDiscoverableInitLib.c` 註冊控制器。
     - 網路驅動：`Silicon/Marvell/Drivers/Net/Pp2Dxe/Pp2Dxe.c`（走 UEFI Driver Binding Model）產生 `EFI_SIMPLE_NETWORK_PROTOCOL` 實例，供上層 MNP/IP4/UDP4/PXE 使用。
   - BDS 階段：UEFI Boot Manager 透過 `EFI_PXE_BASE_CODE_PROTOCOL` 呼叫 NIC 進行 DHCP/TFTP。
4. 類似的 USB NIC 路徑：`Drivers/ASIX/Bus/Usb/UsbNetworking/Ax88179/DriverBinding.c` → `SimpleNetwork.c`。

**重點**：整條流程中網路只是「一個網口的收送封包」，沒有 forwarding、沒有 bridge、沒有 switch fabric 控制。

## 4. 範本檔案（要新增某類東西時，照抄哪裡）

| 要新增什麼 | 照抄哪個檔案 / 目錄 | 注意事項 |
|---|---|---|
| 一塊新主機板（Platform） | `Platform/ARM/JunoPkg/`、`Platform/Marvell/Armada70x0Db/`、`Platform/RaspberryPi/*` 擇一最接近者整包複製 | 必改：`.dec` 的 `PACKAGE_GUID`、`.dsc/.fdf` 的 `PLATFORM_GUID`、`PLATFORM_NAME`、`OUTPUT_DIRECTORY`；確保所有 PCD/Library override 都指向自家 Silicon |
| 新 SoC/晶片共用模組（Silicon Pkg） | `Silicon/Marvell/Armada7k8k/`、`Silicon/NXP/LX2160A/` | 僅放可被多個 Board 共用的程式；不要寫死特定 Board 參數（用 PCD/Library 抽象） |
| 新 UEFI DXE 驅動（通用，走 Driver Binding） | `Drivers/ASIX/Bus/Usb/UsbNetworking/Ax88179/`（含 `DriverBinding.c`, `ComponentName.c`, `SimpleNetwork.c`, `.inf`） | 四件組：Driver entry、DriverBinding、ComponentName、Protocol 實作；在 `.inf` 的 `[Protocols]` 正確列出 produced / consumed |
| 新 UEFI GOP（顯示）驅動 | `Drivers/DisplayLink/DisplayLinkPkg/DisplayLinkGop/` | 參考其 `Gop.c` + `UsbDisplayLink.c` 結構 |
| 新 UEFI 應用程式（.efi） | `Drivers/OptionRomPkg/Application/BltLibSample/` | `MODULE_TYPE = UEFI_APPLICATION`，主入口 `ShellAppMain` 或 `UefiMain` |
| 新 Library（供其他模組 link） | `Drivers/OptionRomPkg/Library/FrameBufferBltLib/`、`Features/Intel/Debugging/BeepDebugFeaturePkg/Library/BeepLib/` | 一併提供 `*Null` 版本（例：`BeepLibNull.c/.inf`），讓未使用者可短路掉 |
| 新 Feature（可被多平台 include） | `Features/Intel/UserInterface/LogoFeaturePkg/`、`Features/Intel/PowerManagement/S3FeaturePkg/` | 必備 `Include/{Feature}.dsc`、`Include/PreMemory.fdf`、`Include/PostMemory.fdf`，給 Platform 一行 `!include` 就能啟用 |
| 新 ACPI 表 / ASL | `Features/Intel/OutOfBandManagement/IpmiFeaturePkg/BmcAcpi/BmcSsdt/BmcSsdt.asl` | 搭配 `BmcAcpi.c` 用 `InstallAcpiTable` 載入 |
| 新 SMBIOS Type | `Features/Intel/SystemInformation/SmbiosFeaturePkg/SmbiosBasicDxe/Type2BaseBoardManufacturerFunction.c` 等 Type 檔 | 每個 Type 一個 `.c`，在 `SmbiosBasicEntryPoint.c` 註冊 |
| 新檔案系統驅動 | `Features/Ext4Pkg/Ext4Dxe/`（完整 File/Inode/Extents/Partition 組合） | 需產生 `EFI_SIMPLE_FILE_SYSTEM_PROTOCOL`，掛在 Partition 上 |

## 5. 本專案適用的開發規則

> 規則依 EDK II 慣例與此 repo 實際檔案驗證。目的是讓未來 AI 寫出來的程式能進得了 upstream。

1. **分層依賴單向**（Platform → Silicon/Drivers/Features → edk2 核心）。違反會讓 Silicon/Driver 無法在別的板子重用。
2. **每個模組必備三件組**：`Xxx.c`（或多個）、`Xxx.inf`（宣告 sources/packages/libraryclasses/protocols/guids/pcd）、必要時 `Xxx.uni`。缺 `.inf` 的 C 檔不會被編譯。
3. **命名慣例**：
   - 型別 `UPPER_CASE_WITH_UNDERSCORES`；函式/變數 PascalCase；全域常數 `g` 前綴；`.inf` 內 `MODULE_TYPE` 必填（`DXE_DRIVER`、`UEFI_APPLICATION`、`PEIM`、`DXE_SMM_DRIVER`、`UEFI_DRIVER` 等）。
   - 檔名採 PascalCase，`.inf` / `.dec` / `.dsc` / `.fdf` 副檔名必須正確。
4. **協定/PPI/GUID 要有正式唯一 GUID**；在 `.dec` 的 `[Guids]`/`[Protocols]`/`[Ppis]` 註冊後才能被 `.inf` 引用。
5. **PCD 用來抽掉 board-specific 常數**，不要在 Silicon 原始碼寫死位址/時脈/GPIO 編號。板子差異放 `Platform/{Board}Pkg` 的 `.dsc` 覆寫 PCD。
6. **UEFI Driver Model**：產生/消耗 Protocol 的驅動要實作 `DriverBinding.c`（Supported/Start/Stop）和 `ComponentName.c`。範本見 `Drivers/ASIX/.../Ax88179/`。
7. **Library 必附 Null 版本**（當不想啟用 feature 時可短路），見 `Features/Intel/Debugging/BeepDebugFeaturePkg/Library/BeepLib/BeepLibNull.*`。
8. **錯誤處理**：所有返回型別為 `EFI_STATUS` 的函式必須一路回傳狀態；分配的資源（記憶體、Event、Protocol 安裝）失敗路徑要反向釋放。不要 `ASSERT` 處理外部輸入；`ASSERT` 只用於內部不可能狀態。
9. **檔案長度**：單一 `.c` 建議 < 1500 行；若超過（例如 `Silicon/Marvell/Drivers/Net/Pp2Dxe/Mvpp2Lib.c`），考慮拆成多個子檔。此規則目前**未自動檢查**。
10. **授權表頭**：每個新檔案開頭要有 SPDX（`BSD-2-Clause-Patent`）與 Copyright 聲明，與 `License.txt` 一致。
11. **沒有單元測試文化**：UEFI 模組通常沒有 host-side unit test；驗證走「實際平台/QEMU 開機」。新增硬體相關模組時應至少提供 QEMU 或可模擬的 `Platform/Qemu/*` 建置目標以供審查者驗證。
12. **不可為 switch data-plane 加 ad-hoc 程式碼**：本 repo 範圍限於 UEFI 韌體；L2/L3 轉發、VLAN、STP 等應在上層作業系統實作，不可塞進 DXE 驅動。

## 6. 機器檢查

### 已設定（存在於 repo 或上游 edk2 標準流程）
- `build`（EDK II 編譯器前端）會驗證 `.inf/.dsc/.fdf` 的一致性、Library 解析、PCD 覆寫。這是最實際的「結構檢查」。
- `CODEOWNERS` / `REVIEWERS`：PR review 自動指派。
- 授權檔：`License.txt`、`License-History.txt`。

### 待設定（建議新增，皆目前此 repo 內未見設定檔）
| 檢查 | 工具 / 指令 | 目的 |
|---|---|---|
| 提交/碼風格 | `python edk2/BaseTools/Scripts/PatchCheck.py HEAD~1..HEAD` | 檢查 commit message 與 patch 的 EDK II 風格 |
| 程式碼格式 | `uncrustify -c edk2/.pytool/Plugin/UncrustifyCheck/uncrustify.cfg -F files.txt` | 統一 C 檔格式 |
| 靜態風格 | ECC（`edk2/BaseTools/Source/Python/Ecc/`） | 命名、include、Protocol/GUID 一致性 |
| 多平台建置矩陣 | GitHub Actions / stuart_ci_build | 每個 `Platform/*/*.dsc` 至少在 CI build 一次（DEBUG + RELEASE） |
| 檔案大小門檻 | 自訂腳本 + `wc -l` | 警告 `.c` 超過 1500 行、函式超過 200 行 |
| 依賴邊界 | 自訂腳本：掃 `.inf` 的 `[Packages]` | 禁止 `Silicon/*/*.inf` 引用 `Platform/*.dec` |
| SPDX / 版權 | `reuse lint` 或自訂 grep | 保證每個新檔案含正確授權頭 |

> 若打算把本 repo 作為某 switch 產品的 UEFI 層，建議至少把「多平台建置矩陣」與「依賴邊界」兩項設成必過，否則很容易在不同 board 間互污染。
