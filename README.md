<html>
  <head>
    <meta charset="utf-8">
    <title>Snow AR Camera (Fixed)</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0, viewport-fit=cover">

    <style>
      /* --- 基本設定 --- */
      html, body {
        margin: 0; padding: 0; width: 100%; height: 100%;
        overflow: hidden; background-color: black;
        font-family: sans-serif;
        /* スマホでのスクロールバウンスなどを防ぐ */
        overscroll-behavior: none;
      }

      /* 動画表示エリア（見た目用） */
      .responsive-video {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        object-fit: cover; /* ここで見た目のクロップはできている */
      }
      #camera-feed { z-index: 1; }
      /* 見た目上もスクリーン合成を適用 */
      #snow-layer { z-index: 2; mix-blend-mode: screen; pointer-events: none; }

      /* --- UIパーツ --- */
      
      /* シャッターボタンの親枠 */
      #shutter-container {
        position: fixed; bottom: 30px; left: 50%;
        transform: translateX(-50%);
        width: 80px; height: 80px;
        z-index: 100;
        cursor: pointer;
        -webkit-tap-highlight-color: transparent; 
        user-select: none;
      }

      /* 円形ゲージ（SVG） */
      .progress-ring {
        position: absolute; top: 0; left: 0;
        width: 80px; height: 80px;
        transform: rotate(-90deg);
      }
      .progress-ring__circle {
        transition: stroke-dashoffset 0.1s linear;
        stroke: #ff3b30;
        stroke-width: 4;
        fill: transparent;
      }

      /* 内側の白いボタン */
      #shutter-btn {
        position: absolute; top: 10px; left: 10px;
        width: 60px; height: 60px;
        background-color: white;
        border-radius: 50%;
        transition: all 0.2s;
      }
      
      /* 録画中のボタン変形 */
      #shutter-container.recording #shutter-btn {
        width: 30px; height: 30px;
        top: 25px; left: 25px;
        border-radius: 4px;
        background-color: #ff3b30;
      }

      /* 撮影完了メッセージ */
      #flash {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background: white; opacity: 0; pointer-events: none; z-index: 200;
        transition: opacity 0.2s;
      }
      
      /* 処理用の隠しキャンバス */
      #work-canvas { display: none; }
    </style>
  </head>

  <body>

    <video id="camera-feed" class="responsive-video" autoplay muted playsinline></video>
    <video id="snow-layer" class="responsive-video" src="snow.mp4" loop muted playsinline webkit-playsinline crossorigin="anonymous"></video>

    <div id="shutter-container">
      <svg class="progress-ring">
        <circle class="progress-ring__circle" stroke-dasharray="240 240" stroke-dashoffset="240" r="38" cx="40" cy="40"/>
      </svg>
      <div id="shutter-btn"></div>
    </div>

    <div id="flash"></div>

    <canvas id="work-canvas"></canvas>

    <script>
      // 変数定義
      const cameraVideo = document.getElementById('camera-feed');
      const snowVideo = document.getElementById('snow-layer');
      const canvas = document.getElementById('work-canvas');
      const ctx = canvas.getContext('2d');
      const shutterContainer = document.getElementById('shutter-container');
      const progressCircle = document.querySelector('.progress-ring__circle');
      
      const radius = progressCircle.r.baseVal.value;
      const circumference = radius * 2 * Math.PI;
      
      progressCircle.style.strokeDasharray = `${circumference} ${circumference}`;
      progressCircle.style.strokeDashoffset = circumference;

      let mediaRecorder;
      let recordedChunks = [];
      let isRecording = false;
      let recordingStartTime;
      let animationFrameId;
      let selectedMimeType = '';
      
      let pressTimer;
      let isLongPress = false;
      const LONG_PRESS_DURATION = 500;

      // ==========================================
      // 1. カメラと動画の準備
      // ==========================================
      async function initApp() {
        try {
          const stream = await navigator.mediaDevices.getUserMedia({
            video: { 
              facingMode: 'environment', 
              width: { ideal: 1280 },
              height: { ideal: 720 } 
            },
            audio: false 
          });
          cameraVideo.srcObject = stream;
          
          // 動画がロードされたら再生開始
          snowVideo.oncanplay = () => {
            snowVideo.play().catch(() => {
              document.body.addEventListener('touchstart', () => snowVideo.play(), {once:true});
            });
          };

        } catch (err) {
          alert("カメラが起動できへんかったわ。権限を確認してな！");
          console.error(err);
        }
      }

      // ==========================================
      // 2. 合成処理（Canvasに描画）★ここが修正の肝や！★
      // ==========================================
      function drawCompositeFrame() {
        // キャンバスサイズをカメラ映像に合わせる
        const cw = cameraVideo.videoWidth;
        const ch = cameraVideo.videoHeight;
        if (canvas.width !== cw || canvas.height !== ch) {
          canvas.width = cw;
          canvas.height = ch;
        }

        // 1. カメラを描く（通常モード）
        ctx.globalCompositeOperation = 'source-over';
        ctx.drawImage(cameraVideo, 0, 0, cw, ch);

        // 2. 雪を「スクリーン合成」で重ねる
        // これで黒が抜けるはずや
        ctx.globalCompositeOperation = 'screen';

        // --- ここからクロップ計算 ---
        const vw = snowVideo.videoWidth;
        const vh = snowVideo.videoHeight;

        // 動画の準備ができてないときはスキップ
        if (vw === 0 || vh === 0) {
           if (isRecording) animationFrameId = requestAnimationFrame(drawCompositeFrame);
           return;
        }

        // アスペクト比を比較
        const videoAspect = vw / vh;
        const canvasAspect = cw / ch;

        let sx, sy, sw, sh; // ソース（動画）側の切り取り範囲

        if (canvasAspect > videoAspect) {
          // キャンバスの方が横長 → 動画の上下をカットして横幅を合わせる
          sw = vw;
          sh = vw / canvasAspect;
          sx = 0;
          sy = (vh - sh) / 2; // 中央寄せ
        } else {
          // キャンバスの方が縦長（または同じ）→ 動画の左右をカットして縦幅を合わせる
          sh = vh;
          sw = vh * canvasAspect;
          sx = (vw - sw) / 2; // 中央寄せ
          sy = 0;
        }

        // 計算した範囲で描画！
        // drawImage(元画像, 切り取り開始X, Y, 切り取り幅, 高さ, 描画先X, Y, 描画先幅, 高さ)
        ctx.drawImage(snowVideo, sx, sy, sw, sh, 0, 0, cw, ch);
        // --- クロップ計算ここまで ---

        // 動画撮影中なら次のフレームも予約
        if (isRecording) {
          animationFrameId = requestAnimationFrame(drawCompositeFrame);
        }
      }

      // ==========================================
      // 3. 静止画撮影
      // ==========================================
      function takePhoto() {
        drawCompositeFrame();
        
        const flash = document.getElementById('flash');
        flash.style.opacity = 1;
        setTimeout(() => flash.style.opacity = 0, 200);

        const dataURL = canvas.toDataURL('image/png');
        downloadFile(dataURL, `snow_photo_${Date.now()}.png`);
      }

      // ==========================================
      // 4. 動画撮影
      // ==========================================
      function startRecording() {
        isRecording = true;
        isLongPress = true;
        shutterContainer.classList.add('recording');
        recordingStartTime = Date.now();

        drawCompositeFrame();

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
          console.error("録画設定エラー:", e);
          mediaRecorder = new MediaRecorder(stream);
          selectedMimeType = 'video/webm';
        }

        mediaRecorder.ondataavailable = (event) => {
          if (event.data.size > 0) recordedChunks.push(event.data);
        };

        mediaRecorder.onstop = () => {
          const blob = new Blob(recordedChunks, { type: selectedMimeType || 'video/webm' });
          const url = URL.createObjectURL(blob);
          
          let ext = 'webm';
          if (selectedMimeType && selectedMimeType.includes('mp4')) {
            ext = 'mp4';
          }
          
          downloadFile(url, `snow_video_${Date.now()}.${ext}`);
          recordedChunks = [];
        };

        mediaRecorder.start();
        updateGauge();
      }

      function stopRecording() {
        isRecording = false;
        shutterContainer.classList.remove('recording');
        cancelAnimationFrame(animationFrameId);
        
        if (mediaRecorder && mediaRecorder.state !== 'inactive') {
          mediaRecorder.stop();
        }
        
        progressCircle.style.strokeDashoffset = circumference;
      }

      function updateGauge() {
        if (!isRecording) return;
        
        const MAX_RECORD_TIME = 10000;
        const elapsed = Date.now() - recordingStartTime;
        const progress = Math.min(elapsed / MAX_RECORD_TIME, 1);
        
        const offset = circumference - (progress * circumference);
        progressCircle.style.strokeDashoffset = offset;

        if (progress < 1) {
          requestAnimationFrame(updateGauge);
        } else {
          stopRecording();
        }
      }

      // ==========================================
      // 5. 操作イベント管理
      // ==========================================
      
      const startPress = (e) => {
        e.preventDefault();
        isLongPress = false;
        
        pressTimer = setTimeout(() => {
          startRecording(); 
        }, LONG_PRESS_DURATION);
      };

      const endPress = (e) => {
        e.preventDefault();
        clearTimeout(pressTimer);

        if (isRecording) {
          stopRecording();
        } else {
          takePhoto();
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

      window.onload = initApp;

    </script>
  </body>
</html>
