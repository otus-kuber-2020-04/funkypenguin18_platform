FROM nginx:latest
RUN useradd -u 1001 user
WORKDIR /app
RUN chown -R user:user /app/
COPY ./config/nginx.conf /etc/nginx/nginx.conf

USER user
EXPOSE 8000
