<div class="flex">
  <div>
    <div>
      <video id="webcam" width="{width}px" height="{height}px"></video>
      <canvas id="canvas" src={src} width="{width}px" height="{height}px"/>
    </div>

    <button id="record" disabled>Start Recording</button>
    <p id="message"></p>
  </div>
  <div>
    <div>
      <input id="input" type="text" bind:value={inputValue} on:change={encode}/>
    </div>
    <div>
      <div id="qrCode"/>
      <canvas id="qrCode2" height="400px" width="400px"/>
    </div>
  </div>
</div>

<script lang="ts">
  import { onMount } from 'svelte';
  import { createFFmpeg, fetchFile } from "@ffmpeg/ffmpeg";
  import jsQR from 'jsqr';
  
  import { qrcode, modes, ecLevel } from 'qrcode.es';
  
  const width = 400;
  const height = 400;

  let inputValue = "";
  let src = "";
  const ffmpeg = createFFmpeg({ log: true });

  onMount(() => {  
    const webcam: HTMLVideoElement = document.getElementById('webcam');
    const recordBtn: HTMLButtonElement = document.getElementById('record');

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

      await ffmpeg.load();
    })();
  
    const transcode = async (webcamData) => {
      const message = document.getElementById('message');
      const name = 'record.webm';
      
      message.innerHTML = 'Loading ffmpeg-core.js';
      
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

      console.log('frames')
      console.log(frames)

      for (let frameIndex = 0; frameIndex < frames.length; ++frameIndex) {
        let imageData: ImageData = new ImageData(new Uint8ClampedArray(frames[frameIndex]), width, height);

        var canvas: HTMLCanvasElement = document.getElementById('canvas');
        var ctx = canvas.getContext('2d');

        ctx.putImageData(imageData, width, height)

        console.log(imageData)
                
        ctx.putImageData(imageData, imageData.height, imageData.width)

        const code = jsQR(imageData.data, width, height);
  
        if (code) {
          console.log('code');
          console.log(code);
        } else {
          console.log("No code");
        }

      }
    }

  });

  let qrCode = null

  async function encode() {
    if (inputValue == "") return;

    const qrCodeSetting = {
        size: 300,
        minVersion: 1,
        maxVersion: 13, // 330 bytes
        ecLevel: ecLevel.MEDIUM,
    };

    const element = document.getElementById("qrCode"); //Element must be an instance of HTMLCanvasElement or HTMLDivElement
    
    if (qrCode == null) {
      qrCode = new qrcode(element); //Initializing the QrCode
    }
    
    console.log(qrCode)

    await qrCode.generate(inputValue, qrCodeSetting); // Function that generates the QrCode
    
    //let image = this.qrCode.getImage(); // Function to get the data Url of the QrCode Image

  }
</script>