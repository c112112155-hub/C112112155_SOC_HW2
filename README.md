# C112112155_SOC_HW2

## 作業題目

![HW2](https://github.com/C111112102/F115112106-SOC_HW2/blob/main/Image/HW2.jpg)

### 題目一

查找：

- User Guide
- Programming Guide
- BSP

並了解 Zynq-7020 Interrupt 架構。

### 題目二

找出 UG 中與 Interrupt 相關的 Port。

### 題目三

實作：

- LED 跑馬燈
- SW 觸發 Interrupt
- ISR 改變跑馬燈方向

---

Zynq-7020 中斷（Interrupt）查詢與實作

---

## 參考文件與頁碼定位

| 文件 | 版本／來源 | 本作業使用的位置 |
|------|-----------|------------------|
| UG585 Zynq-7000 SoC Technical Reference Manual | v1.12.1（2017/12/06） | Chapter 1 §1.1（p.26）；Chapter 7 Interrupts；Chapter 14 GPIO；Chapter 9 §9.3.3（p.261）；Appendix B（GIC 暫存器） |
| UG821 Zynq-7000 SoC Software Developers Guide | Xilinx | PDF 頁 14–20 |
| PG144 AXI GPIO LogiCORE IP Product Guide | Xilinx | PDF 頁 146–157（AXI GPIO 中斷暫存器） |
| EGO-XZ7 硬體手冊 V1.2 | E-Elements，2017/03/25 | 頁 27–28（LED／按鈕腳位） |

軟體驅動函式名稱與實際行為，以 Vitis／SDK 2018.3 產生的 BSP 原始碼為準。

---

## Q1:

**題目：從 Zynq-7020 手冊找出與 BSP（Board Support Package）、ISR 相關的章節。**

### BSP 相關

| 項目 | 位置 | 說明 |
|------|------|------|
| UG585 Chapter 1 §1.1 Overview | p.26 | Xilinx 為 PS 與 PL 週邊提供裸機（Stand-alone）與 Linux 驅動程式；其他作業系統與 BSP 由合作夥伴提供。表示 UG585 主要描述硬體，軟體端要看另外的文件。 |
| UG821 Software Developers Guide | PDF 頁 14–20 | 說明 Zynq-7000 的軟體開發流程，包含 BSP 與驅動的使用方式。 |
| BSP 原始碼（SDK 2018.3） | `xparameters.h`、`xscugic.h`、`xgpio.h` | 實際的中斷 ID 巨集與 GIC、GPIO 驅動函式。 |

### ISR 相關

| 項目 | 位置 | 說明 |
|------|------|------|
| UG585 Chapter 7 Interrupts | Chapter 7 | 本作業最直接的章節：說明 GIC、私有／共享／軟體中斷，以及 PL 中斷（IRQ_F2P）對應的中斷 ID。 |
| UG585 Chapter 9 §9.3.3 Interrupt Service Routine | p.261 | DMA 控制器的 ISR 程式設計指南，作為週邊 ISR 的寫法範例。 |
| UG585 Chapter 14 GPIO | Chapter 14 | PS 端 GPIO 的中斷說明，與 PL 端 AXI GPIO 做對照。 |
| PG144 AXI GPIO | PDF 頁 146–157 | AXI GPIO 的中斷暫存器（`GIER`、`IP IER`、`IP ISR`），ISR 內要用它清除中斷。 |

### 中斷架構摘要

PL 的按鈕訊號經 AXI GPIO 產生 `ip2intc_irpt`，接到 PS 的 `IRQ_F2P`，進入 GIC，GIC 再通知 CPU 執行對應的 ISR。

---

## Q2:

**題目：找出中斷控制器相關的暫存器：IRQ #（port）、pending register（port status）、mask register（port enable）。**

### 1. IRQ #（Port）

| 項目 | 內容 | 位置 |
|------|------|------|
| PL 到 PS 的中斷訊號組 | PS-PL Signal Groups 中的 Interrupt Request FPGA to Processor，即 `IRQ_F2P` | UG585 p.56–57 |
| IRQ ID 對照表 | `IRQ_F2P[2:0]` → 61~63；`IRQ_F2P[7:3]` → 64~68；`IRQ_F2P[15:8]` → 84~91 | UG585 p.231 |
| 本作業使用 | `IRQ_F2P[0]` → **IRQ 61** | `xparameters.h` 內的 `XPAR_FABRIC_AXI_GPIO_1_IP2INTC_IRPT_INTR` 為 61U |

### 2. Pending Register（中斷狀態）

| 層級 | 暫存器 | 說明 |
|------|--------|------|
| GIC Distributor | `ICDISPR` / `ICDICPR` | 設定／清除 pending 位元。IRQ 61 在第 2 組（`ICDISPR1`、`ICDICPR1`）的 bit 29。 |
| GIC CPU Interface | `ICCHPIR` | 目前最高優先權的 pending 中斷。 |
| GIC CPU Interface | `ICCIAR` | CPU 讀取後得到發生的中斷 ID。 |
| AXI GPIO（PL 端） | `IP ISR` | 記錄 GPIO 通道的中斷狀態，ISR 內需寫入清除。 |

### 3. Mask Register（中斷啟用）

| 層級 | 暫存器 | 說明 |
|------|--------|------|
| GIC Distributor | `ICDISER` / `ICDICER` | 啟用／停用中斷。IRQ 61 為 `ICDISER1` 的 bit 29。 |
| GIC CPU Interface | `ICCPMR` | 優先權遮罩，低於此值的中斷不會送給 CPU。 |
| AXI GPIO（PL 端） | `IP IER`、`GIER` | 通道中斷啟用與全域中斷啟用。 |

### 暫存器位址（GIC）

| 暫存器 | 位址 |
|--------|------|
| `ICDISER1` | 0xF8F01104 |
| `ICDISPR1` | 0xF8F01204 |
| `ICDICPR1` | 0xF8F01284 |
| `ICCPMR` | 0xF8F00104 |
| `ICCIAR` | 0xF8F0010C |
| `ICCEOIR` | 0xF8F00110 |
| `ICCHPIR` | 0xF8F00118 |

GIC 暫存器的詳細欄位在 UG585 Appendix B 的 Generic Interrupt Controller 模組。

---

## Q3:

