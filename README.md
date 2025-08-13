<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Encrypted Chat</title>
  <style>
    body {
      font-family: "Segoe UI Emoji", Arial, sans-serif;
      background-color: #000;
      color: white;
      margin: 0;
      padding: 0;
    }
    #loginPage, #chatPage, #lockScreen {
      max-width: 600px;
      margin: auto;
      padding: 20px;
    }
    #chat {
      height: 400px;
      overflow-y: auto;
      border: 1px solid #333;
      padding: 10px;
      background-color: #111;
      background-size: cover;
      background-position: center;
      border-radius: 10px;
    }
    .message {
      padding: 8px 12px;
      margin: 5px;
      border-radius: 15px;
      max-width: 70%;
      word-wrap: break-word;
      font-size: 16px;
    }
    .sent {
      background: white;
      color: black;
      text-align: right;
      margin-left: auto;
      cursor: pointer;
    }
    .received {
      background: black;
      color: white;
      text-align: left;
      border: 1px solid #444;
      margin-right: auto;
    }
    input, button {
      padding: 10px;
      margin: 5px 0;
      border-radius: 5px;
      border: none;
      font-size: 16px;
    }
    #message {
      width: 60%;
    }
    #sendBtn {
      width: 15%;
      background: #0b93f6;
      color: white;
    }
    #attachBtn {
      width: 15%;
      background: #28a745;
      color: white;
    }
    #loginPage input {
      width: 100%;
      margin-bottom: 10px;
    }
    #joinBtn {
      background: #0b93f6;
      color: white;
      width: 100%;
    }
    img.chat-media, video.chat-media {
      max-width: 100%;
      border-radius: 10px;
      display: block;
    }
    /* Settings modal */
    #settingsModal {
      display:none;
      position:fixed;
      top:50%;
      left:50%;
      transform:translate(-50%,-50%);
      background:white;
      color:black;
      padding:20px;
      border-radius:8px;
      box-shadow:0 0 10px rgba(0,0,0,0.3);
      z-index:999;
    }
    #settingsBtn {
      position:absolute;
      top:10px;
      right:10px;
      background:none;
      border:none;
      font-size:20px;
      cursor:pointer;
      color:white;
    }
    /* Lock screen */
    #lockScreen {
      text-align: center;
    }
    #unlockBtn {
      background: #0b93f6;
      color: white;
      width: 100%;
    }
  </style>
