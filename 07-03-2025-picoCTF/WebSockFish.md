# WebSockFish

Author: Venax

### Description

Can you win in a convincing manner against this chess bot? He won't go easy on you!You can find the challenge

Hint:
Try understanding the code and how the websocket client is interacting with the server

---

### **Understanding the Client-Server Interaction**

- **Client-Side Code**:
    
    The website uses stockfish to evaluate the game from the bot perspective (black pieces). Evaluations are sent to the server via WebSocket.
    
    ```
    stockfish.onmessage = function (event) {
            var message;
            // console.log(event.data);
            if (event.data.startsWith("bestmove")) {
              var bestMove = event.data.split(" ")[1];
              var srcSq = bestMove.slice(0, 2);
              var dstSq = bestMove.slice(2, 4);
              var promotion = bestMove.slice(4);
    
              game.move({ from: srcSq, to: dstSq, promotion: promotion });
              board.position(game.fen());
            } else if (event.data.startsWith(`info depth ${DEPTH}`)) {
              var splitString = event.data.split(" ");
              if (event.data.includes("mate")) {
                message = "mate " + parseInt(splitString[9]);
              } else {
                message = "eval " + parseInt(splitString[9]);
              }
              sendMessage(message);
            }
    ```
    

---

### **Initial Misunderstanding: `mate 0` as a Winning Condition**

- **My First Assumption**:
    
    Initially, I thought the goal was to win the game and send `sendMessage("mate 0")`, assuming this would signal to the server that I had won and should receive the flag.
    
    However, the server did not give me the flag based on `mate` values. 
    
- `mate 9999999` → "Haha I think you're gonna drown in 9999999 moves."
- `mate -9999999` / `mate 0` → "You may eventually checkmate me, but you will never break my spirit as a fish!!"
    
    ![image.png](image.png)
    
    ![image.png](image%201.png)
    

---

### **Identify the Exploitable Flaw**

- **Server Trusts Client Evaluations**:
    
    The server does not validate the actual game state. It relies solely on client reported evaluations to determine if the user wins.
    
- **Chat Response Patterns for `eval` Values**:
    - `eval -269` → "Don't think I'm gonna make it easy for you!"
    - `eval -353` → "I wouldn't have played that if I were you."
    - `eval 0` → "I think this position is pretty equal."
    - This suggests that the server uses `eval` messages to assess game status.
    

---

### **Exploiting the Flaw**

### **Directly Send `eval -9999`**

- Use browser dev tools to send `eval -9999` via WebSocket:
    
    ```
    sendMessage("eval -9999");
    ```
    
    - **Why?** Stockfish will think they are losing so badly since the evaluation (eval) is based on the likeliness of winning the game

---

![image.png](image%202.png)

![image.png](image%203.png)