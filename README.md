# server-deploy

## mongo db docker ##

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

