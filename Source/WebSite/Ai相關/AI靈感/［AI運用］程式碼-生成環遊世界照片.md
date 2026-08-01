

import React, { useState, useRef, useEffect } from 'react';
import { 
  Upload, MapPin, Download, Globe, Sparkles, Image as ImageIcon, Copy, Check, Camera, Plane, User, Navigation,
  Star, Lightbulb, Wrench, Users, Mic, FileText, MessageCircle, Phone, Mail, ArrowRight, Play, Youtube
} from 'lucide-react';

// --- Types ---
interface LocationData {
  displayName: string;
  lat: number;
  lng: number;
  region?: string; 
}

interface UploadedImage {
  base64: string; 
  rawBase64: string; 
  mimeType: string;
  previewUrl: string;
}

interface GeneratedImageState {
  status: 'idle' | 'loading' | 'success' | 'error';
  imageUrl?: string;
  caption?: string;
  error?: string;
}

// --- Constants & Config ---
const KEYWORDS = [
  { label: "夕陽", value: "唯美的夕陽光線，金色時刻 (Golden Hour)" },
  { label: "美食", value: "享受當地特色美食，令人垂涎" },
  { label: "文青", value: "文藝氣息，膠卷風格，質感，淺景深，復古穿搭" },
  { label: "海灘", value: "陽光沙灘比基尼，度假放鬆，藍天白雲，海灘裝" },
  { label: "夜景", value: "繁華的城市夜景，霓虹燈光，Cyberpunk風格，時尚穿搭" },
  { label: "古蹟", value: "歷史悠久的建築氛圍，懷舊，宏偉，當地傳統服飾元素" },
  { label: "大自然", value: "壯麗的山川風景，清新自然，廣角鏡頭，戶外機能服飾" },
  { label: "奢華", value: "奢華高端的旅遊體驗，精緻，香檳，晚宴服裝" },
  { label: "探險", value: "背包客冒險風格，充滿活力，動態感，登山裝備" }
];

// --- Gemini API Service ---
const apiKey = ""; // API Key is injected by the environment

const stripBase64Prefix = (base64: string) => {
  return base64.replace(/^data:image\/(png|jpeg|jpg|webp);base64,/, '');
};

const generateInstagramCaption = async (location: string, style: string) => {
  try {
    const prompt = `
      請為一張在「${location}」拍攝的旅遊照片寫一篇 Instagram 貼文。
      
      風格要求：${style || '開心、放鬆'}
      語言：繁體中文 (Traditional Chinese)
      
      內容結構：
      1. 一句吸引人的開頭 (Hook)
      2. 2-3句關於當下的心情或景點描述
      3. 結尾互動問題
      4. 5-8個相關的 Hashtags
      
      請不要包含任何 "這是你的貼文" 之類的開場白，直接給出貼文內容。加入適當的 Emoji。
    `;

    const response = await fetch(
      `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          contents: [{ parts: [{ text: prompt }] }]
        })
      }
    );

    const data = await response.json();
    if (data.error) throw new Error(data.error.message);
    return data.candidates?.[0]?.content?.parts?.[0]?.text || "無法生成文案";
  } catch (error) {
    console.error("Caption Error:", error);
    return "AI 正在休息，暫時無法寫出文案，請稍後再試！";
  }
};

const generateTravelPhoto = async (image: UploadedImage, location: string, userPrompt: string) => {
  try {
    // Updated prompt to explicitly request clothing changes based on location and style
    const prompt = `
      Based on the reference image, create a realistic travel photograph of this person at ${location}.
      
      CRITICAL INSTRUCTION: Change the person's clothing and overall look to perfectly match the location, weather, and the specific style described in: "${userPrompt || 'casual travel outfit'}".
      
      For example, if the location is a beach, they should be wearing appropriate beachwear. If it's a formal landmark or "luxury" style is selected, they should be dressed accordingly.
      
      Maintain the person's facial identity and defining features from the reference image, but completely integrate them into the new scene with appropriate attire and lighting. High resolution, 4k, detailed travel photography.
    `;

    const response = await fetch(
      `https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-image-preview:generateContent?key=${apiKey}`,
      {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          contents: [{
            parts: [
              { text: prompt },
              {
                inlineData: {
                  mimeType: image.mimeType,
                  data: image.rawBase64
                }
              }
            ]
          }],
          generationConfig: {
            responseModalities: ["IMAGE"],
          }
        })
      }
    );

    const data = await response.json();
    
    if (data.error) {
       throw new Error(data.error.message || "生成失敗，可能是圖片觸發了安全過濾");
    }

    const base64Image = data.candidates?.[0]?.content?.parts?.find((p: any) => p.inlineData)?.inlineData?.data;
    
    if (!base64Image) throw new Error("AI 沒有回傳圖片，請換個描述試試看");
    
    return `data:image/png;base64,${base64Image}`;

  } catch (error: any) {
    console.error("Image Gen Error:", error);
    throw new Error(error.message || "圖片生成失敗");
  }
};


// --- Components ---

// 1. Real Leaflet Map Component
const InteractiveMap: React.FC<{ onSelect: (loc: LocationData) => void, selected: LocationData | null }> = ({ onSelect, selected }) => {
  const mapContainerRef = useRef<HTMLDivElement>(null);
  const mapInstanceRef = useRef<any>(null);
  const markerRef = useRef<any>(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const link = document.createElement('link');
    link.rel = 'stylesheet';
    link.href = 'https://unpkg.com/leaflet@1.9.4/dist/leaflet.css';
    document.head.appendChild(link);

    const script = document.createElement('script');
    script.src = 'https://unpkg.com/leaflet@1.9.4/dist/leaflet.js';
    script.async = true;
    document.body.appendChild(script);

    script.onload = () => {
      setLoading(false);
      initMap();
    };

    return () => {
      if (mapInstanceRef.current) {
        mapInstanceRef.current.remove();
        mapInstanceRef.current = null;
      }
      document.head.removeChild(link);
      document.body.removeChild(script);
    };
  }, []);

  const initMap = () => {
    if (!mapContainerRef.current || !(window as any).L) return;

    const L = (window as any).L;

    const map = L.map(mapContainerRef.current).setView([23.5, 121], 3);
    mapInstanceRef.current = map;

    L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
      attribution: '&copy; <a href="https://www.openstreetmap.org/copyright">OpenStreetMap</a> contributors'
    }).addTo(map);

    const customIcon = L.icon({
        iconUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-icon.png',
        shadowUrl: 'https://unpkg.com/leaflet@1.9.4/dist/images/marker-shadow.png',
        iconSize: [25, 41],
        iconAnchor: [12, 41],
        popupAnchor: [1, -34],
    });

    map.on('click', async (e: any) => {
      const { lat, lng } = e.latlng;

      if (markerRef.current) {
        markerRef.current.setLatLng([lat, lng]);
      } else {
        markerRef.current = L.marker([lat, lng], { icon: customIcon }).addTo(map);
      }

      try {
        const response = await fetch(`https://nominatim.openstreetmap.org/reverse?format=json&lat=${lat}&lon=${lng}&zoom=10&accept-language=zh-TW`);
        const data = await response.json();
        
        let displayName = "未知地點";
        if (data.address) {
            const city = data.address.city || data.address.town || data.address.county || data.address.state;
            const country = data.address.country;
            if (city && country) displayName = `${country}, ${city}`;
            else displayName = data.display_name.split(',')[0]; 
        }

        markerRef.current.bindPopup(`<b>${displayName}</b><br>已選擇`).openPopup();
        
        onSelect({
            displayName: displayName,
            lat: lat,
            lng: lng
        });

      } catch (err) {
        console.error("Geocoding failed", err);
        onSelect({
            displayName: `${lat.toFixed(2)}, ${lng.toFixed(2)}`,
            lat: lat,
            lng: lng
        });
      }
    });
  };

  return (
    <div className="relative w-full h-full bg-slate-100 rounded-xl overflow-hidden">
      <div id="map" ref={mapContainerRef} className="w-full h-full z-0" />
      
      {loading && (
        <div className="absolute inset-0 flex items-center justify-center bg-slate-50 z-10">
          <div className="flex flex-col items-center gap-2">
             <div className="w-6 h-6 border-2 border-blue-600 border-t-transparent rounded-full animate-spin"></div>
             <span className="text-xs text-slate-500">載入世界地圖中...</span>
          </div>
        </div>
      )}

      <div className="absolute bottom-2 left-2 bg-white/90 backdrop-blur px-2 py-1 rounded-md text-[10px] text-slate-500 z-[400] pointer-events-none shadow-sm border border-slate-200">
         使用 OpenStreetMap 圖資
      </div>
    </div>
  );
};

