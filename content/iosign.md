---
title: "iOS IPA Signing Service"
---

This is a private IPA signing service, it is not under credential protection as UDIDs are unique.

<form id="signForm">
  <div class="custom-file-container">
    <label for="ipaFile" style="white-space: nowrap;">Upload IPA File:</label>
    <input type="file" id="ipaFile" name="file" accept=".ipa" style="display: none;">
    <label for="ipaFile" id="ipaFileLabel" class="custom-file-label">Choose file</label>
  </div>
  <br><br>
  <label for="udid">Device UDID:</label>
  <input type="password" id="udid" name="udid" placeholder="Enter your device UDID" required autocomplete="current-password" class="form-input">
  <br><br>
  <button type="submit" class="text-button">Sign IPA</button>
</form>

<br>
<div class="loader" id="loader"></div>
    
<div class="progress-container" id="progressContainer">
  <label for="progressFill">Uploading:</label>
  <div class="custom-progress-bar">
    <div class="custom-progress-fill" id="progressFill"></div>
  </div>
  <span id="progressPercent">0%</span>
</div>
  
<div id="message" class="message"></div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/qrcodejs/1.0.0/qrcode.min.js"></script>
<script>
  const form = document.getElementById('signForm');
  const messageDiv = document.getElementById('message');
  const loader = document.getElementById('loader');
  const progressContainer = document.getElementById('progressContainer');
  const progressFill = document.getElementById('progressFill');
  const progressPercent = document.getElementById('progressPercent');
  const ipaFile = document.getElementById("ipaFile");
  const CHUNK_SIZE = 50 * 1024 * 1024;

  let isUploading = false;
  let activeXhr = null;
        
  let currentProgress = 0;
  let targetProgress = 0;
  let xhr;
        
  const isAppleDevice = () => {
    return /iPhone|iPad|iPod|AppleWatch|Vision/i.test(navigator.userAgent);
  };

  ipaFile.onchange = (event) => {
    if (event.target.files.length > 0) {
      document.getElementById("ipaFileLabel").textContent = event.target.files[0].name;
    }
  };

  const showError = (msg) => {
    loader.style.display = 'none'; 
    progressContainer.style.display = 'none';
    messageDiv.classList.remove('success');
    messageDiv.classList.add('error');
    messageDiv.innerHTML = msg;
    messageDiv.style.display = 'block';
    isUploading = false;
  }; 

  const updateProgress = (percent) => {
    progressFill.style.width = `${percent}%`;
    progressPercent.textContent = `${Math.round(percent)}%`;
  }; 

  const sendRequest = (endpoint, formData, onProgress = null) => { 
    return new Promise((resolve, reject) => {
      const xhr = new XMLHttpRequest();
      activeXhr = xhr;

      if (onProgress) {
        xhr.upload.onprogress = onProgress;
      }

      xhr.open('POST', endpoint, true);

      xhr.onload = () => {
        if (xhr.status >= 200 && xhr.status < 300) {
          try {
            const response = JSON.parse(xhr.responseText);
            resolve(response);
          } catch (e) {
            reject("Invalid JSON response from server");
          }
        } else {
          try {
            const err = JSON.parse(xhr.responseText);
            reject(err.message || `Server Error: ${xhr.status}`);
          } catch (e) {
            reject(`Server Error: ${xhr.status}`);
          }
        }
      };

      xhr.onerror = () => reject("Network Error");
      xhr.onabort = () => reject("Aborted by user");

      xhr.send(formData);
    });
  }; 
        
  form.onsubmit = async (e) => {
    e.preventDefault();
    if (isUploading) return;

    const fileInput = document.getElementById('ipaFile');
    const udidInput = document.getElementById('udid');
    const file = fileInput.files[0];
    const udid = udidInput.value.trim();

    if (!file) return showError("Please select a file.");
    if (!udid) return showError("Please enter UDID.");

    isUploading = true;
    loader.style.display = 'block';
    progressContainer.style.display = 'block';
    messageDiv.style.display = 'none';
    updateProgress(0);

    try {
      console.log("Starting Upload Init...");
      const initData = new FormData();
      initData.append('udid', udid);
      
      const initResponse = await sendRequest('https://iosign.emy.sh/upload/init', initData);
      const uploadId = initResponse.upload_id;
      
      if (!uploadId) throw "No upload ID received.";

      const totalSize = file.size;
      const totalChunks = Math.ceil(totalSize / CHUNK_SIZE);
      let chunkIndex = 0;
      let start = 0;

      while (start < totalSize) {
        const end = Math.min(start + CHUNK_SIZE, totalSize);
        const chunk = file.slice(start, end);
        
        const chunkData = new FormData();
        chunkData.append('upload_id', uploadId);
        chunkData.append('chunk_index', chunkIndex);
        chunkData.append('file', chunk, "blob");

        const onChunkProgress = (event) => {
          if (event.lengthComputable) {
            const loadedGlobal = start + event.loaded;
            const percent = (loadedGlobal / totalSize) * 100;
            updateProgress(percent);
          }
        };

        await sendRequest('https://iosign.emy.sh/upload/chunk', chunkData, onChunkProgress);
        
        start += CHUNK_SIZE;
        chunkIndex++;
      }

      progressContainer.style.display = 'none';
      messageDiv.classList.add('success');
      messageDiv.innerHTML = `<strong>Upload complete. Signing IPA, please wait...</strong>`;
      messageDiv.style.display = 'block';

      const finishData = new FormData();
      finishData.append('upload_id', uploadId);

      const result = await sendRequest('https://iosign.emy.sh/upload/finish', finishData);

      console.log("Signing Complete");
      loader.style.display = 'none';
      messageDiv.innerHTML = `
        <strong>${result.message}</strong>
        <div class="links">
          <a href="${result.ipa_url}" target="_blank">Download Signed IPA</a>
          <br>
          <a href="${result.ota_url}" target="_blank">Download OTA Plist</a>
        </div>
        <br>
      `;

      if (isAppleDevice()) {
        messageDiv.innerHTML += `
          <a href="${result.install_url}" class="install-button">Install on Device</a>
        `;
      } else {
        var qrSize = Math.floor(window.innerWidth * 0.15);
        if (qrSize < 150) qrSize = 150; 
        
        messageDiv.innerHTML += `
          <strong>Scan this QR code with the iOS Camera app to install the IPA</strong>
          <br><br>
          <div id="qrCode" style="width: ${qrSize}px; height: ${qrSize}px; margin: 0 auto;"></div>
        `;
        
        new QRCode(document.getElementById("qrCode"), {
          text: result.install_url,
          width: qrSize,
          height: qrSize,
          colorDark: "#000000",
          colorLight: "#ffffff",
          correctLevel: QRCode.CorrectLevel.M
        });
      }

    } catch (error) {
      showError(error);
    } finally {
      isUploading = false;
      activeXhr = null;
    }
  };
</script>
