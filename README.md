<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Encrypted Chat App</title>
  <style>
    body { font-family: "Segoe UI Emoji", Arial, sans-serif; background: #000; color:#fff; margin:0; }
    .page { display:none; padding:20px; max-width:600px; margin:auto; }
    #authPage{display:block;}
    #chat { height: 420px; overflow-y: auto; padding: 10px; background: #111; border: 1px solid #333; border-radius:10px; }
    .wrapper { margin: 10px 0; display:flex; flex-direction:column; }
    .name-tag { font-size:12px; opacity:.7; margin: 0 0 4px 4px; }
    .message { padding: 8px 12px; border-radius: 14px; max-width: 70%; word-wrap: break-word; font-size:16px; }
    .sent { background:#fff; color:#000; align-self:flex-end; cursor:pointer; }
    .received { background:#000; color:#fff; border:1px solid #444; align-self:flex-start; }
    .msg-time { font-size: 12px; opacity:.6; margin-top:3px; align-self:flex-end; }
    .seen-status { font-size: 12px; margin-left: 6px; opacity:.7; }
    .chat-media { max-width:100%; border-radius:10px; display:block; margin-top:6px; }
    .reply-preview { font-size: 12px; background: #222; padding: 6px; border-left: 3px solid #888; margin-bottom: 6px; border-radius: 6px; }
    #replyBox { display:none; background:#222; padding:8px; margin:8px 0; border-radius:6px; }
    input, button { padding:10px; border:none; border-radius:6px; margin:5px 0; font-size:16px; }
    button { background:#0b93f6; color:#fff; cursor:pointer; }
    #fileInput { background:#333; color:#fff; }
    a { color:#9cf; }
  </style>
</head>
<body>

<!-- Auth -->
<div id="authPage" class="page">
  <h2>Sign Up</h2>
  <input id="signupUser" placeholder="Username">
  <input id="signupPass" placeholder="Password" type="password">
  <button onclick="signUp()">Sign Up</button>

  <h2 style="margin-top:20px;">Login</h2>
  <input id="loginUser" placeholder="Username">
  <input id="loginPass" placeholder="Password" type="password">
  <button onclick="login()">Login</button>
</div>

<!-- Join Room -->
<div id="roomPage" class="page">
  <h2>Join Room</h2>
  <input id="room" placeholder="Room Name">
  <input id="key" placeholder="Secret Key">
  <button onclick="joinRoom()">Join</button>
</div>

<!-- Chat -->
<div id="chatPage" class="page">
  <h2 style="margin:0 0 10px 0;">Encrypted Chat</h2>
  <div id="chat"></div>

  <div id="replyBox">
    Replying to: <span id="replyPreview"></span>
    <button style="background:#555;margin-left:10px;" onclick="cancelReply()">Cancel</button>
  </div>

  <input id="message" placeholder="Type message...">
  <div>
    <input type="file" id="fileInput" accept="image/*,video/*">
    <button onclick="sendMessage()">Send</button>
  </div>
</div>

<!-- Firebase v8 -->
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-database.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-storage.js"></script>

<script>
  // === Firebase config (your project) ===
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

  // === Simple user DB (username/password + UID) ===
  const authDb = db.ref("users");

  let myId = null;     // unique uid
  let myName = null;   // username
  let roomName = "";
  let secretKey = "";
  let replyToMsg = null;

  // === Unicode-safe XOR encryption (your original scheme) ===
  function encrypt(text, key) {
    const utf8Text = unescape(encodeURIComponent(text));
    let result = '';
    for (let i = 0; i < utf8Text.length; i++) {
      result += String.fromCharCode(utf8Text.charCodeAt(i) ^ key.charCodeAt(i % key.length));
    }
    return btoa(result);
  }
  function decrypt(text, key) {
    try {
      const decoded = atob(text);
      let result = '';
      for (let i = 0; i < decoded.length; i++) {
        result += String.fromCharCode(decoded.charCodeAt(i) ^ key.charCodeAt(i % key.length));
      }
      return decodeURIComponent(escape(result));
    } catch (e) {
      return "[Encrypted]";
    }
  }

  // === Helpers ===
  function genUid() {
    try {
      const arr = new Uint32Array(4);
      crypto.getRandomValues(arr);
      return 'u_' + Array.from(arr).map(n => n.toString(36)).join('') + Date.now().toString(36);
    } catch {
      return 'u_' + Math.random().toString(36).slice(2) + Date.now().toString(36);
    }
  }

  // === Auth ===
  function signUp() {
    const username = document.getElementById("signupUser").value.trim();
    const password = document.getElementById("signupPass").value.trim();
    if (!username || !password) return alert("Enter username & password");
    authDb.child(username).once("value", snap => {
      if (snap.exists()) return alert("Username already taken!");
      const uid = genUid();
      authDb.child(username).set({ password, uid, createdAt: Date.now() });
      alert("Account created! You can now log in.");
    });
  }

  function login() {
    const username = document.getElementById("loginUser").value.trim();
    const password = document.getElementById("loginPass").value.trim();
    if (!username || !password) return alert("Enter username & password");
    authDb.child(username).once("value", snap => {
      if (snap.exists() && snap.val().password === password) {
        myId = snap.val().uid || username; // fallback to username if older account
        myName = username;
        document.getElementById("authPage").style.display = "none";
        document.getElementById("roomPage").style.display = "block";
      } else {
        alert("Invalid login credentials");
      }
    });
  }

  // === Join room & listeners ===
  function joinRoom() {
    roomName = document.getElementById("room").value.trim();
    secretKey = document.getElementById("key").value.trim();
    if (!roomName || !secretKey) return alert("Please enter room name and secret key");

    document.getElementById("roomPage").style.display = "none";
    document.getElementById("chatPage").style.display = "block";

    const roomRef = db.ref(roomName);

    roomRef.on("child_added", snapshot => renderMessage(snapshot));
    roomRef.on("child_changed", snapshot => updateMessageSeen(snapshot));
    roomRef.on("child_removed", snapshot => {
      const id = snapshot.key;
      const node = document.querySelector(`.message[data-id="${id}"]`);
      if (node && node.parentElement) node.parentElement.remove();
    });

    document.getElementById("fileInput").onchange = (e) => {
      const file = e.target.files[0];
      if (file) sendMedia(file);
      e.target.value = "";
    };
  }

  function renderMessage(snapshot) {
    const data = snapshot.val();
    const id = snapshot.key;

    const senderToken = data.senderId || data.sender; // support old messages
    const senderName = data.senderName || "Unknown";
    const isMine = senderToken === myId || senderToken === myName;

    const decrypted = decrypt(data.text, secretKey);

    const wrapper = document.createElement("div");
    wrapper.className = "wrapper";

    // name tag (show for others)
    if (!isMine) {
      const tag = document.createElement("div");
      tag.className = "name-tag";
      tag.textContent = senderName;
      wrapper.appendChild(tag);
    }

    const bubble = document.createElement("div");
    bubble.className = "message";
    bubble.setAttribute("data-id", id);

    // Reply preview
    if (data.replyTo) {
      const replyDiv = document.createElement("div");
      replyDiv.className = "reply-preview";
      replyDiv.textContent = decrypt(data.replyTo, secretKey);
      bubble.appendChild(replyDiv);
    }

    // Body: media or text
    if (typeof decrypted === "string" && decrypted.startsWith("file:")) {
      const fileUrl = decrypted.slice(5);
      const link = document.createElement("a");
      link.href = fileUrl;
      link.target = "_blank";
      link.download = fileUrl.split("/").pop();

      if (/\.(jpeg|jpg|png|gif|webp)$/i.test(fileUrl)) {
        const img = document.createElement("img");
        img.src = fileUrl;
        img.className = "chat-media";
        link.appendChild(img);
      } else if (/\.(mp4|webm|ogg)$/i.test(fileUrl)) {
        const video = document.createElement("video");
        video.src = fileUrl;
        video.controls = true;
        video.className = "chat-media";
        link.appendChild(video);
      } else {
        link.textContent = "Download file";
      }
      bubble.appendChild(link);
    } else {
      bubble.appendChild(document.createTextNode(decrypted));
    }

    if (isMine) {
      bubble.classList.add("sent");
      const seen = document.createElement("span");
      seen.className = "seen-status";
      seen.textContent = data.seen ? " ✓✓" : " ✓";
      seen.setAttribute("data-seen-for", id);
      bubble.appendChild(seen);
      bubble.onclick = () => {
        if (confirm("Delete this message?")) {
          db.ref(roomName).child(id).remove();
        }
      };
    } else {
      bubble.classList.add("received");
      if (!data.seen) db.ref(roomName).child(id).update({ seen: true });
      bubble.onclick = () => {
        replyToMsg = data.text; // keep encrypted
        document.getElementById("replyPreview").textContent = decrypted;
        document.getElementById("replyBox").style.display = "block";
        document.getElementById("message").focus();
      };
    }

    const time = document.createElement("div");
    time.className = "msg-time";
    const date = new Date(data.timestamp || Date.now());
    time.textContent = date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });

    wrapper.appendChild(bubble);
    wrapper.appendChild(time);

    const chat = document.getElementById("chat");
    chat.appendChild(wrapper);
    chat.scrollTop = chat.scrollHeight;
  }

  function updateMessageSeen(snapshot) {
    const data = snapshot.val();
    const id = snapshot.key;
    const seenNode = document.querySelector(`.seen-status[data-seen-for="${id}"]`);
    if (seenNode) seenNode.textContent = data.seen ? " ✓✓" : " ✓";
  }

  // === Send text ===
  function sendMessage() {
    const input = document.getElementById("message");
    const msg = input.value.trim();
    if (!msg) return;

    const encrypted = encrypt(msg, secretKey);
    const replyEncrypted = replyToMsg ? replyToMsg : null;

    db.ref(roomName).push({
      text: encrypted,
      replyTo: replyEncrypted,
      senderId: myId,         // NEW: unique user id
      senderName: myName,     // NEW: username
      sender: myId,           // legacy field for older clients
      timestamp: Date.now(),
      seen: false
    });

    replyToMsg = null;
    document.getElementById("replyBox").style.display = "none";
    input.value = "";
    input.focus();
  }

  // === Send media ===
  function sendMedia(file) {
    if (!file) return;
    const path = `${roomName}/${Date.now()}_${file.name}`;
    const meta = { contentType: file.type || undefined };
    const uploadTask = storage.ref(path).put(file, meta);
    uploadTask.on('state_changed',
      null,
      (err) => {
        console.error(err);
        alert("Upload failed: " + (err && err.message ? err.message : err));
      },
      () => {
        uploadTask.snapshot.ref.getDownloadURL().then(url => {
          const encryptedUrl = encrypt("file:" + url, secretKey);
          db.ref(roomName).push({
            text: encryptedUrl,
            senderId: myId,     // NEW
            senderName: myName, // NEW
            sender: myId,       // legacy
            timestamp: Date.now(),
            seen: false
          });
        });
      }
    );
  }

  function cancelReply() {
    replyToMsg = null;
    document.getElementById("replyBox").style.display = "none";
  }
</script>

</body>
</html>