// 2. Updated Teacher Intro Component
const TeacherIntro: React.FC = () => {
  return (
    <div className="mt-16 bg-slate-50 rounded-[2.5rem] border border-slate-200 overflow-hidden">
        
        {/* Section Title */}
        <div className="text-center py-12 px-6">
             <h2 className="text-3xl font-bold text-slate-800 mb-8 inline-block relative">
                別讓您的企業錯過 AI 浪潮！
                <span className="absolute -bottom-2 left-0 w-full h-1 bg-blue-500 rounded-full"></span>
             </h2>
             
             <div className="max-w-4xl mx-auto">
                <div className="bg-white rounded-3xl p-8 md:p-12 shadow-sm border border-slate-100 text-center relative overflow-hidden group hover:shadow-md transition-shadow">
                    <div className="relative z-10">
                        <h1 className="text-3xl md:text-5xl font-extrabold text-slate-900 mb-6 bg-clip-text text-transparent bg-gradient-to-r from-slate-900 to-slate-700">
                            黃敬峰 企業AI培訓/陪跑專家
                        </h1>
                        <p className="text-xl text-slate-600 mb-8 font-medium">從策略到實戰，引領您的團隊高效轉型</p>
                        <a href="#contact" className="inline-flex items-center gap-2 px-8 py-4 bg-gradient-to-r from-blue-600 to-indigo-600 text-white rounded-full font-bold text-lg hover:shadow-lg hover:shadow-blue-500/30 transition-all transform hover:-translate-y-1">
                            立即聯繫，開啟 AI 轉型之旅 <ArrowRight size={20} />
                        </a>
                    </div>
                    {/* Background decoration */}
                    <div className="absolute top-0 right-0 w-64 h-64 bg-blue-50 rounded-full blur-3xl opacity-50 -mr-20 -mt-20"></div>
                    <div className="absolute bottom-0 left-0 w-48 h-48 bg-purple-50 rounded-full blur-3xl opacity-50 -ml-16 -mb-16"></div>
                </div>
             </div>
        </div>

        {/* Bio Section */}
        <div className="px-6 md:px-12 pb-16">
            <div className="text-center mb-12">
                <h2 className="text-2xl font-bold text-slate-800">認識您的數位轉型領航員</h2>
            </div>
            
            <div className="flex flex-col lg:flex-row items-center gap-12 max-w-6xl mx-auto">
                <div className="lg:w-5/12 text-center">
                    <div className="relative inline-block group">
                        <div className="absolute inset-0 bg-blue-600 rounded-full blur opacity-20 group-hover:opacity-30 transition-opacity"></div>
                        <img 
                            src="https://i.ibb.co/hFqZC7NM/DSC00301-3.jpg" 
                            className="relative w-64 h-64 md:w-80 md:h-80 object-cover rounded-full border-8 border-white shadow-xl" 
                            alt="黃敬峰老師照片" 
                            onError={(e) => e.currentTarget.src='https://placehold.co/300x300/CCCCCC/FFFFFF?text=阿峰老師'}
                        />
                    </div>
                    <h4 className="mt-6 text-2xl font-bold text-slate-800">黃敬峰 (阿峰老師)</h4>
                </div>
                
                <div className="lg:w-7/12">
                    <h3 className="text-xl font-bold text-slate-800 mb-6 flex items-center gap-2">
                        <Star className="text-yellow-500 fill-yellow-500" /> 經歷亮點
                    </h3>
                    <ul className="space-y-4">
                        {[
                            "企業 AI 知名培訓老師",
                            "輔導逾 400 間企業導入 AI 與數位工具",
                            "前行政院青年事務委員會副召集人",
                            "新北市青年事務委員",
                            "國立東華大學創新育成中心企業輔導顧問"
                        ].map((item, i) => (
                            <li key={i} className="flex items-start gap-3 p-4 bg-white rounded-xl shadow-sm border border-slate-100 hover:border-blue-200 transition-colors">
                                <div className="mt-1 min-w-[20px]"><Check size={20} className="text-blue-600" /></div>
                                <span className="text-lg text-slate-700 font-medium">{item}</span>
                            </li>
                        ))}
                    </ul>
                </div>
            </div>
        </div>

        {/* Features Section */}
        <div className="bg-white py-16 px-6 md:px-12 border-t border-slate-200">
             <div className="max-w-6xl mx-auto">
                <h2 className="text-2xl font-bold text-slate-800 text-center mb-12">阿峰老師授課特色</h2>
                <div className="grid md:grid-cols-3 gap-8">
                    {[
                        { icon: Lightbulb, title: "深入淺出，化繁為簡", desc: "將複雜的 AI 技術轉化為簡單易懂的實用知識，讓不同背景的學員都能輕鬆掌握。", color: "text-yellow-500 bg-yellow-50" },
                        { icon: Wrench, title: "實戰導向，即學即用", desc: "課程設計強調實用性，結合實際工作場景，透過大量實操演練，確保學員立即提升工作效率。", color: "text-blue-500 bg-blue-50" },
                        { icon: Users, title: "互動式教學，活潑生動", desc: "課堂氣氛輕鬆活潑，鼓勵提問與討論，透過小組活動與案例分析，提高學習參與度。", color: "text-emerald-500 bg-emerald-50" }
                    ].map((feature, i) => (
                        <div key={i} className="bg-slate-50 rounded-2xl p-8 text-center hover:bg-white hover:shadow-xl transition-all duration-300 border border-slate-100 group">
                            <div className={`w-16 h-16 rounded-2xl mx-auto flex items-center justify-center mb-6 ${feature.color} group-hover:scale-110 transition-transform`}>
                                <feature.icon size={32} />
                            </div>
                            <h5 className="text-xl font-bold text-slate-800 mb-4">{feature.title}</h5>
                            <p className="text-slate-600 leading-relaxed">{feature.desc}</p>
                        </div>
                    ))}
                </div>
             </div>
        </div>

        {/* Resources Section */}
        <div className="py-16 px-6 md:px-12 bg-slate-50">
             <div className="max-w-6xl mx-auto">
                <h2 className="text-2xl font-bold text-slate-800 text-center mb-12">更多 AI 學習資源</h2>
                <div className="grid md:grid-cols-3 gap-6">
                    <a href="https://line.me/ti/g2/o6oRaGIHTzZ1nEofxnT9Rbv7_ZHAX-rylbJfKA?utm_source=invitation&utm_medium=link_copy&utm_campaign=default" target="_blank" rel="noreferrer" className="block group">
                        <div className="bg-white rounded-2xl p-8 text-center h-full border border-slate-200 hover:border-green-500 hover:shadow-lg hover:shadow-green-500/10 transition-all">
                            <div className="w-14 h-14 rounded-full bg-green-50 text-green-600 mx-auto flex items-center justify-center mb-4 group-hover:bg-green-600 group-hover:text-white transition-colors">
                                <Users size={28} />
                            </div>
                            <h5 className="text-lg font-bold text-slate-800 mb-2">AI 學員社群</h5>
                            <p className="text-sm text-slate-500">加入 LINE 社群，與其他學員交流心得，獲取最新資訊。</p>
                        </div>
                    </a>
                    
                    <a href="https://podcasts.apple.com/us/podcast/%E4%B8%8B%E7%8F%AD%E5%AD%B8ai-ai%E5%A4%9C%E5%A4%9Ctalk/id1777590042" target="_blank" rel="noreferrer" className="block group">
                        <div className="bg-white rounded-2xl p-8 text-center h-full border border-slate-200 hover:border-purple-500 hover:shadow-lg hover:shadow-purple-500/10 transition-all">
                            <div className="w-14 h-14 rounded-full bg-purple-50 text-purple-600 mx-auto flex items-center justify-center mb-4 group-hover:bg-purple-600 group-hover:text-white transition-colors">
                                <Mic size={28} />
                            </div>
                            <h5 className="text-lg font-bold text-slate-800 mb-2">AI Podcast</h5>
                            <p className="text-sm text-slate-500">收聽「下班學AI」，利用通勤時間，輕鬆掌握 AI 新知。</p>
                        </div>
                    </a>

                    <a href="https://vocus.cc/salon/StartupForYou" target="_blank" rel="noreferrer" className="block group">
                        <div className="bg-white rounded-2xl p-8 text-center h-full border border-slate-200 hover:border-blue-500 hover:shadow-lg hover:shadow-blue-500/10 transition-all">
                            <div className="w-14 h-14 rounded-full bg-blue-50 text-blue-600 mx-auto flex items-center justify-center mb-4 group-hover:bg-blue-600 group-hover:text-white transition-colors">
                                <FileText size={28} />
                            </div>
                            <h5 className="text-lg font-bold text-slate-800 mb-2">AI 文章專欄</h5>
                            <p className="text-sm text-slate-500">閱讀方格子專欄，深度了解 AI 趨勢與實戰技巧。</p>
                        </div>
                    </a>
                </div>
             </div>
        </div>

        {/* Clients Section */}
        <div className="bg-white py-16 px-6 md:px-12 border-t border-slate-200">
             <div className="max-w-6xl mx-auto text-center">
                 <h2 className="text-2xl font-bold text-slate-800 mb-12">超過 400+ 企業與機構的信賴推薦</h2>
                 <div className="bg-slate-50 rounded-2xl p-8 border border-slate-100">
                     <div className="flex flex-wrap justify-center items-center gap-8 md:gap-12 opacity-80 grayscale hover:grayscale-0 transition-all duration-500">
                        {/* Logos Placeholder logic since actual URLs might break, styling them as uniform boxes if broken */}
                        {[
                            "https://i.ibb.co/msCc9JR/2025-05-25-10-45-21.png", "https://i.ibb.co/XZYkkqkN/2025-05-25-10-45-15.png",
                            "https://i.ibb.co/NgCknD5y/2025-05-25-10-45-10.png", "https://i.ibb.co/qMkdYMpT/2025-05-25-10-45-06.png",
                            "https://i.ibb.co/RpDhbqwv/2025-05-25-10-45-01.png", "https://i.ibb.co/YFzLftPm/2025-05-25-10-44-57.png",
                            "https://i.ibb.co/yFrHXv20/2025-05-25-10-44-52.png", "https://i.ibb.co/5xszGtxd/2025-05-25-10-44-48.png",
                            "https://i.ibb.co/mF1RLB1v/2025-05-25-10-44-43.png", "https://i.ibb.co/whj6F03m/2025-05-25-10-44-39.png",
                            "https://i.ibb.co/bjdHY2b4/2025-05-25-10-44-34.png", "https://i.ibb.co/fYQYTbmC/2025-05-25-10-44-29.png"
                        ].map((src, i) => (
                            <img key={i} src={src} alt="Client Logo" className="h-12 md:h-16 object-contain hover:scale-110 transition-transform" onError={(e) => {e.currentTarget.style.display='none'}} />
                        ))}
                     </div>
                 </div>
             </div>
        </div>

        {/* Moments Section */}
        <div className="py-16 px-6 md:px-12 bg-slate-50">
            <div className="max-w-6xl mx-auto">
                <h2 className="text-2xl font-bold text-slate-800 text-center mb-12">精彩瞬間</h2>
                <div className="grid md:grid-cols-2 gap-8">
                    {[
                        { img: "https://i.ibb.co/HfjhD0GR/A7C00941.jpg", title: "台灣生物科技公司", text: "為中高階主管帶來AI策略課程，深入探討產業應用與未來趨勢。" },
                        { img: "https://i.ibb.co/C3KVdVkb/IMG-2831.jpg", title: "馬來西亞大學 MBA 課程", text: "分享AI商業應用實戰經驗，啟發學員的創新思維與國際視野。" }
                    ].map((item, i) => (
                        <div key={i} className="bg-white rounded-2xl overflow-hidden shadow-sm border border-slate-200 hover:shadow-lg transition-all">
                            <div className="aspect-video relative overflow-hidden">
                                <img src={item.img} alt={item.title} className="w-full h-full object-cover hover:scale-105 transition-transform duration-700" onError={(e) => e.currentTarget.src='https://placehold.co/600x400/EEEEEE/AAAAAA?text=教學照片'} />
                            </div>
                            <div className="p-6">
                                <h5 className="text-xl font-bold text-slate-800 mb-2">{item.title}</h5>
                                <p className="text-slate-600">{item.text}</p>
                            </div>
                        </div>
                    ))}
                </div>
            </div>
        </div>

        {/* Testimonials Images */}
        <div className="bg-white py-16 px-6 md:px-12 border-t border-slate-200">
            <div className="max-w-6xl mx-auto">
                <h2 className="text-2xl font-bold text-slate-800 text-center mb-12">聽聽他們怎麼說</h2>
                <div className="grid md:grid-cols-3 gap-6">
                    {["https://i.ibb.co/mVXNmGBk/2025-05-25-10-46-41.png", "https://i.ibb.co/pj7TLPnf/2025-05-25-10-46-46.png", "https://i.ibb.co/HpqhfxQN/2025-05-25-10-46-51.png"].map((src, i) => (
                        <div key={i} className="bg-slate-50 rounded-2xl p-2 border border-slate-200 shadow-sm hover:shadow-md transition-shadow">
                            <img src={src} alt="Testimonial" className="w-full h-auto rounded-xl" onError={(e) => e.currentTarget.src='https://placehold.co/400x300/EEEEEE/AAAAAA?text=學員見證'} />
                        </div>
                    ))}
                </div>
            </div>
        </div>

        {/* Videos Section */}
        <div className="py-16 px-6 md:px-12 bg-slate-50">
            <div className="max-w-6xl mx-auto text-center">
                <h2 className="text-2xl font-bold text-slate-800 mb-12">學員與課程見證</h2>
                
                <h3 className="text-lg font-semibold text-slate-600 mb-6 flex items-center justify-center gap-2">
                    <Youtube className="text-red-600" /> 學員見證影片
                </h3>
                <div className="grid md:grid-cols-3 gap-6 mb-12">
                     {["FBIRyZjBLTE", "X3nILy__5Uo", "7SNi9iGwuI0"].map((id, i) => (
                         <div key={i} className="aspect-video bg-black rounded-xl overflow-hidden shadow-lg relative group">
                             <iframe 
                                src={`https://www.youtube.com/embed/${id}`} 
                                title="YouTube video player" 
                                className="w-full h-full"
                                allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" 
                                allowFullScreen
                             ></iframe>
                         </div>
                     ))}
                </div>

                <h3 className="text-lg font-semibold text-slate-600 mb-6 flex items-center justify-center gap-2">
                    <Youtube className="text-red-600" /> 課程見證影片
                </h3>
                <div className="grid md:grid-cols-2 gap-6 max-w-4xl mx-auto">
                     {["ZiJ7qPxjPes", "Eg4rLnLGn8Q"].map((id, i) => (
                         <div key={i} className="aspect-video bg-black rounded-xl overflow-hidden shadow-lg">
                             <iframe 
                                src={`https://www.youtube.com/embed/${id}`} 
                                title="YouTube video player" 
                                className="w-full h-full"
                                allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" 
                                allowFullScreen
                             ></iframe>
                         </div>
                     ))}
                </div>
            </div>
        </div>

        {/* Contact Section */}
        <div id="contact" className="py-16 px-6 md:px-12 bg-white border-t border-slate-200">
            <div className="max-w-4xl mx-auto text-center">
                 <h2 className="text-3xl font-bold text-slate-800 mb-4">立即聯繫，開啟 AI 轉型之旅</h2>
                 <p className="text-xl text-slate-600 mb-12">讓阿峰老師為您量身打造最適合的 AI 導入方案</p>
                 
                 <div className="grid md:grid-cols-3 gap-8">
                     <a href="https://line.me/ti/g2/o6oRaGIHTzZ1nEofxnT9Rbv7_ZHAX-rylbJfKA?utm_source=invitation&utm_medium=link_copy&utm_campaign=default" target="_blank" rel="noreferrer" className="flex flex-col items-center gap-4 p-6 rounded-2xl hover:bg-slate-50 transition-colors group">
                         <div className="w-16 h-16 bg-[#06C755] text-white rounded-full flex items-center justify-center text-3xl shadow-lg group-hover:scale-110 transition-transform">
                             <MessageCircle size={32} fill="white" />
                         </div>
                         <div className="text-slate-800 font-bold text-lg">LINE 社群</div>
                     </a>

                     <a href="tel:0976715102" className="flex flex-col items-center gap-4 p-6 rounded-2xl hover:bg-slate-50 transition-colors group">
                         <div className="w-16 h-16 bg-blue-600 text-white rounded-full flex items-center justify-center text-3xl shadow-lg group-hover:scale-110 transition-transform">
                             <Phone size={32} fill="white" />
                         </div>
                         <div className="text-slate-800 font-bold text-lg">0976-715-102</div>
                     </a>

                     <a href="mailto:nikeshoxmiles@gmail.com" className="flex flex-col items-center gap-4 p-6 rounded-2xl hover:bg-slate-50 transition-colors group">
                         <div className="w-16 h-16 bg-red-500 text-white rounded-full flex items-center justify-center text-3xl shadow-lg group-hover:scale-110 transition-transform">
                             <Mail size={32} />
                         </div>
                         <div className="text-slate-800 font-bold text-lg">寄送 Email</div>
                     </a>
                 </div>
            </div>
        </div>

    </div>
  );
};

