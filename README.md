# VLT CNC 拋光系統

> 基於 Siemens SINUMERIK ONE 的五軸 CNC 拋光控制系統，C# 客製化 HMI 整合加工參數管理、即時補償與多站協調架構。

![Platform](https://img.shields.io/badge/platform-Windows-blue)
![Siemens](https://img.shields.io/badge/Siemens-SINUMERIK%20ONE-009999)
![TIA Portal](https://img.shields.io/badge/TIA%20Portal-SCL%20%2F%20LAD-009999)
![CSharp](https://img.shields.io/badge/C%23-.NET%204.8-512BD4)
![Status](https://img.shields.io/badge/status-Single--Station%20In%20Production%20%7C%20Multi--Station%20In%20Development-blue)

---

## 概覽

VLT CNC 拋光系統是一套針對工業拋光製程設計的 CNC 客製化控制方案，以 Siemens SINUMERIK ONE 為核心，結合 C# HMI（3GL 架構）實現五軸加工參數管理、即時電流補償與 NC 程式自動生成。

**單站架構**已完成並交付客戶試車；**多站架構**（1 主站 + 11 子站協調控制）目前仍在開發中，本文件同時涵蓋兩個階段的技術說明。

此為**公司私人專案**，本 repo 僅作作品集架構說明用途，不包含原始碼、DB 位址或任何生產敏感資訊。

---

## 系統架構

### 單站架構（已交付）

```
┌─────────────────────────────────────────┐
│         C# 3GL HMI（Windows Forms）      │
│  模式切換 │ NC程式管理 │ 補償設定 │ 監控  │
└──────────────────┬──────────────────────┘
                   │ DataSvc / FileSvc
                   │ (Siemens Operate Services)
┌──────────────────▼──────────────────────┐
│          SINUMERIK ONE                  │
│  PLC（TIA Portal）+ NCK                 │
│  五軸插補 │ TRAORI │ 電流補償 │ R 變數   │
└──────────────────┬──────────────────────┘
                   │
         ┌─────────▼─────────┐
         │   五軸拋光機台      │
         │  X / Y / Z / B / C │
         └───────────────────┘
```

### 多站架構（開發中）

```
┌─────────────────────────────────────────┐
│         C# 3GL HMI（Windows Forms）      │
│  站號選擇 │ 模式切換 │ NC程式管理 │ 監控  │
└──────────────────┬──────────────────────┘
                   │ DataSvc / FileSvc
                   │ (Siemens Operate Services)
┌──────────────────▼──────────────────────┐
│          主站 SINUMERIK ONE              │
│  DB1700（HMI 資料交換區）               │
│  同步控制區（全站廣播命令保留）          │
└──┬───────────────────────────────────┬──┘
   │ S7 GET / PUT（PROFINET）           │
   │                                    │
┌──▼──────────┐            ┌────────────▼──┐
│ 子站 1~6    │   . . .    │  子站 7~11    │
│ SINUMERIK   │            │  SINUMERIK    │
│ ONE         │            │  ONE          │
│ DB1701~1706 │            │  DB1707~1711  │
└─────────────┘            └───────────────┘
```

---

## 核心功能

### 五軸拋光控制

- **雙模式拋光切換**：定位拋（TRAORI 定向刀尖追隨）與環拋（C 軸連續旋轉）動態切換，依製程需求即時切換
- **無頓點 NC 程式架構**：M 碼與插補指令同行、布輪轉速僅在數值變化時才寫入，消除加工路徑頓點
- **NC 程式表格化管理**：C# HMI 以 CSV 表格管理加工步序，逐列自動生成對應 NC 程式段落，降低程式撰寫門檻

### 即時補償

- **磨耗補償（電流閉環）**：PLC 持續監控主軸電流，比對基準值後計算 Z 軸補償量，透過溫度補償 SD 寫入，自動修正布輪磨耗
- **線速度補償**：偵測布輪半徑縮小後重新計算轉速，維持固定線速度

### 多站協調控制（開發中）

- **1 主站 + 11 子站架構**：HMI 僅連接主站，透過 S7 GET/PUT 間接存取所有子站狀態
- **動態切站機制**：HMI 下拉選單切換站號（1~11），切站期間啟用防衝突 Flag，同步完成後才開放操作
- **全站廣播（站號 99）**：POKE_BLK 一次性將命令寫入全部子站 DB，支援全線同步啟動
- **NC 程式自動派送**：WinSCP SFTP 依站號對應 IP，自動將 NC 程式推送至對應子站 SINUMERIK ONE

### HMI 功能

- 模式切換（Manual / Auto / Reference Point）
- 即時座標顯示、I/O 狀態監控
- NC 程式載入、編輯、執行管理
- 管理員 / 操作員雙層密碼保護
- 全站同步操作（多站架構，站號 99 廣播）

---

## 通訊架構

| 通訊層 | 技術 | 說明 |
|---|---|---|
| HMI ↔ 主站 | Siemens Operate Services DataSvc | 讀寫 DB、NC 狀態 |
| HMI ↔ NC 程式 | FileSvc / WinSCP SFTP | NC 程式上傳與派送 |
| 主站 ↔ 子站 | S7 GET / PUT（PROFINET） | 資料同步，按需觸發 |
| 子站 NCK → 子站 PLC | FB2 / FB3 批次讀 | R 變數鏡像至 DB |

### 多站資料流向

```
子站 NCK R 變數
  → 子站 PLC（FB2/FB3 批次讀）
    → 子站 DB170N
      → 主站 PLC（GET）
        → 主站 DB1700
          → HMI DataSvc 讀取
```

---

## DB 設計原則（多站）

所有站共用**同構 DB 結構**（P / R 雙軌設計）：

- **P 段（Parameter）**：HMI 寫入，PLC 讀取執行——命令、設定值
- **R 段（Response）**：PLC 寫回，HMI 讀取顯示——狀態、座標、補償量
- **主站獨有區段**：同步控制預留區（全站廣播用）

同構設計讓 C# 端以單一位址計算邏輯處理所有站，僅需切換目標 DB 號。

---

## 技術選型

| 層 | 技術 | 版本 / 說明 |
|---|---|---|
| CNC 控制器 | Siemens SINUMERIK ONE | 單站 × 1（已交付）/ 主站 × 1 + 子站 × 11（開發中） |
| PLC 程式 | TIA Portal SCL / LAD | GET/PUT 通訊、補償運算、NCK 鏡像 |
| HMI 框架 | C# Windows Forms 3GL | Siemens Operate Services 整合 |
| HMI API | Siemens Operate Services DataSvc | NCK 讀寫、DB 存取 |
| NC 程式傳輸 | WinSCP（SFTP）/ FileSvc | 多站各自 IP 目標派送 |
| 開發環境 | Visual Studio 2022 x86 | .NET 4.8 |

---

## 我的參與角色

本專案由多方協作完成，包含 Siemens 原廠工程師技術支援及內部團隊協作。我主要負責的範疇：

- PLC 通訊邏輯實作與除錯（TIA Portal SCL）——跨站 GET/PUT 資料同步、POKE_BLK 全站廣播
- C# 3GL HMI 多站切換機制開發——防衝突邏輯、站號路由
- NC 程式自動生成引擎（CSV 表格 → NC code）
- WinSCP SFTP 多站 IP 自動派送整合
- 現場試車協調與問題排除

---

*Built with [Kevin Lai](https://github.com/kapy0312) — 私人公司專案，架構說明用途*