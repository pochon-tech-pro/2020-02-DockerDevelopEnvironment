## 2020-02-DockerDevelopEnvironment

### 概要

- 2020/02時点でのDocker開発環境群 (Docker for Mac)

### 2020-NodeJS

- NodeJS開発環境用

- Dockerfile

```Dockerfile
FROM node:10-alpine
  
ENV APP_PORT 8001
ENV APP_ROOT /app
EXPOSE $APP_PORT
WORKDIR $APP_ROOT
CMD [ "sh" ]

RUN apk update && \
    apk add git openssh curl jq && \
    curl -o- -L https://yarnpkg.com/install.sh | sh

RUN apk add python make g++
```

- docker-compose.yml

```yaml:docker-compose.yml
version: '3.4'
x-logging:
  &default-logging
  driver: "json-file"
  options:
    max-size: "100k"
    max-file: "3"
services:

  app:
    build: ./
    logging: *default-logging
    volumes:
    - ./app/:/app/
    command: sh -c "yarn start"
    ports: 
      - "8001:8001"
```

