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
    #loginPage, #chatPage {
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
      width: 80%;
    }
    #sendBtn {
      width: 18%;
      background: #0b93f6;
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
  </style>
</head>
<body>

  <!-- Page 1: Login -->
  <div id="loginPage">
    <h2>Join Chat Room</h2>
    <input type="text" id="room" placeholder="Room Name"><br>
    <input type="text" id="key" placeholder="Secret Key"><br>
    <button id="joinBtn" onclick="joinRoom()">Join Chat</button>
  </div>

  <!-- Page 2: Chat -->
  <div id="chatPage" style="display:none;">
    <div id="chat"></div>
    <input type="text" id="message" placeholder="Type a message">
    <button id="sendBtn" onclick="sendMessage()">Send</button>
  </div>

  <!-- Firebase SDK v8 -->
  <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-app.js"></script>
  <script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-database.js"></script>

  <script>
    const firebaseConfig = {
      apiKey: "AIzaSyCyXRwlhZiOK3rVYa6lHGg0rHB5CS6h1W4",
      authDomain: "my-chat-3594d.firebaseapp.com",
      databaseURL: "https://my-chat-3594d-default-rtdb.firebaseio.com",
      projectId: "my-chat-3594d",
      storageBucket: "my-chat-3594d.firebasestorage.app",
      messagingSenderId: "208703073410",
      appId: "1:208703073410:web:4ed522eb2cc522c00cd89b",
      measurementId: "G-CVC6XSQ5JW"
    };
    firebase.initializeApp(firebaseConfig);

    let db = firebase.database();
    let roomName = "";
    let secretKey = "";

    // UTF-8 safe XOR encryption for emoji support
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
        div.className = "message received";
        div.textContent = decrypted; // textContent supports emojis
        document.getElementById("chat").appendChild(div);
        document.getElementById("chat").scrollTop = document.getElementById("chat").scrollHeight;
      });
    }

    function sendMessage() {
      let msgInput = document.getElementById("message");
      let msg = msgInput.value.trim();
      if (msg === "") return;

      let encrypted = encrypt(msg, secretKey);
      db.ref(roomName).push({ text: encrypted });

      let div = document.createElement("div");
      div.className = "message sent";
      div.textContent = msg; // Keep emoji intact
      document.getElementById("chat").appendChild(div);
      document.getElementById("chat").scrollTop = document.getElementById("chat").scrollHeight;

      msgInput.value = "";
      msgInput.focus(); // Keep keyboard open on mobile
    }
  </script>

</body>
</html>
