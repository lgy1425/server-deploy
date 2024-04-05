# server-deploy

- 사용하시는 docker version 에 따라 docker-compose / docker compose 를 알맞게 사용하시면 됩니다.

## mongo db docker ##

mongo-compose.yml
```
version: "3.2"
services:
  mongodb:
    image: mongo
    restart: always
    environment:
      MONGO_INITDB_ROOT_USERNAME: sungmin_admin
      MONGO_INITDB_ROOT_PASSWORD: sungmin_admin
    volumes:
      - type: bind
        source: /data/db # local 경로
        target: /data/db  # container 내부에서의 경로
    container_name: "mongodb"
    ports:
      - "16010:27017"
```

위 파일을 작성하고 `docker compose -f mongo-compose.yml up --build -d` 를 실행하면 몽고디비가 실행이 됩니다.
source : /data/db 는 데이터 파일이 쌓이는 경로로 적절하게 바꿔주셔도 됩니다.

## MySQL database ##
현재는 성민의 `mysql://softgroup:cmcm0691@softgroup.co.kr/sm_form` 에 연결되어 있고 member table 만 있습니다.
새로 데이터베이스를 만들고 싶으시다면 스키마는 아래와 같습니다.
```
CREATE TABLE `member` (
  `created_at` datetime DEFAULT NULL,
  `updated_at` datetime DEFAULT NULL,
  `id` int(11) NOT NULL AUTO_INCREMENT
  `username` varchar(30) DEFAULT NULL,
  `password` varchar(255) DEFAULT NULL,
  `admin` int(11) DEFAULT NULL,
  `available` int(11) DEFAULT NULL,
  `hospital_code` varchar(100) DEFAULT NULL,
  `hospital_name` varchar(50) DEFAULT NULL,
  `password_raw` varchar(100) DEFAULT NULL,
  `url` varchar(255) DEFAULT NULL,
  `return_url` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB AUTO_INCREMENT=41 DEFAULT CHARSET=utf8;
```
## Code base download 
git clone https://LeeGY@bitbucket.org/LeeGY/sungmin_form.git 또는 제가 전달드린 zip 파일로 코드를 다운 받습니다.
폴더에는 3개의 폴더가 있습니다.
form-base : Python, Flask 패키지가 포함된 이미지 빌드를 위한 파일
form-api : Backend 서버
form-front : Front end

## Python, Flask Image build
form-base 폴더에서 `docker build . -t form_base` 를 실행합니다.
`docker images` 명령어를 실행 후
`form_base` 가 생성된 것을 확인한다.

## Backend docker compose
form-api 로 이동한다.

DB 경로를 설정한다.
config/db_config.py 에서
```
class ProductionDBConfig(LocalDBConfig):
    CSRF_ENABLED = True
    SQLALCHEMY_DATABASE_URI = os.getenv('DATABASE_URL',
                                        'mysql://softgroup:cmcm0691@softgroup.co.kr/sm_form')
    MONGODB_URL = os.getenv("MONGODB_URL","mongodb://sungmin_admin:soft0691?@118.67.130.214:16010/form")
```
해당 부분에서 mysql, mongo db 경로를 설정한다. 새로운 디비를 만들 것이 아니면 그대로 사용하셔도 됩니다.

이미지 저장폴더 생성합니다.
docker-compose.yml
```
version: "3.1"

services:
  api:
    build: .
    ports:
      - "20021:8000"
    command: >
      sh -c "gunicorn -b 0.0.0.0:8000 production:app"
    environment:
      - LC_ALL=C.UTF-8
    volumes:
      - /home/docker_home/images:/usr/src/images
```

/home/docker_home/images:/usr/src/images --> 이부분을 지정한 이미지폴더로 수정합니다.


`docker compose up --build -d` 를 실행합니다.

docker-compose.yml 파일을 보시면 기본적인 수준의 로드밸런싱을 위해 20021,20022,20023 포드 세개에 똑같은 서버를 띄워놓았습니다.

http://<서버 ip 또는 도메인>:<포트> 로 접속하면

<img width="372" alt="스크린샷 2024-04-05 오후 5 20 32" src="https://github.com/lgy1425/server-deploy/assets/13029243/263b0d7c-b1eb-4111-80a4-43afe5434ca9">

아래와 같이 index page 를 확인할 수 있습니다.



