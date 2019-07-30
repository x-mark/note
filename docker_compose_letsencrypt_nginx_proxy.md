## ubuntu安装最新版本docker

1. 安装docker-compose-letsencrypt-nginx-proxy-companion
```shell
mkdir nginx-proxy 
cd nginx-proxy
git clone https://github.com/evertramos/docker-compose-letsencrypt-nginx-proxy-companion.git .
```

version: '3'
services: 
   mail:
    image: bestwu/ewomailserver
    hostname: mail.zkkr50.com
    container_name: ewomail
    restart: always
    expose:
     - "80"
     - "433"
    ports:
      - "25:25"
      - "143:143"
      - "587:587"
      - "993:993"
      - "109:109"
      - "110:110"
      - "465:465"
      - "995:995"
      - "8080:8080"
    volumes:
      - ./mysql:/ewomail/mysql/data
      - ./vmail:/ewomail/mail
      - ./rainloop:/ewomail/www/rainloop/data
      - ./ssl/certs/:/etc/ssl/certs/
      - ./ssl/private/:/etc/ssl/private/
      - ./ssl/dkim/:/ewomail/dkim/
    environment:
      VIRTUAL_HOST: mail.zkkr50.com
      LETSENCRYPT_HOST: mail.zkkr50.com
      LETSENCRYPT_EMAIL: xmarkdeng@gmail.com
networks:
  default:
    external:
      name: webproxy