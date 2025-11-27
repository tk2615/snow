<html>

  <head>

    <meta charset="utf-8">

    <title>Snow AR Camera (Clean Preview & Stable)</title>

    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0, viewport-fit=cover">

<style>

  /* 最初の見出し（h1）を消す */

  h1:first-of-type {

    display: none !important;

  }

</style>

    <style>

      /* --- 基本設定 --- */

      html, body {

        margin: 0; padding: 0; width: 100%; height: 100%;

        overflow: hidden; background-color: #000;

        font-family: sans-serif;

        overscroll-behavior: none;

      }



      .hidden-source {

        position: absolute; top: 0; left: 0; width: 10px; height: 10px;

        opacity: 0.01; pointer-events: none; z-index: -99;

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

        display: flex; justify-content: center; align-items: center;

        backdrop-filter: blur(4px); -webkit-tap-highlight-color: transparent; 

        transition: background 0.2s;

        display: none;

      }

      .icon-btn:active { background: rgba(255, 255, 255, 0.3); }

      .icon-btn svg { width: 24px; height: 24px; fill: white; }



      #reload-btn { right: 20px; }

      #flip-btn { left: 20px; }



      /* スタート画面 */

      #start-screen {

        position: fixed; top: 0; left: 0; width: 100%; height: 100%;

        background-color: rgba(0, 0, 0, 0.6);

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

        object-fit: contain; background-color: transparent;

        filter: drop-shadow(0 0 10px rgba(0,0,0,0.5));

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

      /* プレビュー動画のクリック無効化（再生バーを出さないため） */

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

      <button id="start-btn">START</button>

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

          snowV1.loop = false;

          snowV2.loop = false;

          await snowV1.play().catch(e => console.warn(e));

          await initCamera(currentFacingMode);

          

          updateDimensions();

          drawCompositeFrame(0);



        } catch (err) {

          showError("カメラエラー:\n" + err.message);

        }

      }



      window.onload = initApp;

      window.addEventListener('resize', () => { needsResize = true; });



      startBtn.addEventListener('click', () => {

        startScreen.style.opacity = '0';

        setTimeout(() => { startScreen.style.display = 'none'; }, 500);

        shutterContainer.style.display = 'block';

        flipBtn.style.display = 'flex';

        reloadBtn.style.display = 'flex';

        if(currentSnowVideo.paused) currentSnowVideo.play().catch(()=>{});

      });



      async function initCamera(facingMode) {

        if (cameraVideo.srcObject) {

          cameraVideo.srcObject.getTracks().forEach(track => track.stop());

        }

        let stream = null;

        try {

          // ★軽量化: 720p (1280x720) を指定して安定化

          stream = await navigator.mediaDevices.getUserMedia({

            video: { 

              facingMode: facingMode,

              width: { ideal: 1280 }, 

              height: { ideal: 720 } 

            },

            audio: false 

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



      // ==========================================

      // 描画ループ (30fps)

      // ==========================================

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

            // ★安定化: 動画が止まっていたら再生

            if (nextSnowVideo.paused) nextSnowVideo.play().catch(()=>{});

            

            const alphaCurrent = Math.max(0, timeLeft / FADE_DURATION);

            const alphaNext = 1.0 - alphaCurrent;

            drawSnowToBuffer(currentSnowVideo, vw, vh, alphaCurrent);

            drawSnowToBuffer(nextSnowVideo, vw, vh, alphaNext);

          } else {

            drawSnowToBuffer(currentSnowVideo, vw, vh, 1.0);

            // 待機中の動画は停止

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

        } else {

            // 再生が開始されてない場合の保険

            if(currentSnowVideo.paused) currentSnowVideo.play().catch(()=>{});

        }



        ctx.globalCompositeOperation = 'source-over';

        ctx.drawImage(cameraVideo, 0, 0, vw, vh);

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

          sw = videoW;

          sh = videoW / canvasAspect;

          sx = 0;

          sy = (videoH - sh) / 2;

        } else {

          sh = videoH;

          sw = videoH * canvasAspect;

          sx = (videoW - sw) / 2;

          sy = 0;

        }



        bufferCtx.globalAlpha = alpha;

        bufferCtx.drawImage(video, sx, sy, sw, sh, 0, 0, cw, ch);

        bufferCtx.globalAlpha = 1.0;

      }



      // ==========================================

      // UI イベント

      // ==========================================

      function showPreview(type, url, filename) {

        if (currentPreviewUrl) URL.revokeObjectURL(currentPreviewUrl);

        currentPreviewUrl = url;

        previewModal.style.display = 'flex';

        shutterContainer.style.display = 'none';

        flipBtn.style.display = 'none';

        reloadBtn.style.display = 'none';



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

          // ★プレビュー再生 (ミュートで自動再生)

          previewVideo.muted = true; 

          previewVideo.play().catch(()=>{});

          

          btnSaveVideo.onclick = () => downloadFile(url, filename);

        }

      }



      function closePreview() {

        previewModal.style.display = 'none';

        previewVideo.pause();

        previewVideo.src = "";

        previewImg.src = "";

        // プレビュー終了時に雪動画を再開

        if(currentSnowVideo.paused) currentSnowVideo.play().catch(()=>{});

        shutterContainer.style.display = 'block';

        flipBtn.style.display = 'flex';

        reloadBtn.style.display = 'flex';

      }

      

      btnClose.addEventListener('click', closePreview);



      document.addEventListener("visibilitychange", () => {

        if (document.visibilityState === "visible") {

          if (previewModal.style.display === 'none' && startScreen.style.display === 'none') {

             if (currentSnowVideo.paused) currentSnowVideo.play().catch(()=>{});

             if (cameraVideo.paused) cameraVideo.play().catch(()=>{});

          }

        }

      });



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

          mediaRecorder = new MediaRecorder(stream, selectedMimeType ? { mimeType: selectedMimeType } : undefined);

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

        if (isRecording) stopRecording();

        else if (!isLongPress) takePhoto();

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
