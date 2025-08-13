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
      height: 430px;
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
      margin: 6px;
      border-radius: 15px;
      max-width: 78%;
      word-wrap: break-word;
      font-size: 15px;
      position: relative;
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
    .meta {
      font-size: 11px;
      opacity: 0.7;
      margin-top: 4px;
    }
    .reply-snippet {
      font-size: 12px;
      opacity: 0.8;
      border-left: 3px solid #888;
      padding-left: 8px;
      margin-bottom: 6px;
      max-height: 3.6em;
      overflow: hidden;
    }
    .actions {
      position: absolute;
      bottom: -18px;
      right: 8px;
      font-size: 12px;
      opacity: 0.8;
    }
    .actions button {
      background: transparent;
      color: inherit;
      border: 1px solid #666;
      border-radius: 10px;
      padding: 2px 6px;
      margin-left: 6px;
      cursor: pointer;
    }
    input, button {
      padding: 10px;
      margin: 5px 0;
      border-radius: 5px;
      border: none;
      font-size: 16px;
    }
    #message {
      width: calc(100% - 190px);
    }
    #sendBtn {
      width: 90px;
      background: #0b93f6;
      color: white;
      margin-left: 6px;
    }
    #attachBtn {
      width: 50px;
      background: #28a745;
      color: white;
      margin-left: 6px;
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
      width: 90%;
      max-width: 420px;
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
    /* Reply composer preview */
    #replyPreview {
      display:none;
      background: #1c1c1c;
      border-left: 3px solid #0b93f6;
      padding: 8px 10px;
      margin-bottom: 8px;
      border-radius: 6px;
      font-size: 13px;
      opacity: 0.9;
    }
    #replyPreview .close {
      float: right;
      cursor: pointer;
      font-weight: bold;
      margin-left: 8px;
    }
    /* Hidden inputs */
    #filePicker {
      display: none;
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

    <!-- Reply preview -->
    <div id="replyPreview">
      <span id="replyClose" class="close">✕</span>
      Replying to:
      <div id="replyText"></div>
    </div>

    <input type="text" id="message" placeholder="Type a message">
    <button id="attachBtn" onclick="triggerAttach()">📎</button>
    <button id="sendBtn" onclick="sendMessage()">Send</button>
    <input id="filePicker" type="file" accept="image/*,video/*">
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

    const db = firebase.database();
    const storage = firebase.storage();

    // ---- PERSISTENT USER ID to avoid merging after reload ----
    let myId = localStorage.getItem("myId");
    if (!myId) {
      myId = "u_" + Date.now().toString(36) + Math.random().toString(36).slice(2,8);
      localStorage.setItem("myId", myId);
    }

    let roomName = "";
    let secretKey = "";
    let replyTarget = null; // { id, text }

    // Lock screen check
    window.addEventListener("load", () => {
      const lockEnabled = localStorage.getItem("appLockEnabled") === "true";
      document.getElementById("lockScreen").style.display = lockEnabled ? "block" : "none";
      document.getElementById("loginPage").style.display = lockEnabled ? "none" : "block";

      const savedBg = localStorage.getItem("chatBackground");
      if (savedBg) document.getElementById("chat").style.backgroundImage = `url(${savedBg})`;
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

    // --- XOR "encryption" (kept from your app) ---
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

    // --- Join room & listeners ---
    let roomRef = null;
    function joinRoom() {
      roomName = document.getElementById("room").value.trim();
      secretKey = document.getElementById("key").value.trim();
      if (!roomName || !secretKey) {
        alert("Please enter room name and secret key");
        return;
      }
      document.getElementById("loginPage").style.display = "none";
      document.getElementById("chatPage").style.display = "block";

      roomRef = db.ref(roomName);
      // Clear UI first
      document.getElementById("chat").innerHTML = "";

      // Load existing and future messages
      roomRef.off(); // safety
      roomRef.on("child_added", renderMessage);
      roomRef.on("child_changed", updateMessageSeen);

      // Mark already loaded messages as seen (for others)
      // and continue to mark as user scrolls/keeps chat open
      observeSeen();
    }

    // --- Render message with reply + seen meta ---
    function renderMessage(snapshot) {
      const data = snapshot.val();
      const msgId = snapshot.key;

      // data fields expected:
      // { text: <encrypted>, sender: <id>, ts: <ms>, replyTo: <id|null>, seenBy: { userId: true } | undefined }

      let decrypted = "";
      try { decrypted = decrypt(data.text, secretKey); }
      catch (e) { decrypted = "[Unable to decrypt]"; }

      const wrap = document.createElement("div");
      wrap.className = "message " + (data.sender === myId ? "sent" : "received");
      wrap.setAttribute("data-id", msgId);

      // Reply block (if any)
      if (data.replyTo) {
        const replyDiv = document.createElement("div");
        replyDiv.className = "reply-snippet";
        replyDiv.textContent = "↪ " + (window._messageCache && window._messageCache[data.replyTo] ? trimForReply(window._messageCache[data.replyTo]) : "Quoted message");
        wrap.appendChild(replyDiv);
      }

      // Media vs text
      if (decrypted.startsWith("file:")) {
        const fileUrl = decrypted.replace("file:", "");
        const link = document.createElement("a");
        link.href = fileUrl; link.target = "_blank"; link.download = fileUrl.split('/').pop();

        if (/\.(jpeg|jpg|png|gif|webp)$/i.test(fileUrl)) {
          const img = document.createElement("img");
          img.src = fileUrl; img.className = "chat-media";
          link.appendChild(img);
        } else if (/\.(mp4|webm|ogg)$/i.test(fileUrl)) {
          const video = document.createElement("video");
          video.src = fileUrl; video.controls = true; video.className = "chat-media";
          link.appendChild(video);
        } else {
          link.textContent = fileUrl;
        }
        wrap.appendChild(link);
        cacheMessageText(msgId, "[media]");
      } else {
        wrap.appendChild(document.createTextNode(decrypted));
        cacheMessageText(msgId, decrypted);
      }

      // Actions (Reply / Delete-own)
      const actions = document.createElement("div");
      actions.className = "actions";
      const replyBtn = document.createElement("button");
      replyBtn.textContent = "Reply";
      replyBtn.onclick = () => setReplyTarget(msgId, window._messageCache[msgId] || "");
      actions.appendChild(replyBtn);

      if (data.sender === myId) {
        const delBtn = document.createElement("button");
        delBtn.textContent = "Delete";
        delBtn.onclick = () => {
          if (confirm("Delete this message?")) roomRef.child(msgId).remove();
          wrap.remove();
        };
        actions.appendChild(delBtn);
      }
      wrap.appendChild(actions);

      // Meta: time + seen ticks (for own messages)
      const meta = document.createElement("div");
      meta.className = "meta";
      const ts = data.ts ? new Date(data.ts) : new Date();
      let ticks = "";
      if (data.sender === myId) {
        const seenCount = data.seenBy ? Object.keys(data.seenBy).filter(uid => uid !== myId).length : 0;
        ticks = seenCount > 0 ? " ✓✓" : " ✓";
      }
      meta.textContent = ts.toLocaleTimeString() + ticks;
      wrap.appendChild(meta);

      document.getElementById("chat").appendChild(wrap);
      document.getElementById("chat").scrollTop = document.getElementById("chat").scrollHeight;

      // Mark as seen if it's not mine
      if (data.sender !== myId) {
        markSeen(msgId);
      }
    }

    function updateMessageSeen(snapshot) {
      // Update ticks on existing rendered node
      const data = snapshot.val();
      const msgId = snapshot.key;
      const node = document.querySelector(`.message[data-id="${msgId}"]`);
      if (!node) return;
      const meta = node.querySelector(".meta");
      if (!meta) return;
      if (data.sender === myId) {
        const seenCount = data.seenBy ? Object.keys(data.seenBy).filter(uid => uid !== myId).length : 0;
        const timePart = meta.textContent.replace(/ ✓✓| ✓/g, "");
        meta.textContent = timePart + (seenCount > 0 ? " ✓✓" : " ✓");
      }
    }

    // --- Seen handling ---
    function markSeen(messageId) {
      if (!roomRef) return;
      const seenPath = roomRef.child(messageId).child("seenBy").child(myId);
      seenPath.set(true).catch(()=>{});
    }

    function observeSeen() {
      // Mark messages as seen on initial load and when user interacts
      const chat = document.getElementById("chat");
      const markAllVisible = () => {
        const nodes = chat.querySelectorAll(".message.received");
        nodes.forEach(n => {
          const id = n.getAttribute("data-id");
          markSeen(id);
        });
      };
      markAllVisible();
      chat.addEventListener("scroll", () => {
        if (chat.scrollTop + chat.clientHeight >= chat.scrollHeight - 10) {
          markAllVisible();
        }
      });
    }

    // --- Send text or media ---
    function sendMessage() {
      if (!roomRef) return;
      const msgInput = document.getElementById("message");
      const raw = msgInput.value.trim();
      if (raw === "" && !replyTarget) return;

      const payload = {
        text: encrypt(raw || "", secretKey),
        sender: myId,
        ts: Date.now(),
        replyTo: replyTarget ? replyTarget.id : null
      };
      roomRef.push(payload);
      msgInput.value = "";
      clearReplyTarget();
    }

    // Hidden file input for reliable re-use
    document.getElementById("filePicker").addEventListener("change", function (e) {
      const file = e.target.files[0];
      if (!file || !roomName) { this.value = ""; return; }
      const path = `${roomName}/${Date.now()}_${file.name}`;
      const storageRef = storage.ref(path);
      storageRef.put(file).then(snap => snap.ref.getDownloadURL()).then(url => {
        const payload = {
          text: encrypt("file:" + url, secretKey),
          sender: myId,
          ts: Date.now(),
          replyTo: replyTarget ? replyTarget.id : null
        };
        roomRef.push(payload);
        clearReplyTarget();
      }).catch(err => {
        alert("Upload failed");
        console.error(err);
      }).finally(() => {
        // IMPORTANT: reset so it can be picked again next time
        e.target.value = "";
      });
    });

    function triggerAttach() {
      document.getElementById("filePicker").click();
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

    // --- Reply feature (UI state) ---
    function setReplyTarget(id, text) {
      replyTarget = { id, text };
      const preview = document.getElementById("replyPreview");
      document.getElementById("replyText").textContent = trimForReply(text);
      preview.style.display = "block";
    }
    function clearReplyTarget() {
      replyTarget = null;
      document.getElementById("replyPreview").style.display = "none";
      document.getElementById("replyText").textContent = "";
    }
    document.getElementById("replyClose").addEventListener("click", clearReplyTarget);

    function trimForReply(text) {
      if (!text) return "";
      const t = (""+text).replace(/^file:.*/,'[media]');
      return t.length > 120 ? t.slice(0, 117) + "..." : t;
    }

    // Cache message text locally to show reply snippets even before fetch
    window._messageCache = {};
    function cacheMessageText(id, text) {
      window._messageCache[id] = text;
    }
  </script>
</body>
</html>