// --- Main App Component ---
const App: React.FC = () => {
  const [uploadedImage, setUploadedImage] = useState<UploadedImage | null>(null);
  const [selectedLocation, setSelectedLocation] = useState<LocationData | null>(null);
  const [generationState, setGenerationState] = useState<GeneratedImageState>({ status: 'idle' });
  const [customPrompt, setCustomPrompt] = useState('');
  const [copied, setCopied] = useState(false);
  
  const fileInputRef = useRef<HTMLInputElement>(null);

  const handleFileChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0];
    if (!file) return;

    if (file.size > 5 * 1024 * 1024) {
      alert("檔案太大囉！請上傳小於 5MB 的照片。");
      return;
    }

    const reader = new FileReader();
    reader.onload = (e) => {
      const result = e.target?.result as string;
      setUploadedImage({
        base64: result,
        rawBase64: stripBase64Prefix(result),
        mimeType: file.type,
        previewUrl: result,
      });
    };
    reader.readAsDataURL(file);
  };

  const handleKeywordClick = (keywordValue: string) => {
    if (customPrompt.includes(keywordValue)) return;
    const newPrompt = customPrompt ? `${customPrompt}，${keywordValue}` : keywordValue;
    setCustomPrompt(newPrompt);
  };

  const handleGenerate = async () => {
    if (!uploadedImage || !selectedLocation) return;

    setGenerationState({ status: 'loading' });
    try {
      // Parallel execution for speed
      const [imageUrl, caption] = await Promise.all([
        generateTravelPhoto(uploadedImage, selectedLocation.displayName, customPrompt),
        generateInstagramCaption(selectedLocation.displayName, customPrompt)
      ]);

      setGenerationState({ 
        status: 'success', 
        imageUrl: imageUrl,
        caption: caption
      });
    } catch (error: any) {
      setGenerationState({ status: 'error', error: error.message });
    }
  };

  const handleDownload = () => {
    if (generationState.imageUrl) {
      const link = document.createElement('a');
      link.href = generationState.imageUrl;
      link.download = `wanderlust-feng-${Date.now()}.png`;
      document.body.appendChild(link);
      link.click();
      document.body.removeChild(link);
    }
  };

  const handleCopyCaption = () => {
    if (generationState.caption) {
      // Using modern clipboard API with fallback
      if (navigator.clipboard && navigator.clipboard.writeText) {
         navigator.clipboard.writeText(generationState.caption);
      } else {
         const textArea = document.createElement("textarea");
         textArea.value = generationState.caption;
         document.body.appendChild(textArea);
         textArea.select();
         document.execCommand('copy');
         document.body.removeChild(textArea);
      }
      setCopied(true);
      setTimeout(() => setCopied(false), 2000);
    }
  };

  return (
    <div className="min-h-screen flex flex-col font-sans text-slate-800 bg-slate-50">
      {/* Header */}
      <header className="bg-white border-b border-slate-200 sticky top-0 z-50">
        <div className="max-w-[1600px] mx-auto px-4 sm:px-6 lg:px-8 h-16 flex items-center justify-between">
          <div className="flex items-center gap-2">
            <div className="w-8 h-8 bg-blue-600 rounded-lg flex items-center justify-center shadow-blue-200 shadow-md">
              <Plane className="w-5 h-5 text-white transform -rotate-45" />
            </div>
            <h1 className="text-xl font-bold bg-gradient-to-r from-blue-600 to-indigo-600 bg-clip-text text-transparent">
              阿峰老師帶你環遊世界AI
            </h1>
          </div>
          <div className="flex items-center gap-4">
             <span className="text-xs font-semibold px-2 py-1 bg-blue-50 text-blue-600 rounded-md border border-blue-100 hidden sm:block">
               Gemini 2.5 Flash 驅動
             </span>
             <a href="#contact" className="w-8 h-8 rounded-full bg-slate-100 flex items-center justify-center hover:bg-slate-200 transition-colors text-slate-600">
                <User size={16} />
             </a>
          </div>
        </div>
      </header>

      <main className="flex-1 max-w-[1600px] mx-auto w-full px-4 sm:px-6 lg:px-8 py-6">
        <div className="grid grid-cols-1 lg:grid-cols-12 gap-6 lg:gap-8 h-full mb-16">
          
          {/* Left Column: Inputs */}
          <div className="lg:col-span-5 flex flex-col gap-6">
            
            {/* Step 1: Upload */}
            <div className="bg-white p-5 rounded-2xl shadow-sm border border-slate-100 transition-all hover:shadow-md">
              <div className="flex items-center gap-2 mb-4">
                <div className="w-6 h-6 rounded-full bg-blue-100 text-blue-600 flex items-center justify-center text-sm font-bold">1</div>
                <h2 className="font-semibold text-lg">上傳您的照片</h2>
              </div>
              
              <div 
                className={`
                  relative border-2 border-dashed rounded-xl p-4 transition-all text-center cursor-pointer group min-h-[160px] flex flex-col justify-center
                  ${uploadedImage ? 'border-blue-500 bg-blue-50/30' : 'border-slate-300 hover:border-blue-400 hover:bg-slate-50'}
                `}
                onClick={() => fileInputRef.current?.click()}
              >
                <input 
                  type="file" 
                  ref={fileInputRef}
                  className="hidden" 
                  accept="image/*" 
                  onChange={handleFileChange} 
                />
                
                {uploadedImage ? (
                  <div className="flex items-center justify-center gap-4">
                    <div className="relative w-24 h-24 rounded-lg overflow-hidden shadow-sm shrink-0 border border-white">
                      <img 
                        src={uploadedImage.previewUrl} 
                        alt="Uploaded preview" 
                        className="w-full h-full object-cover"
                      />
                    </div>
                    <div className="text-left">
                        <p className="text-sm font-medium text-slate-700">照片已上傳</p>
                        <p className="text-xs text-slate-500 mb-2">建議使用半身照效果最佳</p>
                        <span className="inline-flex items-center gap-1 text-xs bg-emerald-100 text-emerald-700 px-2 py-1 rounded-full font-medium">
                           <Check size={10} /> 準備就緒
                        </span>
                    </div>
                  </div>
                ) : (
                  <div className="flex flex-col items-center justify-center gap-2 py-4">
                    <div className="w-12 h-12 rounded-full bg-blue-100 text-blue-500 flex items-center justify-center mb-1 group-hover:scale-110 transition-transform shadow-sm">
                      <Camera size={20} />
                    </div>
                    <div>
                        <p className="font-medium text-slate-700">點擊上傳自拍照</p>
                        <p className="text-xs text-slate-400 mt-1">支援 JPG, PNG (Max 5MB)</p>
                    </div>
                  </div>
                )}
              </div>
            </div>

            {/* Step 2: Map & Prompt */}
            <div className="bg-white p-5 rounded-2xl shadow-sm border border-slate-100 flex-1 flex flex-col transition-all hover:shadow-md">
              <div className="flex items-center justify-between mb-3">
                <div className="flex items-center gap-2">
                  <div className="w-6 h-6 rounded-full bg-blue-100 text-blue-600 flex items-center justify-center text-sm font-bold">2</div>
                  <h2 className="font-semibold text-lg">選擇目的地</h2>
                </div>
                {selectedLocation && (
                  <span className="text-xs font-bold px-3 py-1 bg-emerald-100 text-emerald-700 rounded-full flex items-center gap-1 max-w-[150px] truncate animate-fade-in border border-emerald-200 shadow-sm">
                    <MapPin size={12} className="fill-emerald-700 text-emerald-700" /> {selectedLocation.displayName}
                  </span>
                )}
              </div>
              
              {/* Map Container */}
              <div className="h-[350px] w-full rounded-xl overflow-hidden border border-slate-200 relative z-0 shadow-inner bg-slate-100 mb-5">
                <InteractiveMap 
                  onSelect={setSelectedLocation} 
                  selected={selectedLocation} 
                />
              </div>

              <div className="space-y-4">
                 <div>
                    <label className="block text-sm font-medium text-slate-700 mb-1">您想去哪裡？ (點擊地圖自動填入)</label>
                    <div className="relative">
                        <input 
                        type="text"
                        value={selectedLocation?.displayName || ''}
                        onChange={(e) => setSelectedLocation(prev => prev ? { ...prev, displayName: e.target.value } : { displayName: e.target.value, lat: 0, lng: 0 })}
                        placeholder="請點擊上方地圖選擇地點..."
                        className="w-full pl-10 pr-4 py-3 border border-slate-300 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-blue-500 transition-shadow text-sm shadow-sm"
                        />
                        <Globe className="absolute left-3 top-3.5 text-slate-400 w-4 h-4" />
                    </div>
                 </div>

                <div>
                    <label className="block text-sm font-medium text-slate-700 mb-2">風格 / 關鍵字 (AI 參考用)</label>
                    
                    {/* Keywords Chips */}
                    <div className="flex flex-wrap gap-2 mb-3">
                    {KEYWORDS.map((kw) => (
                        <button
                        key={kw.label}
                        onClick={() => handleKeywordClick(kw.value)}
                        className="px-3 py-1.5 bg-slate-50 hover:bg-blue-50 text-slate-600 hover:text-blue-600 border border-slate-200 hover:border-blue-300 rounded-lg text-xs font-medium transition-all active:scale-95 shadow-sm"
                        >
                        {kw.label}
                        </button>
                    ))}
                    </div>

                    <textarea 
                        value={customPrompt}
                        onChange={(e) => setCustomPrompt(e.target.value)}
                        placeholder="描述您想要的畫面感覺，例如: 穿著和服，手中拿著抹茶冰淇淋..."
                        className="w-full px-4 py-3 border border-slate-300 rounded-xl focus:ring-2 focus:ring-blue-500 focus:border-blue-500 transition-shadow text-sm resize-none h-24 shadow-sm"
                    />
                </div>
              </div>
            </div>
          </div>

          {/* Right Column: Action & Result */}
          <div className="lg:col-span-7 flex flex-col gap-6">
             
             {/* Action Button */}
             <div className="flex justify-center lg:justify-start">
                <button
                  onClick={handleGenerate}
                  disabled={!uploadedImage || !selectedLocation || generationState.status === 'loading'}
                  className={`
                    w-full py-4 rounded-2xl font-bold text-xl shadow-xl flex items-center justify-center gap-3 transition-all transform border border-white/20
                    ${!uploadedImage || !selectedLocation 
                      ? 'bg-slate-200 text-slate-400 cursor-not-allowed shadow-none' 
                      : 'bg-gradient-to-r from-blue-600 via-indigo-600 to-purple-600 text-white hover:shadow-blue-500/40 hover:-translate-y-1 active:translate-y-0'}
                  `}
                >
                  {generationState.status === 'loading' ? (
                    <>
                      <div className="w-6 h-6 border-3 border-white/30 border-t-white rounded-full animate-spin" />
                      正在為您訂機票... (生成中)
                    </>
                  ) : (
                    <>
                      <Sparkles className="w-6 h-6" />
                      開始生成旅遊照 & IG 文案
                    </>
                  )}
                </button>
             </div>

             {/* Result Area */}
             <div className={`
                flex-1 bg-white rounded-3xl shadow-sm border border-slate-100 p-2 relative overflow-hidden flex flex-col min-h-[600px] transition-all duration-500
                ${generationState.status === 'success' ? 'ring-4 ring-blue-500/10 shadow-xl' : ''}
             `}>
                
                {generationState.status === 'idle' && (
                  <div className="flex-1 flex flex-col items-center justify-center text-slate-400 gap-6 p-10 text-center bg-slate-50/50 rounded-2xl border border-dashed border-slate-200 m-2">
                    <div className="w-32 h-32 rounded-full bg-white flex items-center justify-center relative shadow-sm">
                       <ImageIcon size={48} className="text-slate-200 absolute" />
                       <div className="absolute -right-2 -bottom-2 w-12 h-12 bg-blue-100 rounded-full flex items-center justify-center text-blue-500 ring-4 ring-slate-50">
                          <MapPin size={24} className="fill-blue-500 text-blue-500" />
                       </div>
                    </div>
                    <div>
                        <h3 className="text-xl font-bold text-slate-600 mb-2">準備好出發了嗎？</h3>
                        <p className="text-slate-500 max-w-md mx-auto">
                            只要上傳一張照片並點擊地圖，阿峰老師的 AI 就能帶您瞬間移動到世界各地！
                        </p>
                    </div>
                  </div>
                )}

                {generationState.status === 'loading' && (
                  <div className="flex-1 flex flex-col items-center justify-center gap-8 animate-pulse p-10 bg-slate-50/50 rounded-2xl m-2">
                     <div className="relative w-40 h-40">
                        <div className="absolute inset-0 rounded-full border-4 border-slate-200"></div>
                        <div className="absolute inset-0 rounded-full border-4 border-blue-500 border-t-transparent animate-spin"></div>
                        <Plane className="absolute inset-0 m-auto text-blue-500 w-16 h-16 animate-bounce" />
                     </div>
                     <div className="text-center space-y-3">
                        <p className="text-2xl font-bold text-slate-700">正在創造美好回憶...</p>
                        <p className="text-slate-500">
                          正在融合當地光影與氛圍<br/>
                          並為您構思 {selectedLocation?.displayName} 的精彩文案
                        </p>
                     </div>
                  </div>
                )}

                {generationState.status === 'error' && (
                   <div className="flex-1 flex flex-col items-center justify-center text-red-500 gap-6 p-12 text-center bg-red-50/30 rounded-2xl m-2">
                     <div className="w-20 h-20 bg-white rounded-full flex items-center justify-center shadow-sm text-4xl ring-4 ring-red-50">
                        ⚠️
                     </div>
                     <div>
                        <h3 className="text-2xl font-bold text-slate-800 mb-2">哎呀！出了一點小狀況</h3>
                        <p className="text-slate-600 mb-6 max-w-md mx-auto">{generationState.error}</p>
                        <button 
                        onClick={() => setGenerationState({ status: 'idle' })}
                        className="px-8 py-3 bg-white border border-slate-200 text-slate-700 rounded-xl hover:bg-slate-50 font-bold transition-colors shadow-sm"
                        >
                        再試一次
                        </button>
                     </div>
                   </div>
                )}

                {generationState.status === 'success' && generationState.imageUrl && (
                  <div className="flex flex-col h-full animate-fade-in">
                    {/* Image Section */}
                    <div className="flex-1 relative group bg-slate-900 overflow-hidden rounded-t-2xl min-h-[500px]">
                         <img 
                          src={generationState.imageUrl} 
                          alt="Generated Travel Photo" 
                          className="w-full h-full object-contain bg-black/40 backdrop-blur-xl"
                        />
                        <div className="absolute bottom-0 left-0 right-0 p-6 bg-gradient-to-t from-black/80 via-black/40 to-transparent opacity-0 group-hover:opacity-100 transition-opacity flex justify-between items-end">
                            <div className="text-white/80 text-xs font-mono">
                                Generated by Gemini 2.5
                            </div>
                            <button 
                              onClick={handleDownload}
                              className="flex items-center gap-2 px-6 py-3 bg-white text-slate-900 rounded-full hover:bg-blue-50 transition-all font-bold shadow-lg transform active:scale-95"
                            >
                              <Download size={18} />
                              下載高清大圖
                            </button>
                        </div>
                    </div>

                    {/* Caption Section */}
                    {generationState.caption && (
                      <div className="bg-white border-t border-slate-200 flex flex-col lg:flex-row shadow-[0_-10px_40px_-15px_rgba(0,0,0,0.1)] relative z-10">
                        <div className="flex-1 p-6">
                            <h3 className="font-bold text-slate-800 flex items-center gap-2 mb-3">
                                <div className="w-8 h-8 rounded-full bg-gradient-to-tr from-yellow-400 via-red-500 to-purple-500 p-[2px]">
                                    <div className="w-full h-full bg-white rounded-full flex items-center justify-center">
                                        <span className="text-[10px] font-bold">IG</span>
                                    </div>
                                </div>
                                您的專屬貼文
                            </h3>
                            <div className="p-4 bg-slate-50 rounded-xl text-slate-700 leading-relaxed whitespace-pre-wrap font-medium border border-slate-100 max-h-[200px] overflow-y-auto custom-scrollbar text-sm">
                                {generationState.caption}
                            </div>
                        </div>
                        <div className="p-6 border-t lg:border-t-0 lg:border-l border-slate-200 flex items-center justify-center bg-slate-50 lg:w-[220px]">
                          <button
                            onClick={handleCopyCaption}
                            className={`
                              w-full py-4 rounded-xl flex items-center justify-center gap-2 text-sm font-bold transition-all shadow-sm
                              ${copied 
                                ? 'bg-green-500 text-white border-green-600' 
                                : 'bg-white border border-slate-300 text-slate-700 hover:bg-white hover:border-blue-400 hover:text-blue-600'}
                            `}
                          >
                            {copied ? (
                              <>
                                <Check size={18} /> 已複製到剪貼簿
                              </>
                            ) : (
                              <>
                                <Copy size={18} /> 複製貼文內容
                              </>
                            )}
                          </button>
                        </div>
                      </div>
                    )}
                  </div>
                )}
             </div>
          </div>
        </div>
        
        {/* Updated Teacher Introduction Section */}
        <TeacherIntro />
      </main>
    </div>
  );
};

export default App;










2025-11-24
09:50

#ai點子 
#ai靈感 
#ai運用 