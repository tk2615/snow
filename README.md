<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Red Box Test</title>
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    <script src="https://aframe.io/releases/1.4.2/aframe.min.js"></script>
    <script src="https://raw.githack.com/AR-js-org/AR.js/master/aframe/build/aframe-ar.js"></script>

    <script>
      AFRAME.registerComponent('auto-play-video', {
        init: function () {
          var video = document.querySelector('#snow-video');
          var statusText = document.querySelector('#status-text');

          var attemptPlay = function() {
            video.play().then(() => {
              if(statusText) statusText.innerHTML = "再生中！<br>赤い箱は見えてるか？";
            }).catch(function(error) {
              if(statusText) statusText.innerHTML = "タップして再生！";
              document.body.addEventListener('click', function() { 
                video.play(); 
                if(statusText) statusText.innerHTML = "再生中！<br>赤い箱は見えてるか？";
              }, { once: true });
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
    
    <div id="status-text" style="
      position: fixed; top: 20px; left: 0; width: 100%; 
      text-align: center; color: red; font-weight: bold; font-size: 20px; 
      z-index: 10; text-shadow: 2px 2px 0 #fff;">
      読み込み中...
    </div>

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
          position="-1.5 0 -4" 
          width="2" 
          height="1.5"
          material="shader: flat; transparent: false; opacity: 1; side: double;">
        </a-video>

        <a-box 
          position="1.5 0 -4" 
          color="red" 
          scale="1 1 1">
        </a-box>

      </a-entity>

    </a-scene>
  </body>
</html>
