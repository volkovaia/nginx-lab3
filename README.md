# nginx-lab3
# Часть 1
Создание SSL-сертификата:
```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-keyout ssl/nginx-selfsigned.key -out ssl/nginx-selfsigned.crt \
-subj "/C=RU/ST=Moscow/L=Moscow/O=IT/CN=localhost"
```
<img width="974" height="135" alt="image" src="https://github.com/user-attachments/assets/1255aca7-2ce8-4342-a2f2-66dfdc99af63" />
<br><br>
nginx.conf:

```bash
# Это обработка сетевых соединений, должна быть в любом случае, даже пустая
events {}

http {
    # Общие настройки SSL
    ssl_certificate /etc/nginx/ssl/nginx-selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx-selfsigned.key;

    server {
        # nginx слушает обычный незащищенный порт
        listen 80;
        # это домены, для которых будет работать правило автоматического редиректа
        server_name project1.local project2.local;
        # всё, что есть в http живёт по тому же адресу, но с https
        return 301 https://$host$request_uri;
    }

    # виртуальный хост для проекта 1
    server {
        # nginx слушает защищенный порт и ожидает шифрование
        listen 443 ssl;
        # если запрос приходит на этот домен, nginx открывает этот блок
        server_name project1.local;
        # указывает, где физически лежат файлы первого проекта и какой файл открывать первым
        location / {
            root /usr/share/nginx/html/project1;
            index index.html;
        }
    }

    # аналогично для второго домена
    server {
        listen 443 ssl;
        server_name project2.local;

        location / {
            root /usr/share/nginx/html/project2;
            index index.html;
        }

        # alias, при запросе project2.local/assets/ файл будет искаться в /project2/static/
        # (root добавляет путь из URL к пути на диске, а alias полностью подменяет его,
        # так что при запросе project2.local/assets/secret.txt nginx идёт не в assets, а в /project2/static/) 
        location /assets/ {
            alias /usr/share/nginx/html/project2/static/;
        }
    }
}

```
<br><br>
Для проверки записываем содержимое страниц, чтобы потом продемонстрировать открытие двух разных проектов на одноме сервере.
```bash
echo "<h1>Это 1 проект</h1>" > lab3/project1/index.html
```
```bash
echo "<h1>Это 2 проект</h1>" > lab3/project2/index.html 
```
Секретный файл для проверки alias:
```bash
echo "Secret file found via alias!" > lab3/project2/static/secret.txt
```
<br><br>
Запуск контейнера:

```bash
# docker run: создание и запуска нового контейнера;
# -d (detached): запуск контейнера в фоновом режиме
docker run -d --name nginx-lab \

  # -p (publish):проброс портов компьютера с портами внутри контейнера;
  # 80:80: проброс стандартных HTTP-портов
  # 443:443: проброс защищенного HTTPS-порта
  # то есть запрос попадает на порт 80, Docker перенаправляет его внутрь контейнера (тоже на порт 80), где его подхватывает Nginx
  -p 80:80 -p 443:443 \

  # -v (volume): команда для монтирования папок или файлов
  # /etc/nginx/nginx.conf: подменяем стандартный конфиг Nginx своим
  # :ro (read-only): контейнер может только читать этот файл
  -v "$(cygpath -m $(pwd)/nginx.conf)":/etc/nginx/nginx.conf:ro \

  # проброс папки с сертификатами
  -v "$(cygpath -m $(pwd)/ssl)":/etc/nginx/ssl:ro \

  # пробос папок с HTML-файлами
  -v "$(cygpath -m $(pwd)/project1)":/usr/share/nginx/html/project1:ro \
  -v "$(cygpath -m $(pwd)/project2)":/usr/share/nginx/html/project2:ro \
  nginx
```

