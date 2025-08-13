db.ref(roomName).on("child_added", function(snapshot) {
  let data = snapshot.val();
  let decrypted = decrypt(data.text, secretKey);

  // Always create a new message bubble (no grouping)
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

  // Apply sent/received styles
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

  // Append as a new message every time
  document.getElementById("chat").appendChild(div);
  document.getElementById("chat").scrollTop = document.getElementById("chat").scrollHeight;
});
