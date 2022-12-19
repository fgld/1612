### Локальная установка flask+gunicorn+nginx ### 

## 1. Установка необходимых пакетов ##

	sudo apt update
	sudo apt install python3-pip python3-dev nginx

## 2. Создание каталога для приложения и виртуальной среды ##

  # 2.1  Установка virtualenv #	
	sudo pip3 install virtualenv
  # 2.2 Создание каталога для приложения #
	mkdir flaskapp && cd $_
  # 2.3 Создание виртуальной среды с именем env #
	virtualenv env
  # 2.4 Активация виртуальной среды #
	source env/bin/activate
  # 2.5  Установка flask и gunicorn испольюзуя pip #
	pip3 install flask gunicorn

## 3. Создание простого проекта и точки входа wsgi ##

  # 3.1 Нужно создать простой проект #
	nano app.py
  	# Вставить следующее содержимое в файл app.py #
from flask import Flask
app = Flask(__name__)
@app.route('/')
def hello_world():
    return "Hello World!"
if __name__ == '__main__':
    app.run(debug=True,host='0.0.0.0')
  	# Сохраняем и выходим #

  # 3.2 Далее нужно создать файл, который будет служить точкой входа для нашего приложения. Этот файл будет сообщать серверу gunicorn как взаимодействовать с приложением #
	nano wsgi.py
  	# Вставить следующее содержимое в файл wsgi.py #
from app import app

if __name__ == "__main__":
    app.run()
  	# Сохраняем и выходим #

  # 3.3 Можно проверить способность gunicorn обслуживать проект, введя следующую команду #
	gunicorn --bind 0.0.0.0:5000 wsgi:app
  	# Ты должен получить вывод похожий на этот: #
  	# [2022-12-18 13:38:13 +0000] [1235] [INFO] Starting gunicorn 20.1.0 #
  	# [2022-12-18 13:38:13 +0000] [1235] [INFO] Listening at: http://0.0.0.0:5000 (1235) #
  	# [2022-12-18 13:38:13 +0000] [1235] [INFO] Using worker: sync #
  	# [2022-12-18 13:38:13 +0000] [1236] [INFO] Booting worker with pid: 1236 #

  	# Нажимаем CTRL+C, чтобы остановить сервер разработки flask #
 
  # 3.4 Деактивируем виртуальную среду командой: #
	deactivate

## 4. Создание systemd ##

  # 4.1 Создаем службу systemd следующей командой: #
	sudo nano /etc/systemd/system/flaskapp.service
	# Вставляем в него следующее содержимое: #
[Unit]
# [Unit] Определяет метаданные и зависимости #
Description=Gunicorn instance to serve myproject
# Сообщает системе инициализации запускать это после network.taget #
After=network.target
# Мы предоставим нашей учетной записи пользователя право собственности на процесс, поскольку он владеет всеми соответствующими файлами (Не забудь указать своего пользователя) #
[Service]
# [Service] определяет пользователя и группу под которыми нам процесс будет запускаться #
# Не забудь указать в поле User своего пользователя #
User=asakovich
# Указываем группу-владельца www-data, благодаря этому Nginx может просто взаимодействовать с gunicorn проццессом #
Group=www-data
# Указываем расположение рабочего каталога и переменную окружения, чтобы система инициализации знала, где находятся наши исполняемые файлы #
WorkingDirectory=/home/asakovich/flaskapp/
Environment="PATH=/home/asakovich/flaskapp/env/bin"
# Указываем команду для запуска службы # 
ExecStart=/home/asakovich/flaskapp/env/bin/gunicorn --workers 3 --bind unix:app.sock -m 007 wsgi:app
[Install]
# Эта строка сообщит systemd с чем связать службу, если мы разрешим ее запуск при загрузке #
WantedBy=multi-user.target

	# Сохраняем и выходим #
  # 4.2 Активируем сервис вводом команд : #
	sudo systemctl start flaskapp
	sudo systemctl enable flaskapp
	# файл app.sock будет создан автоматически #

## 5. Настройка nginx ##
  
  # 5.1 Создаем файл flaskapp внутри /etc/nginx/sites-available
	sudo nano /etc/nginx/sites-available/flaskapp
	# Вставляем в него следующее содержимое: #
server {
listen 80;
server_name 127.0.0.1;

location / {
  include proxy_params;
  proxy_pass http://unix:/home/asakovich/flaskapp/app.sock;
    }
}                
	# Сохраняем и выходим #	

  # 5.2 Проверяем все ли в порядке с конфигурационным файлом: #
	sudo nginx -t
	# Если все хорошо, то ты должен получить такой вывод: #
	# nginx: the configuration file /etc/nginx/nginx.conf syntax is ok #
	# nginx: configuration file /etc/nginx/nginx.conf test is successful #
 
  # 5.3 Активируем конфигурацию #
	sudo ln -s /etc/nginx/sites-available/flaskapp /etc/nginx/sites-enabled

  # 5.4 Перезапускаем nginx
	sudo systemctl restart nginx

  # 5.5 Теперь, перейдя в браузере по 127.0.0.1, ты должен увидеть надпись Hello World! 

###
