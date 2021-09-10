FROM debian:buster-slim

RUN  apt-get update && apt-get -y upgrade && apt-get -y install wget nginx-common libgeoip-dev \
     && cd && wget https://github.com/techwise-technologies/tsuki-dev/releases/download/v1.20.1/nginx-1.20.1_linux-gnu-amd64.deb \
     && echo "$TECHWISE_NGINX_SHA256 nginx-1.20.1_linux-gnu-amd64.deb" | sha256sum -c - && \
     && dpkg -i nginx-1.20.1_linux-gnu-amd64.deb \
     && apt-get --fix-broken -y install \
     && rm -rf nginx-1.20.1_linux-gnu-amd64.deb

RUN  mkdir /usr/local/nginx \
     && touch /usr/local/nginx/uwsgi_temp

COPY ./nginx/nginx.conf /etc/nginx/nginx.conf

EXPOSE 80

STOPSIGNAL SIGQUIT

CMD ["nginx", "-g", "daemon off;"]
