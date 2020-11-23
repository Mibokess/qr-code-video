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
      
      await ffmpeg.run('-i', name, '-vf', 'format=rgba,fps=1,scale=320:240', '-f', 'rawvideo', 'out.bin');
      
      message.innerHTML = 'Complete transcoding';

      const data = ffmpeg.FS('readFile', `out.bin`);
      const frames = [];

      const frameSize = 4 * width * height;

      for (let i = 0; i < data.length - frameSize; i += frameSize) {
        frames.push(data.slice(i, i + frameSize))
      }

      for (let frameIndex = 0; frameIndex < frames.length; ++frameIndex) {
        let imageData = new ImageData(width, height);

        for (let i = 0; i < imageData.length; i += 4) {
          imageData[i] = frames[frameIndex][i]; 
          imageData[i + 1] = frames[frameIndex][i + 1]; 
          imageData[i + 2] = frames[frameIndex][i + 2]; 
          imageData[i + 3] = frames[frameIndex][i + 3]; 
        }

        const code = jsQR(imageData, width, height);
  
        if (code) {
          console.log('code');
          console.log(code);
        } else {
          console.log("No code");
        }

      }
    }
  });
</script>