FROM nginx
COPY ./nginx.conf /etc/nginx/nginx.conf

RUN mkdir /app
RUN chown -R 1001 /app
RUN touch /var/run/nginx.pid && chown -R 1001 /var/run/nginx.pid
RUN chown -R 1001 /var/cache/nginx

EXPOSE 8000
USER 1001
