FROM node:13-alpine
 
RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.aliyun.com/g' /etc/apk/repositories && \
    apk add --no-cache git python libstdc++ && \
    npm install -g cnpm --registry=https://registry.npm.taobao.org && \
    cnpm install hexo-cli -g

EXPOSE 4000
