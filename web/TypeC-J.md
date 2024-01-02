# 🤥Type C-J🤥
![js](https://img.shields.io/badge/PHP-777BB4?style=for-the-badge&logo=php&logoColor=white)

## 개요
 - php Type Juggling에 관련한 취약점 문제입니다.
 - 1단계지만 풀고 이해하는데 애좀 먹은것같습니다.

## Preview
![image](https://github.com/ChCh0i/dreamhack/assets/108965611/3afbded2-65b1-4eb4-93a0-f46a348b4840)

 - 간단한 로그인폼이 있는것을 확인할수있습니다.

## Source
```
<html>
<head>
<link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.2/css/bootstrap.min.css">
<title>Type c-j</title>
</head>
<body>
    <!-- Fixed navbar -->
    <nav class="navbar navbar-default navbar-fixed-top">
      <div class="container">
        <div class="navbar-header">
          <a class="navbar-brand" href="/">Type c-j</a>
        </div>
        <div id="navbar">
          <ul class="nav navbar-nav">
            <li><a href="/">Index page</a></li>
          </ul>
        </div><!--/.nav-collapse -->
      </div>
    </nav><br/><br/><br/>
    <div class="container">
    <?php
    function getRandStr($length = 10) {
        $characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
        $charactersLength = strlen($characters);
        $randomString = '';
    
        for ($i = 0; $i < $length; $i++) {
            $randomString .= $characters[mt_rand(0, $charactersLength - 1)];
        }
        return $randomString;

    }
    require_once('flag.php');
    error_reporting(0);
    $id = getRandStr();
    $pw = sha1("1");
    // POST request
    if ($_SERVER["REQUEST_METHOD"] == "POST") {
      $input_id = $_POST["input1"] ? $_POST["input1"] : "";
      $input_pw = $_POST["input2"] ? $_POST["input2"] : "";
      sleep(1);

      if((int)$input_id == $id && strlen($input_id) === 10){
        echo '<h4>ID pass.</h4><br>';
        if((int)$input_pw == $pw && strlen($input_pw) === 8){
            echo "<pre>FLAG\n";
            echo $flag;
            echo "</pre>";
          }
        } else{
          echo '<h4>Try again.</h4><br>';
        }
      }else {
      echo '<h3>Fail...</h3>';
     }
    ?> 
    </div> 
</body>
</html>
```

## 취약점
 - 먼저 어떤식으로 접근을 해야 플래그를 얻을수 있는지 보겠습니다.
```
if((int)$input_id == $id && strlen($input_id) === 10){
        echo '<h4>ID pass.</h4><br>';
        if((int)$input_pw == $pw && strlen($input_pw) === 8){
            echo "<pre>FLAG\n";
            echo $flag;
            echo "</pre>";
          }
        } else{
          echo '<h4>Try again.</h4><br>';
        }
      }else {
      echo '<h3>Fail...</h3>';
     }
```
 - 위 코드를 보았을때 입력한id 값과 임의의$id라는 변수를 비교하고 문자열의 길이가 10글자일때 id가 pass되는것을 알수있습니다.
 - 두 번째도 마찬가지로 pw도 id와 똑같이 비교합니다.
 - 그럼 id값이 어떤식으로 비교되는지 알아야하기에 소스코드를 보면
```
function getRandStr($length = 10) {
        $characters = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';
        $charactersLength = strlen($characters);
        $randomString = '';
    
        for ($i = 0; $i < $length; $i++) {
            $randomString .= $characters[mt_rand(0, $charactersLength - 1)];
        }
        return $randomString;

    }
```
 - $id값을 생성하는 함수입니다.
 - 0~Z까지의 문자를 랜덤으로 저장합니다.
```
$pw = sha1("1");
```
 - $pw를 저장하는 부분입니다. 1의 값을 sha1로 해시값으로 변환시켜 저장합니다.
 - pw는 간단히 얻을수있지만 id를 구하는 과정에서 굉장히 어려워했던것 같습니다.

## payload
 - payload 를 살펴보겠습니다.
 - pw를 구하는 방법은 1의 값을 sha1로 해싱하여 8글자까지 가져오면 그게 input_pw로 들어갈 값이 됩니다.
```
356A192B
```
 - 다음으로는 id를 구하는 과정입니다.
 - 먼저 아까 id를 비교하는 연산자를 확인해보겠습니다.
```
if((int)$input_id == $id && strlen($input_id) === 10)
```
 - id를 비교하는 연산자를 확인해보면 $input_id가 int형으로 형변환이 되는것을 확인할수있고 그뒤 $id 와 느슨한 비교연산자 "==" 를 통하여 비교하는것을 확인할수 있습니다.
 - 느슨한 비교연산자를 통해 $input_id의 값이 int 형으로 형변환이 되며 $id의 값도 자동형변환이 일어나 int형으로 형변환이 일어나게 됩니다.
 - 그렇게되면 "a"... 와 같이 string형의 문자열을 입력하게되면 0으로 처리되며 $id의 값도 마찬가지로 0으로 처리되게 됩니다.
 - 그럼 input_id의 값을 0으로 채워주고 문자열길이비교 10글자 까지만 채워주게되면 id는 pass 되게 됩니다.
 - 입력값
```
id : 0000000000
pw : 356A192B
```

![image](https://github.com/ChCh0i/dreamhack/assets/108965611/e191f506-266f-460f-acfd-ad543a70d1ad)

 - 위 와 같이 flag를 획득한것을 볼수 있습니다.
