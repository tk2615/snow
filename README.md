<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Black BG Snow AR</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <meta name="apple-mobile-web-app-capable" content="yes">

    <script src="https://aframe.io/releases/1.4.2/aframe.min.js"></script>
    <script src="https://raw.githack.com/AR-js-org/AR.js/master/aframe/build/aframe-ar.js"></script>

    <script>
      // 【重要】スマホで動画を再生させるためのスイッチ
      // 画面をタップすると動画がスタートする仕掛けや
      AFRAME.registerComponent('click-to-play', {
        init: function () {
          var video = document.querySelector('#snow-video');
          var startMsg = document.querySelector('#start-message');
          
          var playVideo = function() {
            video.play();
            // 再生したら案内メッセージを消す
            if(startMsg) startMsg.style.display = 'none';
          };

          document.body.addEventListener('click', playVideo);
          document.body.addEventListener('touchstart', playVideo);
        }
      });
    </script>
  </head>

  <body style="margin: 0; overflow: hidden;">
    
    <a-scene embedded arjs="sourceType: webcam; debugUIEnabled: false;" vr-mode-ui="enabled: false">

      <a-assets>
        < 
          id="snow-" 
          src="https://raw.githubusercontent.com/mrdoob/three.js/master/examples/textures/sintel.mp4" 
          preload="auto" 
          loop="true" 
          muted="true"
          playsinline 
          webkit-playsinline
          crossOrigin="anonymous">
        </>
        </a-assets>

      <a-entity camera look-controls click-to-play>
        
        <a- 
          src="#snow.mp4" 
          position="0 0 -3" 
          width="4" 
          height="2.25"
          scale="1.5 1.5 1.5"
          material="shader: flat; transparent: true; blending: additive; opacity: 0.9;">
        </a->

      </a-entity>

    </a-scene>

    <div id="start-message" style="
      position: fixed; 
      top: 50%; 
      left: 50%; 
      transform: translate(-50%, -50%); 
      z-index: 10; 
      color: white; 
      background: rgba(0,0,0,0.5); 
      padding: 20px; 
      border-radius: 10px;
      text-align: center; 
      font-family: sans-serif; 
      pointer-events: none;">
      画面をタップして<br>雪を降らせるで！
    </div>

  </body>
</html>
