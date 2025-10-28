---
title: Line-API-群組推播
date: 2025-10-28 14:51:26
categories:
  - GCP
tags:
  - Cloud Run
---
##### 自己做了一個Side Project，會用到Line API的推播，但後來測試發現一次只能有一個Line帳號收到Line BOT的推播訊息，這篇記錄把原本的只有單人能收到推播訊息，改成把訊息推播到群組，讓有加入群組的人都可以收到推播訊息。
<!-- more -->
#
#
#
#
---
### 把機器人邀請進入要推播的群組
1. ##### 首先要先把機器人邀請進入群組
2. ##### 要記得先到Line Develope的Messaging API頁籤把Allow bot to join group chats打開
{% asset_image open_group_chat.jpg open_group_chat %}
#
#
#
#
---
### Cloud Run 設定
1. ##### 更新node.js後端程式碼，新增路由 /lineWebhook
``` text
const functions = require('@google-cloud/functions-framework');
const fetch = require('node-fetch');

const TOKEN = "{TOKEN}"; // 長期有效 token
const USER_ID = "{接收者的 User ID}"; // 接收者的 User ID

// 處理 CORS
const setCorsHeaders = (res) => {
  res.set("Access-Control-Allow-Origin", "*");
  res.set("Access-Control-Allow-Methods", "POST, GET, OPTIONS"); // 加入 GET 方法
  res.set("Access-Control-Allow-Headers", "Content-Type");
}

// ===============================================
// ✅ 新增：處理 Line Webhook 請求 (/lineWebhook)
// ===============================================
const handleLineWebhook = async (req, res) => {
  // **注意：Line Webhook 不需要 CORS 處理**

  // 1. 處理預檢請求 (雖然 Line 不會發 OPTIONS，但為保險仍保留)
  if (req.method === "OPTIONS") {
    return res.status(204).send('');
  }

  // 2. 確保是 POST 請求
  if (req.method !== "POST") {
    return res.status(405).send("Method Not Allowed. Only POST is allowed for Line Webhook.");
  }
  
  try {
    // 取得 Line 傳送過來的 events 陣列
    const events = req.body.events;

    // 遍歷所有事件
    for (const event of events) {
      console.log(`Processing event: ${JSON.stringify(event)}`);

      // 僅處理 'message' 事件 (如使用者傳送文字訊息)
      if (event.type === 'message' && event.message.type === 'text') {
        const userText = event.message.text;
        const replyToken = event.replyToken;
        
        let replyMessage = `您說了: ${userText} (此為自動回覆)`;
        
        // TODO: 在這裡加入您的業務邏輯，例如：
        // if (userText === '天氣') { replyMessage = '今天天氣很好。'; }

        // 回覆訊息給使用者 (使用 Line Reply API)
        await replyLineMessage(replyToken, replyMessage);
      } 
      // TODO: 可在此處加入其他事件處理，如 postback, follow, unfollow 等...
    }

    // **重要：Line 期望收到 HTTP 200 OK，表示 Webhook 接收成功**
    return res.status(200).send('OK');

  } catch (err) {
    console.error("Line Webhook Error:", err);
    // 即使處理失敗，建議仍返回 200，避免 Line 重複發送請求
    return res.status(200).send('Webhook Processed with Errors');
  }
};

// 輔助函數：回覆 Line 訊息
const replyLineMessage = async (replyToken, text) => {
  const url = "https://api.line.me/v2/bot/message/reply";
  const body = JSON.stringify({
    replyToken: replyToken,
    messages: [{ type: "text", text: text }],
  });

  const r = await fetch(url, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${TOKEN}`,
    },
    body: body,
  });

  if (!r.ok) {
    console.error("Line Reply API Failed:", await r.json());
  }
};

// API：推播line訊息
const sendLineMessage = async (req, res) => {

  // 設定 CORS
  setCorsHeaders(res); // 設定 CORS
  //res.set("Access-Control-Allow-Origin", "*");
  //res.set("Access-Control-Allow-Methods", "POST, OPTIONS");
  //res.set("Access-Control-Allow-Headers", "Content-Type");

  // 處理預檢請求
  if (req.method === "OPTIONS") {
    return res.status(204).send('');
  }

  // GET 測試路徑
  if (req.method === "GET") {
    return res.status(200).send("✅ LINE Proxy Service is running");
  }

  if (req.method === "POST") {
    try {
      const { message, to } = req.body || {};

      const r = await fetch("https://api.line.me/v2/bot/message/push", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          "Authorization": `Bearer ${TOKEN}`,
        },
        body: JSON.stringify({
          to: to || USER_ID,
          messages: [{ type: "text", text: message || "No message" }],
        }),
      });

      const data = await r.json();
      return res.status(200).json(data);

    } catch (err) {
      console.error(err);
      return res.status(500).send("發送失敗" + err);
    }
  }

  // 其他方法
  res.status(405).send("Method Not Allowed");
};

// API：取得目前時間
const getCurrentTime = (req, res) => {
  setCorsHeaders(res); // 設定 CORS

  if (req.method === "OPTIONS") {
    return res.status(204).send('');
  }

  if (req.method === "GET") {
    const now = new Date();
    return res.status(200).json({ time: now.toISOString() });
  }

  res.status(405).send("Method Not Allowed");
};

// 路由器函數
const router = (req, res) => {
  const path = req.path; // 取得 URL 路徑

  switch (path) {
    case '/sendLineMessage':
      return sendLineMessage(req, res);
    case '/getCurrentTime':
      return getCurrentTime(req, res);
    case '/lineWebhook': // 新增 Line Webhook 路徑
      return handleLineWebhook(req, res);      
    default:
      res.status(404).send('Not Found');
  }
};

functions.http('router', router); // 將 router 函數指定為進入點
```

2. ##### 儲存後重新佈署
{% asset_image build.jpg build %}
#
#
#
#
#
---
### Line Develope 設定
1. ##### 到Line Develope的Messaging API頁籤把Webhook URL替換成剛剛的Cloud Run網址，並且要記得把Use webhook打開
{% asset_image open_webhook.jpg open_webhook %}
2. ##### 點選Verify按鈕，如果正常呼叫會出現Success的提示視窗。
#
#
#
#
---
### 取得goupId
1. ##### 到Line群組織中隨便發個訊息。
2. ##### 再回到Cloud Run觀看Log，可以看到我們剛剛發送訊息後，Log記錄到的Payload內容。其中有一個欄位goupId就是我們要的東西!
{% asset_image test_result.jpg test_result %}
#
#
#
#
---
### 測試
1. ##### 把line推播API的to參數換成剛剛取得的goupId。
``` text
    const r = await fetch("https://api.line.me/v2/bot/message/push", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "Authorization": `Bearer ${TOKEN}`,
      },
      body: JSON.stringify({
        to: to || {把剛剛的groupId貼到這裡來},
        messages: [{ type: "text", text: message || "No message" }],
      }),
    });
```
#
#
#
#
---