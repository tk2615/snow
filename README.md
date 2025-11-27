<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Perfect Fit AR</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0">

    <style>
      /* 【重要】リセット設定
         画面の余白を完全にゼロにする
      */
      html, body {
        margin: 0;
        padding: 0;
        width: 100%;
        height: 100%;
        overflow: hidden; /* スクロール禁止 */
        background-color: black;
      }

      /* 【共通設定】動画を画面いっぱいに強制固定するクラス
         position: fixed; → スクロールしても動かない絶対固定
         top/left/bottom/right: 0; → 四隅に張り付ける
         object-fit: cover; → アスペクト比を維持したまま、隙間なく画面を埋める（はみ出た部分はカット）
      */
      .responsive-video {
        position: fixed;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        object-fit: cover; 
      }

      /* 1階：カメラ映像（下） */
      #camera-feed {
        z-index: 1;
      }

      /* 2階：雪の動画（上） */
      #snow-layer {
        z-index: 2;
        /* スクリーン合成：黒を透明にする */
        mix-blend-mode: screen; 
        /* タップ操作を透過（下のボタン等を押せるようにする保険） */
        pointer-events: none;
      }

      /* スタートボタン（自動再生失敗時用） */
      #start-btn {
        position: fixed;
        top: 50%; left: 50%;
        transform: translate(-50%, -50%);
        z-index: 999;
        padding: 15px 30px;
        background: rgba(0,0,0,0.5);
        color: white;
        border: 2px solid white;
        border-radius: 30px;
        font-weight: bold;
        display: none;
      }
    </style>
  </head>

  <body>

    <video id="camera-feed" class="responsive-video" autoplay muted playsinline></video>

    <video id="snow-layer" class="responsive-video" src="snow.mp4" loop muted playsinline webkit-playsinline></video>

    <button id="start-btn">雪を降らせる！</button>

    <script>
      // カメラ起動
      async function startCamera() {
        const video = document.getElementById('camera-feed');
        try {
          // environment = 背面カメラ
          // width/height ideal: フルHD画質を要求してボヤけ防止
          const stream = await navigator.mediaDevices.getUserMedia({
            video: { 
              facingMode: 'environment',
              width: { ideal: 1920 },
              height: { ideal: 1080 } 
            },
            audio: false
          });
          video.srcObject = stream;
        } catch (err) {
          alert("カメラが使えへん！設定を確認してな。");
        }
      }

      // 動画再生管理
      function initSnow() {
        const snow = document.getElementById('snow-layer');
        const btn = document.getElementById('start-btn');

        // 強制再生トライ
        const playPromise = snow.play();
        
        if (playPromise !== undefined) {
          playPromise.then(() => {
            console.log("自動再生成功！");
          }).catch(() => {
            console.log("ブロックされた。ボタン出すで。");
            btn.style.display = 'block';
          });
        }

        // ボタンクリックで再生
        btn.addEventListener('click', () => {
          snow.play();
          btn.style.display = 'none';
        });

        // 画面タッチでも再生（念には念を）
        document.body.addEventListener('touchstart', () => {
          if(snow.paused) { snow.play(); btn.style.display = 'none'; }
        }, {once:true});
      }

      window.onload = function() {
        startCamera();
        initSnow();
      };
    </script>
  </body>
</html>
