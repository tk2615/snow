<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <title>Debug Snow AR</title>
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
              console.log("再生成功！");
              if(statusText) statusText.innerHTML = "再生中！<br>(黒い四角が見えれば成功)";
            }).catch(function(error) {
              console.log("ブロックされた！");
              if(statusText) statusText.innerHTML = "再生停止中...<br>画面をタップして！";
              document.body.addEventListener('click', function() { 
                video.play(); 
                if(statusText) statusText.innerHTML = "再生中！<br>(黒い四角が見えれば成功)";
              }, { once: true });
            });
          };

          if (video.readyState > 0) {
            attemptPlay();
          } else {
            video.addEventListener('loadeddata', attemptPlay);
          }
          
          // エラー検知：もしファイルが見つからない時はここでわかる
          video.addEventListener('error', function(e) {
            if(statusText) statusText.innerHTML = "エラー発生！<br>snow.mp4が見つからへんか、壊れてるで！";
            statusText.style.color = "red";
          });
        }
      });
    </script>
  </head>

  <body style="margin: 0; overflow: hidden;">
    
    <div id="status-text" style="
      position: fixed; top: 20px; left: 0; width: 100%; 
      text-align: center; color: yellow; font-weight: bold; font-size: 20px; 
      z-index: 10; text-shadow: 2px 2px 0 #000;">
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
          position="0 0 -4" 
          width="4" 
          height="2.25"
          material="shader: flat; transparent: false; blending: none; opacity: 1;">
        </a-video>

      </a-entity>

    </a-scene>
  </body>
</html>