</head>
<body>

  <!-- Lock Screen -->
  <div id="lockScreen" style="display:none;">
    <h2>App Locked 🔒</h2>
    <input type="password" id="unlockPassword" placeholder="Enter App Lock Password">
    <button id="unlockBtn">Unlock</button>
  </div>

  <!-- Page 1: Login -->
  <div id="loginPage">
    <h2>Join Chat Room</h2>
    <input type="text" id="room" placeholder="Room Name"><br>
    <input type="text" id="key" placeholder="Secret Key"><br>
    <button id="joinBtn" onclick="joinRoom()">Join Chat</button>
  </div>

  <!-- Page 2: Chat -->
  <div id="chatPage" style="display:none; position:relative;">
    <button id="settingsBtn">⚙️</button>
    <button onclick="changeBackground()" style="background:#555;color:white;margin-bottom:10px;">
      Change Chat Background
    </button>
    <div id="chat"></div>
    <input type="text" id="message" placeholder="Type a message">
    <button id="attachBtn" onclick="attachFile()">📎</button>
    <button id="sendBtn" onclick="sendMessage()">Send</button>
  </div>

  <!-- Settings Modal -->
  <div id="settingsModal">
    <h3>App Lock Settings</h3>
    <label>
      New PIN/Password:
      <input type="password" id="newAppPassword" />
    </label>
    <br><br>
    <label>
      <input type="checkbox" id="enableAppLock" /> Enable App Lock
    </label>
    <br><br>
    <button id="saveSettings">Save</button>
    <button id="closeSettings">Close</button>
  </div>

  <!-- Firebase SDK v8 -->
  <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-app.js"></script>
  <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-database.js"></script>
  <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-storage.js"></script>

  <script>
    const firebaseConfig = {
      apiKey: "AIzaSyCyXRwlhZiOK3rVYa6lHGg0rHB5CS6h1W4",
      authDomain: "my-chat-3594d.firebaseapp.com",
      databaseURL: "https://my-chat-3594d-default-rtdb.firebaseio.com",
      projectId: "my-chat-3594d",
      storageBucket: "my-chat-3594d.appspot.com",
      messagingSenderId: "208703073410",
      appId: "1:208703073410:web:4ed522eb2cc522c00cd89b",
      measurementId: "G-CVC6XSQ5JW"
    };
    firebase.initializeApp(firebaseConfig);

    let db = firebase.database();
    let storage = firebase.storage();
    let roomName = "";
    let secretKey = "";
    let myId = Date.now().toString();

    // Lock screen check
    window.addEventListener("load", () => {
      const lockEnabled = localStorage.getItem("appLockEnabled") === "true";
      if (lockEnabled) {
        document.getElementById("lockScreen").style.display = "block";
        document.getElementById("loginPage").style.display = "none";
      } else {
        document.getElementById("lockScreen").style.display = "none";
        document.getElementById("loginPage").style.display = "block";
      }
      const savedBg = localStorage.getItem("chatBackground");
      if (savedBg) {
        document.getElementById("chat").style.backgroundImage = `url(${savedBg})`;
      }
    });

    document.getElementById("unlockBtn").addEventListener("click", () => {
      const pass = document.getElementById("unlockPassword").value;
      const savedPass = localStorage.getItem("appLockPassword");
      if (pass === savedPass) {
        document.getElementById("lockScreen").style.display = "none";
        document.getElementById("loginPage").style.display = "block";
      } else {
        alert("Incorrect password.");
      }
    });

    // Settings modal
    document.getElementById("settingsBtn").addEventListener("click", () => {
      document.getElementById("settingsModal").style.display = "block";
      document.getElementById("newAppPassword").value = localStorage.getItem("appLockPassword") || "";
      document.getElementById("enableAppLock").checked = localStorage.getItem("appLockEnabled") === "true";
    });
    document.getElementById("closeSettings").addEventListener("click", () => {
      document.getElementById("settingsModal").style.display = "none";
    });
    document.getElementById("saveSettings").addEventListener("click", () => {
      const newPass = document.getElementById("newAppPassword").value;
      const enabled = document.getElementById("enableAppLock").checked;
      if (enabled && !newPass) {
        alert("Please enter a password to enable App Lock.");
        return;
      }
      if (newPass) localStorage.setItem("appLockPassword", newPass);
      localStorage.setItem("appLockEnabled", enabled);
      alert("Settings saved!");
      document.getElementById("settingsModal").style.display = "none";
    });

    function encrypt(text, key) {
      let utf8Text = unescape(encodeURIComponent(text));
      let result = '';
      for (let i = 0; i < utf8Text.length; i++) {
        result += String.fromCharCode(utf8Text.charCodeAt(i) ^ key.charCodeAt(i % key.length));
      }
      return btoa(result);
    }

    function decrypt(text, key) {
      let decoded = atob(text);
      let result = '';
      for (let i = 0; i < decoded.length; i++) {
        result += String.fromCharCode(decoded.charCodeAt(i) ^ key.charCodeAt(i % key.length));
      }
      return decodeURIComponent(escape(result));
    }

    function joinRoom() {
      roomName = document.getElementById("room").value.trim();
      secretKey = document.getElementById("key").value.trim();
      if (!roomName || !secretKey) {
        alert("Please enter room name and secret key");
        return;
      }
      document.getElementById("loginPage").style.display = "none";
      document.getElementById("chatPage").style.display = "block";
      db.ref(roomName).on("child_added", function(snapshot) {
        let data = snapshot.val();
        let decrypted = decrypt(data.text, secretKey);
        let div = document.createElement("div");
        div.setAttribute("data-id", snapshot.key);
        if (decrypted.startsWith("file:")) {
          let fileUrl = decrypted.replace("file:", "");
          let link = document.createElement("a");
          link.href = fileUrl;
          link.target = "_blank";
          link.download = fileUrl.split('/').pop();
          if (fileUrl.match(/\.(jpeg|jpg|png|gif)$/i)) {
            let img = document.createElement("img");
            img.src = fileUrl;
            img.className = "chat-media";
            link.appendChild(img);
          } else if (fileUrl.match(/\.(mp4|webm|ogg)$/i)) {
            let video = document.createElement("video");
            video.src = fileUrl;
            video.controls = true;
            video.className = "chat-media";
            link.appendChild(video);
          }
          div.appendChild(link);
        } else {
          div.textContent = decrypted;
        }
        if (data.sender === myId) {
          div.className = "message sent";
          div.onclick = function () {
            if (confirm("Delete this message?")) {
              db.ref(roomName).child(snapshot.key).remove();
              div.remove();
            }
          };
        } else {
          div.className = "message received";
        }
        document.getElementById("chat").appendChild(div);
        document.getElementById("chat").scrollTop = document.getElementById("chat").scrollHeight;
      });
    }

    function sendMessage() {
      let msgInput = document.getElementById("message");
      let msg = msgInput.value.trim();
      if (msg === "") return;
      let encrypted = encrypt(msg, secretKey);
      db.ref(roomName).push({ text: encrypted, sender: myId });
      msgInput.value = "";
      msgInput.focus();
    }

    function attachFile() {
      const fileInput = document.createElement("input");
      fileInput.type = "file";
      fileInput.accept = "image/*,video/*";
      fileInput.onchange = function(e) {
        const file = e.target.files[0];
        if (!file) return;
        const storageRef = storage.ref(`${roomName}/${Date.now()}_${file.name}`);
        storageRef.put(file).then(snapshot => {
          snapshot.ref.getDownloadURL().then(url => {
            let encryptedUrl = encrypt("file:" + url, secretKey);
            db.ref(roomName).push({ text: encryptedUrl, sender: myId });
          });
        });
      };
      fileInput.click();
    }

    function changeBackground() {
      const fileInput = document.createElement("input");
      fileInput.type = "file";
      fileInput.accept = "image/*";
      fileInput.onchange = function(e) {
        const file = e.target.files[0];
        const reader = new FileReader();
        reader.onload = function(event) {
          const imgData = event.target.result;
          document.getElementById("chat").style.backgroundImage = `url(${imgData})`;
          localStorage.setItem("chatBackground", imgData);
        };
        if (file) reader.readAsDataURL(file);
      };
      fileInput.click();
    }
  </script>
</body>
</html>
