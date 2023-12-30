# ❤Simple_Sqli❤
![js](https://img.shields.io/badge/MySQL-00000F?style=for-the-badge&logo=mysql&logoColor=white) ![js](https://img.shields.io/badge/Visual_Studio_Code-0078D4?style=for-the-badge&logo=visual%20studio%20code&logoColor=white)

## 개요
 - dreamhack web카테고리에서 가장 간단한 Sql injection 공격기법에 관한 문제를 풀이한 것입니다.

## Preview
![image](https://github.com/ChCh0i/dreamhack/assets/108965611/f083a7b5-5adf-417f-828f-abea02daae1b)

 - 간단한 로그인폼
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
    db.execute('create table users(userid char(100), userpassword char(100));')
    db.execute(f'insert into users(userid, userpassword) values ("guest", "guest"), ("admin", "{binascii.hexlify(os.urandom(16)).decode("utf8")}");')
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
        userid = request.form.get('userid')
        userpassword = request.form.get('userpassword')
        res = query_db(f'select * from users where userid="{userid}" and userpassword="{userpassword}"')
        if res:
            userid = res[0]
            if userid == 'admin':
                return f'hello {userid} flag is {FLAG}'
            return f'<script>alert("hello {userid}");history.go(-1);</script>'
        return '<script>alert("wrong");history.go(-1);</script>'

app.run(host='0.0.0.0', port=8000)
```
 - 소스코드를 보면 userid=admin 일 경우 flag를 읽어들이는것을 확인할수 있습니다.

## 취약점
 - 소스를 보았을때 database를 참조하는것이 아닌 실행될때마다 생성하는것을 확인할수있고 userid=guest&userpassword=guest 와 userid=admin&userpassword={binascii.hexlify(os.urandom(16)).decode("utf8")} 로 처리되는것을 확인할수 있습니다.
 - 취약한 점으로는 sql문에 대한 블랙리스트처리를 안함과 바인드함수 미사용이 있습니다.
 - 두 번째로는 소스코드에 admin이라는 userid 제공에 있습니다.

## payload
 - userid 폼에 더블쿼터를 이용하여 명령문 -- - 를 활용해 userid값을 받고 뒤의 내용들을 주석처리합니다
```
admin" -- -
```
![image](https://github.com/ChCh0i/dreamhack/assets/108965611/1d29d435-25e1-4410-84af-8960dc4e4afc)
