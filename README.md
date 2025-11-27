<html>
  <head>
    <meta charset="utf-8">
    <title>Snow AR Camera (Native FOV)</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0, viewport-fit=cover">

    <style>
      /* --- 基本設定 --- */
      html, body {
        margin: 0; padding: 0; width: 100%; height: 100%;
        overflow: hidden; background-color: #000; /* 余白は黒 */
        font-family: sans-serif;
        overscroll-behavior: none;
      }

      /* 素材（非表示） */
      .hidden-source {
        position: absolute; top: 0; left: 0;
        width: 10px; height: 10px;
        opacity: 0.01;
        pointer-events: none;
        z-index: -99;
      }

      /* メイン表示＆録画用キャンバス */
      #work-canvas {
        /* 画面中央に配置 */
        position: fixed;
        top: 50%; left: 50%;
        transform: translate(-50%, -50%);
        /* 画面からはみ出さないように最大サイズを制限 */
        max-width: 100%;
        max-height: 100%;
        /* アスペクト比を維持して収める */
        object-fit: contain; 
        z-index: 1;
      }

      /* --- UIパーツ --- */
      .icon-btn {
        position: fixed; top: 20px;
        z-index: 500;
        width: 44px; height: 44px;
        background: rgba(0, 0, 0, 0.3);
        border: 1px solid rgba(255, 255, 255, 0.5);
        border-radius: 50%;
        color: white;
        cursor: pointer;
        display: flex; justify-content: center; align-items: center;
        backdrop-filter: blur(4px);
        -webkit-tap-highlight-color: transparent; 
        transition: background 0.2s;
      }
      .icon-btn:active { background: rgba(255, 255, 255, 0.3); }
      .icon-btn svg { width: 24px; height: 24px; fill: white; }

      #reload-btn { right: 20px; }
      #flip-btn { left: 20px; display: none; }

      /* スタート画面 */
      #start-screen {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background-color: rgba(0,0,0,0.9);
        z-index: 999;
        display: flex; justify-content: center; align-items: center; flex-direction: column;
        color: white;
        padding: 20px;
        box-sizing: border-box;
      }
      #start-btn {
        padding: 15px 40px; font-size: 18px;
        background: #ff3b30; color: white;
        border: none; border-radius: 30px;
        cursor: pointer; font-weight: bold;
        margin-bottom: 20px;
      }
      #status-msg { 
        color: #ffffff; font-size: 16px; text-align: center; line-height: 1.5;
        white-space: pre-wrap;
      }

      /* プレビュー画面 */
      #preview-modal {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background-color: rgba(0,0,0,0.95);
        z-index: 2000;
        display: none;
        flex-direction: column;
        justify-content: center; align-items: center;
        color: white;
      }
      #preview-img, #preview-video {
        max-width: 90%; max-height: 70%;
        border-radius: 8px; box-shadow: 0 0 20px rgba(0,0,0,0.5);
        margin-bottom: 20px; object-fit: contain;
      }
      .preview-text { font-size: 14px; margin-bottom: 20px; color: #ccc; }
      .preview-buttons { display: flex; gap: 20px; }
      .btn { padding: 12px 30px; border-radius: 25px; border: none; font-size: 16px; font-weight: bold; cursor: pointer; }
      .btn-save { background-color: #ff3b30; color: white; }
      .btn-close { background-color: #555; color: white; }


      /* シャッターボタン */
      #shutter-container {
        position: fixed; bottom: 30px; left: 50%;
        transform: translateX(-50%);
        width: 80px; height: 80px;
        z-index: 100;
        cursor: pointer;
        -webkit-tap-highlight-color: transparent; 
        user-select: none;
        display: none;
      }
      .progress-ring {
        position: absolute; top: 0; left: 0;
        width: 80px; height: 80px;
        transform: rotate(-90deg);
      }
      .progress-ring__circle {
        transition: stroke-dashoffset 0.1s linear;
        stroke: #ff3b30; stroke-width: 4; fill: transparent;
      }
      #shutter-btn {
        position: absolute; top: 10px; left: 10px;
        width: 60px; height: 60px;
        background-color: white; border-radius: 50%;
        transition: all 0.2s;
        display: flex; justify-content: center; align-items: center;
      }
      #camera-icon {
        width: 32px; height: 32px;
        fill: #333;
        transition: opacity 0.2s;
      }
      #shutter-container.recording #shutter-btn {
        width: 30px; height: 30px; top: 25px; left: 25px;
        border-radius: 4px; background-color: #ff3b30;
      }
      #shutter-container.recording #camera-icon {
        opacity: 0;
      }
      #flash {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background: white; opacity: 0; pointer-events: none; z-index: 200;
        transition: opacity 0.2s;
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
      <button id="start-btn">カメラを起動する</button>
      <div id="status-msg"></div>
    </div>

    <div id="preview-modal">
      <img id="preview-img">
      <video id="preview-video" controls playsinline></video>
      <p id="preview-msg-photo" class="preview-text">画像長押しで保存してください</p>
      <div class="preview-buttons">
        <button id="btn-save-video" class="btn btn-save" style="display:none;">動画を保存</button>
        <button id="btn-close" class="btn btn-close">閉じる</button>
      </div>
    </div>

    <video id="camera-feed" class="hidden-source" autoplay muted playsinline></video>
    <video id="snow-1" class="hidden-source" src="snow.mp4" muted playsinline webkit-playsinline></video>
    <video id="snow-2" class="hidden-source" src="snow.mp4" muted playsinline webkit-playsinline></video>

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
      // ==========================================
      // DOM & 変数
      // ==========================================
      const cameraVideo = document.getElementById('camera-feed');
      const snowV1 = document.getElementById('snow-1');
      const snowV2 = document.getElementById('snow-2');
      let currentSnowVideo = snowV1;
      let nextSnowVideo = snowV2;
      const FADE_DURATION = 1.0;

      const canvas = document.getElementById('work-canvas');
      const ctx = canvas.getContext('2d');
      const bufferCanvas = document.createElement('canvas');
      const bufferCtx = bufferCanvas.getContext('2d');

      const shutterContainer = document.getElementById('shutter-container');
      const progressCircle = document.querySelector('.progress-ring__circle');
      const startScreen = document.getElementById('start-screen');
      const startBtn = document.getElementById('start-btn');
      const statusMsg = document.getElementById('status-msg');
      const flipBtn = document.getElementById('flip-btn');

      const previewModal = document.getElementById('preview-modal');
      const previewImg = document.getElementById('preview-img');
      const previewVideo = document.getElementById('preview-video');
      const previewMsgPhoto = document.getElementById('preview-msg-photo');
      const btnSaveVideo = document.getElementById('btn-save-video');
      const btnClose = document.getElementById('btn-close');
      
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

      function log(msg) {
        statusMsg.innerHTML = msg;
        console.log(msg);
      }

      // ==========================================
      // カメラ初期化 (最大画角・ネイティブ解像度)
      // ==========================================
      async function initCamera(facingMode) {
        if (cameraVideo.srcObject) {
          cameraVideo.srcObject.getTracks().forEach(track => track.stop());
        }

        let stream = null;
        try {
          log(`カメラ起動中 (${facingMode === 'user' ? '自撮り' : '外向き'})...`);
          
          // ★変更点：理想のWidth/Heightを指定せず、センサーのネイティブ比率を優先する
          // ただし画質は落としたくないので、idealに大きい数字を指定して「高画質」だけ要求する
          stream = await navigator.mediaDevices.getUserMedia({
            video: { 
              facingMode: facingMode,
              // 特定のアスペクト比(1280x720など)を強要しないことで、センサー全域(4:3など)を取得させる
              width: { ideal: 4096 }, 
              height: { ideal: 2160 } 
            },
            audio: false 
          });
        } catch (err) {
          log("高画質指定失敗...標準設定で再試行");
          try {
             stream = await navigator.mediaDevices.getUserMedia({
              video: { facingMode: facingMode }, // 何も指定しない
              audio: false 
            });
          } catch(e) {
            throw e;
          }
        }

        cameraVideo.srcObject = stream;
        return new Promise((resolve) => {
          cameraVideo.onloadedmetadata = () => {
            resolve();
          };
        });
      }

      flipBtn.addEventListener('click', async () => {
        currentFacingMode = (currentFacingMode === 'environment') ? 'user' : 'environment';
        flipBtn.style.pointerEvents = 'none';
        flipBtn.style.opacity = 0.5;
        try {
          await initCamera(currentFacingMode);
          log("");
        } catch (err) {
          log("カメラ切り替えエラー: " + err.message);
        } finally {
          flipBtn.style.pointerEvents = 'auto';
          flipBtn.style.opacity = 1;
        }
      });

      // ==========================================
      // 起動
      // ==========================================
      startBtn.addEventListener('click', async () => {
        startBtn.disabled = true;
        startBtn.textContent = "起動中...";
        log("初期化中...");

        try {
          snowV1.loop = false;
          snowV2.loop = false;
          await snowV1.play();
          
          await initCamera(currentFacingMode);

          startScreen.style.display = 'none';
          shutterContainer.style.display = 'block';
          flipBtn.style.display = 'flex';
          
          drawCompositeFrame(); 

        } catch (err) {
          console.error(err);
          startBtn.disabled = false;
          startBtn.textContent = "再試行";
          log("エラー: " + err.message);
        }
      });

      // ==========================================
      // 描画ループ (映像全体を表示＝黒帯あり)
      // ==========================================
      function drawCompositeFrame() {
        // カメラ映像の生サイズ
        const vw = cameraVideo.videoWidth;
        const vh = cameraVideo.videoHeight;

        if (vw === 0 || vh === 0) {
           requestAnimationFrame(drawCompositeFrame);
           return;
        }

        // Canvasの表示サイズ（ブラウザ画面のサイズ）を取得
        const screenW = window.innerWidth;
        const screenH = window.innerHeight;

        // Canvas自体の解像度は画面サイズに合わせる
        if (canvas.width !== screenW || canvas.height !== screenH) {
          canvas.width = screenW;
          canvas.height = screenH;
          bufferCanvas.width = screenW;
          bufferCanvas.height = screenH;
        }

        // バッファクリア
        bufferCtx.globalCompositeOperation = 'source-over';
        bufferCtx.fillStyle = '#000000';
        bufferCtx.fillRect(0, 0, screenW, screenH);

        // ★ここがミソ：カメラ映像を「画面に収まるように(contain)」計算する
        const videoAspect = vw / vh;
        const screenAspect = screenW / screenH;
        
        let drawX, drawY, drawW, drawH;

        if (screenAspect > videoAspect) {
          // 画面の方が横長（映像の左右に黒帯）
          drawH = screenH;
          drawW = drawH * videoAspect;
          drawX = (screenW - drawW) / 2;
          drawY = 0;
        } else {
          // 画面の方が縦長（映像の上下に黒帯）
          drawW = screenW;
          drawH = drawW / videoAspect;
          drawX = 0;
          drawY = (screenH - drawH) / 2;
        }

        // --- 雪動画のクロスフェード処理 ---
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
            
            // 雪も、カメラが表示されているエリア(drawX,Y,W,H)だけに描画する
            drawSnowToBuffer(currentSnowVideo, drawX, drawY, drawW, drawH, alphaCurrent);
            drawSnowToBuffer(nextSnowVideo, drawX, drawY, drawW, drawH, alphaNext);
          } else {
            drawSnowToBuffer(currentSnowVideo, drawX, drawY, drawW, drawH, 1.0);
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
          }
        }

        // --- メイン合成 ---
        // 1. カメラを描画（計算したエリアに）
        ctx.globalCompositeOperation = 'source-over';
        // 黒背景でクリアしてから描く
        ctx.fillStyle = '#000000';
        ctx.fillRect(0, 0, screenW, screenH);
        
        ctx.drawImage(cameraVideo, drawX, drawY, drawW, drawH);

        // 2. 雪（バッファ）を合成
        ctx.globalCompositeOperation = 'screen';
        ctx.drawImage(bufferCanvas, 0, 0);

        requestAnimationFrame(drawCompositeFrame);
      }

      // 雪動画を指定エリアに描画
      function drawSnowToBuffer(video, dx, dy, dw, dh, alpha) {
        if (alpha <= 0.01) return;
        const vw = video.videoWidth;
        const vh = video.videoHeight;
        if (vw === 0 || vh === 0) return;

        // クロップ処理（雪動画のアスペクト比を維持して、描画エリアを埋める）
        const drawAspect = dw / dh;
        const videoAspect = vw / vh;
        let sx, sy, sw, sh;

        if (drawAspect > videoAspect) {
          sw = vw;
          sh = vw / drawAspect;
          sx = 0;
          sy = (vh - sh) / 2;
        } else {
          sh = vh;
          sw = vh * drawAspect;
          sx = (vw - sw) / 2;
          sy = 0;
        }

        bufferCtx.globalAlpha = alpha;
        bufferCtx.drawImage(video, sx, sy, sw, sh, dx, dy, dw, dh);
        bufferCtx.globalAlpha = 1.0;
      }


      // ==========================================
      // プレビュー・撮影・保存
      // ==========================================
      function showPreview(type, url, filename) {
        if (currentPreviewUrl) URL.revokeObjectURL(currentPreviewUrl);
        currentPreviewUrl = url;

        previewModal.style.display = 'flex';
        shutterContainer.style.display = 'none';
        flipBtn.style.display = 'none';
        document.getElementById('reload-btn').style.display = 'none';

        if (type === 'photo') {
          previewImg.style.display = 'block';
          previewVideo.style.display = 'none';
          previewMsgPhoto.style.display = 'block';
          btnSaveVideo.style.display = 'none';
          previewImg.src = url;
        } else {
          previewImg.style.display = 'none';
          previewVideo.style.display = 'block';
          previewMsgPhoto.style.display = 'none';
          btnSaveVideo.style.display = 'block';
          previewVideo.src = url;
          previewVideo.play().catch(()=>{});
          btnSaveVideo.onclick = () => downloadFile(url, filename);
        }
      }

      function closePreview() {
        previewModal.style.display = 'none';
        previewVideo.pause();
        previewVideo.src = "";
        previewImg.src = "";
        
        shutterContainer.style.display = 'block';
        flipBtn.style.display = 'flex';
        document.getElementById('reload-btn').style.display = 'flex';
      }
      
      btnClose.addEventListener('click', closePreview);


      function takePhoto() {
        if (shutterLock) return;
        shutterLock = true;

        const flash = document.getElementById('flash');
        flash.style.opacity = 1;
        setTimeout(() => flash.style.opacity = 0, 200);

        const dataURL = canvas.toDataURL('image/png');
        showPreview('photo', dataURL);

        setTimeout(() => { shutterLock = false; }, 1000);
      }

      function startRecording() {
        isRecording = true;
        isLongPress = true;
        shutterContainer.classList.add('recording');
        recordingStartTime = Date.now();

        const stream = canvas.captureStream(30);
        const mimeTypes = ['video/mp4;codecs=avc1', 'video/mp4', 'video/webm;codecs=h264', 'video/webm'];
        const selectedMimeType = mimeTypes.find(type => MediaRecorder.isTypeSupported(type)) || '';

        try {
          const options = selectedMimeType ? { mimeType: selectedMimeType } : undefined;
          mediaRecorder = new MediaRecorder(stream, options);
        } catch (e) {
          mediaRecorder = new MediaRecorder(stream);
        }

        mediaRecorder.ondataavailable = (event) => {
          if (event.data.size > 0) recordedChunks.push(event.data);
        };

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
        const MAX_RECORD_TIME = 10000;
        const elapsed = Date.now() - recordingStartTime;
        const progress = Math.min(elapsed / MAX_RECORD_TIME, 1);
        progressCircle.style.strokeDashoffset = circumference - (progress * circumference);

        if (progress < 1) requestAnimationFrame(updateGauge);
        else stopRecording();
      }

      const startPress = (e) => {
        if(e.cancelable) e.preventDefault();
        if (isPressing) return;
        isPressing = true;
        isLongPress = false;
        pressTimer = setTimeout(() => startRecording(), LONG_PRESS_DURATION);
      };

      const endPress = (e) => {
        if(e.cancelable) e.preventDefault();
        if (!isPressing) return;
        isPressing = false;
        clearTimeout(pressTimer);
        if (isRecording) {
          stopRecording();
        } else {
          if (!isLongPress) takePhoto();
        }
      };

      shutterContainer.addEventListener('mousedown', startPress);
      shutterContainer.addEventListener('touchstart', startPress, {passive: false});
      shutterContainer.addEventListener('mouseup', endPress);
      shutterContainer.addEventListener('mouseleave', endPress);
      shutterContainer.addEventListener('touchend', endPress, {passive: false});

      function downloadFile(url, filename) {
        const a = document.createElement('a');
        a.href = url;
        a.download = filename;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
      }
    </script>
  </body>
</html>
