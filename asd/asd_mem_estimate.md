# Signature Processing — RAM Usage Analysis

## 1. 各階段記憶體使用量

| 階段 | 同時存活峰值 (KiB) | 該階段新增峰值 (KiB) | 說明 |
|---|---:|---:|---|
| 壓縮檔 In-Memory 下載 | 24.36 | 24.36 | memfd 與 readback heap buffer 短暫同時存在 |
| 壓縮檔 SHA-256 驗證 | 12.18 | 0 | 保留壓縮檔 heap buffer，未額外配置記憶體 |
| Manifest 處理 | 19.10 | 6.92 | 版本資料解碼、驗證及清單處理 |
| Decompress | 86.01 | 73.16 | 解壓資料、項目表及原壓縮檔同時存在 |
| Feature 驗證 | 86.01 | 0 | 重用已解壓資料，未額外配置記憶體 |
| CTX 驗證 / Base64 Decode | 92.29 | 6.28 | 21 個 CTX 逐項解碼並釋放暫存記憶體 |
| Encrypt + 落地 | **219.86** | **133.84** | 原資料、Hex Buffer、Ciphertext Buffer 及暫存路徑表同時存在 |

> **注意：**
> - 各階段存在記憶體重疊及保留關係，不可直接將各列峰值相加。
> - 「同時存活峰值」包含前面階段尚未釋放的 Buffer。
> - 「該階段新增峰值」表示相對該階段開始時的記憶體增加量。
> - 整體記憶體使用峰值出現在 **Encrypt + 落地** 階段，約 **219.86 KiB**。
> - 清理完成後，追蹤的 Heap 與 memfd 剩餘量均為 **0 bytes**。

---

## 2. 測試資料統計

| 項目 | 數值 |
|---|---:|
| 壓縮檔大小 | 12,471 bytes |
| 解壓後檔案數量 | 26 |
| 解壓後資料總大小 | 69,257 bytes |
| CTX 驗證數量 | 21 |
| 整體同時存活記憶體峰值 | **225,133 bytes** |
| 整體峰值換算 | **219.86 KiB** |
| 清理後 Heap 剩餘量 | 0 bytes |
| 清理後 memfd 剩餘量 | 0 bytes |

---

## 3. 記憶體峰值詳細分析

最大記憶體使用量發生於加密 `blockfile2026101501` 時。

由於加密階段仍保留完整解壓資料，因此原始資料、加密輸入及輸出 Buffer 會同時存在。

| 記憶體配置項目 | 使用量 (Bytes) | 說明 |
|---|---:|---|
| 壓縮檔 + 解壓資料 + 相關表格 | 88,077 | 保留壓縮檔、完整解壓資料與相關 Metadata |
| 加密落地暫存路徑表 | 6,656 | 儲存加密輸出所需的暫存路徑資訊 |
| Blockfile Hex Buffer | 55,879 | 加密處理使用的 Hex 資料 Buffer |
| Blockfile Ciphertext Buffer | 74,521 | 加密後的 Ciphertext 資料 Buffer |
| **總計（同時存活峰值）** | **225,133** | **約 219.86 KiB** |

### 記憶體峰值計算

```text
Compressed + Decompressed + Metadata =  88,077 bytes
Temporary Path Table                 =   6,656 bytes
Blockfile Hex Buffer                 =  55,879 bytes
Blockfile Ciphertext Buffer          =  74,521 bytes
----------------------------------------------------
Total Peak Memory                    = 225,133 bytes
                                     ≈ 219.86 KiB
```

---

## 4. 記憶體使用量總結

| 指標 | 結果 |
|---|---|
| 最大記憶體使用階段 | Encrypt + 落地 |
| 最大記憶體使用峰值 | **219.86 KiB** |
| 單階段最大新增峰值 | **133.84 KiB** |
| 主要記憶體使用來源 | 解壓資料、Hex Buffer、Ciphertext Buffer |
| CTX 處理方式 | 逐項 Base64 Decode，完成後釋放暫存 |
| 加密階段記憶體特性 | 原始解壓資料與加密 Buffer 同時存在 |
| 清理後 Heap / memfd 剩餘量 | **0 bytes** |

### 結論

1. **最大 RAM 使用量為 219.86 KiB**，發生於 Encrypt + 落地階段。
2. **Decompress 階段峰值為 86.01 KiB**，主要來自原壓縮檔與完整解壓資料同時存在。
3. **CTX 驗證峰值為 92.29 KiB**，透過逐項解碼與釋放暫存 Buffer，避免 21 個 CTX 的解碼暫存同時累積。
4. **加密階段為主要記憶體使用瓶頸**，因為完整解壓資料、Hex Buffer 與 Ciphertext Buffer 同時存在。
5. **各階段記憶體峰值不可直接加總**，應以整個執行流程中的最大同時存活量評估。
6. **清理完成後，追蹤的 Heap 與 memfd 剩餘量均為 0 bytes**，表示追蹤範圍內的記憶體資源已完成釋放。

> **備註：** 本報告統計的是追蹤範圍內的 Heap Buffer 與 memfd 使用量，並不等同於 Process 的完整 RSS、PSS 或實際實體 RAM 使用量。
