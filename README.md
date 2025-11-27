<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Snow AR Loop</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <meta name="apple-mobile-web-app-capable" content="yes">

    <script src="https://aframe.io/releases/1.4.2/aframe.min.js"></script>
    <script src="https://raw.githack.com/AR-js-org/AR.js/master/aframe/build/aframe-ar.js"></script>

    <script>
      // 自動再生を強力にサポートするスクリプト
      AFRAME.registerComponent('auto-play-video', {
        init: function () {
          var video = document.querySelector('#snow-video');

          var attemptPlay = function() {
            video.play().catch(function(error) {
              console.log("自動再生ブロックされたわ。タップ待ちに切り替えるで。");
              // 万が一ブロックされたら、画面のどこを触っても再生するように仕込む
              document.body.addEventListener('click', function() { video.play(); }, { once: true });
              document.body.addEventListener('touchstart', function() { video.play(); }, { once: true });
            });
          };

          if (video.readyState > 0) {
            attemptPlay();
          } else {
            video.addEventListener('loadeddata', attemptPlay);
          }
        }
      });
    </script>
  </head>

  <body style="margin: 0; overflow: hidden;">
    
    <a-scene embedded arjs="sourceType: webcam; debugUIEnabled: false;" vr-mode-ui="enabled: false">

      <a-assets>
        <video 
          id="snow-video" 
          src="snow.mp4" 
          preload="auto" 
          autoplay="true"
          loop="true" 
          muted="true"
          playsinline 
          webkit-playsinline
          crossOrigin="anonymous">
        </video>
      </a-assets>

      <a-entity camera look-controls auto-play-video>
        
        <a-video 
          src="#snow-video" 
          position="0 0 -4" 
          width="5" 
          height="3"
          scale="1.5 1.5 1.5"
          material="shader: flat; transparent: true; blending: additive; opacity: 1;">
        </a-video>

      </a-entity>

    </a-scene>
  </body>
</html>
