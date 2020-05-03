FROM python:3.8.1-alpine3.11

ENV SERVER_PORT 8000
ENV SERVER_STATIC_CONTENT_DIR "/app/"

RUN set -x \
    && mkdir $SERVER_STATIC_CONTENT_DIR \
    && addgroup -g 1001 -S appgroup \
    && adduser -S -D -H -u 1001 -s /sbin/nologin -G appgroup -g appuser appuser

EXPOSE $SERVER_PORT

USER appuser

CMD python -m http.server $SERVER_PORT --directory $SERVER_STATIC_CONTENT_DIR

