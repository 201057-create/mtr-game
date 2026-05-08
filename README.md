<!DOCTYPE html>
<html lang="zh-HK">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>MTR 3D 終極駕駛模擬</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
    <style>
        body { margin: 0; overflow: hidden; background-color: #000; font-family: '-apple-system', sans-serif; user-select: none; touch-action: none; }
        
        /* 遊戲容器，接收拖曳事件 */
        #game-container { position: absolute; top: 0; left: 0; width: 100vw; height: 100vh; cursor: grab; }
        #game-container:active { cursor: grabbing; }

        /* ======= 畫面共用設定 ======= */
        .fullscreen-menu {
            position: fixed; top: 0; left: 0; width: 100vw; height: 100vh;
            background: rgba(10, 10, 15, 0.8); backdrop-filter: blur(10px);
            z-index: 9999; display: flex; flex-direction: column;
            justify-content: center; align-items: center; color: white; transition: opacity 0.5s ease;
        }
        .menu-content {
            text-align: center; max-width: 950px; padding: 40px; 
            background: rgba(255, 255, 255, 0.05); border-radius: 20px; 
            border: 1px solid rgba(255,255,255,0.15); box-shadow: 0 20px 50px rgba(0,0,0,0.6);
        }
        .menu-title { font-size: 50px; margin: 0 0 20px 0; letter-spacing: 2px; text-shadow: 2px 2px 5px #000; }
        .menu-title span { color: #ed1d24; transition: color 0.3s; }
        .start-btn {
            background: linear-gradient(135deg, #ed1d24, #b71c1c); color: white;
            font-size: 28px; font-weight: bold; padding: 15px 60px; border: 2px solid rgba(255,255,255,0.5);
            border-radius: 40px; cursor: pointer; box-shadow: 0 8px 25px rgba(0,0,0,0.5);
            transition: all 0.3s; margin-top: 20px;
        }
        .start-btn:hover { transform: scale(1.05); filter: brightness(1.2); }
        .start-btn:active { transform: scale(0.95); }
        .menu-footer { position: absolute; bottom: 20px; right: 20px; color: #777; font-size: 14px; }

        /* ======= 第一層：歡迎畫面 ======= */
        #welcome-screen { opacity: 1; }
        .welcome-subtitle { font-size: 24px; color: #aaa; margin-bottom: 40px; letter-spacing: 5px; }

        /* ======= 第二層：路綫選擇 ======= */
        #route-select-screen { display: none; opacity: 0; }
        
        .route-selector { display: flex; justify-content: center; flex-wrap: wrap; gap: 10px; margin-bottom: 20px; max-height: 50vh; overflow-y: auto; padding: 10px;}
        .route-btn { 
            background: rgba(0,0,0,0.5); border: 2px solid #555; color: #aaa; 
            padding: 12px 25px; font-size: 20px; font-weight: bold; border-radius: 30px; 
            cursor: pointer; transition: all 0.3s; 
        }
        .route-btn:hover { background: rgba(255,255,255,0.1); }
        .route-btn.active { color: #fff; }
        .route-btn.active.twl { border-color: #ed1d24; background: rgba(237,29,36,0.3); box-shadow: 0 0 20px rgba(237,29,36,0.6); }
        .route-btn.active.isl { border-color: #0071CE; background: rgba(0,113,206,0.3); box-shadow: 0 0 20px rgba(0,113,206,0.6); }
        .route-btn.active.ktl { border-color: #00A859; background: rgba(0,168,89,0.3); box-shadow: 0 0 20px rgba(0,168,89,0.6); }
        .route-btn.active.tkl { border-color: #8A2BE2; background: rgba(138,43,226,0.3); box-shadow: 0 0 20px rgba(138,43,226,0.6); } 
        .route-btn.active.eal { border-color: #53B7E8; background: rgba(83,183,232,0.3); box-shadow: 0 0 20px rgba(83,183,232,0.6); } 
        .route-btn.active.sil { border-color: #b2bb1c; background: rgba(178,187,28,0.3); box-shadow: 0 0 20px rgba(178,187,28,0.6); } 
        .route-btn.active.tcl { border-color: #F68F26; background: rgba(246,143,38,0.3); box-shadow: 0 0 20px rgba(246,143,38,0.6); } 
        .route-btn.active.ael { border-color: #008C95; background: rgba(0,140,149,0.3); box-shadow: 0 0 20px rgba(0,140,149,0.6); } 
        .route-btn.active.tml { border-color: #9A3820; background: rgba(154,56,32,0.3); box-shadow: 0 0 20px rgba(154,56,32,0.6); } 
        .route-btn.active.drl { border-color: #FF75B5; background: rgba(255,117,181,0.3); box-shadow: 0 0 20px rgba(255,117,181,0.6); } 
        
        .route-info-title { font-size: 24px; color: #ccc; margin: 0 0 30px 0; transition: color 0.3s; }

        /* ======= 遊戲 UI ======= */
        #ui-layer { opacity: 0; transition: opacity 0.5s ease; pointer-events: none; }
        #ui-layer.visible { opacity: 1; pointer-events: auto; }

        #hud {
            position: absolute; top: 20px; left: 20px;
            background: rgba(0, 0, 0, 0.8); padding: 15px 25px; border-radius: 12px;
            border-left: 5px solid #ed1d24; color: white; pointer-events: none;
            box-shadow: 0 4px 10px rgba(0,0,0,0.5); z-index: 20; transition: border-color 0.3s;
        }
        .hud-text { font-size: 22px; font-weight: bold; margin: 8px 0; }
        .highlight { color: #00ff00; font-family: monospace; font-size: 26px;}
        .warning { color: #ffaa00; }
        .danger { color: #ff4444; }
        .station-name { font-size: 24px; font-weight: 900; text-shadow: 1px 1px 2px black; }
        
        #btn-camera {
            position: absolute; top: 20px; right: 20px; background: rgba(255, 255, 255, 0.2); 
            border: 2px solid white; color: white; padding: 10px 20px; border-radius: 20px; 
            font-size: 18px; font-weight: bold; cursor: pointer; z-index: 10; backdrop-filter: blur(5px); transition: 0.3s;
        }
        #btn-camera.manual-mode { border-color: #ffaa00; color: #ffaa00; }
        #btn-camera.driver-mode { border-color: #00ff00; color: #00ff00; }
        #btn-camera.passenger-mode { border-color: #42a5f5; color: #42a5f5; }

        #message-box {
            position: absolute; top: 20px; left: 50%; transform: translateX(-50%);
            font-size: 20px; font-weight: bold; color: white;
            background: rgba(0, 0, 0, 0.85); padding: 12px 30px; border-radius: 30px;
            border: 2px solid rgba(255,255,255,0.2); box-shadow: 0 4px 15px rgba(0,0,0,0.5);
            text-align: center; pointer-events: none; z-index: 20; white-space: pre-wrap; transition: opacity 0.3s;
        }
        #message-box:empty { display: none; }

        #hologram-box {
            position: absolute; top: 40%; left: 50%; transform: translate(-50%, -50%);
            text-align: center; pointer-events: none; z-index: 10;
        }

        #door-controls {
            display: none; position: absolute; left: 30px; bottom: 30px;
            flex-direction: row; gap: 15px; z-index: 50;
        }
        .door-btn {
            width: 75px; height: 75px; border-radius: 50%; border: 3px solid rgba(255,255,255,0.4);
            cursor: pointer; display: flex; justify-content: center; align-items: center;
            box-shadow: 0 6px 15px rgba(0,0,0,0.6); -webkit-tap-highlight-color: transparent; transition: transform 0.1s;
        }
        .door-btn:active { transform: scale(0.9); }
        .door-btn svg { width: 38px; height: 38px; }
        #btn-door-open { background: linear-gradient(135deg, #42a5f5, #1565c0); color: white; }
        #btn-door-close { background: linear-gradient(135deg, #ffa726, #e65100); color: white; }

        #btn-next-station {
            position: absolute; top: 50%; left: 50%; transform: translate(-50%, -50%);
            padding: 15px 40px; font-size: 24px; font-weight: bold; background-color: #555;
            color: white; border: 3px solid white; border-radius: 30px; cursor: pointer; display: none; z-index: 100;
        }

        #reverser-controls {
            position: absolute; right: 170px; bottom: 30px; z-index: 20; display: flex; flex-direction: column; gap: 15px;
        }
        .dir-btn {
            background: rgba(0,0,0,0.7); color: #777; border: 2px solid #555;
            padding: 12px 20px; border-radius: 12px; font-size: 20px; font-weight: bold;
            cursor: pointer; transition: all 0.2s ease;
        }
        .dir-btn.active { background: rgba(0, 255, 0, 0.2); color: #00ff00; border-color: #00ff00; box-shadow: 0 0 10px rgba(0,255,0,0.5); }
        .dir-btn.active.rev { background: rgba(255, 170, 0, 0.2); color: #ffaa00; border-color: #ffaa00; box-shadow: 0 0 10px rgba(255,170,0,0.5); }

        #lever-container { position: absolute; right: 20px; bottom: 30px; width: 140px; height: 180px; z-index: 20; }
        #lever-track {
            position: absolute; right: 20px; top: 10px; width: 60px; height: 160px;
            background: linear-gradient(to right, #222, #444, #222); border-radius: 30px; border: 3px solid #111; box-shadow: 0 0 10px rgba(0,0,0,0.8) inset;
        }
        #lever-handle {
            width: 90px; height: 50px; background: linear-gradient(to bottom, #f0f0f0, #aaa);
            border-radius: 10px; position: absolute; left: -15px; top: 55px; cursor: grab;
            box-shadow: 0 6px 15px rgba(0,0,0,0.6), inset 0 2px 2px rgba(255,255,255,0.8);
            border: 2px solid #555; touch-action: none; display: flex; justify-content: center; align-items: center; z-index: 2;
        }
        #lever-handle:active { cursor: grabbing; }
        #lever-handle::after { content: '|||'; color: #555; font-weight: bold; letter-spacing: 2px; }
        .lever-label { position: absolute; right: 80px; font-weight: bold; font-size: 20px; text-shadow: 1px 1px 2px #000; pointer-events: none; text-align: right; }
        .lever-label.top { top: 10px; color: #f44336; }
        .lever-label.center { top: 75px; color: #ccc; }
        .lever-label.bottom { bottom: 15px; color: #4CAF50; }
    </style>
</head>
<body>

    <div id="game-container"></div>

    <!-- 第一層：歡迎畫面 -->
    <div id="welcome-screen" class="fullscreen-menu">
        <div class="menu-content">
            <h1 class="menu-title"><span>MTR</span> 3D 駕駛模擬</h1>
            <div class="welcome-subtitle">終極鐵路駕駛體驗</div>
            <button class="start-btn" onclick="goToRouteSelection()">進入系統</button>
        </div>
        <div class="menu-footer">v13.0 終極十綫版 (抵達尾站返回主選單)</div>
    </div>

    <!-- 第二層：路綫選擇畫面 -->
    <div id="route-select-screen" class="fullscreen-menu">
        <div class="menu-content" style="max-width: 950px;">
            <h2 style="margin-top: 0; font-size: 32px;">請選擇值班路綫</h2>
            
            <div class="route-selector">
                <button id="btn-twl" class="route-btn active twl" onclick="selectRoute('TWL')">荃灣綫 Tsuen Wan Line</button>
                <button id="btn-isl" class="route-btn" onclick="selectRoute('ISL')">港島綫 Island Line</button>
                <button id="btn-ktl" class="route-btn" onclick="selectRoute('KTL')">觀塘綫 Kwun Tong Line</button>
                <button id="btn-tkl" class="route-btn" onclick="selectRoute('TKL')">將軍澳綫 TKO Line</button>
                <button id="btn-eal" class="route-btn" onclick="selectRoute('EAL')">東鐵綫 East Rail Line</button>
                <button id="btn-sil" class="route-btn" onclick="selectRoute('SIL')">南港島綫 South Island Line</button>
                <button id="btn-tcl" class="route-btn" onclick="selectRoute('TCL')">東涌綫 Tung Chung Line</button>
                <button id="btn-ael" class="route-btn" onclick="selectRoute('AEL')">機場快綫 Airport Express</button>
                <button id="btn-tml" class="route-btn" onclick="selectRoute('TML')">屯馬綫 Tuen Ma Line</button>
                <button id="btn-drl" class="route-btn" onclick="selectRoute('DRL')">迪士尼綫 Disneyland Resort Line</button>
            </div>
            
            <h3 class="route-info-title" id="subtitle-display">中環 ➔ 荃灣 (共 16 站)</h3>
            <button class="start-btn" id="start-btn" onclick="startGame()">▶ 開始值班</button>
        </div>
    </div>

    <div id="ui-layer">
        <div id="hud">
            <div class="hud-text">目的地: <span id="station-display" class="station-name">中環</span></div>
            <div class="hud-text">速度: <span id="speed-display" class="highlight">0</span> km/h</div>
            <div class="hud-text">距離: <span id="dist-display" class="highlight">0</span> m</div>
        </div>

        <button id="btn-camera" onclick="toggleCameraMode()">🎥 自動視角</button>

        <div id="message-box">歡迎值班！請先開啟車門上客</div>
        <div id="hologram-box"></div>
        
        <button id="btn-next-station" onclick="goToNextStation()">關門後，駛往下一站</button>

        <div id="door-controls">
            <button class="door-btn" id="btn-door-open" onclick="operateDoors('open')" title="開啟車門">
                <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M14 12h8m0 0l-3-3m3 3l-3 3M10 12H2m0 0l3-3m-3 3l3 3M10 4v16M14 4v16"/></svg>
            </button>
            <button class="door-btn" id="btn-door-close" onclick="operateDoors('close')" title="關閉車門">
                <svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M22 12h-8m0 0l3-3m-3 3l3 3M2 12h8m0 0l-3-3m3 3l-3 3M10 4v16M14 4v16"/></svg>
            </button>
        </div>

        <div id="reverser-controls">
            <button class="dir-btn active" id="btn-dir-fwd" onclick="setDirection(1)">⬆️ 前進</button>
            <button class="dir-btn" id="btn-dir-rev" onclick="setDirection(-1)">⬇️ 退後</button>
        </div>

        <div id="lever-container">
            <div id="lever-track">
                <div id="lever-handle"></div>
                <div class="lever-label top">制動</div>
                <div class="lever-label center">惰行</div>
                <div class="lever-label bottom">牽引</div>
            </div>
        </div>
    </div>

    <script>
        // ================= 十大路綫數據設定 =================
        const routeData = {
            'TWL': {
                name: "中環 ➔ 荃灣 (共 16 站)", themeColor: "#ed1d24",
                stations: [
                    { name: "中環 Central", color: 0x8B0000, cssColor: '#8B0000' }, { name: "金鐘 Admiralty", color: 0x1E90FF, cssColor: '#1E90FF' }, 
                    { name: "尖沙咀 Tsim Sha Tsui", color: 0xFFCC00, cssColor: '#FFCC00' }, { name: "佐敦 Jordan", color: 0x8BC34A, cssColor: '#8BC34A' },        
                    { name: "油麻地 Yau Ma Tei", color: 0xCCCCCC, cssColor: '#CCCCCC' }, { name: "旺角 Mong Kok", color: 0xD32F2F, cssColor: '#D32F2F' },       
                    { name: "太子 Prince Edward", color: 0x9C27B0, cssColor: '#9C27B0' }, { name: "深水埗 Sham Shui Po", color: 0x006400, cssColor: '#006400' },
                    { name: "長沙灣 Cheung Sha Wan", color: 0xC2B280, cssColor: '#C2B280' }, { name: "荔枝角 Lai Chi Kok", color: 0xFF7F50, cssColor: '#FF7F50' },
                    { name: "美孚 Mei Foo", color: 0x0000CD, cssColor: '#0000CD' }, { name: "荔景 Lai King", color: 0xB22222, cssColor: '#B22222' },
                    { name: "葵芳 Kwai Fong", color: 0x2E8B57, cssColor: '#2E8B57' }, { name: "葵興 Kwai Hing", color: 0xFFD700, cssColor: '#FFD700' },
                    { name: "大窩口 Tai Wo Hau", color: 0x8B0000, cssColor: '#8B0000' }, { name: "荃灣 Tsuen Wan", color: 0xed1d24, cssColor: '#ed1d24' }
                ]
            },
            'ISL': {
                name: "堅尼地城 ➔ 柴灣 (共 17 站)", themeColor: "#0071CE",
                stations: [
                    { name: "堅尼地城 Kennedy Town", color: 0xADD8E6, cssColor: '#ADD8E6' }, { name: "香港大學 HKU", color: 0x9ACD32, cssColor: '#9ACD32' },
                    { name: "西營盤 Sai Ying Pun", color: 0x8A2BE2, cssColor: '#8A2BE2' }, { name: "上環 Sheung Wan", color: 0xD2B48C, cssColor: '#D2B48C' },
                    { name: "中環 Central", color: 0x8B0000, cssColor: '#8B0000' }, { name: "金鐘 Admiralty", color: 0x1E90FF, cssColor: '#1E90FF' },
                    { name: "灣仔 Wan Chai", color: 0x00FF00, cssColor: '#00FF00' }, { name: "銅鑼灣 Causeway Bay", color: 0xC8A2C8, cssColor: '#C8A2C8' },
                    { name: "天后 Tin Hau", color: 0xFFA500, cssColor: '#FFA500' }, { name: "炮台山 Fortress Hill", color: 0x008000, cssColor: '#008000' },
                    { name: "北角 North Point", color: 0xFF8C00, cssColor: '#FF8C00' }, { name: "鰂魚涌 Quarry Bay", color: 0x008080, cssColor: '#008080' },
                    { name: "太古 Tai Koo", color: 0xDC143C, cssColor: '#DC143C' }, { name: "西灣河 Sai Wan Ho", color: 0xFFD700, cssColor: '#FFD700' },
                    { name: "筲箕灣 Shau Kei Wan", color: 0x00008B, cssColor: '#00008B' }, { name: "杏花邨 Heng Fa Chuen", color: 0xB22222, cssColor: '#B22222' },
                    { name: "柴灣 Chai Wan", color: 0x006400, cssColor: '#006400' }
                ]
            },
            'KTL': {
                name: "黃埔 ➔ 調景嶺 (共 17 站)", themeColor: "#00A859",
                stations: [
                    { name: "黃埔 Whampoa", color: 0xADD8E6, cssColor: '#ADD8E6' }, { name: "何文田 Ho Man Tin", color: 0x90EE90, cssColor: '#90EE90' }, 
                    { name: "油麻地 Yau Ma Tei", color: 0xCCCCCC, cssColor: '#CCCCCC' }, { name: "旺角 Mong Kok", color: 0xD32F2F, cssColor: '#D32F2F' }, 
                    { name: "太子 Prince Edward", color: 0x9C27B0, cssColor: '#9C27B0' }, { name: "石硤尾 Shek Kip Mei", color: 0x9ACD32, cssColor: '#9ACD32' }, 
                    { name: "九龍塘 Kowloon Tong", color: 0x87CEFA, cssColor: '#87CEFA' }, { name: "樂富 Lok Fu", color: 0x2E8B57, cssColor: '#2E8B57' }, 
                    { name: "黃大仙 Wong Tai Sin", color: 0xFFD700, cssColor: '#FFD700' }, { name: "鑽石山 Diamond Hill", color: 0x333333, cssColor: '#333333' }, 
                    { name: "彩虹 Choi Hung", color: 0x0000CD, cssColor: '#0000CD' }, { name: "九龍灣 Kowloon Bay", color: 0xFF0000, cssColor: '#FF0000' }, 
                    { name: "牛頭角 Ngau Tau Kok", color: 0x98FB98, cssColor: '#98FB98' }, { name: "觀塘 Kwun Tong", color: 0xEEEEEE, cssColor: '#EEEEEE' }, 
                    { name: "藍田 Lam Tin", color: 0x1E90FF, cssColor: '#1E90FF' }, { name: "油塘 Yau Tong", color: 0xFFD700, cssColor: '#FFD700' }, 
                    { name: "調景嶺 Tiu Keng Leng", color: 0xADFF2F, cssColor: '#ADFF2F' } 
                ]
            },
            'TKL': {
                name: "北角 ➔ 寶琳 (共 7 站)", themeColor: "#8A2BE2", 
                stations: [
                    { name: "北角 North Point", color: 0xFF8C00, cssColor: '#FF8C00' }, { name: "鰂魚涌 Quarry Bay", color: 0x008080, cssColor: '#008080' }, 
                    { name: "油塘 Yau Tong", color: 0xFFD700, cssColor: '#FFD700' }, { name: "調景嶺 Tiu Keng Leng", color: 0xADFF2F, cssColor: '#ADFF2F' }, 
                    { name: "將軍澳 Tseung Kwan O", color: 0xDC143C, cssColor: '#DC143C' }, { name: "坑口 Hang Hau", color: 0x87CEFA, cssColor: '#87CEFA' }, 
                    { name: "寶琳 Po Lam", color: 0xFF4500, cssColor: '#FF4500' }
                ]
            },
            'EAL': {
                name: "金鐘 ➔ 羅湖 (共 14 站)", themeColor: "#53B7E8", 
                stations: [
                    { name: "金鐘 Admiralty", color: 0x1E90FF, cssColor: '#1E90FF' }, { name: "會展 Exhibition Centre", color: 0x607D8B, cssColor: '#607D8B' },
                    { name: "紅磡 Hung Hom", color: 0xD32F2F, cssColor: '#D32F2F' }, { name: "旺角東 Mong Kok East", color: 0x00BCD4, cssColor: '#00BCD4' },
                    { name: "九龍塘 Kowloon Tong", color: 0x87CEFA, cssColor: '#87CEFA' }, { name: "大圍 Tai Wai", color: 0x006400, cssColor: '#006400' },
                    { name: "沙田 Sha Tin", color: 0xed1d24, cssColor: '#ed1d24' }, { name: "火炭 Fo Tan", color: 0xFFA500, cssColor: '#FFA500' },
                    { name: "大學 University", color: 0x87CEEB, cssColor: '#87CEEB' }, { name: "大埔墟 Tai Po Market", color: 0x4CAF50, cssColor: '#4CAF50' },
                    { name: "太和 Tai Wo", color: 0xFFD700, cssColor: '#FFD700' }, { name: "粉嶺 Fanling", color: 0xFFEB3B, cssColor: '#FFEB3B' },
                    { name: "上水 Sheung Shui", color: 0xFF9800, cssColor: '#FF9800' }, { name: "羅湖 Lo Wu", color: 0x53B7E8, cssColor: '#53B7E8' }
                ]
            },
            'SIL': {
                name: "金鐘 ➔ 海怡半島 (共 5 站)", themeColor: "#b2bb1c", 
                stations: [
                    { name: "金鐘 Admiralty", color: 0x1E90FF, cssColor: '#1E90FF' }, { name: "海洋公園 Ocean Park", color: 0x0080FF, cssColor: '#0080FF' },
                    { name: "黃竹坑 Wong Chuk Hang", color: 0xFFFF00, cssColor: '#FFFF00' }, { name: "利東 Lei Tung", color: 0xFFA500, cssColor: '#FFA500' },
                    { name: "海怡半島 South Horizons", color: 0xb2bb1c, cssColor: '#b2bb1c' }
                ]
            },
            'TCL': {
                name: "香港 ➔ 東涌 (共 8 站)", themeColor: "#F68F26", 
                stations: [
                    { name: "香港 Hong Kong", color: 0xFF8C00, cssColor: '#FF8C00' }, { name: "九龍 Kowloon", color: 0xA9A9A9, cssColor: '#A9A9A9' },
                    { name: "奧運 Olympic", color: 0x4682B4, cssColor: '#4682B4' }, { name: "南昌 Nam Cheong", color: 0xFFFF00, cssColor: '#xFFFF00' },
                    { name: "荔景 Lai King", color: 0xB22222, cssColor: '#B22222' }, { name: "青衣 Tsing Yi", color: 0xADD8E6, cssColor: '#ADD8E6' },
                    { name: "欣澳 Sunny Bay", color: 0x808080, cssColor: '#808080' }, { name: "東涌 Tung Chung", color: 0x8A2BE2, cssColor: '#8A2BE2' }
                ]
            },
            'AEL': {
                name: "香港 ➔ 博覽館 (共 5 站)", themeColor: "#008C95", 
                stations: [
                    { name: "香港 Hong Kong", color: 0xFF8C00, cssColor: '#FF8C00' }, { name: "九龍 Kowloon", color: 0xA9A9A9, cssColor: '#A9A9A9' },
                    { name: "青衣 Tsing Yi", color: 0xADD8E6, cssColor: '#ADD8E6' }, { name: "機場 Airport", color: 0x808080, cssColor: '#808080' },
                    { name: "博覽館 AsiaWorld-Expo", color: 0xA0522D, cssColor: '#A0522D' }
                ]
            },
            'TML': {
                name: "屯門 ➔ 烏溪沙 (共 27 站)", themeColor: "#9A3820", 
                stations: [
                    { name: "屯門 Tuen Mun", color: 0x008080, cssColor: '#008080' }, { name: "兆康 Siu Hong", color: 0x87CEEB, cssColor: '#87CEEB' },
                    { name: "天水圍 Tin Shui Wai", color: 0xFFA500, cssColor: '#FFA500' }, { name: "朗屏 Long Ping", color: 0xFFC0CB, cssColor: '#FFC0CB' },
                    { name: "元朗 Yuen Long", color: 0x40E0D0, cssColor: '#40E0D0' }, { name: "錦上路 Kam Sheung Road", color: 0x8B4513, cssColor: '#8B4513' },
                    { name: "荃灣西 Tsuen Wan West", color: 0xA52A2A, cssColor: '#A52A2A' }, { name: "美孚 Mei Foo", color: 0x0000CD, cssColor: '#0000CD' },
                    { name: "南昌 Nam Cheong", color: 0xFFFF00, cssColor: '#xFFFF00' }, { name: "柯士甸 Austin", color: 0xD2B48C, cssColor: '#D2B48C' },
                    { name: "尖東 East Tsim Sha Tsui", color: 0xFFFF00, cssColor: '#xFFFF00' }, { name: "紅磡 Hung Hom", color: 0xD32F2F, cssColor: '#D32F2F' },
                    { name: "何文田 Ho Man Tin", color: 0x90EE90, cssColor: '#90EE90' }, { name: "土瓜灣 To Kwa Wan", color: 0x87CEFA, cssColor: '#87CEFA' },
                    { name: "宋皇臺 Sung Wong Toi", color: 0xF5DEB3, cssColor: '#F5DEB3' }, { name: "啟德 Kai Tak", color: 0xFFA500, cssColor: '#FFA500' },
                    { name: "鑽石山 Diamond Hill", color: 0x333333, cssColor: '#333333' }, { name: "顯徑 Hin Keng", color: 0x98FB98, cssColor: '#98FB98' },
                    { name: "大圍 Tai Wai", color: 0x006400, cssColor: '#006400' }, { name: "車公廟 Che Kung Temple", color: 0xEEE8AA, cssColor: '#EEE8AA' },
                    { name: "沙田圍 Sha Tin Wai", color: 0xFFB6C1, cssColor: '#FFB6C1' }, { name: "第一城 City One", color: 0xFFA500, cssColor: '#FFA500' },
                    { name: "石門 Shek Mun", color: 0xFFFFE0, cssColor: '#xFFFFE0' }, { name: "大水坑 Tai Shui Hang", color: 0xADD8E6, cssColor: '#ADD8E6' },
                    { name: "恆安 Heng On", color: 0x87CEFA, cssColor: '#87CEFA' }, { name: "馬鞍山 Ma On Shan", color: 0xDDA0DD, cssColor: '#DDA0DD' },
                    { name: "烏溪沙 Wu Kai Sha", color: 0x8B4513, cssColor: '#8B4513' }
                ]
            },
            'DRL': {
                name: "欣澳 ➔ 迪士尼 (共 2 站)", themeColor: "#FF75B5", 
                stations: [
                    { name: "欣澳 Sunny Bay", color: 0x808080, cssColor: '#808080' },
                    { name: "迪士尼 Disneyland Resort", color: 0x006400, cssColor: '#006400' }
                ]
            }
        };

        // ================= 核心變數 =================
        let scene, camera, renderer, worldGroup;
        let train;
        let trainStripeMaterial; 
        
        let selectedRouteId = 'TWL';
        let stationsData = routeData['TWL'].stations;
        let currentStationIndex = 0;
        let stopMarkZ = -12.5; 

        let speed = 0;
        let perfectlyStopped = false;
        let doorProgress = 0, targetDoorProgress = 0;
        
        let doorsArray = [];
        let psdDoorsArray = []; 
        let isOverrun = false; 
        let driverLever3D = null;
        let leverValue = 0; 
        let currentDirection = 1; 
        let doorsClosedMsgShown = false;

        let frontGlass = null; // 記錄車頭牆壁，方便切換視角時隱藏

        let cameraMode = 'auto'; 
        let camAngleX = 0.15, camAngleY = 0.3, camRadius = 85; 
        let passAngleX = 0, passAngleY = 0; 
        let currentFov = 60; 
        let isDragging = false;
        let previousMouseX = 0, previousMouseY = 0;

        let gameStarted = false;

        // ================= UI 路綫與選單切換 =================
        window.goToRouteSelection = function() {
            const welcome = document.getElementById('welcome-screen');
            const routeSelect = document.getElementById('route-select-screen');
            welcome.style.opacity = '0';
            setTimeout(() => {
                welcome.style.display = 'none';
                routeSelect.style.display = 'flex';
                setTimeout(() => routeSelect.style.opacity = '1', 50);
            }, 500);
        }

        window.selectRoute = function(id) {
            selectedRouteId = id;
            stationsData = routeData[id].stations;
            
            // 更新按鈕樣式
            document.querySelectorAll('.route-btn').forEach(btn => btn.classList.remove('active', 'twl', 'isl', 'ktl', 'tkl', 'eal', 'sil', 'tcl', 'ael', 'tml', 'drl'));
            const selectedBtn = document.getElementById('btn-' + id.toLowerCase());
            selectedBtn.classList.add('active', id.toLowerCase());
            
            // 更新 UI 顏色主題
            document.getElementById('subtitle-display').innerText = routeData[id].name;
            document.getElementById('subtitle-display').style.color = routeData[id].themeColor;
            document.getElementById('start-btn').style.background = `linear-gradient(135deg, ${routeData[id].themeColor}, #333)`;
            document.getElementById('btn-next-station').style.backgroundColor = routeData[id].themeColor;
            document.getElementById('hud').style.borderLeftColor = routeData[id].themeColor;

            // 即時更新 3D 車廂飾條顏色
            if (trainStripeMaterial) {
                trainStripeMaterial.color.set(routeData[id].themeColor);
            }

            createWorld();
        }

        // ================= 初始設置 =================
        function init() {
            const container = document.getElementById('game-container');
            scene = new THREE.Scene();
            scene.background = new THREE.Color(0x111111); 
            scene.fog = new THREE.Fog(0x111111, 50, 1000);

            camera = new THREE.PerspectiveCamera(currentFov, window.innerWidth / window.innerHeight, 0.1, 2000);
            
            renderer = new THREE.WebGLRenderer({ antialias: true });
            renderer.setSize(window.innerWidth, window.innerHeight);
            renderer.shadowMap.enabled = true;
            container.appendChild(renderer.domElement);

            scene.add(new THREE.AmbientLight(0xffffff, 0.4));
            const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
            dirLight.position.set(0, 100, 0);
            scene.add(dirLight);

            createWorld(); 
            createTrain();

            window.addEventListener('resize', onWindowResize, false);
            setupLeverControl(); 
            setupCameraControls(); 
            animate();
        }

        function startGame() {
            gameStarted = true;
            currentStationIndex = 0;
            stopMarkZ = -12.5;
            speed = 0;
            train.position.set(0, 0, 0); 
            updateHUDStation();

            const routeSelect = document.getElementById('route-select-screen');
            const uiLayer = document.getElementById('ui-layer');
            
            routeSelect.style.opacity = '0';
            setTimeout(() => { routeSelect.style.display = 'none'; }, 500);
            
            uiLayer.classList.add('visible');
        }

        function updateHUDStation() {
            const stDisplay = document.getElementById('station-display');
            stDisplay.innerText = stationsData[currentStationIndex].name;
            stDisplay.style.color = stationsData[currentStationIndex].cssColor;
        }

        function createTextTexture(text, themeCssColor) {
            const canvas = document.createElement('canvas');
            canvas.width = 512; canvas.height = 128;
            const ctx = canvas.getContext('2d');
            ctx.fillStyle = '#ffffff'; ctx.fillRect(0, 0, 512, 128);
            ctx.fillStyle = themeCssColor; ctx.fillRect(0, 0, 512, 15); ctx.fillRect(0, 113, 512, 15);
            ctx.fillStyle = '#000000'; ctx.font = 'bold 44px sans-serif'; ctx.textAlign = 'center'; ctx.textBaseline = 'middle';
            ctx.fillText(text, 256, 64);
            return new THREE.CanvasTexture(canvas);
        }

        function createWorld() {
            if (worldGroup) { scene.remove(worldGroup); }
            worldGroup = new THREE.Group();
            scene.add(worldGroup);
            psdDoorsArray = []; 

            const trackLength = 150000; 
            const trackCenterZ = -73000;

            const bedGeo = new THREE.PlaneGeometry(80, trackLength);
            const bedMat = new THREE.MeshLambertMaterial({ color: 0x222222 });
            const trackBed = new THREE.Mesh(bedGeo, bedMat);
            trackBed.rotation.x = -Math.PI / 2; trackBed.position.y = -0.9; trackBed.position.z = trackCenterZ;
            worldGroup.add(trackBed);

            const railGeo = new THREE.BoxGeometry(0.5, 0.5, trackLength);
            const railMat = new THREE.MeshStandardMaterial({ color: 0x888888, metalness: 0.8 });
            const railLeft = new THREE.Mesh(railGeo, railMat); railLeft.position.set(-2, -0.6, trackCenterZ); worldGroup.add(railLeft);
            const railRight = new THREE.Mesh(railGeo, railMat); railRight.position.set(2, -0.6, trackCenterZ); worldGroup.add(railRight);

            const tunnelMat = new THREE.MeshLambertMaterial({ color: 0x151515 }); 
            const leftWallGeo = new THREE.BoxGeometry(1, 18, trackLength);
            const leftWall = new THREE.Mesh(leftWallGeo, tunnelMat); leftWall.position.set(-10, 8, trackCenterZ); worldGroup.add(leftWall);
            const rightWallGeo = new THREE.BoxGeometry(1, 18, trackLength);
            const rightWall = new THREE.Mesh(rightWallGeo, tunnelMat); rightWall.position.set(20.5, 8, trackCenterZ); worldGroup.add(rightWall);
            const ceilingGeo = new THREE.BoxGeometry(32, 1, trackLength);
            const ceiling = new THREE.Mesh(ceilingGeo, tunnelMat); ceiling.position.set(5.25, 16.5, trackCenterZ); worldGroup.add(ceiling);

            const poleGeo = new THREE.CylinderGeometry(0.2, 0.2, 10);
            const poleMat = new THREE.MeshLambertMaterial({ color: 0x555555 });
            for (let i = 200; i > -145000; i -= 50) {
                const pole = new THREE.Mesh(poleGeo, poleMat); pole.position.set(-8, 4, i); worldGroup.add(pole);
            }

            const psdMat = new THREE.MeshStandardMaterial({ color: 0x888888, metalness: 0.7, roughness: 0.3 });
            const psdGlassMat = new THREE.MeshStandardMaterial({ color: 0xaaddff, transparent: true, opacity: 0.35, metalness: 0.8, roughness: 0.1, side: THREE.DoubleSide });

            stationsData.forEach((st, index) => {
                const platformGroup = new THREE.Group();
                let stStopMarkZ = -12.5 - (index * 4000); 
                let stCenterZ = stStopMarkZ + 105.5; 

                const platformGeo = new THREE.BoxGeometry(16, 3, 212);
                const platformMat = new THREE.MeshLambertMaterial({ color: 0xcccccc });
                const platform = new THREE.Mesh(platformGeo, platformMat);
                platform.position.set(12.5, -0.6, 0); platformGroup.add(platform);

                const stCeilingGeo = new THREE.BoxGeometry(16, 0.5, 212);
                const stCeilingMat = new THREE.MeshLambertMaterial({ color: 0xe0e0e0 }); 
                const stCeiling = new THREE.Mesh(stCeilingGeo, stCeilingMat);
                stCeiling.position.set(12.5, 9.0, 0); 
                platformGroup.add(stCeiling);

                const beamGeo = new THREE.BoxGeometry(16, 0.6, 2);
                const beamMat = new THREE.MeshLambertMaterial({ color: 0xbbbbbb });
                for(let z = 100; z > -100; z -= 20) {
                    const beam = new THREE.Mesh(beamGeo, beamMat);
                    beam.position.set(12.5, 8.7, z); platformGroup.add(beam);
                }

                const psdUpperWallGeo = new THREE.BoxGeometry(0.5, 3.15, 212); 
                const psdUpperWallMat = new THREE.MeshLambertMaterial({ color: 0x222222 }); 
                const psdUpperWall = new THREE.Mesh(psdUpperWallGeo, psdUpperWallMat);
                psdUpperWall.position.set(4.6, 7.175, 0); platformGroup.add(psdUpperWall);
                
                const psdUpperStripeGeo = new THREE.BoxGeometry(0.55, 0.3, 212);
                const psdUpperStripe = new THREE.Mesh(psdUpperStripeGeo, new THREE.MeshLambertMaterial({ color: st.color }));
                psdUpperStripe.position.set(4.6, 7.175, 0); platformGroup.add(psdUpperStripe);

                const yellowStripGeo = new THREE.BoxGeometry(0.5, 0.1, 212);
                const yellowMat = new THREE.MeshLambertMaterial({ color: 0xffcc00 });
                const yellowStrip = new THREE.Mesh(yellowStripGeo, yellowMat);
                yellowStrip.position.set(6.5, 0.95, 0); platformGroup.add(yellowStrip);

                const wallGeo = new THREE.BoxGeometry(1, 15, 212);
                const wallMat = new THREE.MeshLambertMaterial({ color: st.color });
                const wall = new THREE.Mesh(wallGeo, wallMat);
                wall.position.set(20, 7.5, 0); platformGroup.add(wall);

                const endWallGeo = new THREE.BoxGeometry(15.4, 10, 1); 
                const frontEndWall = new THREE.Mesh(endWallGeo, wallMat); frontEndWall.position.set(12.3, 4.5, 105.5); platformGroup.add(frontEndWall);
                const backEndWall = new THREE.Mesh(endWallGeo, wallMat); backEndWall.position.set(12.3, 4.5, -105.5); platformGroup.add(backEndWall);

                const signGeo = new THREE.PlaneGeometry(10, 2.5);
                const signTex = createTextTexture(st.name, st.cssColor);
                const signMat = new THREE.MeshBasicMaterial({ map: signTex });
                for(let z = 80; z > -80; z -= 40) {
                    const sign = new THREE.Mesh(signGeo, signMat);
                    sign.position.set(19.4, 5.0, z); sign.rotation.y = -Math.PI / 2; platformGroup.add(sign);
                }

                for(let z = 90; z > -90; z -= 30) {
                    const pLight = new THREE.PointLight(0xffffff, 0.6, 60);
                    pLight.position.set(10, 8.5, z); platformGroup.add(pLight);
                }

                const stopBoardGeo = new THREE.BoxGeometry(2, 4, 0.5);
                const stopBoardMat = new THREE.MeshBasicMaterial({ color: 0xffaa00 }); 
                const stopBoard = new THREE.Mesh(stopBoardGeo, stopBoardMat);
                stopBoard.position.set(4, 2.9, stStopMarkZ - stCenterZ); platformGroup.add(stopBoard); 

                const psdHeaderGeo = new THREE.BoxGeometry(0.4, 0.6, 212);
                const psdHeader = new THREE.Mesh(psdHeaderGeo, psdMat); psdHeader.position.set(4.6, 5.3, 0); platformGroup.add(psdHeader);
                const psdStripeGeo = new THREE.BoxGeometry(0.45, 0.4, 212);
                const psdStripe = new THREE.Mesh(psdStripeGeo, new THREE.MeshLambertMaterial({ color: st.color })); psdStripe.position.set(4.6, 4.7, 0); platformGroup.add(psdStripe);
                const psdFooterGeo = new THREE.BoxGeometry(0.4, 0.3, 212);
                const psdFooter = new THREE.Mesh(psdFooterGeo, psdMat); psdFooter.position.set(4.6, 1.05, 0); platformGroup.add(psdFooter);

                let previousFixedEndZ = -106;
                for (let c = 0; c < 8; c++) {
                    for (let d of [-8, 0, 8]) {
                        let doorCenterZ = c * 26.5 + d - 93;
                        let fixedLength = (doorCenterZ - 1.6) - previousFixedEndZ;
                        if (fixedLength > 0) {
                            let fixedGeo = new THREE.BoxGeometry(0.1, 4.1, fixedLength);
                            let fixedMesh = new THREE.Mesh(fixedGeo, psdGlassMat); fixedMesh.position.set(4.6, 2.95, previousFixedEndZ + fixedLength / 2); platformGroup.add(fixedMesh);
                            let mapGeo = new THREE.BoxGeometry(0.12, 0.8, Math.min(fixedLength * 0.6, 2));
                            let mapMat = new THREE.MeshLambertMaterial({ color: 0xdddddd });
                            let mapMesh = new THREE.Mesh(mapGeo, mapMat); mapMesh.position.set(4.6, 3.1, previousFixedEndZ + fixedLength / 2); platformGroup.add(mapMesh);
                        }
                        let slidingGeo = new THREE.BoxGeometry(0.15, 4.1, 1.6);
                        let psdDoorLeft = new THREE.Mesh(slidingGeo, psdGlassMat); psdDoorLeft.position.set(4.55, 2.95, doorCenterZ - 0.8); platformGroup.add(psdDoorLeft);
                        let psdDoorRight = new THREE.Mesh(slidingGeo, psdGlassMat); psdDoorRight.position.set(4.55, 2.95, doorCenterZ + 0.8); platformGroup.add(psdDoorRight);
                        psdDoorsArray.push({ mesh: psdDoorLeft, baseZ: doorCenterZ - 0.8, direction: -1 }); psdDoorsArray.push({ mesh: psdDoorRight, baseZ: doorCenterZ + 0.8, direction: 1 });
                        previousFixedEndZ = doorCenterZ + 1.6;
                    }
                }
                let finalFixedLength = 106 - previousFixedEndZ;
                if (finalFixedLength > 0) {
                    let fixedGeo = new THREE.BoxGeometry(0.1, 4.1, finalFixedLength); let fixedMesh = new THREE.Mesh(fixedGeo, psdGlassMat);
                    fixedMesh.position.set(4.6, 2.95, previousFixedEndZ + finalFixedLength / 2); platformGroup.add(fixedMesh);
                }
                platformGroup.position.z = stCenterZ; worldGroup.add(platformGroup);
            });
        }

        function createTrain() {
            train = new THREE.Group();
            doorsArray = [];
            const numCars = 8; const carSpacing = 26.5;

            const chassisMat = new THREE.MeshStandardMaterial({ color: 0x333333, metalness: 0.3 });
            const bodyMat = new THREE.MeshStandardMaterial({ color: 0xeeeeee, metalness: 0.6, roughness: 0.3 });
            const floorMat = new THREE.MeshStandardMaterial({ color: 0x999999, roughness: 0.9 });
            const ceilingMat = new THREE.MeshStandardMaterial({ color: 0xffffff, roughness: 0.5 });
            const seatMat = new THREE.MeshStandardMaterial({ color: 0xc0c0c0, metalness: 0.8, roughness: 0.2 }); 
            const poleMat = new THREE.MeshStandardMaterial({ color: 0xed1d24, metalness: 0.3, roughness: 0.4 }); 
            const windowMat = new THREE.MeshStandardMaterial({ color: 0x111111, metalness: 0.9, transparent: true, opacity: 0.6 }); 
            const doorMat = new THREE.MeshStandardMaterial({ color: 0xbbbbbb, metalness: 0.8 });
            const acMat = new THREE.MeshStandardMaterial({ color: 0xaaaaaa });
            const frontMat = new THREE.MeshStandardMaterial({ color: 0x111111 }); 
            const headlightMat = new THREE.MeshBasicMaterial({ color: 0xffffee }); 
            
            // 初始化為荃灣綫顏色，之後由 selectRoute 更新
            trainStripeMaterial = new THREE.MeshLambertMaterial({ color: routeData['TWL'].themeColor });

            for (let c = 0; c < numCars; c++) {
                let carGroup = new THREE.Group();
                let isHead = (c === 0);

                const chassisGeo = new THREE.BoxGeometry(6, 0.5, 26);
                const chassis = new THREE.Mesh(chassisGeo, chassisMat); chassis.position.y = 0.25; carGroup.add(chassis);
                
                const floorGeo = new THREE.BoxGeometry(5.8, 0.2, 25);
                const floor = new THREE.Mesh(floorGeo, floorMat); floor.position.y = 0.8; carGroup.add(floor);
                
                const ceilingGeo = new THREE.BoxGeometry(5.8, 0.2, 25);
                const ceiling = new THREE.Mesh(ceilingGeo, ceilingMat); ceiling.position.y = 6.6; carGroup.add(ceiling);

                const wallSegments = [ { length: 3.7, z: -10.65 }, { length: 6.4, z: -4.0 }, { length: 6.4, z: 4.0 }, { length: 3.7, z: 10.65 } ];
                wallSegments.forEach(seg => {
                    if (seg.length > 2) { 
                        const lowerGeo = new THREE.BoxGeometry(0.2, 1.6, seg.length);
                        let lwR = new THREE.Mesh(lowerGeo, bodyMat); lwR.position.set(2.8, 1.6, seg.z); carGroup.add(lwR);
                        let lwL = new THREE.Mesh(lowerGeo, bodyMat); lwL.position.set(-2.8, 1.6, seg.z); carGroup.add(lwL);
                        const upperGeo = new THREE.BoxGeometry(0.2, 2.8, seg.length);
                        let uwR = new THREE.Mesh(upperGeo, bodyMat); uwR.position.set(2.8, 5.2, seg.z); carGroup.add(uwR);
                        let uwL = new THREE.Mesh(upperGeo, bodyMat); uwL.position.set(-2.8, 5.2, seg.z); carGroup.add(uwL);
                        
                        let pilFrontZ = seg.z - seg.length/2 + 0.25; let pilFrontR = new THREE.Mesh(new THREE.BoxGeometry(0.2, 1.4, 0.5), bodyMat); pilFrontR.position.set(2.8, 3.1, pilFrontZ); carGroup.add(pilFrontR);
                        let pilFrontL = new THREE.Mesh(new THREE.BoxGeometry(0.2, 1.4, 0.5), bodyMat); pilFrontL.position.set(-2.8, 3.1, pilFrontZ); carGroup.add(pilFrontL);
                        let pilBackZ = seg.z + seg.length/2 - 0.25; let pilBackR = new THREE.Mesh(new THREE.BoxGeometry(0.2, 1.4, 0.5), bodyMat); pilBackR.position.set(2.8, 3.1, pilBackZ); carGroup.add(pilBackR);
                        let pilBackL = new THREE.Mesh(new THREE.BoxGeometry(0.2, 1.4, 0.5), bodyMat); pilBackL.position.set(-2.8, 3.1, pilBackZ); carGroup.add(pilBackL);

                        const winR = new THREE.Mesh(new THREE.BoxGeometry(0.25, 1.4, seg.length - 1.0), windowMat); winR.position.set(2.8, 3.1, seg.z); carGroup.add(winR);
                        const winL = new THREE.Mesh(new THREE.BoxGeometry(0.25, 1.4, seg.length - 1.0), windowMat); winL.position.set(-2.8, 3.1, seg.z); carGroup.add(winL);
                        
                        const seatBaseR = new THREE.Mesh(new THREE.BoxGeometry(0.6, 0.15, seg.length - 1.0), seatMat); seatBaseR.position.set(2.3, 1.35, seg.z); carGroup.add(seatBaseR);
                        const seatBackR = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.8, seg.length - 1.0), seatMat); seatBackR.position.set(2.6, 1.8, seg.z); carGroup.add(seatBackR);
                        const seatBaseL = new THREE.Mesh(new THREE.BoxGeometry(0.6, 0.15, seg.length - 1.0), seatMat); seatBaseL.position.set(-2.3, 1.35, seg.z); carGroup.add(seatBaseL);
                        const seatBackL = new THREE.Mesh(new THREE.BoxGeometry(0.15, 0.8, seg.length - 1.0), seatMat); seatBackL.position.set(-2.6, 1.8, seg.z); carGroup.add(seatBackL);
                    } else {
                        const wallR = new THREE.Mesh(new THREE.BoxGeometry(0.2, 5.8, seg.length), bodyMat); wallR.position.set(2.8, 3.7, seg.z); carGroup.add(wallR);
                        const wallL = new THREE.Mesh(new THREE.BoxGeometry(0.2, 5.8, seg.length), bodyMat); wallL.position.set(-2.8, 3.7, seg.z); carGroup.add(wallL);
                    }
                });

                for (let pz of [-8, 0, 8]) {
                    const topFrameGeo = new THREE.BoxGeometry(0.2, 1.15, 1.6);
                    const tfR = new THREE.Mesh(topFrameGeo, bodyMat); tfR.position.set(2.8, 6.025, pz); carGroup.add(tfR);
                    const tfL = new THREE.Mesh(topFrameGeo, bodyMat); tfL.position.set(-2.8, 6.025, pz); carGroup.add(tfL);
                }

                if (!isHead) {
                    const endWall = new THREE.Mesh(new THREE.BoxGeometry(5.8, 5.8, 0.2), bodyMat); endWall.position.set(0, 3.7, -12.4); carGroup.add(endWall);
                    const gangway = new THREE.Mesh(new THREE.BoxGeometry(4.8, 5.1, 1.5), new THREE.MeshStandardMaterial({color: 0x222222})); gangway.position.set(0, 3.35, -13.25); carGroup.add(gangway);
                }
                const backWall = new THREE.Mesh(new THREE.BoxGeometry(5.8, 5.8, 0.2), bodyMat); backWall.position.set(0, 3.7, 12.4); carGroup.add(backWall);

                const poleGeo = new THREE.CylinderGeometry(0.04, 0.04, 5.8);
                for (let pz of [-8, -4, 0, 4, 8]) {
                    const pole = new THREE.Mesh(poleGeo, poleMat); pole.position.set(0, 3.7, pz); carGroup.add(pole);
                    if (pz === -8 || pz === 0 || pz === 8) {
                        const pR1 = new THREE.Mesh(poleGeo, poleMat); pR1.position.set(1.5, 3.7, pz - 1.0); carGroup.add(pR1);
                        const pL1 = new THREE.Mesh(poleGeo, poleMat); pL1.position.set(-1.5, 3.7, pz - 1.0); carGroup.add(pL1);
                    }
                }

                for (let lz of [-6, 6]) { const iLight = new THREE.PointLight(0xffffee, 0.6, 15); iLight.position.set(0, 6.0, lz); carGroup.add(iLight); }

                const roof = new THREE.Mesh(new THREE.BoxGeometry(5.4, 0.8, 25), bodyMat); roof.position.y = 7.1; carGroup.add(roof);
                for(let z of [-8, 0, 8]) { const ac = new THREE.Mesh(new THREE.BoxGeometry(3, 0.8, 4), acMat); ac.position.set(0, 7.6, z); carGroup.add(ac); }

                if (isHead) {
                    const front = new THREE.Mesh(new THREE.BoxGeometry(5.9, 5.8, 0.5), frontMat); front.position.set(0, 3.7, -12.5); carGroup.add(front);
                    frontGlass = front; // 綁定車頭牆壁，方便司機視角隱藏
                    
                    const headlightGeo = new THREE.CircleGeometry(0.4, 16);
                    const hlLeft = new THREE.Mesh(headlightGeo, headlightMat); hlLeft.position.set(-1.8, 2.0, -12.76); hlLeft.rotation.y = Math.PI; carGroup.add(hlLeft);
                    const hlRight = new THREE.Mesh(headlightGeo, headlightMat); hlRight.position.set(1.8, 2.0, -12.76); hlRight.rotation.y = Math.PI; carGroup.add(hlRight);
                    const desk = new THREE.Mesh(new THREE.BoxGeometry(5.8, 1.2, 1.5), new THREE.MeshStandardMaterial({ color: 0x222222, metalness: 0.5, roughness: 0.8 })); desk.position.set(0, 2.8, -11.5); carGroup.add(desk);
                    const screen = new THREE.Mesh(new THREE.BoxGeometry(0.8, 0.6, 0.1), new THREE.MeshBasicMaterial({ color: 0x004400 })); screen.position.set(-1.5, 3.5, -11.8); screen.rotation.x = -Math.PI / 6; carGroup.add(screen);
                    
                    const leverMesh = new THREE.Mesh(new THREE.CylinderGeometry(0.05, 0.05, 0.4), new THREE.MeshStandardMaterial({ color: 0xcccccc }));
                    leverMesh.position.set(-0.8, 3.6, -11.5); leverMesh.rotation.x = Math.PI / 4; carGroup.add(leverMesh); driverLever3D = leverMesh; 

                    const partitionWall = new THREE.Mesh(new THREE.BoxGeometry(5.8, 5.8, 0.2), bodyMat); partitionWall.position.set(0, 3.7, -10.0); carGroup.add(partitionWall);
                    const partitionDoor = new THREE.Mesh(new THREE.BoxGeometry(1.2, 4.2, 0.22), new THREE.MeshStandardMaterial({ color: 0x999999, metalness: 0.5, roughness: 0.4 })); partitionDoor.position.set(0, 2.9, -10.0); carGroup.add(partitionDoor);
                    const partitionWin = new THREE.Mesh(new THREE.BoxGeometry(0.7, 1.2, 0.25), windowMat); partitionWin.position.set(0, 3.6, -10.0); carGroup.add(partitionWin);
                }

                // 列車飾條
                const stripeGeo = new THREE.BoxGeometry(6.05, 0.4, 25);
                const stripe = new THREE.Mesh(stripeGeo, trainStripeMaterial); stripe.position.y = 0.8; stripe.name = "stripe"; carGroup.add(stripe);

                const doorGeo = new THREE.BoxGeometry(0.15, 4.65, 1.6); const doorWinGeo = new THREE.BoxGeometry(0.2, 1.4, 1.0); 
                for (let i = 0; i < 3; i++) {
                    let doorZ = -8 + i * 8; 
                    let doorRightLeft = new THREE.Group(); let drlMesh = new THREE.Mesh(doorGeo, doorMat); let drlWin = new THREE.Mesh(doorWinGeo, windowMat); drlWin.position.y = -0.025; doorRightLeft.add(drlMesh); doorRightLeft.add(drlWin); doorRightLeft.position.set(2.95, 3.125, doorZ - 0.8); 
                    let doorRightRight = new THREE.Group(); let drrMesh = new THREE.Mesh(doorGeo, doorMat); let drrWin = new THREE.Mesh(doorWinGeo, windowMat); drrWin.position.y = -0.025; doorRightRight.add(drrMesh); doorRightRight.add(drrWin); doorRightRight.position.set(2.95, 3.125, doorZ + 0.8);
                    carGroup.add(doorRightLeft); carGroup.add(doorRightRight);
                    doorsArray.push({ mesh: doorRightLeft, baseZ: doorZ - 0.8, direction: -1, side: 'right' }); doorsArray.push({ mesh: doorRightRight, baseZ: doorZ + 0.8, direction: 1, side: 'right' });
                    
                    let doorLeftLeft = new THREE.Group(); let dllMesh = new THREE.Mesh(doorGeo, doorMat); let dllWin = new THREE.Mesh(doorWinGeo, windowMat); dllWin.position.y = -0.025; doorLeftLeft.add(dllMesh); doorLeftLeft.add(dllWin); doorLeftLeft.position.set(-2.95, 3.125, doorZ - 0.8); 
                    let doorLeftRight = new THREE.Group(); let dlrMesh = new THREE.Mesh(doorGeo, doorMat); let dlrWin = new THREE.Mesh(doorWinGeo, windowMat); dlrWin.position.y = -0.025; doorLeftRight.add(dlrMesh); doorLeftRight.add(dlrWin); doorLeftRight.position.set(-2.95, 3.125, doorZ + 0.8);
                    carGroup.add(doorLeftLeft); carGroup.add(doorLeftRight);
                    doorsArray.push({ mesh: doorLeftLeft, baseZ: doorZ - 0.8, direction: -1, side: 'left' }); doorsArray.push({ mesh: doorLeftRight, baseZ: doorZ + 0.8, direction: 1, side: 'left' });
                }
                carGroup.position.z = c * carSpacing; train.add(carGroup);
            }
            train.position.set(0, 0, 0); scene.add(train);
        }

        // ================= 邏輯與控制 =================
        function operateDoors(action) {
            const holoBox = document.getElementById('hologram-box');
            if (!holoBox) return; 

            if (action === 'open' && perfectlyStopped) {
                targetDoorProgress = 1; 
                doorsClosedMsgShown = false;
                holoBox.innerHTML = `<div style="background: rgba(0,0,0,0.7); padding: 20px; border-radius: 15px; display: inline-block;"><svg viewBox="0 0 24 24" fill="none" stroke="#42a5f5" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" style="width: 80px; height: 80px;"><path d="M14 12h8m0 0l-3-3m3 3l-3 3M10 12H2m0 0l3-3m-3 3l3 3M10 4v16M14 4v16"/></svg><br><span style="color: #42a5f5; font-size: 24px;">趟門打開</span></div>`;
                setTimeout(() => { if(targetDoorProgress === 1) holoBox.innerHTML = ""; }, 1000);
                
                // 檢查是否尾站
                if (currentStationIndex >= stationsData.length - 1) {
                    document.getElementById('btn-next-station').style.display = "none";
                    document.getElementById('message-box').style.display = "none";
                    
                    // 顯示成功過關訊息
                    setTimeout(() => {
                        holoBox.innerHTML = `<div style="background: rgba(0,0,0,0.9); padding: 40px; border-radius: 20px; border: 4px solid #00ff00; box-shadow: 0 0 30px rgba(0,255,0,0.5);"><h1 style="color: #00ff00; margin: 0 0 10px 0; font-size: 40px;">🎉 旅程完成！</h1><p style="color: white; font-size: 24px; margin: 0;">成功抵達終點站：${stationsData[currentStationIndex].name}</p><p style="color: #aaa; margin-top: 20px;">系統將於 3 秒後返回主選單...</p></div>`;
                        
                        // 3秒後返回主畫面
                        setTimeout(() => {
                            resetToMainMenu();
                        }, 3000);
                    }, 1000);
                } else {
                    document.getElementById('btn-next-station').style.display = "none";
                }

            } else if (action === 'close') {
                targetDoorProgress = 0;
                holoBox.innerHTML = `<div style="background: rgba(0,0,0,0.7); padding: 20px; border-radius: 15px; display: inline-block;"><svg viewBox="0 0 24 24" fill="none" stroke="#ffa726" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round" style="width: 80px; height: 80px;"><path d="M22 12h-8m0 0l3-3m-3 3l3 3M2 12h8m0 0l-3-3m3 3l-3 3M10 4v16M14 4v16"/></svg><br><span style="color: #ffa726; font-size: 24px;">車門關閉</span></div>`;
                setTimeout(() => { if(targetDoorProgress === 0) holoBox.innerHTML = ""; }, 1000);
            }
        }
        
        function resetToMainMenu() {
            gameStarted = false;
            document.getElementById('hologram-box').innerHTML = "";
            document.getElementById('ui-layer').classList.remove('visible');
            document.getElementById('door-controls').style.display = 'none';
            document.getElementById('message-box').style.display = 'block';
            document.getElementById('message-box').innerText = "歡迎值班！請先開啟車門上客";
            document.getElementById('message-box').style.color = "white";
            
            leverValue = 0;
            const handle = document.getElementById('lever-handle');
            const rect = document.getElementById('lever-track').getBoundingClientRect();
            handle.style.top = (rect.height / 2) - 25 + 'px';
            
            const routeSelect = document.getElementById('route-select-screen');
            routeSelect.style.display = 'flex';
            setTimeout(() => { routeSelect.style.opacity = '1'; }, 50);
        }

        function goToNextStation() {
            if (currentStationIndex < stationsData.length - 1) {
                currentStationIndex++; updateHUDStation(); stopMarkZ = -12.5 - (currentStationIndex * 4000);
                document.getElementById('btn-next-station').style.display = "none"; document.getElementById('door-controls').style.display = "none";
                perfectlyStopped = false; isOverrun = false;
                document.getElementById('message-box').innerText = "請推動手柄前往下一站！"; document.getElementById('message-box').style.color = "white";
            }
        }

        function setupCameraControls() {
            const container = document.getElementById('game-container');
            const switchToManual = () => {
                if (cameraMode === 'passenger' || cameraMode === 'driver') return; 
                if(cameraMode !== 'manual') { cameraMode = 'manual'; const btn = document.getElementById('btn-camera'); btn.innerText = "🎥 手動視角"; btn.className = 'manual-mode'; }
            };

            container.addEventListener('mousedown', (e) => { isDragging = true; switchToManual(); previousMouseX = e.clientX; previousMouseY = e.clientY; });
            container.addEventListener('mousemove', (e) => {
                if (isDragging) { 
                    let deltaX = e.clientX - previousMouseX; let deltaY = e.clientY - previousMouseY;
                    if (cameraMode === 'passenger' || cameraMode === 'driver') {
                        passAngleX -= deltaX * 0.005; passAngleY -= deltaY * 0.005; 
                        if (passAngleY < -Math.PI/2) passAngleY = -Math.PI/2; if (passAngleY > Math.PI/2) passAngleY = Math.PI/2;
                    } else { camAngleX -= deltaX * 0.005; camAngleY += deltaY * 0.005; }
                    previousMouseX = e.clientX; previousMouseY = e.clientY; 
                }
            });
            container.addEventListener('mouseup', () => isDragging = false); container.addEventListener('mouseleave', () => isDragging = false);

            container.addEventListener('wheel', (e) => {
                if (cameraMode === 'passenger' || cameraMode === 'driver') {
                    currentFov += e.deltaY * 0.05; if (currentFov < 20) currentFov = 20; if (currentFov > 110) currentFov = 110; camera.fov = currentFov; camera.updateProjectionMatrix();
                } else {
                    switchToManual(); camRadius += e.deltaY * 0.05; if (camRadius < 10) camRadius = 10; if (camRadius > 200) camRadius = 200; 
                }
            }, { passive: true });

            let initialPinchDistance = null;
            container.addEventListener('touchstart', (e) => { 
                switchToManual(); 
                if (e.touches.length === 1) { isDragging = true; previousMouseX = e.touches[0].clientX; previousMouseY = e.touches[0].clientY; } 
                else if (e.touches.length === 2) { isDragging = false; initialPinchDistance = Math.hypot(e.touches[0].clientX - e.touches[1].clientX, e.touches[0].clientY - e.touches[1].clientY); }
            }, { passive: false });
            
            container.addEventListener('touchmove', (e) => {
                if(e.target.tagName !== 'BUTTON') e.preventDefault(); 
                if (isDragging && e.touches.length === 1) {
                    let deltaX = e.touches[0].clientX - previousMouseX; let deltaY = e.touches[0].clientY - previousMouseY;
                    if (cameraMode === 'passenger' || cameraMode === 'driver') {
                        passAngleX -= deltaX * 0.008; passAngleY -= deltaY * 0.008; 
                        if (passAngleY < -Math.PI/2) passAngleY = -Math.PI/2; if (passAngleY > Math.PI/2) passAngleY = Math.PI/2;
                    } else { camAngleX -= deltaX * 0.008; camAngleY += deltaY * 0.008; }
                    previousMouseX = e.touches[0].clientX; previousMouseY = e.touches[0].clientY;
                } else if (e.touches.length === 2 && initialPinchDistance !== null) {
                    let currentDistance = Math.hypot(e.touches[0].clientX - e.touches[1].clientX, e.touches[0].clientY - e.touches[1].clientY);
                    let delta = initialPinchDistance - currentDistance;
                    if (cameraMode === 'passenger' || cameraMode === 'driver') { currentFov += delta * 0.1; if (currentFov < 20) currentFov = 20; if (currentFov > 110) currentFov = 110; camera.fov = currentFov; camera.updateProjectionMatrix(); } 
                    else { camRadius += delta * 0.5; if (camRadius < 10) camRadius = 10; if (camRadius > 200) camRadius = 200; }
                    initialPinchDistance = currentDistance;
                }
            }, { passive: false });
            container.addEventListener('touchend', (e) => { if (e.touches.length < 2) initialPinchDistance = null; if (e.touches.length === 0) isDragging = false; });
        }

        window.toggleCameraMode = function() {
            const btn = document.getElementById('btn-camera');
            currentFov = 60; camera.fov = currentFov; camera.updateProjectionMatrix(); passAngleX = 0; passAngleY = 0;
            if (cameraMode === 'auto' || cameraMode === 'manual') { 
                cameraMode = 'driver'; btn.innerText = "👀 司機視角"; btn.className = 'driver-mode'; 
                if (frontGlass) frontGlass.visible = false; // 隱藏車頭牆壁
            } else if (cameraMode === 'driver') {
                cameraMode = 'passenger'; btn.innerText = "🚶 乘客視角"; btn.className = 'passenger-mode'; 
                if (frontGlass) frontGlass.visible = true; // 恢復顯示車頭牆壁
            } else { 
                cameraMode = 'auto'; btn.innerText = "🎥 自動視角"; btn.className = ''; 
                if (frontGlass) frontGlass.visible = true; // 恢復顯示車頭牆壁
            }
        };

        function setDirection(dir) {
            if (!gameStarted) return;
            if (Math.abs(speed) > 0.01) return; 
            currentDirection = dir;
            document.getElementById('btn-dir-fwd').className = dir === 1 ? "dir-btn active" : "dir-btn";
            document.getElementById('btn-dir-rev').className = dir === -1 ? "dir-btn active rev" : "dir-btn";
        }

        function setupLeverControl() {
            const handle = document.getElementById('lever-handle');
            const track = document.getElementById('lever-track');
            let isDraggingLever = false;

            function updateLever(clientY) {
                if (!gameStarted) return;
                const rect = track.getBoundingClientRect();
                let y = clientY - rect.top; y = Math.max(0, Math.min(y, rect.height)); 
                let rawValue = (y / (rect.height / 2)) - 1;
                if (Math.abs(rawValue) < 0.15) rawValue = 0;
                
                if (Math.abs(rawValue) > 0 && (doorProgress > 0.05 || targetDoorProgress > 0)) {
                    rawValue = 0; 
                }
                leverValue = rawValue;
                let handleY = (leverValue + 1) * (rect.height / 2) - 25; 
                handle.style.top = handleY + 'px';
            }

            handle.addEventListener('mousedown', () => { isDraggingLever = true; });
            window.addEventListener('mousemove', (e) => { if (isDraggingLever) updateLever(e.clientY); });
            window.addEventListener('mouseup', () => { isDraggingLever = false; });
            handle.addEventListener('touchstart', (e) => { isDraggingLever = true; e.preventDefault(); updateLever(e.touches[0].clientY); }, { passive: false });
            window.addEventListener('touchmove', (e) => { if (isDraggingLever) { if(e.target === handle || e.target === track) e.preventDefault(); updateLever(e.touches[0].clientY); } }, { passive: false });
            window.addEventListener('touchend', () => { isDraggingLever = false; });
        }

        function onWindowResize() { camera.aspect = window.innerWidth / window.innerHeight; camera.updateProjectionMatrix(); renderer.setSize(window.innerWidth, window.innerHeight); }

        function animate() {
            requestAnimationFrame(animate);

            if (driverLever3D) driverLever3D.rotation.x = (Math.PI / 4) + (leverValue * Math.PI / 8);

            if (doorProgress !== targetDoorProgress) {
                doorProgress += (targetDoorProgress - doorProgress) * 0.1;
                if (Math.abs(doorProgress - targetDoorProgress) < 0.01) {
                    doorProgress = targetDoorProgress;
                    if (doorProgress === 0 && perfectlyStopped && !doorsClosedMsgShown) {
                        doorsClosedMsgShown = true;
                        if (currentStationIndex < stationsData.length - 1) {
                            const msgBox = document.getElementById('message-box');
                            msgBox.innerText = "車門及幕門已關妥，請拉動手柄發車。";
                            msgBox.style.color = "white";
                            setTimeout(() => { if (doorsClosedMsgShown && msgBox.innerText.includes("發車")) msgBox.innerText = ""; }, 2000);
                            document.getElementById('btn-next-station').style.display = "block";
                        }
                    }
                }
                doorsArray.forEach(door => { if (door.side === 'right') door.mesh.position.z = door.baseZ + (doorProgress * 1.3 * door.direction); });
                psdDoorsArray.forEach(psd => { psd.mesh.position.z = psd.baseZ + (doorProgress * 1.3 * psd.direction); });
            }

            if (!gameStarted) {
                camAngleX += 0.002;
            } else {
                if (leverValue > 0) {
                    if (currentDirection === 1) { if (speed < 0) speed += 0.04; else speed += 0.015 * leverValue; } 
                    else { if (speed > 0) speed -= 0.04; else speed -= 0.012 * leverValue; }
                    perfectlyStopped = false; isOverrun = false; document.getElementById('door-controls').style.display = 'none';
                    if(Math.abs(speed) > 0.5 && document.getElementById('message-box').innerText.includes("發車")) document.getElementById('message-box').innerText = "";
                } else if (leverValue < 0) {
                    if (speed > 0) speed -= 0.04 * Math.abs(leverValue); else if (speed < 0) speed += 0.04 * Math.abs(leverValue);
                    if (Math.abs(speed) < 0.02) speed = 0; 
                } else { 
                    if (speed > 0) { speed -= 0.003; if (speed < 0) speed = 0; } else if (speed < 0) { speed += 0.003; if (speed > 0) speed = 0; }
                }

                if (speed > 4.5) speed = 4.5; if (speed < -1.5) speed = -1.5; if (Math.abs(speed) < 0.001) speed = 0; 
                train.position.z -= speed;
            }

            let trainHeadZ = train.position.z - 12.5;
            let distanceToStop = trainHeadZ - stopMarkZ; 

            if (cameraMode === 'driver') {
                let camX = train.position.x - 1.5;
                let camY = train.position.y + 3.8;
                let camZ = train.position.z - 10.5;
                camera.position.set(camX, camY, camZ);
                camera.lookAt(
                    camX - Math.sin(passAngleX) * Math.cos(passAngleY) * 10, 
                    camY + Math.sin(passAngleY) * 10, 
                    camZ - Math.cos(passAngleX) * Math.cos(passAngleY) * 10
                );
            } else if (cameraMode === 'passenger') {
                let camX = train.position.x - 1.8;
                let camY = train.position.y + 2.15;
                let camZ = train.position.z - 3.0;
                camera.position.set(camX, camY, camZ);
                camera.lookAt(
                    camX + Math.cos(passAngleX) * Math.cos(passAngleY) * 10, 
                    camY + Math.sin(passAngleY) * 10, 
                    camZ - Math.sin(passAngleX) * Math.cos(passAngleY) * 10
                );
            } else {
                if (cameraMode === 'auto' && gameStarted) {
                    let targetAngleX = 0.15; let targetAngleY = 0.3; let targetRadius = 85; 
                    if (speed < 1.5 && Math.abs(distanceToStop) < 180) { targetAngleX = -0.5; targetAngleY = 0.1; targetRadius = 45; }
                    camAngleX += (targetAngleX - camAngleX) * 0.05; camAngleY += (targetAngleY - camAngleY) * 0.05; camRadius += (targetRadius - camRadius) * 0.05;
                }
                if (camAngleY < -0.15) camAngleY = -0.15; if (camAngleY > Math.PI/2) camAngleY = Math.PI/2;
                
                let focusX = train.position.x; let focusY = train.position.y + 4; let focusZ = train.position.z - 10;
                let targetCamX = focusX + camRadius * Math.sin(camAngleX) * Math.cos(camAngleY); let targetCamY = focusY + camRadius * Math.sin(camAngleY); let targetCamZ = focusZ + camRadius * Math.cos(camAngleX) * Math.cos(camAngleY);
                if (targetCamY < 0.2) targetCamY = 0.2;
                camera.position.x += (targetCamX - camera.position.x) * 0.2; camera.position.y += (targetCamY - camera.position.y) * 0.2; camera.position.z += (targetCamZ - camera.position.z) * 0.2;
                
                if(!camera.currentLookAt) camera.currentLookAt = new THREE.Vector3(focusX, focusY, focusZ);
                camera.currentLookAt.x += (focusX - camera.currentLookAt.x) * 0.2; camera.currentLookAt.y += (focusY - camera.currentLookAt.y) * 0.2; camera.currentLookAt.z += (focusZ - camera.currentLookAt.z) * 0.2;
                camera.lookAt(camera.currentLookAt);
            }

            if (gameStarted) {
                document.getElementById('speed-display').innerText = Math.abs(speed * 30).toFixed(0); 
                const distEl = document.getElementById('dist-display');
                if (distanceToStop > 0) { distEl.innerText = distanceToStop.toFixed(0); distEl.className = distanceToStop < 300 ? "warning highlight" : "highlight"; } 
                else { distEl.innerText = distanceToStop.toFixed(0); distEl.className = "danger highlight"; }

                if (distanceToStop < -2.5) { 
                    isOverrun = true;
                } else {
                    isOverrun = false; 
                    if (speed === 0 && distanceToStop < 200 && !perfectlyStopped) {
                        if (distanceToStop <= 2.5 && distanceToStop >= -2.5) {
                            perfectlyStopped = true; train.position.z -= distanceToStop; distanceToStop = 0; 
                            document.getElementById('door-controls').style.display = 'flex';
                            
                            // 尾站提示
                            if (currentStationIndex >= stationsData.length - 1) {
                                document.getElementById('message-box').innerText = "🌟 抵達終點站！請開啟車門完成旅程";
                                document.getElementById('message-box').style.color = "#00ff00";
                            } else {
                                document.getElementById('message-box').innerText = "🌟 精準對齊停車！請開啟車門";
                                document.getElementById('message-box').style.color = "#00ff00";
                            }
                        }
                    }
                }
            }

            renderer.render(scene, camera);
        }

        window.onload = function() { init(); };
    </script>
</body>
</html>
