# Docker Autoheal 钉钉机器人魔改版
P.S.: 本项目为 [willfarrell/docker-autoheal](https://github.com/willfarrell/docker-autoheal) 的魔改版，主要修改了：
* 新增环境变量 `DING_WEBHOOK_URL`，用于适配钉钉机器人需要构造的`json`参数
* `Dockerfile` 添加 `阿里云`源，加速国内构建
* `docker-compose` 启动，引入`.env`

## 步骤
1. 在 `.env` 文件中填入你的钉钉机器人`webhook`地址。例如：`DING_WEBHOOK_URL=https://oapi.dingtalk.com/robot/send?access_token=xxxxxx`
2. 在钉钉机器人中设置 `自定义关键词` 为 `autoheal`。如图：
   ![设置自定义关键词](https://cdn.jsdelivr.net/gh/leopold7/CDN2@main/static/images/begs/2022/12/20221215113311.png)
3. 在需要监控的容器中添加一个 `label` 为 `autoheal=true`，且容器支持 `healthcheck`，例如：
```yaml
version: '3.9'
services:
    web-app:
        image: web-app:latest
        restart: unless-stopped
        ports:
            - "8080:8080"
        labels:
            - autoheal=true
        healthcheck:
           test: [ "CMD", "curl", "-fs", "http://localhost:8080/xxx/ping" ]
           interval: 30s
           timeout: 10s
           retries: 3
           start_period: 120s
```

## 效果
当已经添加了 `label` 为  `autoheal=true` 的容器，经过 `healthcheck` 检测为 `unhealthy` 时，会尝试执行重启操作，并执行钉钉webhook `DING_WEBHOOK_URL`。


声名：本项目基于 [willfarrell/docker-autoheal V1.2.0](https://github.com/willfarrell/docker-autoheal)，对于（除钉钉魔改外的）其他配置，请参考原说明文档：
---

# Docker Autoheal

Monitor and restart unhealthy docker containers. 
This functionality was proposed to be included with the addition of `HEALTHCHECK`, however didn't make the cut.
This container is a stand-in till there is native support for `--exit-on-unhealthy` https://github.com/docker/docker/pull/22719.

## Supported tags and Dockerfile links
- [`latest` (*Dockerfile*)](https://github.com/willfarrell/docker-autoheal/blob/main/Dockerfile) - Built daily
- [`1.1.0` (*Dockerfile*)](https://github.com/willfarrell/docker-autoheal/blob/1.1.0/Dockerfile)
- [`v0.7.0` (*Dockerfile*)](https://github.com/willfarrell/docker-autoheal/blob/v0.7.0/Dockerfile)

![](https://img.shields.io/docker/pulls/willfarrell/autoheal "Total docker pulls") [![](https://images.microbadger.com/badges/image/willfarrell/autoheal.svg)](http://microbadger.com/images/willfarrell/autoheal "Docker layer breakdown")

## How to use
### UNIX socket passthrough
```bash
docker run -d \
    --name autoheal \
    --restart=always \
    -e AUTOHEAL_CONTAINER_LABEL=all \
    -v /var/run/docker.sock:/var/run/docker.sock \
    willfarrell/autoheal
```
### TCP socket
```bash
docker run -d \
    --name autoheal \
    --restart=always \
    -e AUTOHEAL_CONTAINER_LABEL=all \
    -e DOCKER_SOCK=tcp://HOST:PORT \
    -v /path/to/certs/:/certs/:ro \
    willfarrell/autoheal
```
a) Apply the label `autoheal=true` to your container to have it watched.

b) Set ENV `AUTOHEAL_CONTAINER_LABEL=all` to watch all running containers. 

c) Set ENV `AUTOHEAL_CONTAINER_LABEL` to existing label name that has the value `true`.

Note: You must apply `HEALTHCHECK` to your docker images first. See https://docs.docker.com/engine/reference/builder/#healthcheck for details.
See https://docs.docker.com/engine/security/https/ for how to configure TCP with mTLS

The certificates, and keys need these names:
* ca.pem
* client-cert.pem
* client-key.pem

### Change Timezone
If you need the timezone to match the local machine, you can map the `/etc/localtime` into the container.
```
docker run ... -v /etc/localtime:/etc/localtime:ro
```


## ENV Defaults
```
AUTOHEAL_CONTAINER_LABEL=autoheal
AUTOHEAL_INTERVAL=5   # check every 5 seconds
AUTOHEAL_START_PERIOD=0   # wait 0 seconds before first health check
AUTOHEAL_DEFAULT_STOP_TIMEOUT=10   # Docker waits max 10 seconds (the Docker default) for a container to stop before killing during restarts (container overridable via label, see below)
DOCKER_SOCK=/var/run/docker.sock   # Unix socket for curl requests to Docker API
CURL_TIMEOUT=30     # --max-time seconds for curl requests to Docker API
WEBHOOK_URL=""    # post message to the webhook if a container was restarted (or restart failed)
```

### Optional Container Labels
```
autoheal.stop.timeout=20        # Per containers override for stop timeout seconds during restart
```

## Testing
```bash
docker build -t autoheal .

docker run -d \
    -e AUTOHEAL_CONTAINER_LABEL=all \
    -v /var/run/docker.sock:/var/run/docker.sock \
    autoheal                                                                        
```
