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







