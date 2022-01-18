FROM nginx:1.20.2-alpine

COPY ./nginx.conf /etc/nginx/conf.d/default.conf

WORKDIR /app

EXPOSE 8000

# because of user 1001 required
RUN chmod -R 777 /var/cache/nginx /run
USER 1001
