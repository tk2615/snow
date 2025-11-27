<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Black Key Snow AR</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no, maximum-scale=1.0">
    <script src="https://aframe.io/releases/1.4.2/aframe.min.js"></script>

    <style>
      body, html { margin: 0; padding: 0; overflow: hidden; width: 100%; height: 100%; background-color: #000; }
      #real-camera { position: fixed; top: 0; left: 0; width: 100%; height: 100%; object-fit: cover; z-index: 1; }
      #ar-scene { position: fixed; top: 0; left: 0; width: 100%; height: 100%; z-index: 2; pointer-events: none; }
    </style>

    <script>
      // 1. 黒色を透明にする「黒抜きシェーダー」を定義
      AFRAME.registerShader('black-key', {
        schema: {
          src: {type: 'map', is: 'uniform'},
          threshold: {type: 'float', is: 'uniform', default: 0.1}, // この数字以下の暗さは透明にする！
          opacity: {type: 'float', is: 'uniform', default: 1.0}
        },
        vertexShader: `
          varying vec2 vUv;
          void main() {
            vUv = uv;
            gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
          }
        `,
        fragmentShader: `
          uniform sampler2D src;
          uniform float threshold;
          uniform float opacity;
          varying vec2 vUv;
          void main() {
            vec4 color = texture2D(src, vUv);
            // 色の明るさ（輝度）を計算
            float brightness = dot(color.rgb, vec3(0.299, 0.587, 0.114));
            
            // 明るさが「しきい値」より低ければ透明に、高ければそのまま表示
            // smoothstepを使って境界を少し滑らかにする
            float alpha = smoothstep(0.0, threshold, brightness);
            
            gl_FragColor = vec4(color.rgb, alpha * opacity);
          }
        `
      });

      // 2. カメラ起動
      async function startRealCamera() {
        const videoElement = document.getElementById('real-camera');
        try {
          const stream = await navigator.mediaDevices.getUserMedia({ video: { facingMode: 'environment' }, audio: false });
          videoElement.srcObject = stream;
        } catch (err) {
          alert("カメラ許可してな！");
        }
      }

      // 3. 強制再生
      function forcePlaySnow() {
        const snowVideo = document.getElementById('snow-video-asset');
        const play = () => {
            snowVideo.play().then(()=>console.log("再生!")).catch(e=>console.log("ブロック!", e));
        };
        play();
        document.body.addEventListener('touchstart', play, {once:true});
        document.body.addEventListener('click', play, {once:true});
      }

      window.onload = function() { startRealCamera(); forcePlaySnow(); };
    </script>
  </head>

  <body>
    <video id="real-camera" autoplay muted playsinline></video>

    <a-scene id="ar-scene" embedded renderer="alpha: true;" vr-mode-ui="enabled: false">
      <a-assets>
        <video id="snow-video-asset" src="snow.mp4" preload="auto" autoplay loop muted playsinline webkit-playsinline crossOrigin="anonymous"></video>
      </a-assets>

      <a-entity camera position="0 0 0" look-controls="enabled: false">
        
        <a-entity 
          geometry="primitive: plane; width: 9; height: 5;"
          position="0 0 -5"
          material="shader: black-key; src: #snow-video-asset; threshold: 0.2; opacity: 1; side: double;">
        </a-entity>

      </a-entity>
    </a-scene>
  </body>
</html>
