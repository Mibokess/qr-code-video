<div>
  <div>
    <video id="webcam" width="{width}px" height="{height}px"></video>
    <canvas id="canvas" src={src} width="{width}px" height="{height}px"/>
  </div>

  <button id="record" disabled>Start Recording</button>
  <p id="message"></p>
</div>

<script lang="js">
  import { onMount } from 'svelte';
  import { createFFmpeg, fetchFile } from "@ffmpeg/ffmpeg";
  import jsQR from 'jsqr';
  
  const width = 320;
  const height = 240;

  let src = "";
  const ffmpeg = createFFmpeg({ log: true });

  onMount(() => {  
    const webcam = document.getElementById('webcam');
    const recordBtn = document.getElementById('record');

    const startRecording = () => {
      const rec = new MediaRecorder(webcam.srcObject);
      const chunks = [];
  
      recordBtn.textContent = 'Stop Recording';
      recordBtn.onclick = () => {
        rec.stop();
        recordBtn.textContent = 'Start Recording';
        recordBtn.onclick = startRecording;
      }

      rec.ondataavailable = e => chunks.push(e.data);
      rec.onstop = async () => {
        transcode(new Uint8Array(await (new Blob(chunks)).arrayBuffer()));
      };
      rec.start();
    };

    (async () => {
      webcam.srcObject = await navigator.mediaDevices.getUserMedia({ video: true, audio: false });
      
      await webcam.play();

      recordBtn.disabled = false;
      recordBtn.onclick = startRecording;
    })();
  
    const transcode = async (webcamData) => {
      const message = document.getElementById('message');
      const name = 'record.webm';
      
      message.innerHTML = 'Loading ffmpeg-core.js';
      
      await ffmpeg.load();
      
      message.innerHTML = 'Start transcoding';
      
      ffmpeg.FS('writeFile', name, await fetchFile(webcamData));
      
      await ffmpeg.run('-i', name, '-vf', 'fps=1,scale=320:240', 'out%d.jpg');
      
      message.innerHTML = 'Complete transcoding';

      for(let i = 1; i < 2; i++) {
        try {
          const data = ffmpeg.FS('readFile', `out${i}.jpg`);

          var canvas = document.getElementById('canvas');
          var context = canvas.getContext('2d');
          var img = new Image;
          img.src = URL.createObjectURL(new Blob([data.buffer], { type: 'img/jpg' }));
          
          img.onload = () => {
            canvas.width = img.width;
            canvas.height = img.height;
            context.drawImage(img, 0, 0);

            var imageData = context.getImageData(0, 0, img.width, img.height);
            
            const code = jsQR(imageData, width, height);

            if (code) {
              console.log('code');
              console.log(code);
            } else {
              console.log("No code");
            }
          }
        } catch (e) {
          console.log(e);
          break;
        }
      }
    }
  });
</script>