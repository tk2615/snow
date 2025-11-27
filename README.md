<html>
  <head>
    <meta charset="utf-8">
    <title>Snow AR Camera (Final Fix)</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0, viewport-fit=cover">
    <style>
      h1:first-of-type { display: none !important; }
      
      html, body {
        margin: 0; padding: 0; width: 100%; height: 100%;
        overflow: hidden; background-color: #000;
        font-family: sans-serif;
        overscroll-behavior: none;
      }

      /* ソース動画は非表示やけど、DOMには存在させる */
      .hidden-source {
        position: absolute; top: 0; left: 0; width: 1px; height: 1px;
        opacity: 0.001; pointer-events: none; z-index: -99;
      }

      #work-canvas {
        position: fixed; top: 50%; left: 50%;
        transform: translate(-50%, -50%);
        width: 100%; height: 100%;
        object-fit: cover; 
        z-index: 1; display: block;
      }

      /* --- UIパーツ --- */
      .icon-btn {
        position: fixed; top: 20px; z-index: 500;
        width: 44px; height: 44px;
        background: rgba(0, 0, 0, 0.3);
        border: 1px solid rgba(255, 255, 255, 0.5);
        border-radius: 50%; color: white; cursor: pointer;
        display: none; justify-content: center; align-items: center;
        backdrop-filter: blur(4px); -webkit-tap-highlight-color: transparent; 
        transition: background 0.2s;
      }
      .icon-btn:active { background: rgba(255, 255, 255, 0.3); }
      .icon-btn svg { width: 24px; height: 24px; fill: white; }

      #reload-btn { right: 20px; }
      #flip-btn { left: 20px; }

      /* スタート画面 */
      #start-screen {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background-color: rgba(0, 0, 0, 0.8);
        z-index: 3000;
        display: flex; flex-direction: column;
        justify-content: space-between; align-items: center;
        padding: 40px 20px; box-sizing: border-box;
        transition: opacity 0.5s ease;
      }

      #howto-container {
        flex: 1; width: 100%; display: flex;
        justify-content: center; align-items: center;
        overflow: hidden; margin-bottom: 20px;
      }
      #howto-img {
        width: 120%; height: 120%;
        object-fit: contain; filter: drop-shadow(0 0 10px rgba(0,0,0,0.5));
      }
      
      #start-btn {
        width: 60%; max-width: 300px; padding: 18px 0; 
        font-size: 20px; font-family: sans-serif;
        background: white; color: black;
        border: none; border-radius: 50px;
        cursor: pointer; font-weight: 900; letter-spacing: 2px;
        box-shadow: 0 4px 15px rgba(255,255,255,0.2);
        transition: transform 0.1s; margin-bottom: 40px; 
      }
      #start-btn:active { transform: scale(0.95); }
      #start-btn:disabled { background: #ccc; cursor: not-allowed; }

      #loading-msg {
        color: white; margin-bottom: 10px; font-size: 14px;
        display: none;
      }

      #error-overlay {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background-color: rgba(0,0,0,0.9); z-index: 4000;
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
        margin-bottom: 20px; object-fit: contain;
      }
      #preview-video { pointer-events: none; }

      .preview-text { font-size: 14px; margin-bottom: 20px; color: #ccc; }
      .preview-buttons { display: flex; gap: 20px; }
      .btn { padding: 12px 30px; border-radius: 30px; border: none; font-size: 16px; font-weight: bold; cursor: pointer; }
      .btn-save { background-color: white; color: black; }
      .btn-close { background-color: #333; color: white; border: 1px solid #555; }

      #shutter-container {
        position: fixed; bottom: 30px; left: 50%;
        transform: translateX(-50%);
        width: 80px; height: 80px;
        z-index: 100; cursor: pointer;
        -webkit-tap-highlight-color: transparent; user-select: none;
        display: none;
      }
      .progress-ring {
        position: absolute; top: 0; left: 0; width: 80px; height: 80px; transform: rotate(-90deg);
      }
      .progress-ring__circle {
        transition: stroke-dashoffset 0.1s linear; stroke: #ff3b30; stroke-width: 4; fill: transparent;
      }
      #shutter-btn {
        position: absolute; top: 10px; left: 10px; width: 60px; height: 60px;
        background-color: white; border-radius: 50%;
        transition: all 0.2s; display: flex; justify-content: center; align-items: center;
      }
      #camera-icon { width: 32px; height: 32px; fill: #333; transition: opacity 0.2s; }
      #shutter-container.recording #shutter-btn {
        width: 30px; height: 30px; top: 25px; left: 25px; border-radius: 4px; background-color: #ff3b30;
      }
      #shutter-container.recording #camera-icon { opacity: 0; }
      #flash {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background: white; opacity: 0; pointer-events: none; z-index: 200; transition: opacity 0.2s;
      }
    </style>
  </head>

  <body>

    <button id="reload-btn" class="icon-btn" onclick="location.reload()">
      <svg viewBox="0 0 24 24"><path d="M17.65 6.35C16.2 4.9 14.21 4 12 4c-4.42 0-7.99 3.58-7.99 8s3.57 8 7.99 8c3.73 0 6.84-2.55 7.73-6h-2.08c-.82 2.33-3.04 4-5.65 4-3.31 0-6-2.69-6-6s2.69-6 6-6c1.66 0 3.14.69 4.22 1.78L13 11h7V4l-2.35 2.35z"/></svg>
    </button>
    <button id="flip-btn" class="icon-btn">
      <svg viewBox="0 0 24 24"><path d="M20 4h-3.17L15 2H9L7.17 4H4c-1.1 0-2 .9-2 2v12c0 1.1.9 2 2 2h16c1.1 0 2-.9 2-2V6c0-1.1-.9-2-2-2zm-5 11.5V13H9v2.5L5.5 12 9 8.5V11h6V8.5l3.5 3.5-3.5 3.5z"/></svg>
    </button>

    <div id="start-screen">
      <div id="howto-container">
        <img id="howto-img" src="howto.png" alt="How to use">
      </div>
      <p id="loading-msg">Camera Initializing...</p>
      <button id="start-btn" disabled>LOADING...</button>
    </div>

    <div id="error-overlay">
      <h3 style="margin-bottom:10px;">Error</h3>
      <div id="error-text"></div>
      <button class="btn btn-close" onclick="location.reload()" style="margin-top:20px;">Retry</button>
    </div>

    <div id="preview-modal">
      <img id="preview-img">
      <video id="preview-video" autoplay loop playsinline muted></video>
      <p id="preview-msg-photo" class="preview-text">＊画像を長押しで保存してください</p>
      <div class="preview-buttons">
        <button id="btn-save-video" class="btn btn-save" style="display:none;">Download</button>
        <button id="btn-close" class="btn btn-close">Back</button>
      </div>
    </div>

    <video id="camera-feed" class="hidden-source" autoplay muted playsinline webkit-playsinline></video>
    
    <video id="snow-1" class="hidden-source" muted playsinline webkit-playsinline preload="auto"></video>
    <video id="snow-2" class="hidden-source" muted playsinline webkit-playsinline preload="auto"></video>

    <canvas id="work-canvas"></canvas>

    <div id="shutter-container">
      <svg class="progress-ring">
        <circle class="progress-ring__circle" stroke-dasharray="240 240" stroke-dashoffset="240" r="38" cx="40" cy="40"/>
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
      // DOM要素取得
      const cameraVideo = document.getElementById('camera-feed');
      const snowV1 = document.getElementById('snow-1');
      const snowV2 = document.getElementById('snow-2');
      const canvas = document.getElementById('work-canvas');
      const ctx = canvas.getContext('2d', { alpha: false, desynchronized: true });

      const startScreen = document.getElementById('start-screen');
      const startBtn = document.getElementById('start-btn');
      const loadingMsg = document.getElementById('loading-msg');
      const shutterContainer = document.getElementById('shutter-container');
      const flipBtn = document.getElementById('flip-btn');
      const reloadBtn = document.getElementById('reload-btn');

      // 変数
      let currentSnowVideo = snowV1;
      let nextSnowVideo = snowV2;
      // フェード時間1秒
      const FADE_DURATION = 1.0; 
      // 次の動画を再生開始するタイミング（終了の2秒前）
      const PRELOAD_TIME = 2.0;

      let currentFacingMode = 'environment';
      let cachedWidth = 0;
      let cachedHeight = 0;
      let needsResize = true;
      let snowBlobUrl = null;

      // 録画関連
      let mediaRecorder;
      let recordedChunks = [];
      let isRecording = false;
      let recordingStartTime;
      const progressCircle = document.querySelector('.progress-ring__circle');
      const radius = progressCircle.r.baseVal.value;
      const circumference = radius * 2 * Math.PI;
      progressCircle.style.strokeDasharray = `${circumference} ${circumference}`;
      progressCircle.style.strokeDashoffset = circumference;
      
      let isPressing = false;
      let pressTimer;
      let isLongPress = false;
      let shutterLock = false;

      // --- エラー表示 ---
      function showError(msg) {
        document.getElementById('error-overlay').style.display = 'flex';
        document.getElementById('error-text').textContent = msg;
        console.error(msg);
      }

      // -------------------------------------------------
      //  初期化フロー（ここを修正！）
      // -------------------------------------------------
      async function initApp() {
        loadingMsg.style.display = 'block';
        
        try {
          // 1. 動画の読み込み (Blob化)
          const response = await fetch('snow.mp4');
          if (!response.ok) throw new Error('snow.mp4 not found');
          const blob = await response.blob();
          snowBlobUrl = URL.createObjectURL(blob);
          
          snowV1.src = snowBlobUrl;
          snowV2.src = snowBlobUrl;
          snowV1.load();
          snowV2.load();

          // 2. カメラ初期化（ここではストリーム取得のみ）
          await initCameraStream(currentFacingMode);
          
          // 3. 準備完了したらSTARTボタン有効化
          startBtn.disabled = false;
          startBtn.textContent = "START";
          loadingMsg.style.display = 'none';

        } catch (err) {
          showError(err.message);
        }
      }

      // カメラストリームの取得とサイズ確定待ち
      async function initCameraStream(facingMode) {
        if (cameraVideo.srcObject) {
          cameraVideo.srcObject.getTracks().forEach(track => track.stop());
        }

        try {
          const stream = await navigator.mediaDevices.getUserMedia({
            video: { 
              facingMode: facingMode,
              width: { ideal: 1280 }, 
              height: { ideal: 720 } 
            },
            audio: false 
          });
          cameraVideo.srcObject = stream;
          
          // ★重要: iPhone等で確実にサイズを取得するための待機
          return new Promise((resolve) => {
            const checkSize = () => {
              if (cameraVideo.videoWidth > 0 && cameraVideo.videoHeight > 0) {
                needsResize = true;
                resolve();
              } else {
                requestAnimationFrame(checkSize);
              }
            };
            // 裏で再生しようとしてみる（ユーザー操作前なので失敗するかもだがサイズ取得には効く）
            cameraVideo.play().catch(()=>{});
            checkSize();
          });
        } catch (err) {
          throw new Error("カメラへのアクセスが拒否されました: " + err.message);
        }
      }

      window.onload = initApp;
      window.addEventListener('resize', () => { needsResize = true; });

      // -------------------------------------------------
      //  STARTボタンクリック（ユーザー操作起点）
      // -------------------------------------------------
      startBtn.addEventListener('click', async () => {
        // ここで確実に再生をキックする
        try {
          await cameraVideo.play();
          await snowV1.play();
          snowV2.pause();
          snowV2.currentTime = 0;
          
          startScreen.style.opacity = '0';
          setTimeout(() => { startScreen.style.display = 'none'; }, 500);
          
          shutterContainer.style.display = 'block';
          flipBtn.style.display = 'flex';
          reloadBtn.style.display = 'flex';
          
          updateDimensions();
          requestAnimationFrame(drawLoop);
          
        } catch (e) {
          showError("再生エラー: " + e.message);
        }
      });

      // カメラ反転
      flipBtn.addEventListener('click', async () => {
        currentFacingMode = (currentFacingMode === 'environment') ? 'user' : 'environment';
        flipBtn.style.opacity = 0.5;
        try { 
          await initCameraStream(currentFacingMode); 
          await cameraVideo.play(); // 反転後も再生
        } catch (err) { 
          console.error(err); 
        } finally { 
          flipBtn.style.opacity = 1; 
        }
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
        }
        needsResize = false;
      }

      // -------------------------------------------------
      //  描画ループ (先読み再生 & クロスフェード)
      // -------------------------------------------------
      function drawLoop() {
        requestAnimationFrame(drawLoop);

        if (needsResize || cachedWidth === 0) {
          updateDimensions();
          if (cachedWidth === 0) return;
        }

        const vw = cachedWidth;
        const vh = cachedHeight;

        // 1. カメラ描画
        ctx.globalCompositeOperation = 'source-over';
        ctx.drawImage(cameraVideo, 0, 0, vw, vh);

        // 2. 動画の制御
        const duration = currentSnowVideo.duration;
        const currentTime = currentSnowVideo.currentTime;

        // durationが取れていない(NaN)ときは再生だけしてリターン
        if (!duration || isNaN(duration)) return;

        const timeLeft = duration - currentTime;

        // ★カクつき対策: 「終了2秒前」になったら次の動画を再生開始（まだ見せない）
        if (timeLeft <= PRELOAD_TIME) {
          if (nextSnowVideo.paused) {
             nextSnowVideo.play().catch(()=>{});
          }
        }

        // ★クロスフェード処理
        if (timeLeft <= FADE_DURATION) {
          // アルファ計算 (1.0 -> 0.0)
          const alphaCurrent = Math.max(0, timeLeft / FADE_DURATION);
          const alphaNext = 1.0 - alphaCurrent;

          drawSnow(currentSnowVideo, vw, vh, 1.0); // ベースは消さない
          drawSnow(nextSnowVideo, vw, vh, alphaNext); // 次の動画を徐々に濃く

          // 動画の入れ替え（完全に終了 or 残り0.05秒）
          if (currentSnowVideo.ended || timeLeft <= 0.05) {
            const temp = currentSnowVideo;
            currentSnowVideo = nextSnowVideo;
            nextSnowVideo = temp;
            
            // 古い動画を停止＆巻き戻し
            nextSnowVideo.pause();
            nextSnowVideo.currentTime = 0;
          }
        } else {
          // 通常再生
          drawSnow(currentSnowVideo, vw, vh, 1.0);
        }
      }

      function drawSnow(video, cw, ch, alpha) {
        if (alpha <= 0.01) return;
        // videoWidthが0なら描画しない
        if (video.videoWidth === 0) return;

        ctx.globalCompositeOperation = 'screen';
        ctx.globalAlpha = alpha;

        const vw = video.videoWidth;
        const vh = video.videoHeight;
        
        const canvasAspect = cw / ch;
        const videoAspect = vw / vh;
        let sx, sy, sw, sh;

        if (canvasAspect > videoAspect) {
          sw = vw;
          sh = vw / canvasAspect;
          sx = 0;
          sy = (vh - sh) / 2;
        } else {
          sh = vh;
          sw = vh * canvasAspect;
          sx = (vw - sw) / 2;
          sy = 0;
        }

        ctx.drawImage(video, sx, sy, sw, sh, 0, 0, cw, ch);
        ctx.globalAlpha = 1.0;
      }

      // -------------------------------------------------
      //  撮影・UI機能
      // -------------------------------------------------
      const previewModal = document.getElementById('preview-modal');
      const previewImg = document.getElementById('preview-img');
      const previewVideo = document.getElementById('preview-video');
      const btnSaveVideo = document.getElementById('btn-save-video');

      function showPreview(type, url, filename) {
        if (window.currentPreviewUrl) URL.revokeObjectURL(window.currentPreviewUrl);
        window.currentPreviewUrl = url;
        previewModal.style.display = 'flex';
        shutterContainer.style.display = 'none';
        flipBtn.style.display = 'none';
        reloadBtn.style.display = 'none';

        if (type === 'photo') {
          previewImg.style.display = 'block';
          previewVideo.style.display = 'none';
          document.getElementById('preview-msg-photo').style.display = 'block';
          btnSaveVideo.style.display = 'none';
          previewImg.src = url;
        } else {
          previewImg.style.display = 'none';
          previewVideo.style.display = 'block';
          document.getElementById('preview-msg-photo').style.display = 'none';
          btnSaveVideo.style.display = 'block';
          previewVideo.src = url;
          previewVideo.play().catch(()=>{});
          btnSaveVideo.onclick = () => {
            const a = document.createElement('a');
            a.href = url;
            a.download = filename;
            document.body.appendChild(a);
            a.click();
            document.body.removeChild(a);
          };
        }
      }

      document.getElementById('btn-close').addEventListener('click', () => {
        previewModal.style.display = 'none';
        previewVideo.pause();
        shutterContainer.style.display = 'block';
        flipBtn.style.display = 'flex';
        reloadBtn.style.display = 'flex';
        // 戻ったら再生再開
        if(currentSnowVideo.paused) currentSnowVideo.play().catch(()=>{});
        if(cameraVideo.paused) cameraVideo.play().catch(()=>{});
      });

      // 撮影
      function takePhoto() {
        if (shutterLock) return;
        shutterLock = true;
        document.getElementById('flash').style.opacity = 1;
        setTimeout(() => document.getElementById('flash').style.opacity = 0, 200);
        const dataURL = canvas.toDataURL('image/png');
        showPreview('photo', dataURL);
        setTimeout(() => { shutterLock = false; }, 1000);
      }

      // 録画
      function startRecording() {
        isRecording = true;
        isLongPress = true;
        shutterContainer.classList.add('recording');
        recordingStartTime = Date.now();
        
        const stream = canvas.captureStream(30);
        const mimeTypes = ['video/mp4;codecs=avc1', 'video/mp4', 'video/webm;codecs=h264', 'video/webm'];
        const selectedMimeType = mimeTypes.find(type => MediaRecorder.isTypeSupported(type)) || '';
        
        try {
          mediaRecorder = new MediaRecorder(stream, selectedMimeType ? { mimeType: selectedMimeType } : undefined);
        } catch (e) { mediaRecorder = new MediaRecorder(stream); }
        
        mediaRecorder.ondataavailable = (e) => { if (e.data.size > 0) recordedChunks.push(e.data); };
        mediaRecorder.onstop = () => {
          const blob = new Blob(recordedChunks, { type: selectedMimeType || 'video/webm' });
          const url = URL.createObjectURL(blob);
          let ext = (selectedMimeType && selectedMimeType.includes('mp4')) ? 'mp4' : 'webm';
          showPreview('video', url, `snow_video_${Date.now()}.${ext}`);
          recordedChunks = [];
        };
        mediaRecorder.start();
        updateGauge();
      }

      function stopRecording() {
        isRecording = false;
        shutterContainer.classList.remove('recording');
        if (mediaRecorder && mediaRecorder.state !== 'inactive') mediaRecorder.stop();
        progressCircle.style.strokeDashoffset = circumference;
      }

      function updateGauge() {
        if (!isRecording) return;
        const progress = Math.min((Date.now() - recordingStartTime) / 10000, 1);
        progressCircle.style.strokeDashoffset = circumference - (progress * circumference);
        if (progress < 1) requestAnimationFrame(updateGauge); else stopRecording();
      }

      // シャッターボタンイベント
      const startPress = (e) => {
        if(e.cancelable) e.preventDefault();
        if (isPressing) return;
        isPressing = true;
        isLongPress = false;
        pressTimer = setTimeout(startRecording, 500);
      };
      const endPress = (e) => {
        if(e.cancelable) e.preventDefault();
        if (!isPressing) return;
        isPressing = false;
        clearTimeout(pressTimer);
        if (isRecording) stopRecording();
        else if (!isLongPress) takePhoto();
      };
      
      shutterContainer.addEventListener('mousedown', startPress);
      shutterContainer.addEventListener('touchstart', startPress, {passive: false});
      shutterContainer.addEventListener('mouseup', endPress);
      shutterContainer.addEventListener('mouseleave', endPress);
      shutterContainer.addEventListener('touchend', endPress, {passive: false});

      // バックグラウンド復帰
      document.addEventListener("visibilitychange", () => {
        if (document.visibilityState === "visible") {
          if (previewModal.style.display === 'none' && startScreen.style.display === 'none') {
             if (currentSnowVideo.paused) currentSnowVideo.play().catch(()=>{});
             if (cameraVideo.paused) cameraVideo.play().catch(()=>{});
          }
        }
      });
    </script>
  </body>
</html>
