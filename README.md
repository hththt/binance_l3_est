這個專案 **`binance_l3_est`** 是由 GitHub 用戶 **OctopusTakopi** 所開發的一個開源工具。其核心目標在於**透過幣安（Binance）提供的市場數據，估算（Estimate）並重構所謂的「Level 3 (L3)」訂單簿數據**。

在加密貨幣交易中，Level 3 數據通常指「逐筆訂單（Individual Orders）」的細節，而一般交易所 API 只提供 Level 2（合併後的價格深度）。

以下是該專案的詳細說明與介紹：

### 1. 專案背景與核心目的

一般情況下，幣安的 API 提供的是 **Level 2 (L2)** 數據，也就是我們看到的「在某個價格共有多少顆幣的掛單」。
然而，專業交易者或造市商（Market Maker）往往需要更精細的 **Level 3 (L3)** 數據，用以觀察：

* **掛單隊列（Queue position）：** 自己的訂單在該價格位階排在第幾個。
* **訂單生命週期：** 每一筆訂單何時掛入、何時被取消或成交。

**`binance_l3_est`** 的目的不是直接從交易所獲取 L3（因為幣安不直接對外提供），而是利用幣安的 **Event Stream（如成交明細與深度更新）**，透過演算法來「推估」每一筆訂單的狀態。

---

### 2. 主要功能與技術特點

根據專案內容，其功能主要包含：

* **即時訂單簿重構：** 監聽幣安的 WebSocket 串流，實時模擬訂單簿的變動。
* **訂單追蹤（Order Tracking）：** 嘗試追蹤特定價格點上的每一筆獨立訂單，並給予一個估計的 ID。
* **隊列位置估計：** 當你下單後，透過該工具可以估算你的訂單前面還有多少掛單量，這對於高頻交易（HFT）中的「排隊策略」至關重要。
* **延遲補償：** 考慮到網絡傳輸延遲，專案中可能包含處理數據包亂序或遺失的機制。

---

### 3. 技術棧

該專案主要使用 **Rust** 語言編寫。
選擇 Rust 的原因是：

* **極高效能：** 處理逐筆成交數據需要極低的延遲與極高的吞吐量。
* **記憶體安全：** 在處理高併發的 WebSocket 數據流時，避免因記憶體問題導致系統崩潰。
* **併發處理：** 能夠輕鬆處理多個交易對的實時數據更新。

---

### 4. 適用場景

這個工具非常適合以下類型的使用者：

1. **高頻交易者（HFT）：** 需要精準掌握訂單簿深度與排隊狀況，以優化掛單策略。
2. **量化研究員：** 分析市場微觀結構（Market Microstructure），例如研究大單撤單對價格的影響。
3. **造市商策略開發：** 評估被动成交（Passive Fill）的機率。

---

### 5. 注意事項（風險提示）

* **估計非真實：** 標題中的 `est` 代表 **Estimate（估計）**。由於幣安官方並不廣播每一筆訂單的 ID 或變動細節，該工具是基於數學邏輯進行的推論，無法保證 100% 與交易所內部的搓合引擎狀態完全一致。
* **API 限制：** 使用此類工具需要極其穩定的網路環境，且需注意幣安對 API 請求頻率（Rate Limit）的限制。

### 如何開始使用？

如果你具備 Rust 開發環境，通常的操作流程是：

1. `git clone` 該儲存庫。
2. 配置幣安的 API（或僅使用 Public WebSocket）。
3. 使用 `cargo run` 編譯並啟動程式。

該專案對於想深入研究**訂單簿微觀結構**的開發者來說，是一個非常具有參考價值的工具。

# BINANCE PERP L3 Order Book Estimator

![Demo GIF](demo2.gif)

This project is a high-performance real-time visualization tool for the Binance perpetual swap order book. It leverages **Execution-Aware Queue Dynamics (EAQD)** to estimate a Level 3 (L3) order book microstructure from Level 2 (L2) events and real-time trade streams, providing deep insights into order queue seniority and market participant behavior.

## Key Features

*   **Real-time Data Architecture**: Low-latency streaming of order book depth and trade events via Binance WebSocket API.
*   **Advanced L3 Estimation**: Moves beyond naive estimation to account for execution priority, cancellation behavior, and market regime.
*   **Microstructure Metrics Panel**: Dedicated real-time dashboard for high-resolution microstructure health indicators.
    *   **Order-to-Trade Ratio (OTR)**: Tracks liquidity provision intensity at Top-1 and Top-20 levels.
    *   **Cancellation-to-Trade Ratio (CTR)**: Monitors cancellation/spoofing velocity across sides.
*   **Dynamic Heatmap Visualization**: Standardized depth heatmap using Z-score normalization with interactive controls.
*   **Interactive Analytics**: Sweep/Liquidity-Cost windows with right-click drag-to-zoom and multi-axis rendering.
*   **K-Means Clustering**: Real-time classification of market participants based on order size and arrival patterns.

## Technical Core

The estimator utilizes several proprietary techniques to maintain a ground-truth-aligned L3 view:

### 1. Execution-Aware Queue Dynamics (EAQD)
Refined logic for inflow and outflow handling. Unlike static models, EAQD understands the difference between fills (FIFO consumption) and cancellations (LIFO/Priority-based reduction).

### 2. Deep Depth Fragmentation (DWF)
Also known as Statistical Order Flow Profiling (SOFP). This system fragments large L2 liquidity additions into multiple virtual orders based on a rolling distribution of market trade sizes.
*   **Whale Bypass**: If an addition is extremely large (e.g., > 20x average trade size), the system bypasses fragmentation, treating it as a single high-conviction "Whale" order.
*   **Robust Multi-Order Cancellation**: Combined with EAQD, the system now handles large LIFO cancellations that span multiple fragments, ensuring accurate queue reduction even when fragmented blocks are removed.

### 3. Marker-Triggered Queue Refining (MTQR)
Integrates the `@trade` stream as a synchronization pulse. When a trade occurs at a specific price, MTQR validates the maker's size against our estimated queue. If a trade exceeds our front-of-queue estimate, the model "snaps" to the ground truth and adjusts seniority accordingly.

### 4. Seniority Decay & Priority Reset
Handles partial fills and order modifications. Partial fills retain their queue position, while modifications that increase size or change price trigger a priority reset, accurately reflecting exchange matching engine logic.

## Usage

#### From Source

Ensure you have Rust installed ([rust-lang.org](https://www.rust-lang.org)).

1. Clone the repository:
   ```bash
   git clone https://github.com/OctopusTakopi/binance_l3_est.git
   cd binance_l3_est
   ```

2. Run the project:
   ```bash
   cargo run -r
   ```

#### UI Controls
*   **Microstructure Toggle**: Use the UI panel to enable/disable the OTR/CTR dashboard.
*   **Heatmap Z-Score**: Adjust the standardization slider to highlight liquidity outliers.
*   **Zoom**: Right-click and drag on the liquidity charts to inspect specific price ranges.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
