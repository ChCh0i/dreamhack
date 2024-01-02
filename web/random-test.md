# 🎃Random-test🎃
![js](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white) 

## 개요
 - dreamhack brute force 문제입니다.

## Source
```
#!/usr/bin/python3
from flask import Flask, request, render_template
import string
import random

app = Flask(__name__)

try:
    FLAG = open("./flag.txt", "r").read()       # flag is here!
except:
    FLAG = "[**FLAG**]"


rand_str = ""
alphanumeric = string.ascii_lowercase + string.digits
for i in range(4):
    rand_str += str(random.choice(alphanumeric))

rand_num = random.randint(100, 200)


@app.route("/", methods = ["GET", "POST"])
def index():
    if request.method == "GET":
        return render_template("index.html")
    else:
        locker_num = request.form.get("locker_num", "")
        password = request.form.get("password", "")

        if locker_num != "" and rand_str[0:len(locker_num)] == locker_num:
            if locker_num == rand_str and password == str(rand_num):
                return render_template("index.html", result = "FLAG:" + FLAG)
            return render_template("index.html", result = "Good")
        else: 
            return render_template("index.html", result = "Wrong!")
            
            
app.run(host="0.0.0.0", port=8000)
```

## 취약점
 - 소스코드를 보았을때 자물쇠의 번호와 비밀번호를 맞추어야 넘어가는 문제입니다.
 - 자물쇠의 번호는 string.ascii_lowercase + string.digits 에서 4글자를 받는것을 확인할수 있습니다.
```
alphanumeric = string.ascii_lowercase + string.digits
for i in range(4):
    rand_str += str(random.choice(alphanumeric))
```

 - 비밀번호는 숫자 100~200 사이의 임의의 숫자를 받는것을 확인할수 있습니다.
```
rand_num = random.randint(100, 200)
```

## payload
 - payload는 burp suite를 통하여 brute force를 진행합니다
![image](https://github.com/ChCh0i/dreamhack/assets/108965611/957a0938-cb68-4b56-9cf1-b21a08934f85)

![image](https://github.com/ChCh0i/dreamhack/assets/108965611/4dc0ed68-b8fc-4bc7-9361-1b9e43a982cd)
