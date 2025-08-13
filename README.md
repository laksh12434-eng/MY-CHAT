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
    margin-bottom: 10px;
    background-size: cover;
    background-position: center;
  }
  .message {
    padding: 8px 12px;
    margin: 5px;
    border-radius: 10px;
    max-width: 70%;
  }
  .mine {
    background-color: #4CAF50;
    margin-left: auto;
    text-align: right;
  }
  .theirs {
    background-color: #555;
    text-align: left;
  }
  input, button {
    padding: 8px;
    margin: 5px;
    border: none;
    border-radius: 5px;
  }
  input {
    width: calc(100% - 16px);
  }
</style>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-database.js"></script>
</head>
<body>

<div id="loginPage">
  <h2>Join Room</h2>
  <input id="room" placeholder="Room Name"><br>
  <input id="key" placeholder="Secret Key"><br>
  <button onclick="joinRoom()">Join</button>
</div>

<div id="chatPage" style="display:none;">
  <h2>Chat Room</h2>
  <div id="chat"></div>
  <input id="msg" placeholder="Type a message" onkeydown="if(event.key==='Enter') sendMessage()">
  <button onclick="sendMessage()">Send</button>
</div>

<script>
  // Firebase config
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

  let myId = localStorage.getItem("myId");
  if (!myId) {
    myId = "u_" + Date.now().toString(36) + Math.random().toString(36).slice(2,8);
    localStorage.setItem("myId", myId);
  }

  let roomRef = null;

  // Option 1 applied — only load new messages
  function joinRoom() {
    const roomName = document.getElementById("room").value.trim();
    const secretKey = document.getElementById("key").value.trim();
    if (!roomName || !secretKey) {
      alert("Please enter room name and secret key");
      return;
    }

    document.getElementById("loginPage").style.display = "none";
    document.getElementById("chatPage").style.display = "block";

    document.getElementById("chat").innerHTML = "";

    if (roomRef) roomRef.off();

    const startTime = Date.now();
    roomRef = db.ref(roomName);
    roomRef.orderByChild("ts").startAt(startTime).on("child_added", snapshot => {
      renderMessage(snapshot);
    });
  }

  function sendMessage() {
    const text = document.getElementById("msg").value.trim();
    if (!text) return;
    const ts = Date.now();
    roomRef.push({
      sender: myId,
      text: text,
      ts: ts
    });
    document.getElementById("msg").value = "";
  }

  function renderMessage(snapshot) {
    const msg = snapshot.val();
    const div = document.createElement("div");
    div.classList.add("message");
    div.classList.add(msg.sender === myId ? "mine" : "theirs");
    div.textContent = msg.text;
    document.getElementById("chat").appendChild(div);
    document.getElementById("chat").scrollTop = document.getElementById("chat").scrollHeight;
  }
</script>

</body>
</html>
