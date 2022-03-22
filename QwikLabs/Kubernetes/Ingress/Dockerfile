FROM nginx:latest
LABEL maintainer="cokemaniaIIDX cokemaniaIIDX@gmail.com"

RUN echo "building nginx server..."

RUN echo "copy conf files"
COPY nginx/nginx.conf /etc/nginx/nginx.conf
COPY nginx/conf.d/healthcheck.conf /etc/nginx/conf.d/healthcheck.conf

RUN echo "copy healthcheck file"
COPY var/www/html/healthcheck.html /usr/share/nginx/html/healthcheck.html