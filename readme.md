# 장고 EC2 배포하기

### WSGI

WSGI는 Web Server Gateway Interface의 줄임말로, 웹 서버 소프트웨어와 파이썬으로 작성된 웹 응용 프로그램 간의 표준 인터페이스이다. 장고는 웹 서버와 직접적으로 통신하는 것이 불가능해 파이썬 패키지인 uWSGI를 설치해 둘을 중간에서 이어줘야 한다.



MobaXterm을 이용해 ubuntu에 올라온 서버를 우선 실행시켜보고 정상적으로 실행되는지 확인!



Ubuntu 인스턴스의 Manage로 들어가서 Networking 탭을 누른후 port를 추가해준다.



### uWSGI 서버 연결하기

#### 1. 패키지 설치하기

```shell
# 가상환경 실행
$ source venv/bin/activate

# uswgi 설치
$ pip install uwsgi
```



#### 2. uWSGI를 이용해 장고 프로젝트 연결하기

```shell
$ uwsgi --http :[포트번호] --home [venv경로] --chdir [장고프로젝트경로] -w [wsgi모듈폴더].wsgi
```

위의 명령어를 실행시켜 프로젝트가 잘 실행되는지 확인한다.



#### 3. ini 파일 만들어서 uWSGI 실행시키기

매번 저 명령어를 사용해 서버열기는 매우 번거롭기 때문에 파일들로 옵션을 저장하여 wsgi가 알아서 서버를 열 수 있도록 만든다.

우선 manage.py 가 있는 디렉토리로 들어온 후에 .config 디렉토리를 만들고 그 안에 uwsgi 디렉토리를 만들고 그안에 [프로젝트명].ini 파일을 만든다

```shell
$ mkdir .config/uwsgi

# uwsgi 디렉토리로 들어와서
$ touch [프로젝트명].ini

# 그 후에 vi를 이용하여 파일 옵션을 설정!!
$ vi [프로젝트명].ini
```

- vi 명령어 => i : 수정, esc: 종료, :wq : 저장후 나가기 ... 그 외에도 다양한 명령어가 있지만 여기선 이 세 명령어로도 충분!

```
[uwsgi]
chdir = /srv/django_project/
module = project_name.wsgi:application
home = /home/ubuntu/myvenv/

uid = deploy
gid = deploy

http = :8080

enable-threads = true
master = true
vacuum = true
pidfile = /tmp/project_name.pid
logto = /var/log/uwsgi/project_name/@(exec://date +%%Y-%%m-%%d).log
log-reopen = true
```

위 와 같이 옵션을 저장해준다

로그를 작성한 경로가 아직 존재하지 않으니 만들어 주고, 배포용 계정이 그 폴더의 소유주여야 로그를 작성할 수 있으니 소유주를 바꿔준다.

```shell
$ sudo mkdir -p /var/log/uwsgi/project_name/
$ sudo chown -R deploy:deploy /var/log/uwsgi/project_name/
```



[프로젝트이름].ini의 옵션을 이용해 uWSGI 서버를 다시킨다.

```shell
$ sudo /home/ubuntu/myvenv/bin/uwsgi -i /srv/django_project/.config/uwsgi/project_name.ini
```











### nginx

nginx는 Dajngo나 Flask와 같은 파이썬 웹 프레임워크에 주로 쓰이며 웹 서버와 장고가 연결될 수 있도록 해준다.

nginx를 설치하기 전까지 Django와 사용자는

```
사용자 <-> uWSGI <-> Django
```

위와 같이 통신을 한다. 일반적으로 사용자의 브라우저는 uWSGI에 직접 요청을 보내지 않고 nginx나 apache와 같은 웹서버를 통해 보낸다. 따라서 nginx를 설치하면

```
사용자 <-> nginx 서버 <-> uWSGI <-> Django
```

와 같이 통신을 하게 된다.





### nginx 와 uSWGI 연결하기

#### nginx 설치

```shell
$ sudo apt-get install nginx
```



#### nginx 설정

1. 유저 설정
   - __nginx.conf__ 파일에서 유저 설정을 진행함

``` shell
$ sudo vim /etc/nginx/nginx.conf
```

를 통해 conf파일을 연 후

```
...
user deploy; <<< 이 부분 수정
worker_processes auto;
pid /run/nginx.pid;
include /etc/nginx/modules-enabled/*.conf;

events {
        worker_connections 768;
        # multi_accept on;
}
....
```



2. conf 파일 만들어 주기
   - .config 디렉토리에 nginx 디렉토리를 새로 만들어 준 후 __[프로젝트명].config__ 파일을 만들어 준다

