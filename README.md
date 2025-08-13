<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Encrypted Chat</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      background-color: #000;
      color: #fff;
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
      border: 1px solid #444;
      padding: 10px;
      background-size: cover;
      background-position: center;
    }
    .message {
      padding: 8px;
      margin: 5px;
      border-radius: 5px;
      word-wrap: break-word;
    }
    .sent {
      background: #1e90ff;
      color: white;
      text-align: right;
    }
    .received {
      background: #333;
      color: white;
      text-align: left;
    }
    .chat-media {
      max-width: 100%;
      border-radius: 8px;
    }
    #settingsModal {
      display: none;
      position: fixed;
      top: 0; left: 0;
      width: 100%;
      height: 100%;
      background: rgba(0,0,0,0.8);
      padding: 20px;
    }
    #settingsModalContent {
      background: #111;
      padding: 20px;
      max-width: 400px;
      margin: auto;
      border-radius: 8px;
    }
    button {
      padding: 8px 12px;
      margin: 5px;
      cursor: pointer;
    }
  </style>
  <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-app.js"></script>
  <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-database.js"></script>
  <script src="https://www.gstatic.com/firebasejs/8.10.1/firebase-storage.js"></script>
  <script src="app-lock.js"></script>
</head>
<body>
  <div id="lockScreen"></div>

  <div id="loginPage">
    <h2>Join Chat</h2>
    <input id="room" placeholder="Room name"><br>
    <input id="key" placeholder="Secret key" type="password"><br>
    <button onclick="joinRoom()">Join</button>
    <button id="settingsBtn">Settings</button>
  </div>

  <div id="chatPage" style="display:none;">
    <div id="chat"></div>
    <input id="message" placeholder="Type a message">
    <button onclick="sendMessage()">Send</button>
    <button onclick="attachFile()">📎</button>
    <button onclick="changeBackground()">🖼</button>
  </div>

  <div id="settingsModal">
    <div id="settingsModalContent">
      <h3>Settings</h3>
      <label><input type="checkbox" id="enableAppLock"> Enable App Lock</label><br>
      <input type="password" id="newAppPassword" placeholder="New password"><br>
      <button id="saveSettings">Save</button>
      <button id="closeSettings">Close</button>
    </div>
  </div>

  <script>
    const firebaseConfig = {
      apiKey: "...",
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
    let lock;

    async function initApp() {
      lock = await AppLock.bootstrap({ appName: "EncryptedChat", inactivityMs: 2 * 60_000 });
      lock.renderLockUI(document.getElementById('lockScreen'), async ({ pin }) => {
        const masterKey = await deriveOrLoadMasterKey();
        await lock.sealMasterKey(masterKey, pin);
        showLogin();
      });
      if (await lock.isLocked()) {
        document.getElementById("loginPage").style.display = "none";
        document.getElementById("chatPage").style.display = "none";
      } else {
        showLogin();
      }
    }

    async function deriveOrLoadMasterKey() {
      const stored = localStorage.getItem("masterKeyRaw");
      if (stored) {
        return Uint8Array.from(atob(stored), c => c.charCodeAt(0));
      }
      const random = crypto.getRandomValues(new Uint8Array(32));
      localStorage.setItem("masterKeyRaw", btoa(String.fromCharCode(...random)));
      return random;
    }

    function showLogin() {
      document.getElementById("lockScreen").style.display = "none";
      document.getElementById("loginPage").style.display = "block";
      const savedBg = localStorage.getItem("chatBackground");
      if (savedBg) {
        document.getElementById("chat").style.backgroundImage = `url(${savedBg})`;
      }
    }

    document.getElementById("settingsBtn").addEventListener("click", () => {
      document.getElementById("settingsModal").style.display = "block";
    });
    document.getElementById("closeSettings").addEventListener("click", () => {
      document.getElementById("settingsModal").style.display = "none";
    });
    document.getElementById("saveSettings").addEventListener("click", async () => {
      const newPass = document.getElementById("newAppPassword").value;
      const enabled = document.getElementById("enableAppLock").checked;
      if (enabled && !newPass) {
        alert("Please enter a password to enable App Lock.");
        return;
      }
      if (enabled && newPass) {
        const masterKey = await deriveOrLoadMasterKey();
        await lock.sealMasterKey(masterKey, newPass);
        alert("App Lock enabled and key sealed!");
      } else if (!enabled) {
        await lock.lockNow();
        alert("App Lock disabled!");
      }
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
      const separator = document.createElement("div");
      separator.style.textAlign = "center";
      separator.style.color = "#aaa";
      separator.textContent = "— New Session —";
      document.getElementById("chat").appendChild(separator);

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
        div.className = (data.sender === myId) ? "message sent" : "message received";
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

    window.addEventListener("load", initApp);
  </script>
</body>
</html>