## Frontend docker compose
- form-front 폴더로 갑니다.
- URL endpoint 설정
config/config.js 에서
```
const prodConfig = {
  API_ENDPOINT: "http://form.talkcrm.co.kr:20020/v1",
  HOST: "http://form.talkcrm.co.kr/",
};
```
front 를 연결할 주소와 API 주소를 설정하면 됩니다.

`docker compose up --build -d` 를 실행합니다.

http://<서버 ip 또는 도메인>:3000 에 접속해서


<img width="686" alt="스크린샷 2024-04-05 오후 5 33 26" src="https://github.com/lgy1425/server-deploy/assets/13029243/75e8b387-b508-42ee-80d2-e319257f79e8">

해당 화면을 확인한다.

## (Optional) Nginx 설정
- 반드시 Nginx 로 할 필요는 없지만 제가 Nginx 로 설정 하여 배포하였기에 안내드립니다.

backend conf
```
upstream form_api {
    ip_hash;
    server 127.0.0.1:20021 weight=10 max_fails=3 fail_timeout=20s;
    server 127.0.0.1:20022 weight=10 max_fails=3 fail_timeout=20s;
    server 127.0.0.1:20023 weight=10 max_fails=3 fail_timeout=30s;
}

server {
        listen 20020;
        server_name _;

        gzip on;
        gzip_proxied any;
        gzip_comp_level 4;
        gzip_types text/css application/javascript image/svg+xml;

        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;


        location / {
                allow all;
                proxy_pass http://form_api;
                proxy_redirect off;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-FOrwarded-For $proxy_add_x_forwarded_for;
        }

}

```

front end / 이미지 conf

```
server {
  listen 7000 default_server;

  server_name _;

  server_tokens off;

  gzip on;
  gzip_proxied any;
  gzip_comp_level 4;
  gzip_types text/css application/javascript image/svg+xml;

  proxy_http_version 1.1;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection 'upgrade';
  proxy_set_header Host $host;
  proxy_cache_bypass $http_upgrade;

  # BUILT ASSETS (E.G. JS BUNDLES)
  # Browser cache - max cache headers from Next.js as build id in url
  # Server cache - valid forever (cleared after cache "inactive" period)
  location /_next/static {
    proxy_cache STATIC;
    proxy_pass http://form_front;
  }

  # STATIC ASSETS (E.G. IMAGES)
  # Browser cache - "no-cache" headers from Next.js as no build id in url
  # Server cache - refresh regularly in case of changes
  location /static {
    proxy_cache STATIC;
    proxy_ignore_headers Cache-Control;
    proxy_cache_valid 60m;
    proxy_pass http://form_front;
  }

  # DYNAMIC ASSETS - NO CACHE
  location / {
    proxy_pass http://form_front;
  }
}


server {
        listen 80;
        server_name form.softgroup.co.kr;

        server_tokens off;

        gzip on;
        gzip_proxied any;
        gzip_comp_level 4;
        gzip_types text/css application/javascript image/svg+xml;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;

        # BUILT ASSETS (E.G. JS BUNDLES)
        # Browser cache - max cache headers from Next.js as build id in url
        # Server cache - valid forever (cleared after cache "inactive" period)
        location /_next/static {
                proxy_cache STATIC;
                proxy_pass http://form_front;
        }

        # STATIC ASSETS (E.G. IMAGES)
        # Browser cache - "no-cache" headers from Next.js as no build id in url
        # Server cache - refresh regularly in case of changes
        location /static {
                proxy_cache STATIC;
                proxy_ignore_headers Cache-Control;
                proxy_cache_valid 60m;
                proxy_pass http://form_front;
        }

        location /images {
                alias /home/docker_home/images;
        }

        # DYNAMIC ASSETS - NO CACHE
        location / {
            proxy_pass http://form_front;
        }

}
```
이미지를
```
location /images {
                alias /home/docker_home/images;
        }
```

이렇게 연결했습니다.

## Dev 환경에서 띄우기
- Backend : 파이썬 가상환경에서 base requirements.txt 를 통해 패키지를 설치 `pip install -r requirements.txt`
- form-api 에서 `python production.py runserver` 를 실행하면 localhost:5000 에 서버가 실행된다.
- Front end : node js 를 설치
- form-front 로 이동 후 npm install 실행
- `npm run dev` 를 실행하면 localhost:3000 에 실행된다.




