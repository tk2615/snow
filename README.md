<html>
  <head>
    <meta charset="utf-8">
    <title>Snow AR Camera (Debug)</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0, viewport-fit=cover">

    <style>
      /* --- 基本設定 --- */
      html, body {
        margin: 0; padding: 0; width: 100%; height: 100%;
        overflow: hidden; background-color: #000033; /* 背景を濃い青に（カメラ死んでる確認用） */
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
        background-color: #000033; /* Canvas自体も青くしとく */
      }

      /* --- UIパーツ --- */
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
        white-space: pre-wrap; /* 改行を表示 */
      }
      .error-text { color: #ff3b30; font-weight: bold; }

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
      }
      #shutter-container.recording #shutter-btn {
        width: 30px; height: 30px; top: 25px; left: 25px;
        border-radius: 4px; background-color: #ff3b30;
      }

      #flash {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background: white; opacity: 0; pointer-events: none; z-index: 200;
        transition: opacity 0.2s;
      }
    </style>
  </head>

  <body>

    <div id="start-screen">
      <button id="start-btn">カメラを起動する</button>
      <div id="status-msg">※カメラの使用を許可してください</div>
    </div>

    <video id="camera-feed" class="hidden-source" autoplay muted playsinline></video>
    <video id="snow-layer" class="hidden-source" src="snow.mp4" loop muted playsinline webkit-playsinline></video>

    <canvas id="work-canvas"></canvas>

    <div id="shutter-container">
      <svg class="progress-ring">
        <circle class="progress-ring__circle" stroke-dasharray="240 240" stroke-dashoffset="240" r="38" cx="40" cy="40"/>
      </svg>
      <div id="shutter-btn"></div>
    </div>

    <div id="flash"></div>

    <script>
      const cameraVideo = document.getElementById('camera-feed');
      const snowVideo = document.getElementById('snow-layer');
      const canvas = document.getElementById('work-canvas');
      const ctx = canvas.getContext('2d');
      const shutterContainer = document.getElementById('shutter-container');
      const progressCircle = document.querySelector('.progress-ring__circle');
      const startScreen = document.getElementById('start-screen');
      const startBtn = document.getElementById('start-btn');
      const statusMsg = document.getElementById('status-msg');
      
      const radius = progressCircle.r.baseVal.value;
      const circumference = radius * 2 * Math.PI;
      progressCircle.style.strokeDasharray = `${circumference} ${circumference}`;
      progressCircle.style.strokeDashoffset = circumference;

      let mediaRecorder;
      let recordedChunks = [];
      let isRecording = false;
      let recordingStartTime;
      let selectedMimeType = '';
      let pressTimer;
      let isLongPress = false;
      const LONG_PRESS_DURATION = 500;

      // ログ表示用ヘルパー
      function log(msg, isError = false) {
        statusMsg.innerHTML = msg;
        if (isError) statusMsg.classList.add('error-text');
        console.log(msg);
      }

      // ==========================================
      // 1. 起動処理（超強化版）
      // ==========================================
      startBtn.addEventListener('click', async () => {
        startBtn.disabled = true;
        startBtn.textContent = "起動中...";
        log("初期化中...");

        try {
          // 1. 動画再生トライ
          try {
            await snowVideo.play();
            log("動画ロードOK...次はカメラ");
          } catch (e) {
            log("動画再生エラー: " + e.message + "\n(タッチで再生できるか試行します)");
          }

          // 2. カメラ取得トライ（再試行ロジック付き）
          let stream = null;
          
          try {
            // トライ1: 指定解像度
            log("カメラ接続中(高画質設定)...");
            stream = await navigator.mediaDevices.getUserMedia({
              video: { facingMode: 'environment', width: { ideal: 1280 }, height: { ideal: 720 } },
              audio: false 
            });
          } catch (err1) {
            log("高画質設定失敗: " + err1.message + "\n標準設定で再試行します...");
            try {
              // トライ2: 何でもいいから映してくれ設定
              stream = await navigator.mediaDevices.getUserMedia({
                video: { facingMode: 'environment' },
                audio: false 
              });
            } catch (err2) {
              throw new Error("カメラ起動に完全失敗: " + err2.message + "\n※ブラウザの設定でカメラ許可を確認してください。\n※HTTPS(鍵マーク)のサイトでないと動きません。");
            }
          }

          if (!stream) throw new Error("ストリームが空です");

          cameraVideo.srcObject = stream;
          log("カメラ取得成功！映像待機中...");
          
          // 映像が実際に来るまで待つ
          cameraVideo.onloadedmetadata = () => {
             log("映像受信開始！");
             startScreen.style.display = 'none';
             shutterContainer.style.display = 'block';
             drawCompositeFrame(); 
          };

        } catch (err) {
          console.error(err);
          startBtn.disabled = false;
          startBtn.textContent = "再試行";
          log(err.message, true);
        }
      });

      // ==========================================
      // 2. 合成ループ
      // ==========================================
      function drawCompositeFrame() {
        const cw = cameraVideo.videoWidth;
        const ch = cameraVideo.videoHeight;

        // まだカメラの準備ができてない場合
        if (cw === 0 || ch === 0) {
           requestAnimationFrame(drawCompositeFrame);
           return;
        }

        if (canvas.width !== cw || canvas.height !== ch) {
          canvas.width = cw;
          canvas.height = ch;
        }

        // --- 背景クリア（デバッグ用：青） ---
        // カメラが描画されれば上書きされて消える。
        // もし青い画面に雪が降ってたら、カメラ映像が透明か来てないってことや。
        ctx.fillStyle = "#000033";
        ctx.fillRect(0, 0, canvas.width, canvas.height);

        // 1. カメラ描画
        ctx.globalCompositeOperation = 'source-over';
        ctx.filter = 'none';
        ctx.drawImage(cameraVideo, 0, 0, cw, ch);

        // 2. 雪の合成（コントラスト強化）
        ctx.globalCompositeOperation = 'screen'; 
        ctx.filter = 'contrast(150%) brightness(60%)'; // 少しマイルドに調整

        const vw = snowVideo.videoWidth;
        const vh = snowVideo.videoHeight;

        if (vw > 0 && vh > 0) {
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
          
          if (snowVideo.paused) snowVideo.play().catch(()=>{});
          ctx.drawImage(snowVideo, sx, sy, sw, sh, 0, 0, cw, ch);
        }

        ctx.filter = 'none';
        requestAnimationFrame(drawCompositeFrame);
      }

      // ==========================================
      // 3. 撮影・録画機能
      // ==========================================
      function takePhoto() {
        const flash = document.getElementById('flash');
        flash.style.opacity = 1;
        setTimeout(() => flash.style.opacity = 0, 200);

        const dataURL = canvas.toDataURL('image/png');
        downloadFile(dataURL, `snow_photo_${Date.now()}.png`);
      }

      function startRecording() {
        isRecording = true;
        isLongPress = true;
        shutterContainer.classList.add('recording');
        recordingStartTime = Date.now();

        const stream = canvas.captureStream(30);
        
        const mimeTypes = [
          'video/mp4;codecs=avc1',
          'video/mp4',
          'video/webm;codecs=h264',
          'video/webm'
        ];
        selectedMimeType = mimeTypes.find(type => MediaRecorder.isTypeSupported(type)) || '';

        try {
          const options = selectedMimeType ? { mimeType: selectedMimeType } : undefined;
          mediaRecorder = new MediaRecorder(stream, options);
        } catch (e) {
          mediaRecorder = new MediaRecorder(stream);
          selectedMimeType = 'video/webm';
        }

        mediaRecorder.ondataavailable = (event) => {
          if (event.data.size > 0) recordedChunks.push(event.data);
        };

        mediaRecorder.onstop = () => {
          const blob = new Blob(recordedChunks, { type: selectedMimeType || 'video/webm' });
          const url = URL.createObjectURL(blob);
          let ext = (selectedMimeType && selectedMimeType.includes('mp4')) ? 'mp4' : 'webm';
          downloadFile(url, `snow_video_${Date.now()}.${ext}`);
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
        isLongPress = false;
        pressTimer = setTimeout(() => startRecording(), LONG_PRESS_DURATION);
      };

      const endPress = (e) => {
        if(e.cancelable) e.preventDefault();
        clearTimeout(pressTimer);
        if (isRecording) stopRecording();
        else takePhoto();
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
