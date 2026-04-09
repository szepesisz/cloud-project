---
title: Docker alapú CI/CD infrastruktúra kiépítése a LoRaWan Kiberbiztonsági kutatócsoport munkájának támogatására
author: Szepesi Szabolcs (DS62C1, C.S. MSc.)
theme:
    name: tokyonight-moon
---

BEVEZETÉS
---

# LoraWan kiberbiztonsági kutatócsoport
- Első féléves saját projektmunka
- Diplomamunka

<!-- pause -->
# Tesztlabor
- Open Telekom Cloud (OTC)
- 4GB RAM, 2 processzor
- Dockerben futó service-ek
  - LoraWan Gateway
  - ELK stack
  - Saját komponensek

<!-- pause -->
  - **Infrastruktúra**

<!-- pause -->

<!-- column_layout: [1, 1] -->
<!-- column: 0 -->
## A választott komponensek

- Gitea
- Drone
    - Drone Docker Runner
- Portainer

<!-- pause -->
<!-- column: 1 -->
## A manage-elt környezet
- oe-lorawan-sniffer


<!-- end_slide -->
BEVEZETÉS
---

# Korábbi állapot

- A forráskód felmásolása
- docker compose: `build .`


<!-- end_slide -->
A PROJEKT CÉLJA
---

![image:width:90%](cloud-project/img/seq.png)
<!-- end_slide -->

Implementáció
---

# deploy-infra
![image:width:90%](cloud-project/img/deploy-infra.png)

<!-- end_slide -->
Implementáció
---

# deploy-infra
```bash
ubuntu@chirpstack:~/docker/deploy-infra$ docker compose ps
NAME         IMAGE                        COMMAND
drone        drone/drone:2                "/bin/drone-server"
drone-runner drone/drone-runner-docker:1  "/bin/drone-runner-d…"
gitea        gitea/gitea:1.25.5           "/usr/bin/entrypoint…"
portainer    portainer/portainer-ce:2.39.0"/portainer -H unix:…"
```
<!-- end_slide -->

Implementáció
---

# deploy-infra
![image:width:90%](cloud-project/img/portainer_infra.png)

<!-- end_slide -->
Implementáció
---

Gitea
---
- GitHub helyett, mivel belső hálózaton kell használni

<!-- column_layout: [1, 1] -->
<!-- column: 0 -->

```conf
$ cat ~/.ssh/config
Host chirpstack
User ubuntu
Hostname 164.30.6.180
PreferredAuthentications publickey
IdentityFile ~/.ssh/lora_rsa
AddKeysToAgent yes
LocalForward 8100 127.0.0.1:8100
LocalForward 8101 127.0.0.1:8101
LocalForward 8122 127.0.0.1:8122
LocalForward 8102 127.0.0.1:8102
LocalForward 8103 127.0.0.1:8103
LocalForward 8104 127.0.0.1:8104
```

<!-- column: 1 -->

```conf
$ cat ~/.ssh/config
...
Host gitea
User git
Hostname localhost
Port 8122
PreferredAuthentications publickey
IdentityFile ~/.ssh/gitea
```

<!-- end_slide -->

Gitea
---

```yaml {2-4| 2,10-11 | 2,12-20 | all}
services:
  gitea:
    image: gitea/gitea:${GITEA_VERSION:-latest}
    container_name: gitea
    environment:
      USER_UID: 1000
      USER_GID: 1000
      GITEA__WEBHOOK__ALLOWED_HOST_LIST: "drone"
      GITEA__SERVER__ROOT_URL: "http://localhost:8101/"
    volumes:
      - $DATA_DIR/$COMPOSE_PROJECT_NAME/gitea/data:/data
    ports:
      - 8101:3000
      - 8122:22
    networks:
      - infranet

networks:
  infranet:
```

<!-- end_slide -->

Drone
---

```yaml
  drone:
    image: drone/drone:${DRONE_VERSION:-latest}
    container_name: drone
    environment:
      DRONE_GITEA_SERVER: http://gitea:3000
      DRONE_GITEA_CLIENT_ID: ${DRONE_GITEA_CLIENT_ID}
      DRONE_GITEA_CLIENT_SECRET: ${DRONE_GITEA_CLIENT_SECRET}
      DRONE_RPC_SECRET: ${DRONE_RPC_SECRET}
      DRONE_USER_CREATE: username:szepesisz,admin:true
      DRONE_SERVER_HOST: localhost:8102
      DRONE_SERVER_PROTO: http
    volumes:
      - $DATA_DIR/$COMPOSE_PROJECT_NAME/drone/data:/data
    ports:
      - "8102:80"
    depends_on:
      - gitea
    networks:
      - infranet
```

<!-- end_slide -->

Drone
---

Gitea intregráció

![image:width:90%](cloud-project/img/gitea_oauth.png)


<!-- end_slide -->


Drone Docker Runner
---

```yaml
  drone-runner:
    image: drone/drone-runner-docker:${DRONE_DOCKER_RUNNER_VERSION:-latest}
    container_name: drone-runner
    environment:
      DRONE_RPC_PROTO: http
      DRONE_RPC_HOST: drone:80
      DRONE_RPC_SECRET: ${DRONE_RPC_SECRET}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      - drone
    networks:
      - infranet
```

<!-- end_slide -->

Portainer
---

```yaml
  portainer:
    container_name: portainer
    image: portainer/portainer-ce:${PORTAINER_VERSION:-latest}
    command: -H unix:///var/run/docker.sock
    restart: always
    ports:
      - 8100:9000
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - $DATA_DIR/$COMPOSE_PROJECT_NAME/portainer/data:/data
    networks:
      - infranet

```
<!-- end_slide -->

