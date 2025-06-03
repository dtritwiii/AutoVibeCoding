# 即時多人白板協作平台技術文件

## 系統概述

即時多人白板協作平台是基於網頁的協作工具，讓多位用戶能在同一虛擬白板上即時互動、繪圖及溝通。本平台採用現代化技術架構，實現高效能的多人協作體驗。

> ℹ️ **注意**：本平台類似於 Miro 或 Figma，但專注於優化多人即時協作體驗。

## 核心技術架構

### 客戶端架構

```
React + TypeScript
├── Canvas API (繪圖核心)
├── Zustand (狀態管理)
├── Material UI/Chakra UI (介面元件)
└── Socket.IO Client (即時通訊)
```

### 服務端架構

```
Node.js + Express
├── Socket.IO (WebSocket通訊)
├── Redis (狀態管理與緩存)
├── MongoDB (資料持久化)
└── JWT (身份驗證)
```

## 關鍵功能實現

### 1. 即時協作機制

即時協作依賴於高效的狀態同步及衝突解決方案：

```javascript
// Socket.IO 事件監聽範例
socket.on('draw-data', (data) => {
  // 驗證資料
  if (!validateDrawData(data)) return;
  
  // 廣播到房間內所有其他用戶
  socket.to(data.roomId).emit('draw-data', {
    userId: socket.userId,
    timestamp: Date.now(),
    payload: data.payload
  });
  
  // 儲存到資料庫
  storeDrawOperation(data);
});
```

> 💡 **提示**：使用操作轉換演算法(OT)處理併發編輯衝突，確保所有用戶看到相同的白板狀態。

### 2. 渲染優化實現

Canvas 渲染優化是系統效能的關鍵：

```typescript
// 局部渲染優化範例
function renderChanges(changes: ChangeSet) {
  const ctx = canvas.getContext('2d');
  
  // 僅渲染變化的區域
  changes.regions.forEach(region => {
    const {x, y, width, height} = region;
    ctx.clearRect(x, y, width, height);
    
    // 從對象存儲中獲取並僅渲染該區域內的元素
    const elements = getElementsInRegion(region);
    elements.forEach(element => renderElement(ctx, element));
  });
}
```

## 資料模型與 API

### 資料模型

| 集合名稱 | 主要欄位 | 用途 |
|---------|---------|------|
| users | id, name, email, hashedPassword | 用戶身份與授權 |
| boards | id, title, ownerId, createdAt | 白板基本資訊 |
| board_contents | boardId, elements[], version | 白板內容存儲 |
| room_sessions | roomId, participants[], startTime | 協作會話追蹤 |

### 主要 API 端點

1. **認證 API**
   - `POST /api/auth/register` - 註冊新用戶
   - `POST /api/auth/login` - 用戶登入獲取 JWT

2. **白板管理**
   - `GET /api/boards` - 取得用戶白板列表
   - `POST /api/boards` - 創建新白板

3. **Socket.IO 事件**
   - `join-room` - 加入協作房間
   - `draw-data` - 發送繪圖操作
   - `object-update` - 更新物件屬性

## 部署與擴展性

### 部署架構

```
[用戶] → [CloudFront CDN] → [負載均衡器] → [EC2 容器集群]
                                              ↓
                            [Redis集群] ← → [MongoDB Atlas]
```

> ⚠️ **警告**：生產環境應配置適當的資料備份策略，並啟用 Redis 持久化。

### 水平擴展能力

系統設計支援水平擴展，透過：

1. 無狀態的 API 服務設計
2. Socket.IO 適配器支援多實例共享連接
3. Redis 發布/訂閱機制實現跨實例通訊

## 效能考量

1. **網路優化**
   - 批量更新減少通訊頻率
   - 訊息壓縮降低頻寬使用

2. **計算優化**
   - 使用 Web Workers 處理複雜計算
   - 採用增量渲染減輕 Canvas 負擔

## 故障排除

| 常見問題 | 可能原因 | 解決方案 |
|---------|---------|---------|
| 協作延遲過高 | 網路連接問題或伺服器負載過高 | 檢查網路連接、優化事件觸發頻率、增加伺服器資源 |
| 白板同步丟失 | WebSocket 連接中斷 | 實現自動重連機制、增加操作日誌與回放功能 |
| 渲染效能下降 | Canvas 元素過多 | 實現分層渲染、虛擬化僅渲染可視區域 |

---

本文件提供了即時多人白板協作平台的技術概覽，詳細實現細節請參考完整的開發文件和程式碼庫。