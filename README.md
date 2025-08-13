<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Encrypted Chat</title>
  <style>
    body {
      font-family: Arial, sans-serif;
      background-color: #000;
      color: white;
      margin: 0;
      padding: 0;
    }
    #loginPage, #chatPage {
      display: none;
      padding: 10px;
    }
    #chat {
      height: 80vh;
      overflow-y: auto;
      background-size: cover;
      padding: 10px;
    }
    .message {
      padding: 8px 12px;
      margin: 5px;
      border-radius: 10px;
      max-width: 60%;
      word-wrap: break-word;
      position: relative;
    }
    .sent {
      background: #4CAF50;
      color: white;
      margin-left: auto;
      cursor: pointer;
    }
    .received {
      background: #555;
      color: white;
      margin-right: auto;
      cursor: pointer;
    }
    #message {
      width: 70%;
      padding: 8px;
    }
    button {
      padding: 8px 12px;
      margin-left: 5px;
      cursor: pointer;
    }
    .chat-media {
      max-width: 200px;
      max-height: 200px;
      display: block;
      margin-top: 5px;
      border-radius: 10px;
    }
    .reply-preview {
      font-size: 12px;
      opacity: 0.7;
      border-left: 2px solid #ccc;
      padding-left: 5px;
      margin-bottom: 5px;
    }
    .status {
      font-size: 10px;
      margin-left: 5px;
      opacity: 0.7;
    }
  </style>
</head>
<body>
  <div id="loginPage">
    <h2>Join Chat</h2>
    <input type="text" id="room" placeholder="Room name">
    <input type="text" id="key" placeholder="Secret key">
    <button onclick="joinRoom()">Join</button>
  </div>

  <div id="chatPage">
    <div id="chat"></div>
    <input type="text" id="message" placeholder="Type your message">
    <button onclick="sendMessage()">Send</button>
    <button onclick="attachFile()">📎</button>
    <button onclick="changeBackground()">🖼️</button>
  </div>

  <!-- Firebase SDK -->
  <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-app.js"></script>
  <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-database.js"></script>
  <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-storage.js"></script>
  
  <script>
    // Firebase Config
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

    const db = firebase.database();
    const storage = firebase.storage();
    let roomName = "";
    let secretKey = "";
    const myId = localStorage.getItem("myId") || Date.now().toString();
    localStorage.setItem("myId", myId);

    let replyToMessageId = null;
    let replyToMessageText = "";

    // Encryption
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

      const savedBg = localStorage.getItem("chatBackground");
      if (savedBg) {
        document.getElementById("chat").style.backgroundImage = `url(${savedBg})`;
      }

      db.ref(roomName).on("child_added", function(snapshot) {
        const data = snapshot.val();
        const decrypted = decrypt(data.text, secretKey);

        const div = document.createElement("div");
        div.setAttribute("data-id", snapshot.key);

        // Reply preview
        if (data.replyTo && data.replyText) {
          const replyPreview = document.createElement("div");
          replyPreview.className = "reply-preview";
          replyPreview.textContent = "↪ " + data.replyText;
          div.appendChild(replyPreview);
        }

        // File or text
        if (decrypted.startsWith("file:")) {
          const fileUrl = decrypted.replace("file:", "");
          const link = document.createElement("a");
          link.href = fileUrl;
          link.target = "_blank";
          link.download = fileUrl.split('/').pop();
          if (fileUrl.match(/\.(jpeg|jpg|png|gif)$/i)) {
            const img = document.createElement("img");
            img.src = fileUrl;
            img.className = "chat-media";
            link.appendChild(img);
          } else if (fileUrl.match(/\.(mp4|webm|ogg)$/i)) {
            const video = document.createElement("video");
            video.src = fileUrl;
            video.controls = true;
            video.className = "chat-media";
            link.appendChild(video);
          }
          div.appendChild(link);
        } else {
          div.appendChild(document.createTextNode(decrypted));
        }

        // Seen status
        const statusSpan = document.createElement("span");
        statusSpan.className = "status";

        if (data.sender === myId) {
          div.className = "message sent";
          div.onclick = function() {
            if (confirm("Reply to or delete this message?")) {
              replyToMessageId = snapshot.key;
              replyToMessageText = decrypted;
              document.getElementById("message").placeholder = "Replying to: " + decrypted.slice(0, 30);
            }
          };
          statusSpan.textContent = data.seen ? "👁️" : "✅";
          div.appendChild(statusSpan);
        } else {
          div.className = "message received";
          div.onclick = function() {
            replyToMessageId = snapshot.key;
            replyToMessageText = decrypted;
            document.getElementById("message").placeholder = "Replying to: " + decrypted.slice(0, 30);
          };
          // Mark seen
          db.ref(roomName).child(snapshot.key).update({ seen: true });
        }

        document.getElementById("chat").appendChild(div);
        document.getElementById("chat").scrollTop = document.getElementById("chat").scrollHeight;
      });
    }

    function sendMessage() {
      const msgInput = document.getElementById("message");
      const msg = msgInput.value.trim();
      if (!msg) return;
      const encrypted = encrypt(msg, secretKey);

      const dataToSend = { text: encrypted, sender: myId, seen: false };
      if (replyToMessageId) {
        dataToSend.replyTo = replyToMessageId;
        dataToSend.replyText = replyToMessageText;
        replyToMessageId = null;
        replyToMessageText = "";
        msgInput.placeholder = "Type your message";
      }

      db.ref(roomName).push(dataToSend);
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
            const encryptedUrl = encrypt("file:" + url, secretKey);
            db.ref(roomName).push({ text: encryptedUrl, sender: myId, seen: false });
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

    document.getElementById("loginPage").style.display = "block";
  </script>
</body>
</html>
