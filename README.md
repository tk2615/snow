<html>
  <head>
    <meta charset="utf-8">
    <title>Snow AR Camera (Color LUTs)</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0, viewport-fit=cover">
    
    <link rel="preload" href="snow.mp4" as="video" type="video/mp4">

    <style>
      /* 最初の見出し（h1）を消す */
      h1:first-of-type { display: none !important; }
    </style>
    <style>
      /* --- 基本設定 --- */
      html, body {
        margin: 0; padding: 0; width: 100%; height: 100%;
        overflow: hidden; background-color: #000;
        font-family: sans-serif; overscroll-behavior: none;
      }
      .hidden-source {
        position: absolute; top: 0; left: 0; width: 10px; height: 10px;
        opacity: 0.01; pointer-events: none; z-index: -99;
      }
      #work-canvas {
        position: fixed; top: 50%; left: 50%;
        transform: translate(-50%, -50%);
        width: 100%; height: 100%;
        object-fit: cover; z-index: 1; display: block;
      }
      /* --- UIパーツ --- */
      .icon-btn {
        position: fixed; top: 20px; z-index: 500;
        width: 44px; height: 44px;
        background: rgba(0, 0, 0, 0.3);
        border: 1px solid rgba(255, 255, 255, 0.5);
        border-radius: 50%; color: white; cursor: pointer;
        display: flex; justify-content: center; align-items: center;
        backdrop-filter: blur(4px); -webkit-tap-highlight-color: transparent; 
        transition: background 0.2s; display: none; /* 初期は非表示 */
      }
      .icon-btn:active { background: rgba(255, 255, 255, 0.3); }
      .icon-btn svg { width: 24px; height: 24px; fill: white; }
      
      #reload-btn { right: 20px; }
      #flip-btn { left: 20px; }
      /* フィルターボタン（フリップの隣に追加） */
      #filter-btn { left: 80px; }

      /* 現在のフィルター名表示 */
      #filter-name-toast {
        position: fixed; top: 80px; left: 50%; transform: translateX(-50%);
        background: rgba(0,0,0,0.6); color: white; padding: 8px 16px;
        border-radius: 20px; font-size: 14px; font-weight: bold;
        pointer-events: none; opacity: 0; transition: opacity 0.3s;
        z-index: 600; letter-spacing: 1px;
      }

      /* スタート画面 */
      #start-screen {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background-color: rgba(0, 0, 0, 0.2); 
        backdrop-filter: blur(8px); 
        -webkit-backdrop-filter: blur(8px);
        z-index: 3000;
        display: flex; flex-direction: column;
        justify-content: space-between; align-items: center;
        padding: 40px 20px; box-sizing: border-box;
        transition: opacity 0.4s ease, transform 0.4s ease;
      }
      #howto-container {
        flex: 1; width: 100%; display: flex;
        justify-content: center; align-items: center;
        overflow: hidden; margin-bottom: 20px;
      }
      #howto-img {
        width: 120%; height: 120%;
        object-fit: contain; background-color: transparent;
        filter: drop-shadow(0 0 15px rgba(0,0,0,0.8));
      }
      
      #start-btn {
        width: 60%; max-width: 300px; padding: 18px 0; 
        font-size: 20px; font-family: sans-serif;
        background: rgba(255, 255, 255, 0.9); color: black;
        border: none; border-radius: 50px;
        cursor: pointer; font-weight: 900; letter-spacing: 2px;
        box-shadow: 0 4px 20px rgba(0,0,0,0.4);
        transition: transform 0.1s, background 0.3s; margin-bottom: 40px; 
      }
      #start-btn:active { transform: scale(0.95); }
      #start-btn:disabled {
        background: rgba(200, 200, 200, 0.5); color: #555; cursor: wait; transform: none; box-shadow: none;
      }
      
      #error-overlay {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background-color: rgba(0,0,0,0.8); z-index: 4000;
        display: none; flex-direction: column;
        justify-content: center; align-items: center;
        padding: 20px; color: #ff3b30; text-align: center;
      }
      #error-text { font-size: 16px; line-height: 1.5; color: white; }

      /* プレビュー画面 */
      #preview-modal {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background-color: rgba(0,0,0,0.95); z-index: 2000;
        display: none; flex-direction: column;
        justify-content: center; align-items: center; color: white;
      }
      #preview-img, #preview-video {
        max-width: 90%; max-height: 75%;
        border-radius: 8px; box-shadow: 0 0 20px rgba(0,0,0,0.5);
        margin-bottom: 30px; object-fit: contain;
      }
      #preview-video { pointer-events: auto; }
      
      .preview-text { font-size: 14px; margin-bottom: 20px; color: #ccc; }
      .preview-buttons { display: flex; gap: 20px; }
      .btn { padding: 12px 30px; border-radius: 30px; border: none; font-size: 16px; font-weight: bold; cursor: pointer; }
      .btn-save { background-color: white; color: black; }
      .btn-close { background-color: #333; color: white; border: 1px solid #555; }

      /* シャッターボタン */
      #shutter-container {
        position: fixed; bottom: 30px; left: 50%; transform: translateX(-50%);
        bottom: calc(30px + env(safe-area-inset-bottom));
        width: 104px; height: 104px; z-index: 100; cursor: pointer;
        -webkit-tap-highlight-color: transparent; user-select: none; display: none;
      }
      .progress-ring {
        position: absolute; top: 0; left: 0; width: 104px; height: 104px; transform: rotate(-90deg);
      }
      .progress-ring__circle {
        transition: stroke-dashoffset 0.1s linear; stroke: #ff3b30; stroke-width: 4; fill: transparent;
      }
      #shutter-btn {
        position: absolute; top: 13px; left: 13px; width: 78px; height: 78px;
        background-color: white; border-radius: 50%;
        transition: all 0.2s; display: flex; justify-content: center; align-items: center;
      }
      #camera-icon { width: 42px; height: 42px; fill: #333; transition: opacity 0.2s; }
      #shutter-container.recording #shutter-btn {
        width: 40px; height: 40px; top: 32px; left: 32px; border-radius: 4px; background-color: #ff3b30;
      }
      #shutter-container.recording #camera-icon { opacity: 0; }
      
      #flash {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background: white; opacity: 0; pointer-events: none; z-index: 200; transition: opacity 0.2s;
      }

      /* ログ画面 */
      #admin-log {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background-color: rgba(0, 0, 0, 0.85);
        color: #00ff00; font-family: "Courier New", Courier, monospace;
        z-index: 9999; display: none;
        flex-direction: column; justify-content: center; align-items: center;
        padding: 40px; box-sizing: border-box;
      }
      #admin-log h2 { border-bottom: 2px solid #00ff00; padding-bottom: 10px; margin-bottom: 20px; }
      .log-section { margin-bottom: 30px; width: 100%; max-width: 600px; }
      .log-row { display: flex; justify-content: space-between; border-bottom: 1px dashed #333; padding: 10px 0; font-size: 18px; }
      .log-row span:last-child { font-weight: bold; }
      .close-log {
        margin-top: 30px; padding: 10px 30px; background: transparent; 
        border: 2px solid #00ff00; color: #00ff00; font-size: 16px; cursor: pointer;
      }
      .close-log:hover { background: #00ff00; color: black; }
    </style>
  </head>
  <body>
    <div id="admin-log">
      <h2>ADMIN LOG (Local Device)</h2>
      <div class="log-section">
        <h3>[ TOTAL (All Time) ]</h3>
        <div class="log-row"><span>Access:</span><span id="log-total-access">0</span></div>
        <div class="log-row"><span>Photo Taken:</span><span id="log-total-photo">0</span></div>
        <div class="log-row"><span>Video Taken:</span><span id="log-total-video">0</span></div>
      </div>
      <div class="log-section">
        <h3>[ TODAY (<span id="log-date"></span>) ]</h3>
        <div class="log-row"><span>Access:</span><span id="log-today-access">0</span></div>
        <div class="log-row"><span>Photo Taken:</span><span id="log-today-photo">0</span></div>
        <div class="log-row"><span>Video Taken:</span><span id="log-today-video">0</span></div>
      </div>
      <button class="close-log" onclick="toggleLog()">CLOSE (Cmd+K)</button>
    </div>

    <button id="reload-btn" class="icon-btn">
      <svg viewBox="0 0 24 24"><path d="M17.65 6.35C16.2 4.9 14.21 4 12 4c-4.42 0-7.99 3.58-7.99 8s3.57 8 7.99 8c3.73 0 6.84-2.55 7.73-6h-2.08c-.82 2.33-3.04 4-5.65 4-3.31 0-6-2.69-6-6s2.69-6 6-6c1.66 0 3.14.69 4.22 1.78L13 11h7V4l-2.35 2.35z"/></svg>
    </button>
    <button id="flip-btn" class="icon-btn">
      <svg viewBox="0 0 24 24"><path d="M20 4h-3.17L15 2H9L7.17 4H4c-1.1 0-2 .9-2 2v12c0 1.1.9 2 2 2h16c1.1 0 2-.9 2-2V6c0-1.1-.9-2-2-2zm-5 11.5V13H9v2.5L5.5 12 9 8.5V11h6V8.5l3.5 3.5-3.5 3.5z"/></svg>
    </button>
    
    <button id="filter-btn" class="icon-btn">
      <svg viewBox="0 0 24 24"><path d="M7.5 5.6L10 7 8.6 4.5 10 2 7.5 3.4 5 2 6.4 4.5 5 7zM19.5 15.4L17 14l1.4 2.5L17 19l2.5-1.4L22 19l-1.4-2.5L22 14zM22 2l-2.5 1.4L17 2l1.4 2.5L17 7l2.5-1.4L22 7l-1.4-2.5zm-8.8 9.2L7.83 6.42 5 16.17l9.75-2.83zM6.69 14.89l1.63-5.62 3.99 3.99-5.62 1.63z"/></svg>
    </button>

    <div id="filter-name-toast">Normal</div>

    <div id="start-screen">
      <div id="howto-container">
        <img id="howto-img" src="howto.png" alt="How to use">
      </div>
      <button id="start-btn" disabled>Loading...</button>
    </div>

    <div id="error-overlay">
      <h3 style="margin-bottom:10px;">Error</h3>
      <div id="error-text"></div>
      <button class="btn btn-close" onclick="location.reload()" style="margin-top:20px;">Retry</button>
    </div>

    <div id="preview-modal">
      <img id="preview-img">
      <video id="preview-video" controls playsinline></video>
      <p id="preview-msg-photo" class="preview-text">＊画像を長押しで保存してください</p>
      <div class="preview-buttons">
        <button id="btn-save-video" class="btn btn-save" style="display:none;">Download</button>
        <button id="btn-close" class="btn btn-close">Back</button>
      </div>
    </div>

    <video id="camera-feed" class="hidden-source" autoplay muted playsinline></video>
    <video id="snow-1" class="hidden-source" src="snow.mp4" muted playsinline webkit-playsinline preload="auto"></video>
    <video id="snow-2" class="hidden-source" src="snow.mp4" muted playsinline webkit-playsinline preload="auto"></video>

    <canvas id="work-canvas"></canvas>

    <div id="shutter-container">
      <svg class="progress-ring">
        <circle class="progress-ring__circle" stroke-dasharray="314 314" stroke-dashoffset="314" r="50" cx="52" cy="52"/>
      </svg>
      <div id="shutter-btn">
        <svg id="camera-icon" viewBox="0 0 24 24">
          <path d="M12 12c2.21 0 4-1.79 4-4s-1.79-4-4-4-4 1.79-4 4 1.79 4 4 4zm0 2c-2.67 0-8 1.34-8 4v2h16v-2c0-2.66-5.33-4-8-4z" opacity="0"/>
          <path d="M9 2L7.17 4H4c-1.1 0-2 .9-2 2v12c0 1.1.9 2 2 2h16c1.1 0 2-.9 2-2V6c0-1.1-.9-2-2-2h-3.17L15 2H9zm3 15c-2.76 0-5-2.24-5-5s2.24-5 5-5 5 2.24 5 5-2.24 5-5 5z"/>
        </svg>
      </div>
    </div>
    <div id="flash"></div>

    <script>
      // ログ管理
      const StatsManager = {
        key: 'snow_ar_stats_v1',
        data: { total: { access: 0, photo: 0, video: 0 }, days: {} },
        init() {
          const stored = localStorage.getItem(this.key);
          if (stored) this.data = JSON.parse(stored);
          this.increment('access');
        },
        save() {
          localStorage.setItem(this.key, JSON.stringify(this.data));
          this.updateUI();
        },
        getTodayStr() {
          const d = new Date();
          return `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`;
        },
        increment(type) {
          if (!this.data.total[type]) this.data.total[type] = 0;
          this.data.total[type]++;
          const today = this.getTodayStr();
          if (!this.data.days[today]) this.data.days[today] = { access: 0, photo: 0, video: 0 };
          if (!this.data.days[today][type]) this.data.days[today][type] = 0;
          this.data.days[today][type]++;
          this.save();
        },
        updateUI() {
          const today = this.getTodayStr();
          const todayData = this.data.days[today] || { access: 0, photo: 0, video: 0 };
          document.getElementById('log-total-access').textContent = this.data.total.access;
          document.getElementById('log-total-photo').textContent = this.data.total.photo;
          document.getElementById('log-total-video').textContent = this.data.total.video;
          document.getElementById('log-date').textContent = today;
          document.getElementById('log-today-access').textContent = todayData.access;
          document.getElementById('log-today-photo').textContent = todayData.photo;
          document.getElementById('log-today-video').textContent = todayData.video;
        }
      };

      document.addEventListener('keydown', (e) => {
        if ((e.metaKey || e.ctrlKey) && e.key === 'k') {
          e.preventDefault();
          toggleLog();
        }
      });
      function toggleLog() {
        const log = document.getElementById('admin-log');
        if (log.style.display === 'flex') { log.style.display = 'none'; } 
        else { StatsManager.updateUI(); log.style.display = 'flex'; }
      }

      // --- フィルター設定 (LUT風) ---
      const filterBtn = document.getElementById('filter-btn');
      const filterNameToast = document.getElementById('filter-name-toast');
      
      const filters = [
        { name: "Normal", value: "none" },
        { name: "Retro", value: "sepia(0.4) contrast(1.1) brightness(0.9) saturate(0.8)" },
        { name: "Film", value: "contrast(1.2) saturate(1.2) brightness(1.1)" },
        { name: "B&W", value: "grayscale(1) contrast(1.2)" },
        { name: "Cool", value: "saturate(0.9) hue-rotate(10deg) contrast(1.1) brightness(1.05)" },
        { name: "Warm", value: "sepia(0.2) saturate(1.3) contrast(1.05)" }
      ];
      
      let currentFilterIndex = 0;
      
      filterBtn.addEventListener('click', () => {
        currentFilterIndex = (currentFilterIndex + 1) % filters.length;
        const currentFilter = filters[currentFilterIndex];
        
        // トースト表示
        filterNameToast.textContent = currentFilter.name;
        filterNameToast.style.opacity = 1;
        setTimeout(() => { filterNameToast.style.opacity = 0; }, 1500);
      });

      const cameraVideo = document.getElementById('camera-feed');
      const snowV1 = document.getElementById('snow-1');
      const snowV2 = document.getElementById('snow-2');
      let currentSnowVideo = snowV1;
      let nextSnowVideo = snowV2;
      const FADE_DURATION = 1.0; 

      const canvas = document.getElementById('work-canvas');
      const ctx = canvas.getContext('2d', { alpha: false, desynchronized: true });
      const bufferCanvas = document.createElement('canvas');
      const bufferCtx = bufferCanvas.getContext('2d', { alpha: false, desynchronized: true });

      const shutterContainer = document.getElementById('shutter-container');
      const progressCircle = document.querySelector('.progress-ring__circle');
      const startScreen = document.getElementById('start-screen');
      const startBtn = document.getElementById('start-btn');
      const flipBtn = document.getElementById('flip-btn');
      const reloadBtn = document.getElementById('reload-btn');

      const previewModal = document.getElementById('preview-modal');
      const previewImg = document.getElementById('preview-img');
      const previewVideo = document.getElementById('preview-video');
      const previewMsgPhoto = document.getElementById('preview-msg-photo');
      const btnSaveVideo = document.getElementById('btn-save-video');
      const btnClose = document.getElementById('btn-close');
      const errorOverlay = document.getElementById('error-overlay');
      const errorText = document.getElementById('error-text');
      
      let currentPreviewUrl = null;
      const radius = progressCircle.r.baseVal.value;
      const circumference = radius * 2 * Math.PI;
      progressCircle.style.strokeDasharray = `${circumference} ${circumference}`;
      progressCircle.style.strokeDashoffset = circumference;

      let mediaRecorder;
      let recordedChunks = [];
      let isRecording = false;
      let recordingStartTime;
      let pressTimer;
      let isLongPress = false;
      let isPressing = false;
      let shutterLock = false; 
      const LONG_PRESS_DURATION = 500;

      let currentFacingMode = 'environment';
      let cachedWidth = 0;
      let cachedHeight = 0;
      let needsResize = true;
      let lastFrameTime = 0;
      const TARGET_FPS = 30; 
      const FRAME_INTERVAL = 1000 / TARGET_FPS;

      function showError(msg) {
        errorOverlay.style.display = 'flex';
        errorText.textContent = msg;
        console.error(msg);
      }

      async function initApp() {
        try {
          StatsManager.init();
          startBtn.textContent = "Loading...";
          startBtn.disabled = true;

          const waitForVideo = (video) => {
            return new Promise((resolve) => {
              if (video.readyState >= 3) return resolve();
              const onCanPlay = () => {
                video.removeEventListener('canplay', onCanPlay);
                resolve();
              };
              video.addEventListener('canplay', onCanPlay);
              video.onerror = () => { console.warn("Video Warning"); resolve(); };
              if(!video.src) video.src = "snow.mp4";
              video.load();
            });
          };

          const timeoutPromise = new Promise(resolve => setTimeout(resolve, 3000));
          const startupTasks = Promise.all([
            waitForVideo(snowV1),
            waitForVideo(snowV2),
            initCamera(currentFacingMode)
          ]);

          await Promise.race([startupTasks, timeoutPromise]);

          updateDimensions();
          drawCompositeFrame(0);

          startBtn.textContent = "START";
          startBtn.disabled = false;
          snowV1.play().catch(e => console.log("Auto-play blocked"));

        } catch (err) {
          showError("Error: " + err.message);
          startBtn.textContent = "Error";
        }
      }

      document.addEventListener('DOMContentLoaded', initApp);
      window.addEventListener('resize', () => { needsResize = true; });

      reloadBtn.addEventListener('click', () => {
        if (cameraVideo.srcObject) {
          cameraVideo.srcObject.getTracks().forEach(track => track.stop());
        }
        setTimeout(() => { window.location.reload(); }, 100);
      });

      startBtn.addEventListener('click', () => {
        if(currentSnowVideo.paused) currentSnowVideo.play().catch(e => console.warn(e));
        startScreen.style.opacity = '0';
        startScreen.style.transform = 'scale(1.1)';
        setTimeout(() => { startScreen.style.display = 'none'; }, 400);
        
        shutterContainer.style.display = 'block';
        flipBtn.style.display = 'flex';
        reloadBtn.style.display = 'flex';
        filterBtn.style.display = 'flex'; // フィルターボタン表示
      });

      async function initCamera(facingMode) {
        if (cameraVideo.srcObject) {
          cameraVideo.srcObject.getTracks().forEach(track => track.stop());
        }
        let stream = null;
        try {
          stream = await navigator.mediaDevices.getUserMedia({
            video: { facingMode: facingMode, width: { ideal: 1280 }, height: { ideal: 720 } },
            audio: true 
          });
        } catch (err) {
          try {
             stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: facingMode }, audio: false });
          } catch(e) { throw e; }
        }
        cameraVideo.srcObject = stream;
        return new Promise((resolve) => {
          cameraVideo.onloadedmetadata = () => {
            needsResize = true;
            resolve();
          };
        });
      }

      flipBtn.addEventListener('click', async () => {
        currentFacingMode = (currentFacingMode === 'environment') ? 'user' : 'environment';
        flipBtn.style.pointerEvents = 'none';
        flipBtn.style.opacity = 0.5;
        try { await initCamera(currentFacingMode); } catch (err) { console.error(err); } 
        finally { flipBtn.style.pointerEvents = 'auto'; flipBtn.style.opacity = 1; }
      });

      function updateDimensions() {
        const vw = cameraVideo.videoWidth;
        const vh = cameraVideo.videoHeight;
        if (vw === 0 || vh === 0) return;
        cachedWidth = vw;
        cachedHeight = vh;
        if (canvas.width !== vw || canvas.height !== vh) {
          canvas.width = vw;
          canvas.height = vh;
          bufferCanvas.width = vw;
          bufferCanvas.height = vh;
        }
        needsResize = false;
      }

      function drawCompositeFrame(timestamp) {
        requestAnimationFrame(drawCompositeFrame);
        const elapsed = timestamp - lastFrameTime;
        if (elapsed < FRAME_INTERVAL) return;
        lastFrameTime = timestamp - (elapsed % FRAME_INTERVAL);

        if (needsResize || cachedWidth === 0) {
          updateDimensions();
          if (cachedWidth === 0) return;
        }

        const vw = cachedWidth;
        const vh = cachedHeight;
        
        bufferCtx.globalCompositeOperation = 'source-over';
        bufferCtx.fillStyle = '#000000';
        bufferCtx.fillRect(0, 0, vw, vh);

        const duration = currentSnowVideo.duration;
        const currentTime = currentSnowVideo.currentTime;

        if (duration && duration > 0) {
          const timeLeft = duration - currentTime;
          if (timeLeft <= FADE_DURATION) {
            if (nextSnowVideo.paused) {
              nextSnowVideo.currentTime = 0;
              nextSnowVideo.play().catch(()=>{});
            }
            const alphaCurrent = Math.max(0, timeLeft / FADE_DURATION);
            const alphaNext = 1.0 - alphaCurrent;
            drawSnowToBuffer(currentSnowVideo, vw, vh, alphaCurrent);
            drawSnowToBuffer(nextSnowVideo, vw, vh, alphaNext);
          } else {
            drawSnowToBuffer(currentSnowVideo, vw, vh, 1.0);
            if (!nextSnowVideo.paused) {
              nextSnowVideo.pause();
              nextSnowVideo.currentTime = 0;
            }
          }
          if (currentSnowVideo.ended || timeLeft <= 0) {
            const temp = currentSnowVideo;
            currentSnowVideo = nextSnowVideo;
            nextSnowVideo = temp;
            nextSnowVideo.pause();
            nextSnowVideo.currentTime = 0;
            if(currentSnowVideo.paused) currentSnowVideo.play().catch(()=>{});
          }
        } else {
           if(currentSnowVideo.paused) currentSnowVideo.play().catch(()=>{});
        }

        // --- 描画 (フィルター適用) ---
        // 1. フィルターを適用してカメラ映像を描画
        ctx.globalCompositeOperation = 'source-over';
        ctx.filter = filters[currentFilterIndex].value;
        ctx.drawImage(cameraVideo, 0, 0, vw, vh);
        
        // 2. フィルターをリセットして雪を描画
        ctx.filter = 'none';
        ctx.globalCompositeOperation = 'screen';
        ctx.drawImage(bufferCanvas, 0, 0);
      }

      function drawSnowToBuffer(video, cw, ch, alpha) {
        if (alpha <= 0.01) return;
        const videoW = video.videoWidth;
        const videoH = video.videoHeight;
        if (videoW === 0 || videoH === 0) return;

        const canvasAspect = cw / ch;
        const videoAspect = videoW / videoH;
        let sx, sy, sw, sh;
        if (canvasAspect > videoAspect) {
          sw = videoW; sh = videoW / canvasAspect; sx = 0; sy = (videoH - sh) / 2;
