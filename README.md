<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Snow AR Camera</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0">

    <style>
      /* --- 基本設定 --- */
      html, body {
        margin: 0; padding: 0; width: 100%; height: 100%;
        overflow: hidden; background-color: black;
        font-family: sans-serif;
      }

      /* 動画表示エリア（見た目用） */
      .responsive-video {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        object-fit: cover;
      }
      #camera-feed { z-index: 1; }
      #snow-layer { z-index: 2; mix-blend-mode: screen; pointer-events: none; }

      /* --- UIパーツ --- */
      
      /* シャッターボタンの親枠 */
      #shutter-container {
        position: fixed; bottom: 30px; left: 50%;
        transform: translateX(-50%);
        width: 80px; height: 80px;
        z-index: 100;
        cursor: pointer;
        /* タップ時のハイライト無効化 */
        -webkit-tap-highlight-color: transparent; 
        user-select: none;
      }

      /* 円形ゲージ（SVG） */
      .progress-ring {
        position: absolute; top: 0; left: 0;
        width: 80px; height: 80px;
        transform: rotate(-90deg); /* 12時の位置からスタート */
      }
      .progress-ring__circle {
        transition: stroke-dashoffset 0.1s linear; /* 滑らかに動かす */
        stroke: #ff3b30; /* 録画中の赤色 */
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
        width: 30px; height: 30px; /* 小さくなる */
        top: 25px; left: 25px;
        border-radius: 4px; /* 四角くなる */
        background-color: #ff3b30;
      }

      /* 撮影完了メッセージ */
      #flash {
        position: fixed; top: 0; left: 0; width: 100%; height: 100%;
        background: white; opacity: 0; pointer-events: none; z-index: 200;
        transition: opacity 0.2s;
      }
      
      /* 処理用の隠しキャンバス（画面には出さない） */
      #work-canvas { display: none; }
    </style>
  </head>

  <body>

    <video id="camera-feed" class="responsive-video" autoplay muted playsinline></video>
    <video id="snow-layer" class="responsive-video" src="snow.mp4" loop muted playsinline webkit-playsinline></video>

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
      
      // ゲージの円周の長さ（計算用）
      const radius = progressCircle.r.baseVal.value;
      const circumference = radius * 2 * Math.PI;
      
      // 初期設定：ゲージを隠す（offsetを最大にする）
      progressCircle.style.strokeDasharray = `${circumference} ${circumference}`;
      progressCircle.style.strokeDashoffset = circumference;

      let mediaRecorder;
      let recordedChunks = [];
      let isRecording = false;
      let recordingStartTime;
      let animationFrameId;
      
      // 長押し判定用タイマー
      let pressTimer;
      let isLongPress = false;
      const LONG_PRESS_DURATION = 500; // 0.5秒長押しで動画モード

      // ==========================================
      // 1. カメラと動画の準備
      // ==========================================
      async function initApp() {
        try {
          const stream = await navigator.mediaDevices.getUserMedia({
            video: { facingMode: 'environment', width: {ideal: 1280}, height: {ideal: 720} },
            audio: false // 今回は音なし（マイク許可が面倒なため）
          });
          cameraVideo.srcObject = stream;
          
          // 雪動画の再生
          snowVideo.play().catch(() => {
            // 自動再生ブロック時はタッチで開始
            document.body.addEventListener('touchstart', () => snowVideo.play(), {once:true});
          });

        } catch (err) {
          alert("カメラが起動できへんかったわ。権限を確認してな！");
        }
      }

      // ==========================================
      // 2. 合成処理（Canvasに描画）
      // ==========================================
      function drawCompositeFrame() {
        // キャンバスサイズをカメラ映像に合わせる
        if (canvas.width !== cameraVideo.videoWidth) {
          canvas.width = cameraVideo.videoWidth;
          canvas.height = cameraVideo.videoHeight;
        }

        // 1. カメラを描く
        ctx.globalCompositeOperation = 'source-over';
        ctx.drawImage(cameraVideo, 0, 0, canvas.width, canvas.height);

        // 2. 雪を「スクリーン合成」で重ねる（CSSと同じ見た目にする）
        ctx.globalCompositeOperation = 'screen';
        ctx.drawImage(snowVideo, 0, 0, canvas.width, canvas.height);

        // 動画撮影中なら次のフレームも予約
        if (isRecording) {
          animationFrameId = requestAnimationFrame(drawCompositeFrame);
        }
      }

      // ==========================================
      // 3. 静止画撮影
      // ==========================================
      function takePhoto() {
        drawCompositeFrame(); // 現在のフレームを合成
        
        // フラッシュ演出
        const flash = document.getElementById('flash');
        flash.style.opacity = 1;
        setTimeout(() => flash.style.opacity = 0, 200);

        // 画像保存
        const dataURL = canvas.toDataURL('image/png');
        downloadFile(dataURL, `snow_photo_${Date.now()}.png`);
      }

      // ==========================================
      // 4. 動画撮影
      // ==========================================
      function startRecording() {
        isRecording = true;
        isLongPress = true; // 動画モード確定
        shutterContainer.classList.add('recording');
        recordingStartTime = Date.now();

        // 録画ループ開始
        drawCompositeFrame();

        // キャンバスの内容をストリーム化（30fps）
        const stream = canvas.captureStream(30);
        
        // ブラウザごとの対応フォーマットを探す
        let options = { mimeType: 'video/webm' };
        if (!MediaRecorder.isTypeSupported('video/webm')) {
          options = { mimeType: 'video/mp4' }; // Safari用
        }

        try {
          mediaRecorder = new MediaRecorder(stream, options);
        } catch (e) {
          // ダメならデフォルトで
          mediaRecorder = new MediaRecorder(stream);
        }

        mediaRecorder.ondataavailable = (event) => {
          if (event.data.size > 0) recordedChunks.push(event.data);
        };

        mediaRecorder.onstop = () => {
          const blob = new Blob(recordedChunks, { type: 'video/webm' });
          const url = URL.createObjectURL(blob);
          downloadFile(url, `snow_video_${Date.now()}.webm`); // iOSはmp4拡張子の方が良い場合もあるけど一旦webmで
          recordedChunks = [];
        };

        mediaRecorder.start();
        updateGauge(); // ゲージアニメーション開始
      }

      function stopRecording() {
        isRecording = false;
        shutterContainer.classList.remove('recording');
        cancelAnimationFrame(animationFrameId);
        
        if (mediaRecorder && mediaRecorder.state !== 'inactive') {
          mediaRecorder.stop();
        }
        
        // ゲージリセット
        progressCircle.style.strokeDashoffset = circumference;
      }

      // ゲージのアニメーション（最大10秒で一周する設定）
      function updateGauge() {
        if (!isRecording) return;
        
        const MAX_RECORD_TIME = 10000; // 10秒
        const elapsed = Date.now() - recordingStartTime;
        const progress = Math.min(elapsed / MAX_RECORD_TIME, 1);
        
        // ゲージを減らしていく
        const offset = circumference - (progress * circumference);
        progressCircle.style.strokeDashoffset = offset;

        if (progress < 1) {
          requestAnimationFrame(updateGauge);
        } else {
          stopRecording(); // 10秒経ったら強制終了
        }
      }

      // ==========================================
      // 5. 操作イベント管理（タップ vs 長押し）
      // ==========================================
      
      // マウス/タッチ開始
      const startPress = (e) => {
        e.preventDefault(); // スクロール等防止
        isLongPress = false;
        
        // 長押しタイマーセット
        pressTimer = setTimeout(() => {
          startRecording(); // 長押し成立→動画開始
        }, LONG_PRESS_DURATION);
      };

      // マウス/タッチ終了
      const endPress = (e) => {
        e.preventDefault();
        clearTimeout(pressTimer); // タイマー解除

        if (isRecording) {
          // 動画撮影中なら停止
          stopRecording();
        } else {
          // 長押しじゃなかった（ただのタップ）なら静止画撮影
          takePhoto();
        }
      };

      // イベントリスナー登録
      shutterContainer.addEventListener('mousedown', startPress);
      shutterContainer.addEventListener('touchstart', startPress);
      
      shutterContainer.addEventListener('mouseup', endPress);
      shutterContainer.addEventListener('mouseleave', endPress);
      shutterContainer.addEventListener('touchend', endPress);


      // ファイルダウンロード用ヘルパー
      function downloadFile(url, filename) {
        const a = document.createElement('a');
        a.href = url;
        a.download = filename;
        document.body.appendChild(a);
        a.click();
        document.body.removeChild(a);
      }

      // アプリ開始
      window.onload = initApp;

    </script>
  </body>
</html>
