<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>Encrypted Chat</title>
  <style>
    body { font-family: Arial, sans-serif; background-color: #000; color: white; }
    #chat { max-width: 600px; margin: auto; padding: 10px; height: 400px; overflow-y: auto; border: 1px solid #333; }
    .message { padding: 5px; margin: 5px; border-radius: 5px; }
    .sent { background: #0b93f6; color: white; text-align: right; }
    .received { background: #262d31; color: white; text-align: left; }
    input, button { padding: 10px; margin: 5px; }
  </style>
</head>
<body>
  <h2>Enter Room Name and Secret Key</h2>
  <input type="text" id="room" placeholder="Room Name"><br>
  <input type="text" id="key" placeholder="Secret Key"><br>
  <button onclick="joinRoom()">Join Chat</button>

  <div id="chat" style="display:none;"></div>
  <input type="text" id="message" placeholder="Type a message" style="display:none;">
  <button id="sendBtn" onclick="sendMessage()" style="display:none;">Send</button>

  <!-- Firebase v12 SDK -->
  <script type="module">
    import { initializeApp } from "https://www.gstatic.com/firebasejs/12.1.0/firebase-app.js";
    import { getAnalytics } from "https://www.gstatic.com/firebasejs/12.1.0/firebase-analytics.js";
    import { getDatabase, ref, push, onChildAdded } from "https://www.gstatic.com/firebasejs/12.1.0/firebase-database.js";

    // Firebase config from your second code
    const firebaseConfig = {
      apiKey: "AIzaSyCyXRwlhZiOK3rVYa6lHGg0rHB5CS6h1W4",
      authDomain: "my-chat-3594d.firebaseapp.com",
      projectId: "my-chat-3594d",
      storageBucket: "my-chat-3594d.firebasestorage.app",
      messagingSenderId: "208703073410",
      appId: "1:208703073410:web:4ed522eb2cc522c00cd89b",
      measurementId: "G-CVC6XSQ5JW",
      databaseURL: "https://my-chat-3594d-default-rtdb.firebaseio.com" // Added for Realtime DB
    };

    const app = initializeApp(firebaseConfig);
    const analytics = getAnalytics(app);
    const db = getDatabase(app);

    let roomName = "";
    let secretKey = "";

    // XOR encryption
    function encrypt(text, key) {
      let result = '';
      for (let i = 0; i < text.length; i++) {
        result += String.fromCharCode(text.charCodeAt(i) ^ key.charCodeAt(i % key.length));
      }
      return btoa(result);
    }

    function decrypt(text, key) {
      let decoded = atob(text);
      let result = '';
      for (let i = 0; i < decoded.length; i++) {
        result += String.fromCharCode(decoded.charCodeAt(i) ^ key.charCodeAt(i % key.length));
      }
      return result;
    }

    // Join chat room
    window.joinRoom = function() {
      roomName = document.getElementById("room").value.trim();
      secretKey = document.getElementById("key").value.trim();

      if (!roomName || !secretKey) {
        alert("Please enter room name and secret key");
        return;
      }

      document.getElementById("chat").style.display = "block";
      document.getElementById("message").style.display = "inline-block";
      document.getElementById("sendBtn").style.display = "inline-block";

      const roomRef = ref(db, roomName);

      onChildAdded(roomRef, (snapshot) => {
        let data = snapshot.val();
        let decrypted = decrypt(data.text, secretKey);
        let div = document.createElement("div");
        div.className = "message received";
        div.innerText = decrypted;
        document.getElementById("chat").appendChild(div);
        document.getElementById("chat").scrollTop = document.getElementById("chat").scrollHeight;
      });
    }

    // Send encrypted message
    window.sendMessage = function() {
      let msg = document.getElementById("message").value.trim();
      if (msg === "") return;

      let encrypted = encrypt(msg, secretKey);
      push(ref(db, roomName), { text: encrypted });

      let div = document.createElement("div");
      div.className = "message sent";
      div.innerText = msg;
      document.getElementById("chat").appendChild(div);
      document.getElementById("chat").scrollTop = document.getElementById("chat").scrollHeight;

      document.getElementById("message").value = "";
    }
  </script>
</body>
</html>
