---
title: '跑步进入服务器新时代 —— 抛弃nginx'
tags: System
date: 2021-07-22 13:29:39
---

阿里云的服务器已经用了快两年了，期间一直用的nginx处理反代和TLS的问题。但最近遇到一件小事，实在让我想要换一种方式来管理这些服务。

起因是我想起一个webdav用来同步zotero的附件，nginx本身提供了一个webdav插件，问题是它默认没有启用，需要在编译时加选项打开。那么这就意味着我得重新自己编译一个nginx出来。可拉到吧，为了方便部署和维护，为什么不用docker整点什么呢？于是我发现了这个 https://github.com/dgraziotin/docker-nginx-webdav-nononsense。

把nginx打包进docker去提供webdav服务！虽然它非常可用，但这就非常离谱了。我用nginx反向代理nginx提供的webdav服务，这好么？这不好。另外，nginx的配置文件实在难写且烦，我大部分的应用都部署在docker里，我需要一个和docker配合地更好的反代工具。

一个可能的选择是我们直接向上到顶，直接上k8s和它的一系列生态工具，我不是很想这么做：
1. k8s太大了，我这单机1c2g的东西用不着也供不起这尊大佛
2. k8s需要提供自己的yaml，大部分的应用基本只服务到dockerfile层。少数提供docker compose，如果是k8s yaml的话还得自己转

我甚至不想上docker swarm，在我管不过来这台服务器上的服务前，这服务器就会吃不消了。在这时我看到了一个有趣的项目 https://github.com/lucaslorentz/caddy-docker-proxy。

好吧，这看上去就像是一个入门版的ingress或是traefik什么的。但它应该够用了。它还提供了一些swarm的模板，但我用docker compose就够了

```yaml
#caddy.yaml
version: "3.7"

# used for specific settings you have outside of your docker config
# ex: proxies to external servers, storage configuration...
# remove this block entirely if not needed (Only used for Docker Swarm)
#configs:
#  caddy-basic-content:
#    file: ./Caddyfile
#    labels:
#      caddy:

services:
  caddy:
    container_name: caddy
    image: lucaslorentz/caddy-docker-proxy:ci-alpine
    ports:
      - 80:80
      - 443:443
    environment:
      - CADDY_INGRESS_NETWORKS=caddy
      - CADDY_DOCKER_CADDYFILE_PATH=/Caddyfile
    networks:
      - caddy
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      # this volume is needed to keep the certificates
      # otherwise, new ones will be re-issued upon restart
      - caddy_data:/data
      - ./Caddyfile:/Caddyfile
    restart: unless-stopped

networks:
  caddy:
    external: true

volumes:
  caddy_data: {}
```

caddy会帮忙申请证书，可以把之前那个脚本关了。以及另外的，这个玩意会自己从label里面找反代的目标，例如我的pastebin挂在：
```yaml
# bw.yaml
version: '3'
services:
  paste:
    container_name: microbin
    image: microbin-docker
    restart: always
    # ports:
    #  - "127.0.0.1:10001:8080"
    volumes:
     - ./microbin-data:/app/pasta_data
    labels:
      caddy: paste.asternight.site
      caddy.reverse_proxy: "{{upstreams http 8080}}"
      caddy.import: basic_auth
    networks:
      - caddy

networks:
  caddy:
    external: true
```

甚至可以在里面加snippet，我把pastebin的auth挪到了反代层来，好吧，这才是合理的做法！另外，我可能还得单独处理一下我的博客，它是静态文件，不在docker里。在caddy里再挂一个路径
```yaml
- /home/git:/home/git
```
然后修改一下Caddyfile
```json
asternight.site {
    root * /home/git
    encode gzip
    file_server {
        hide .git
    }
    log {
        output file /data/log/hexo.log
    }
}
```

完成了，这一下在运维界进步了10年！