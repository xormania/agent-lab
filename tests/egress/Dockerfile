FROM alpine:3.20.10

RUN apk add --no-cache bash bind-tools ca-certificates curl openssl

RUN addgroup -g 1000 tester \
    && adduser -D -H -u 1000 -G tester tester

COPY tests/egress/cases.sh /tests/egress/cases.sh
RUN chmod 0555 /tests/egress/cases.sh

USER 1000:1000
ENTRYPOINT ["/bin/bash", "/tests/egress/cases.sh"]