```shell
$ cd /srv/django_project/.config
$ mkdir nginx
$ vim project_name.conf
```

3. project_name.config 수정

```
server {
    listen 80;
    server_name *.compute.amazonaws.com;
    charset utf-8;
    client_max_body_size 128M;

    location / {
        uwsgi_pass  unix:///tmp/mysite.sock;
        include     uwsgi_params;
    }
}
```

여기서 listen 포트를 80으로 설정해두었으니 이전에 __project_name.ini__ 파일에서 포트를 8080이라고 설정한 __http = :8080__을 수정해줄 차례다.

4. project_name.ini 수정

```shell
[uwsgi]
chdir = /srv/django_project/
module = project_name.wsgi:application
home = /home/ubuntu/myvenv/

uid = deploy
gid = deploy

socket = /tmp/project_name.sock  <<< 여기서부터
chmod-socket = 666
chown-socket = deploy:deploy     <<< 여기까지가 수정됨

enable-threads = true
master = true
vacuum = true
pidfile = /tmp/project_name.pid
logto = /var/log/uwsgi/project_name/@(exec://date +%%Y-%%m-%%d).log
log-reopen = true
```

원래 쓰여있던 __http = :8000__ 대신에 소켓 정보, 소유자, 권한을 서술한 새로운 3줄을 입력

5. uwsgi.service 파일 작성

   nginx와 uwsgi가 연결되려면 uwsgi를 수동으로 켜서는 안 된다. 서버가 계속 켜져 있으려면 uwsgi가 계속 백그라운드에 실행되어 있을 수 있게 설정 파일을 추가로 만들어 주어야한다.

   uwsgi 디렉토리 안에 __uswgi.service__파일을 만들어서

```
[Unit]
Description=uWSGI service
After=syslog.target

[Service]
ExecStart=/home/ubuntu/myvenv/bin/uwsgi -i /srv/django_project/.config/uwsgi/project_name.ini

Restart=always
KillSignal=SIGQUIT
Type=notify
StandardError=syslog
NotifyAccess=all

[Install]
WantedBy=multi-user.target
```

​	해당 내용을 작성해준다.

​	[service] 항목 아래에 있는 ExecStart 값은 원래 수동으로 입력해서 uwsgi를 실행시켰던

```shell
$ sudo /home/ubuntu/myvenv/bin/uwsgi -i /srv/django_project/.config/uwsgi/project_name.ini
```

이 부분을 옮겨둔 것!!

이 명령어를 service로 등록하여 계속 백그라운드에서 실행되도록 하는 것이다.

그 이후 __uwsgi.service__ 파일을 daemon에 등록해준다

```shell
 sudo ln -f /srv/django_project/.config/uwsgi/uwsgi.service /etc/systemd/system/uwsgi.service
```



#### daemon이란?

리눅스 프로그램이 처음 가동될 때 실행되는 백그라운드 프로세스의 일종으로 메모리에 상주하면서 사용자의 요청을 기다리면서 대기하고 있다가 요청이 발생하면 해당 요청에 맞게 대응을 한다. 그리고 일반적으로 daemon 프로그램의 명령어는 'd'로 끝난다.

reference : http://valuefactory.tistory.com/229



그 이후 daemon을 reload 하고 , uwsgi 서비스를 enable 해준 뒤 restart 해준다

```shell
$ sudo systemctl daemon-reload
$ sudo systemctl enable uwsgi
$ sudo systemctl restart uwsgi
```



6. nginx 설정 파일을 nginx 어플리케이션에 등록하기

​	nginx 설정 파일을 등록하려면 cp 명령어를 이용해 sites-available 디렉토리로 __[프로젝트명].conf__를 복사한다

```shell
$ sudo cp -f /srv/django_project/.config/nginx/project_name.conf /etc/nginx/sites-available/django_project.conf
```

그리고 복사한 이 파일을 __sites-enabled__ 폴더 안에 링크해주고, __sites-enabled__에 원래 있었던 default 파일은 삭제!!

```shell
$ sudo ln -sf /etc/nginx/sites-available/mysite.conf /etc/nginx/sites-enabled/mysite.conf

$ sudo rm /etc/nginx/sites-enbaled/default
```

이제 daemon을 다시 reload 해주고 uwsgi와 nginx를 restart 해준다.



```shell
$ sudo systemctl daemon-reload
$ sudo systemctl restart uwsgi nginx
```



수정 사항이 생길 때마다 위의 명령어를 실행해야 반영됨!!



### Reference

- https://velog.io/@jieuihong/aws-django-2

- https://velog.io/@jieuihong/aws-django-3