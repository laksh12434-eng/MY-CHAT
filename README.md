<MY CHAT>
<html>
<head>
  <meta charset="UTF-8">
  <title>Encrypted Chat App</title>
  <style>
    body { font-family: Arial, sans-serif; background: #f0f0f0; margin: 0; }
    .page { display: none; padding: 20px; }
    #chat { height: 400px; overflow-y: auto; padding: 10px; background: #fff; border: 1px solid #ccc; }
    .message { padding: 8px 12px; border-radius: 10px; margin-bottom: 5px; max-width: 60%; word-wrap: break-word; display: inline-block; }
    .sent { background: #dcf8c6; align-self: flex-end; }
    .received { background: #fff; border: 1px solid #ccc; }
    .msg-time { font-size: 10px; color: gray; margin-top: 2px; }
    .seen-status { font-size: 12px; color: blue; margin-left: 5px; }
    .chat-media { max-width: 200px; border-radius: 8px; display: block; margin-top: 5px; }
    .reply-preview { font-size: 12px; background: #eee; padding: 5px; border-left: 3px solid #999; margin-bottom: 5px; }
    #replyBox { display: none; background: #ddd; padding: 5px; margin-top: 5px; border-radius: 5px; }
  </style>
</head>
<body>

<div id="authPage" class="page" style="display:block;">
  <h2>Sign Up</h2>
  <input id="signupUser" placeholder="Username"><br>
  <input id="signupPass" placeholder="Password" type="password"><br>
  <button onclick="signUp()">Sign Up</button>

  <h2>Login</h2>
  <input id="loginUser" placeholder="Username"><br>
  <input id="loginPass" placeholder="Password" type="password"><br>
  <button onclick="login()">Login</button>
</div>

<div id="roomPage" class="page">
  <h2>Join Room</h2>
  <input id="room" placeholder="Room Name"><br>
  <input id="key" placeholder="Secret Key"><br>
  <button onclick="joinRoom()">Join</button>
</div>

<div id="chatPage" class="page">
  <div id="chat"></div>
  
  <div id="replyBox">
    Replying to: <span id="replyPreview"></span>
    <button onclick="cancelReply()">Cancel</button>
  </div>
  
  <input id="message" placeholder="Type message...">
  <input type="file" id="fileInput" onchange="sendMedia(this.files[0])">
  <button onclick="sendMessage()">Send</button>
</div>

<!-- Firebase Scripts -->
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-app.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-database.js"></script>
<script src="https://www.gstatic.com/firebasejs/8.10.0/firebase-storage.js"></script>

<script>
  // Your Firebase config
  const firebaseConfig = {
    apiKey: "YOUR_API_KEY",
    authDomain: "YOUR_DOMAIN",
    databaseURL: "YOUR_DB_URL",
    projectId: "YOUR_PROJECT_ID",
    storageBucket: "YOUR_BUCKET",
    messagingSenderId: "YOUR_SENDER_ID",
    appId: "YOUR_APP_ID"
  };
  firebase.initializeApp(firebaseConfig);
  const db = firebase.database();
  const storage = firebase.storage();

  // Firebase references
  const authDb = firebase.database().ref("users");
  let myId = null;
  let roomName = "";
  let secretKey = "";
  let replyToMsg = null;

  // Simple encryption/decryption (placeholder)
  function encrypt(text, key) { return btoa(text); }
  function decrypt(text, key) { return atob(text); }

  // SIGNUP
  function signUp() {
    let username = document.getElementById("signupUser").value.trim();
    let password = document.getElementById("signupPass").value.trim();
    if (!username || !password) return alert("Enter username & password");

    authDb.child(username).once("value", snap => {
      if (snap.exists()) alert("Username already taken!");
      else {
        authDb.child(username).set({ password: password });
        alert("Account created! You can now log in.");
      }
    });
  }

  // LOGIN
  function login() {
    let username = document.getElementById("loginUser").value.trim();
    let password = document.getElementById("loginPass").value.trim();
    if (!username || !password) return alert("Enter username & password");

    authDb.child(username).once("value", snap => {
      if (snap.exists() && snap.val().password === password) {
        myId = username;
        document.getElementById("authPage").style.display = "none";
        document.getElementById("roomPage").style.display = "block";
      } else alert("Invalid login credentials");
    });
  }

  // JOIN ROOM
  function joinRoom() {
    roomName = document.getElementById("room").value.trim();
    secretKey = document.getElementById("key").value.trim();
    if (!roomName || !secretKey) return alert("Please enter room name and secret key");

    document.getElementById("roomPage").style.display = "none";
    document.getElementById("chatPage").style.display = "block";

    db.ref(roomName).on("child_added", snapshot => {
      let data = snapshot.val();
      let decrypted = decrypt(data.text, secretKey);

      let msgWrapper = document.createElement("div");
      msgWrapper.style.marginBottom = "10px";

      let bubble = document.createElement("div");
      bubble.setAttribute("data-id", snapshot.key);

      // Reply preview
      if (data.replyTo) {
        let replyDiv = document.createElement("div");
        replyDiv.className = "reply-preview";
        replyDiv.textContent = decrypt(data.replyTo, secretKey);
        bubble.appendChild(replyDiv);
      }

      // Handle media
      if (decrypted.startsWith("file:")) {
        let fileUrl = decrypted.replace("file:", "");
        let link = document.createElement("a");
        link.href = fileUrl;
        link.target = "_blank";
        link.download = fileUrl.split('/').pop();

        if (/\.(jpeg|jpg|png|gif)$/i.test(fileUrl)) {
          let img = document.createElement("img");
          img.src = fileUrl;
          img.className = "chat-media";
          link.appendChild(img);
        } else if (/\.(mp4|webm|ogg)$/i.test(fileUrl)) {
          let video = document.createElement("video");
          video.src = fileUrl;
          video.controls = true;
          video.className = "chat-media";
          link.appendChild(video);
        }
        bubble.appendChild(link);
      } else {
        bubble.appendChild(document.createTextNode(decrypted));
      }

      // Sender styling & actions
      if (data.sender === myId) {
        bubble.className = "message sent";
        let seen = document.createElement("span");
        seen.className = "seen-status";
        seen.textContent = data.seen ? "✓✓" : "✓";
        bubble.appendChild(seen);
        bubble.onclick = () => {
          if (confirm("Delete this message?")) {
            db.ref(roomName).child(snapshot.key).remove();
            msgWrapper.remove();
          }
        };
      } else {
        bubble.className = "message received";
        db.ref(roomName).child(snapshot.key).update({ seen: true });
        bubble.onclick = () => {
          replyToMsg = data.text;
          document.getElementById("replyPreview").textContent = decrypted;
          document.getElementById("replyBox").style.display = "block";
        };
      }

      // Timestamp
      let time = document.createElement("div");
      time.className = "msg-time";
      let date = new Date(data.timestamp || Date.now());
      time.textContent = date.toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });

      msgWrapper.appendChild(bubble);
      msgWrapper.appendChild(time);

      document.getElementById("chat").appendChild(msgWrapper);
      document.getElementById("chat").scrollTop = document.getElementById("chat").scrollHeight;
    });
  }

  // SEND MESSAGE
  function sendMessage() {
    let msgInput = document.getElementById("message");
    let msg = msgInput.value.trim();
    if (msg === "") return;

    let encrypted = encrypt(msg, secretKey);
    let replyEncrypted = replyToMsg ? replyToMsg : null;

    db.ref(roomName).push({
      text: encrypted,
      replyTo: replyEncrypted,
      sender: myId,
      timestamp: Date.now(),
      seen: false
    });

    replyToMsg = null;
    document.getElementById("replyBox").style.display = "none";
    msgInput.value = "";
  }

  // SEND MEDIA
  function sendMedia(file) {
    if (!file) return;
    let storageRef = firebase.storage().ref(`${roomName}/${Date.now()}_${file.name}`);
    storageRef.put(file).then(snapshot => {
      snapshot.ref.getDownloadURL().then(url => {
        let encryptedUrl = encrypt("file:" + url, secretKey);
        db.ref(roomName).push({
          text: encryptedUrl,
          sender: myId,
          timestamp: Date.now(),
          seen: false
        });
      });
    });
  }

  // CANCEL REPLY
  function cancelReply() {
    replyToMsg = null;
    document.getElementById("replyBox").style.display = "none";
  }
</script>

</body>
</html>
