# ❤amocafe❤
![js](https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white)

## 개요
 - dreamhack amocafe문제 입니다.

## Preview
![image](https://github.com/ChCh0i/dreamhack/assets/108965611/e6359260-4e43-4308-b13d-bf70a34954db)

## Source
```
#!/usr/bin/env python3
from flask import Flask, request, render_template

app = Flask(__name__)

try:
    FLAG = open("./flag.txt", "r").read()       # flag is here!
except:
    FLAG = "[**FLAG**]"

@app.route('/', methods=['GET', 'POST'])
def index():
    menu_str = ''
    org = FLAG[10:29]
    org = int(org)
    st = ['' for i in range(16)]

    for i in range (0, 16):
        res = (org >> (4 * i)) & 0xf
        if 0 < res < 12:
            if ~res & 0xf == 0x4:
                st[16-i-1] = '_'
            else:
                st[16-i-1] = str(res)
        else:
            st[16-i-1] = format(res, 'x')
    menu_str = menu_str.join(st)

    # POST
    if request.method == "POST":
        input_str =  request.form.get("menu_input", "")
        if input_str == str(org):
            return render_template('index.html', menu=menu_str, flag=FLAG)
        return render_template('index.html', menu=menu_str, flag='try again...')
    # GET
    return render_template('index.html', menu=menu_str)

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8000)
```

## 취약점
 - 해당 페이지접속시 표시되는 text를 dcripy 하는 문제입니다.
 - 텍스트를 변환하는 코드를 살펴보면 비트연산자를통하여 4*i만큼 나누고 0xf와 and 연산을하여 넘어간다음
 - ~res & 0xf == 0x4 즉 res의값이 0xb 일경우 "_" 로 치환
 - 등 이 있겠습니다.

## payload
```
text = "1_c_3_c_0__ff_3e"
alpha = ["b","c","d","e","f"]
total = 0
count = 15

for i in text:
    if(i=='_'):
        res = 0xb
    elif(i in alpha):
        res = int(i,base=16)
    else:
        res = int(i)
    total = (res << (count*4)) | total
    count -= 1
    
print(total)
```
