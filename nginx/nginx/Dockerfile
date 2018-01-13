FROM nginx:1.13.1-alpine

MAINTAINER takeshi.hirosue@bigtreetc.com

COPY template/nginx.conf /etc/nginx/nginx.conf
COPY template/default.conf /etc/nginx/conf.d/default.conf
COPY template/blacklist.conf /etc/nginx/conf.d/blacklist.conf

RUN mkdir -p /usr/share/nginx/html/limit \
        && echo 'limit 50r/s over request 503' > /usr/share/nginx/html/limit/index.html \
        && rm -rf /var/log/nginx/access.log

EXPOSE 80
STOPSIGNAL SIGTERM

CMD ["nginx", "-g", "daemon off;"]
