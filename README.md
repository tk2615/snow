<html>
  <head>
    <meta charset="utf-8">
    <title>Snow AR Camera (Flip & Icon)</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0, viewport-fit=cover">

    <style>
      /* --- 基本設定 --- */
      html, body {
        margin: 0; padding: 0; width: 100%; height: 100%;
        overflow: hidden; background-color: #000;
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
        position: fixed;
        top: 50%; left: 50%;
        transform: translate(-50%, -50%);
        min-width: 100%; min-height: 100%;
        width: auto; height: auto;
        z-index: 1;
        display: block;
      }

      /* --- UIパーツ --- */
      
      /* 共通ボタン（リロード・カメラ切替）のスタイル */
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

      /* 右上：リロード */
      #reload-btn { right: 20px; }
      
      /* 左上：カメラ切り替え（新機能） */
      #flip-btn { left: 20px; display: none; /* 最初は隠す（カメラ起動後に表示） */ }

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

      /* --- プレビュー画面 --- */
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


      /* シャッターボタンエリア */
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

      /* 外側のゲージ */
      .progress-ring {
        position: absolute; top: 0; left: 0;
        width: 80px; height: 80px;
        transform: rotate(-90deg);
      }
      .progress-ring__circle {
        transition: stroke-dashoffset 0.1s linear;
        stroke: #ff3b30; stroke-width: 4; fill: transparent;
      }

      /* シャッターボタン本体 */
      #shutter-btn {
        position: absolute; top: 10px; left: 10px;
        width: 60px; height: 60px;
        background-color: white; border-radius: 50%;
        transition: all 0.2s;
        display: flex; justify-content: center; align-items: center;
      }

      /* カメラアイコン（SVG） */
      #camera-icon {
        width: 32px; height: 32px;
        fill: #333;
        transition: opacity 0.2s;
      }

      /* 録画中のスタイル */
      #shutter-container.recording #shutter-btn {
        width: 30px; height: 30px; top: 25px; left: 25px;
        border-radius: 4px; background-color: #ff3b30;
      }
      /* 録画中はカメラアイコンを消す */
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
          <path d="M12 12c2.21 0 4-1.79 4-4s-1.79-4-4-4-4 1.79-4 4 1.79 4 4 4zm0 2c-2.67 0-8 1.34-8 4v2h16v-2c0-2.66-5.33-4-8-4z" opacity="0"/> <path d="M9 2L7.17 4H4c-1.1 0-2 .9-2 2v12c0 1.1.9 2 2 2h16c1.1 0 2-.9 2-2V6c0-1.1-.9-2-2-2h-3.17L15 2H9zm3 15c-2.76 0-5-2.24-5-5s2.24-5 5-5 5 2.24 5 5-2.24 5-5 5z"/>
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
      const flipBtn = document.getElementById('flip-btn'); // カメラ切り替えボタン

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

      // カメラ設定管理用
      let currentFacingMode = 'environment'; // 初期は外カメラ

      function log(msg) {
        statusMsg.innerHTML = msg;
        console.log(msg);
      }

      // ==========================================
      // カメラ初期化・切り替えロジック
      // ==========================================
      async function initCamera(facingMode) {
        // 既存のストリームがあれば停止
        if (cameraVideo.srcObject) {
          cameraVideo.srcObject.getTracks().forEach(track => track.stop());
        }

        let stream = null;
        try {
          log(`カメラ起動中 (${facingMode === 'user' ? '自撮り' : '外向き'})...`);
          
          // 指定モードで取得トライ
          stream = await navigator.mediaDevices.getUserMedia({
            video: { 
              facingMode: facingMode, 
              width: { ideal: 1280 }, 
              height: { ideal: 720 } 
            },
            audio: false 
          });
        } catch (err) {
          log("指定モード失敗...標準設定で再試行");
          try {
             // 失敗時は指定なしでトライ（PCなどでfacingModeがない場合への保険）
             stream = await navigator.mediaDevices.getUserMedia({
              video: true,
              audio: false 
            });
          } catch(e) {
            throw e;
          }
        }

        cameraVideo.srcObject = stream;
        
        // 映像が来るまで待機
        return new Promise((resolve) => {
          cameraVideo.onloadedmetadata = () => {
            resolve();
          };
        });
      }

      // 切り替えボタンの処理
      flipBtn.addEventListener('click', async () => {
        // モード反転
        currentFacingMode = (currentFacingMode === 'environment') ? 'user' : 'environment';
        
        // ボタンを一時的に無効化（連打防止）
        flipBtn.style.pointerEvents = 'none';
        flipBtn.style.opacity = 0.5;

        try {
          await initCamera(currentFacingMode);
          log(""); // ログクリア
        } catch (err) {
          log("カメラ切り替えエラー: " + err.message);
        } finally {
          flipBtn.style.pointerEvents = 'auto';
          flipBtn.style.opacity = 1;
        }
      });


      // ==========================================
      // アプリ起動
      // ==========================================
      startBtn.addEventListener('click', async () => {
        startBtn.disabled = true;
        startBtn.textContent = "起動中...";
        log("初期化中...");

        try {
          snowV1.loop = false;
          snowV2.loop = false;
          await snowV1.play();
          
          // 初回カメラ起動
          await initCamera(currentFacingMode);

          // UI切り替え
          startScreen.style.display = 'none';
          shutterContainer.style.display = 'block';
          flipBtn.style.display = 'flex'; // 切り替えボタン表示
          
          drawCompositeFrame(); 

        } catch (err) {
          console.error(err);
          startBtn.disabled = false;
          startBtn.textContent = "再試行";
          log("エラー: " + err.message);
        }
      });


      // ==========================================
      // 描画ループ (クロスフェード)
      // ==========================================
      function drawCompositeFrame() {
        const cw = cameraVideo.videoWidth;
        const ch = cameraVideo.videoHeight;

        if (cw === 0 || ch === 0) {
           requestAnimationFrame(drawCompositeFrame);
           return;
        }

        if (canvas.width !== cw || canvas.height !== ch) {
          canvas.width = cw;
          canvas.height = ch;
          bufferCanvas.width = cw;
          bufferCanvas.height = ch;
        }

        // バッファクリア
        bufferCtx.globalCompositeOperation = 'source-over';
        bufferCtx.fillStyle = '#000000';
        bufferCtx.fillRect(0, 0, cw, ch);

        // クロスフェード計算
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
            drawSnowToBuffer(currentSnowVideo, cw, ch, alphaCurrent);
            drawSnowToBuffer(nextSnowVideo, cw, ch, alphaNext);
          } else {
            drawSnowToBuffer(currentSnowVideo, cw, ch, 1.0);
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

        // 合成：カメラ＋バッファ
        // 自撮り（user）の時は左右反転させると鏡みたいで自然やけど、
        // 録画した時に文字が反転するから、今回はあえてそのまま描画するで。
        ctx.globalCompositeOperation = 'source-over';
        
        // もし自撮りの反転を入れたいならここで ctx.scale(-1, 1) とかやるけど
        // AR合成の位置ズレ防止のため今回は標準のままいくわ
        ctx.drawImage(cameraVideo, 0, 0, cw, ch);
        
        ctx.globalCompositeOperation = 'screen';
        ctx.drawImage(bufferCanvas, 0, 0);

        requestAnimationFrame(drawCompositeFrame);
      }

      function drawSnowToBuffer(video, cw, ch, alpha) {
        if (alpha <= 0.01) return;
        const vw = video.videoWidth;
        const vh = video.videoHeight;
        if (vw === 0 || vh === 0) return;

        const videoAspect = vw / vh;
        const canvasAspect = cw / ch;
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

        bufferCtx.globalAlpha = alpha;
        bufferCtx.drawImage(video, sx, sy, sw, sh, 0, 0, cw, ch);
        bufferCtx.globalAlpha = 1.0;
      }


      // ==========================================
      // プレビュー機能
      // ==========================================
      function showPreview(type, url, filename) {
        if (currentPreviewUrl) URL.revokeObjectURL(currentPreviewUrl);
        currentPreviewUrl = url;

        previewModal.style.display = 'flex';
        shutterContainer.style.display = 'none';
        flipBtn.style.display = 'none'; // プレビュー中は切り替えボタンも隠す

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
        flipBtn.style.display = 'flex'; // ボタン復活
      }
      
      btnClose.addEventListener('click', closePreview);


      // ==========================================
      // 撮影機能
      // ==========================================
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
