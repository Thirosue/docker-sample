FROM nginx:1.13.1-alpine

MAINTAINER takeshi.hirosue@bigtreetc.com

COPY template/default.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
STOPSIGNAL SIGTERM

CMD ["nginx", "-g", "daemon off;"]