<br><br>
<img width="974" height="220" alt="image" src="https://github.com/user-attachments/assets/f0d99ae0-f7c9-4787-8ee0-ec7bffab13bf" />
<br><br>
Проверка работы контейнера: контейнер работает
<img width="974" height="67" alt="image" src="https://github.com/user-attachments/assets/ff70f39a-402b-488a-a7c5-19c9d445effb" />
<br><br>
Потом нужно было настроить локальное сопоставление доменных имен (чтобы не искать сайт в интернете, а отправлять запрос на компьютер)
<br><br>
Через Блокнот от имени администратора открыть C:\Windows\System32\drivers\etc\hosts и в конец дописать: 
127.0.0.1 project1.local 
127.0.0.1 project2.local 
<img width="974" height="622" alt="image" src="https://github.com/user-attachments/assets/b54599b8-10a8-4cc3-ba7f-217d7125031c" />
<br><br>
Windows теперь знает эти домены:
<img width="880" height="253" alt="image" src="https://github.com/user-attachments/assets/d361ec0a-2338-4070-813f-697e3d82a68c" />
<br><br>
При вводе 
http://project1.local
Автоматически перебрасывает на https://
<img width="974" height="511" alt="image" src="https://github.com/user-attachments/assets/3b9b3aa1-6853-403e-bfa1-a6a3ce7cdc4b" />
<br><br>
При нажатии на Дополнительно перейти на project1.local:
<img width="974" height="518" alt="image" src="https://github.com/user-attachments/assets/a7f6ddea-7d8e-47a4-bc5b-8be0d8d1dffb" />
<br><br>
Демонстрация отображения уникальных надписей для каждого пет-проекта и проверка alias:
<img width="1919" height="306" alt="image" src="https://github.com/user-attachments/assets/5253372f-cba7-4738-9ba3-93fb2b16ca1e" />
<img width="1919" height="330" alt="image" src="https://github.com/user-attachments/assets/c1e4caf3-798c-491c-9266-88daa093c495" />
<img width="1919" height="258" alt="image" src="https://github.com/user-attachments/assets/76a680d5-7a15-4ec9-b24f-924c68c00fbb" />


<br><br>
# Часть 2
1. Information Disclosure

В графе Server ничего интересного не написано:
<img width="974" height="278" alt="image" src="https://github.com/user-attachments/assets/7c8bc570-20b7-48de-bd73-a2a3ca084913" />
<br><br>
Если открыть в браузере https://takoy-teatr.ru/robots.txt, то можно узнать пути на 2 html-ные страницы, но и в них тоже ничего инетерсного (на одну страницу войти можно, но это картинка верхней плашки сайта, по второму адресу зайти нельзя):
<img width="974" height="366" alt="image" src="https://github.com/user-attachments/assets/6da9e802-d690-4f49-9df7-3cb1f4cda491" />

<img width="974" height="249" alt="image" src="https://github.com/user-attachments/assets/92bfc3c7-74d7-417a-86ed-f83bc73bb11e" />

<img width="974" height="155" alt="image" src="https://github.com/user-attachments/assets/3a4a6837-9c55-440b-93c4-53e2bf16912d" />
<br><br>
2. Fuzzing
Перебор страниц через ffuf:

```bash
ffuf -u https://takoy-teatr.ru/FUZZ -w "C:\62kcmnpass\62kcmnpass.txt" -mc 200,301,403 -t 50
```
<br><br>
<img width="1252" height="371" alt="image" src="https://github.com/user-attachments/assets/631d9851-6efb-4c9e-8688-55f318f4150d" />
Errors (32085) - это сетевые ошибки, сервер просто сбрасывает соединение. Из текущих 57 тысяч попыток больше половины не получили ответа от сервера.
<br><br>
<img width="939" height="377" alt="image" src="https://github.com/user-attachments/assets/f9e46d4f-db45-43ca-b219-52f79271d5fc" />
Ответ 200 встречается, например, при takoy-teatr.ru/?? или takoy-teatr.ru/???, сервер не выдает ошибку 404, а показывает какую-то стандартную заглушку (то есть вообще должен был возвращаться код ошибки 404). 
При этом size: 430062, у всех ответов со статусом 200 размер файла одинаковый.
<br><br>
3. Path Traversal
При попытке обойти путь результат следующий: 
https://takoy-teatr.ru/images/etc/passwd 
<img width="974" height="420" alt="image" src="https://github.com/user-attachments/assets/4a3ed9ec-1afe-47e7-adff-87a1f8a5de93" />
Ошибка 456 говорит о том, что на сайте активна защита уровня приложения (WAF)
<br><br>
4. Также можно попробовать найти открытые папки системы контроля версий:
https://takoy-teatr.ru/.git/config
<img width="815" height="564" alt="image" src="https://github.com/user-attachments/assets/06c50fbe-8780-4cd0-a3b1-349fdb319e2e" />
Но после ffuf эта опция мне пока недоступна :)
