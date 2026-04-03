/**
 * 全自動財經 Reels 發布系統 (完整實作版)
 */

require('dotenv').config();
const axios = require('axios');

const apiKey = process.env.GEMINI_API_KEY;
const MODEL_URL = `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`;

// --- 輔助工具：指數退避重試 ---
async function fetchWithRetry(data, retries = 5) {
    for (let i = 0; i < retries; i++) {
        try {
            const response = await axios.post(MODEL_URL, data);
            return response;
        } catch (error) {
            const delay = Math.pow(2, i) * 1000;
            if (i === retries - 1) throw error;
            console.log(`⚠️ API 忙碌，${delay/1000}s 後重試...`);
            await new Promise(r => setTimeout(r, delay));
        }
    }
}

// --- 1. 獲取真實市場數據 (範例使用免費 API) ---
async function fetchMarketData() {
    console.log("📊 [1/5] 正在抓取 Yahoo Finance/Alpha Vantage 數據...");
    // 這裡可以換成真實的 API 請求
    return {
        date: new Date().toLocaleDateString(),
        hsi: "-0.8% (16500)",
        us_tech: "Nasdaq +0.2%",
        gold: "$2,300",
        oil: "$110"
    };
}

// --- 2. Gemini 生成腳本 ---
async function generateAIPost(marketData) {
    console.log("✨ [2/5] Gemini 正在寫稿...");
    const payload = {
        contents: [{ parts: [{ text: `市況：${JSON.stringify(marketData)}` }] }],
        systemInstruction: { parts: [{ text: "你是專業財經KOL，寫一段30秒辛辣短片腳本與IG貼文。格式：JSON。" }] },
        generationConfig: {
            responseMimeType: "application/json",
            responseSchema: {
                type: "OBJECT",
                properties: {
                    script: { type: "STRING" },
                    caption: { type: "STRING" },
                    headline: { type: "STRING" }
                }
            }
        }
    };
    const response = await fetchWithRetry(payload);
    return JSON.parse(response.data.candidates[0].content.parts[0].text);
}

// --- 3. 渲染影片 (Creatomate 範例) ---
async function renderVideo(aiContent) {
    console.log("🎬 [3/5] 正在雲端合成影片...");
    const res = await axios.post('https://api.creatomate.com/v1/renders', {
        template_id: '你的模板ID',
        modifications: {
            "Text.Headline": aiContent.headline,
            "Text.Script": aiContent.script
        }
    }, { headers: { 'Authorization': `Bearer ${process.env.VIDEO_RENDER_API_KEY}` } });
    return res.data[0].url; // 返回 MP4 網址
}

// --- 4. 發布至 IG ---
async function publishToInstagram(videoUrl, caption) {
    console.log("🚀 [4/5] 正在上傳至 Instagram...");
    const igId = process.env.IG_ACCOUNT_ID;
    const token = process.env.IG_ACCESS_TOKEN;
    
    // Step A: 建立 Container
    const container = await axios.post(`https://graph.facebook.com/v19.0/${igId}/media`, {
        media_type: 'REELS', video_url: videoUrl, caption: caption, access_token: token
    });
    
    console.log("⏳ 等待 IG 處理 (30s)...");
    await new Promise(r => setTimeout(r, 30000));

    // Step B: 正式發布
    await axios.post(`https://graph.facebook.com/v19.0/${igId}/media_publish`, {
        creation_id: container.data.id, access_token: token
    });
    console.log("✅ 完成！");
}

// --- 執行流程 ---
async function main() {
    try {
        const data = await fetchMarketData();
        const content = await generateAIPost(data);
        const videoUrl = await renderVideo(content);
        await publishToInstagram(videoUrl, content.caption);
    } catch (e) {
        console.error("失敗:", e.message);
    }
}

main();