Demo környezet
---

# deploy-demo

```bash
~/Documents/OE/lora/deploy-demo on  main [?] $ tree
.
├── compose.yml
├── config.yml
├── logging.toml
├── lorawan-sniffer
│   └── config.toml
└── state.yml
```

<!-- end_slide -->
oe-lorawan-sniffer
---

```yaml
services:
  lorawan-sniffer-test:
    image: oe-lorawan/oe-lorawan-sniffer:${LORAWAN_SNIFFER_VERISON:-latest}
    environment:
      - OE_LORAWAN_SNIFFER_CONFIG=/app/logging.toml,/app/config.toml
    volumes:
      - $PORTAINER_STACKS_DIR/$PORTAINER_STACK_ID/lorawan-sniffer/config.toml:/app/config.toml
      - $PORTAINER_STACKS_DIR/$PORTAINER_STACK_ID/logging.toml:/app/logging.toml
    ports:
      - 8103:8000
      - 8104:8001
```

<!-- end_slide -->


Dockerfile
---
# wheel

```bash
ARG PYTHON_VERSION=3.13

FROM python:${PYTHON_VERSION}-slim AS base
ENV PIP_ROOT_USER_ACTION=ignore
RUN apt-get update && \
    yes | apt-get install git && \
    pip install build
COPY . .
RUN python -m build .

FROM python:${PYTHON_VERSION}-slim AS wheel
ENV PIP_ROOT_USER_ACTION=ignore
COPY --from=base dist/*.whl .
```

<!-- end_slide -->

Dockerfile
---
# build

```bash
FROM wheel AS build
RUN pip install $(find ./*.whl)  && \
    rm $(find ./*.whl) && \
    mkdir /app
WORKDIR /app
ENV PYTHONBUFFERED=1
ENTRYPOINT ["app"]
```

<!-- end_slide -->

Dockerfile
---
# test

```bash
FROM build AS test
WORKDIR /
COPY --from=base . .
RUN python \
        -m pip install \
            -r requirements_test.txt  \
            pytest pytest-cov \
            bandit ruff \
    && mkdir \
        -p testresults \
    && python \
        -m pytest ... || true \
    && python \
        -m bandit  ... > testresults/bandit.junit.xml \
    && python \
        -m ruff check > testresults/ruff.junit.xml
```
<!-- end_slide -->

Drone pipeline
---

# Clone

```yaml
kind: pipeline
type: docker
name: default

clone:
  disable: true

steps:
- name: clone
  image: alpine/git
  commands:
  - git clone http://192.168.0.173:8101/oe-lorawan/oe-lorawan-sniffer.git .
  - git fetch --tags
  - git checkout $DRONE_COMMIT
```

<!-- end_slide -->

Drone pipeline
---

# Test

```yaml
- name: test
  image: docker:24.0
  privileged: true
  commands:
  - docker build --target test .

  volumes:
    - name: docker
      path: /var/run/docker.sock

...

volumes:
  - name: docker
    host:
      path: /var/run/docker.sock
```

<!-- end_slide -->

Drone pipeline
---

# Build

```yaml
- name: build
  image: docker:24.0
  privileged: true
  commands:
  - docker build --tag oe-lorawan/oe-lorawan-sniffer --target build .
  - docker tag oe-lorawan/oe-lorawan-sniffer oe-lorawan/oe-lorawan-sniffer:$(docker run oe-lorawan/oe-lorawan-sniffer --version)
  - docker tag oe-lorawan/oe-lorawan-sniffer oe-lorawan/oe-lorawan-sniffer:latest

  volumes:
    - name: docker
      path: /var/run/docker.sock
```

<!-- end_slide -->

Drone pipeline
---

# Deploy

```yaml
- name: deploy
  image: curlimages/curl:8.17.0
  secrets: [ portainer_api_key ]
  environment:
    PORTAINER_API_KEY:
      from_secret: portainer_api_key

  commands:
  - "curl -i -X PUT
     http://192.168.0.173:8100/api/stacks/2/git/redeploy?endpointId=1 -H
     \"X-API-KEY: $PORTAINER_API_KEY\" --json '{ \"ForceUpdate\": false }'"
```

<!-- end_slide -->
Drone pipeline
---



# Token
![image:width:60%](cloud-project/img/portainer_token.png)
![image:width:60%](cloud-project/img/portainer_token_drone.png)

<!-- end_slide -->


Sikeres pipeline futás
---

![image:width:100%](cloud-project/img/drone_pipeline.png)


<!-- end_slide -->

Demo
====
# Get current version

```bash +exec
curl \
    --silent \
    http://localhost:8103/openapi.json \
    | jq -r .info
```

<!-- end_slide -->

Demo
====
# Modify

```bash +exec
cd ~/Documents/OE/lora/LoRaWAN-sniffer
FILE=src/oe/lorawan/sniffer/process.py

sed -i \
  "s/title='.*'/title='LoraWan Sniffer Cloud Demo $(date)'/" \
  $FILE 

git add $FILE

git diff --cached $FILE | grep "^[\+\-] "
```

<!-- end_slide -->


Demo
====
# Push

```bash +exec
cd ~/Documents/OE/lora/LoRaWAN-sniffer
git commit -m "Cloud Demo"
git push gitea feature/cloud_demo
```

<!-- end_slide -->

Demo
====
# Get build status

```bash +exec
curl \
  --silent http://localhost:8102/api/repos/oe-lorawan/oe-lorawan-sniffer/builds \
| jq "[.[0].number, .[0].status]"
```

<!-- end_slide -->
