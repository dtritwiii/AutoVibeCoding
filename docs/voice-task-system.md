# 使用n8n建立語音驅動任務管理系統指南

本指南將幫助您使用n8n創建一個結合NLP和語音識別的語音驅動任務系統，使您能夠通過語音命令管理任務。

## 快速開始

### 系統概述

此工作流將使您能夠：
- 通過語音命令創建、查詢、更新和完成任務
- 設置任務提醒
- 分類和管理個人任務
- 與其他應用程序整合(如日曆和生產力工具)

### 前提條件

1. 一個n8n實例(自託管或雲端)
2. 以下服務的API密鑰：
   - 語音識別服務(如Google Speech-to-Text)
   - NLP服務(如Dialogflow或OpenAI)
   - 任務管理工具(如Todoist、Trello或自建數據庫)
   - 通知服務(如Telegram、Email或Push服務)

## 工作流設計

### 1. 主工作流組成

![n8n工作流概覽](https://example.com/workflow-overview.png)

該系統由以下幾個子工作流組成：

1. **語音捕獲與識別流程**
2. **意圖分析與處理流程**
3. **任務管理流程**
4. **通知與提醒流程**

## 詳細配置指南

### 1. 語音捕獲與識別工作流

#### 步驟1：設置觸發器

1. 添加一個**Webhook觸發器**接收語音數據
   ```
   配置：
   - 方法: POST
   - 路徑: /voice-input
   - 響應模式: 最後一個節點
   ```

2. 添加**HTTP請求**節點連接語音識別服務：
   ```
   配置：
   - 請求方法: POST
   - URL: https://speech.googleapis.com/v1/speech:recognize
   - 認證: OAuth2 / API密鑰
   - 請求數據: 
     {
       "config": {
         "languageCode": "zh-TW",
         "enableAutomaticPunctuation": true
       },
       "audio": {
         "content": "{{$binary.data.base64Encoded}}"
       }
     }
   ```

> ⚠️ **警告**：確保妥善保護API密鑰和語音數據，遵循數據隱私規範。

#### 步驟2：設置錯誤處理

添加**錯誤處理**節點處理語音識別失敗情況：

```
表達式：
if({{$node["HTTP Request"].error}}) {
  return {
    success: false,
    message: "語音識別失敗，請重新嘗試"
  };
}
```

### 2. 意圖分析與處理工作流

#### 步驟1：配置NLP處理

1. 添加**OpenAI**節點(或其他NLP服務)：
   ```
   配置：
   - 操作: 完成文本
   - 模型: gpt-4 (或其他支持的模型)
   - 提示:
     系統消息: "分析以下文本並提取用戶意圖(創建/查詢/更新/完成任務)以及相關實體(任務標題、日期時間、優先級等)。以JSON格式返回結果。"
     用戶消息: {{$node["HTTP Request"].json.results[0].alternatives[0].transcript}}
   ```

2. 添加**函數**節點解析NLP結果：
   ```javascript
   // 解析OpenAI返回的JSON格式數據
   const response = $node["OpenAI"].json.choices[0].message.content;
   try {
     const parsed = JSON.parse(response);
     return {
       intent: parsed.intent,
       entities: parsed.entities,
       rawText: $node["HTTP Request"].json.results[0].alternatives[0].transcript
     };
   } catch (error) {
     return {
       intent: "unknown",
       error: "無法解析意圖"
     };
   }
   ```

### 3. 任務管理工作流

#### 步驟1：任務操作分發

添加**Switch**節點根據意圖執行不同操作：

```
路由:
- 創建任務: {{$node["Function"].json.intent == "create"}}
- 查詢任務: {{$node["Function"].json.intent == "query"}}
- 更新任務: {{$node["Function"].json.intent == "update"}}
- 完成任務: {{$node["Function"].json.intent == "complete"}}
- 預設: true
```

#### 步驟2：配置任務創建功能

1. 添加**HTTP請求**或特定服務節點(如Todoist)：
   ```
   配置:
   - 操作: 創建任務
   - 任務標題: {{$node["Function"].json.entities.title}}
   - 截止日期: {{$node["Function"].json.entities.dueDate}}
   - 優先級: {{$node["Function"].json.entities.priority || "中"}}
   - 描述: {{$node["Function"].json.entities.description || ""}}
   ```

2. 類似地，為其他任務操作(查詢、更新、完成)配置相應節點

> 💡 **提示**：使用時區轉換節點確保日期時間處理正確，特別是處理相對時間表達式時。

### 4. 通知與提醒工作流

#### 步驟1：設置定時觸發器

1. 添加**計劃**節點檢查任務提醒：
   ```
   配置:
   - 頻率: 每5分鐘
   - 時區設置: {{$json.userTimezone || "Asia/Taipei"}}
   ```

2. 添加**HTTP請求**查詢即將到期任務：
   ```
   配置:
   - 請求方法: GET
   - URL: https://api.yourtaskservice.com/tasks/upcoming
   - 參數:
     - minutes: 30 // 提前30分鐘提醒
     - userId: {{$json.userId}}
   ```

#### 步驟2：配置提醒發送

根據用戶首選通知方式，配置以下節點之一：

- **Telegram**：發送任務提醒消息
- **Email**：發送提醒郵件
- **Push通知**：發送移動設備通知

## 進階功能配置

### 1. 日曆整合

添加**Google日曆**節點同步任務到用戶日曆：

```
配置:
- 操作: 創建事件
- 日曆ID: primary
- 開始時間: {{$node["Function"].json.entities.dueDate}}
- 結束時間: {{添加持續時間($node["Function"].json.entities.dueDate, 30, 'minutes')}}
- 總結: {{$node["Function"].json.entities.title}}
```

### 2. 用戶偏好學習

添加**函數**節點記錄用戶習慣：

```javascript
// 保存用戶使用模式