---
layout:     post
title:      Git指令整理
subtitle:   kubernetes环境下nginx配置热加载
date:       2019-03-25
author:     cugblack
header-img: img/post-bg-github-cup.jpg
catalog: true
tags:
    - Kubernetes
    - configMap
    - nginx
---
>kubernetes环境下nginx使用configMap挂载配置并实现热加载

# 前提

在kubernetes环境内，我们使用`deployment`部署nginx，为了方便的实时更新nginx配置，实现热加载，我们需要将nginx的配置挂在出来，结合`configMap`来进行实时修改配置，但是如何实时让容器内的nginx进行配置reload，即执行`nginx -s reload`命令。

## 容器的镜像

容器内我们需要使用工具来实时的检测 `/etc/nginx/conf.d`这个文件夹，因为我们之前已经通过configMap来将我们队配置文件的改变传进容器内，容器内部需要感知这个配置文件的改变并执行`nginx -s reload`来重载nginx配置。

工具选择[inotify-tools](https://github.com/rvoicilas/inotify-tools)这个工具来监控配置的变化并执行重载命令，基础镜像选择centos下的openresty，[Dockerfile](https://raw.githubusercontent.com/cugblack/dockerfile/master/nginx/new/Dockerfile)


```angular2html
FROM openresty/openresty:1.13.6.2-1-centos
ADD auto-reload.sh /root/auto-reload.sh
ADD start.sh /root/start.sh

ADD inotify-tools-3.14-8.el7.x86_64.rpm /root/inotify-tools-3.14-8.el7.x86_64.rpm 
WORKDIR /root/
RUN chmod a+x /root/auto-reload.sh \
    && rpm -ivh inotify-tools-3.14-8.el7.x86_64.rpm \
    && rm -rf inotify-tools-3.14-8.el7.x86_64.rpm
CMD ["/bin/bash", "/root/start.sh"]
``````

[热加载脚本](https://raw.githubusercontent.com/cugblack/dockerfile/master/nginx/new/auto-reload.sh)
```
#!/bin/sh
oldcksum=`cksum /etc/nginx/conf.d/default.conf`

inotifywait -e modify,move,create,delete -mr --timefmt '%d/%m/%y %H:%M' --format '%T' \
/etc/nginx/conf.d/ | while read date time; do

    newcksum=`cksum /etc/nginx/conf.d/default.conf`
    if [ "$newcksum" != "$oldcksum" ]; then
        echo "At ${time} on ${date}, config file update detected."
        oldcksum=$newcksum
        nginx -s reload
    fi

done
```

[启动脚本](https://raw.githubusercontent.com/cugblack/dockerfile/master/nginx/new/start.sh)
```angular2html
#!/bin/bash
/usr/bin/openresty -g "daemon off;" &
bash /root/auto-reload.sh
```
>>附nginx部署deployment的文件 [模板文件](https://raw.githubusercontent.com/cugblack/dockerfile/master/nginx/deployment.yaml)