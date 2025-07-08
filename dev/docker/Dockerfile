# builder stage
FROM golang:1.22-alpine as builder

#RUN apt-get update && apt-get install -y libpam-dev
RUN apk --no-cache add git gcc musl-dev linux-pam-dev openssl

# add user
RUN adduser --disabled-password --gecos "" --home /opt/rdpgw --uid 1001 rdpgw

# certificate
RUN mkdir -p /opt/rdpgw && cd /opt/rdpgw && \
    random=$(cat /dev/urandom | tr -dc 'a-zA-Z0-9' | fold -w 32 | head -n 1) && \
    openssl genrsa -des3 -passout pass:$random -out server.pass.key 2048 && \
    openssl rsa -passin pass:$random -in server.pass.key -out key.pem && \
    rm server.pass.key && \
    openssl req -new -sha256 -key key.pem -out server.csr \
    -subj "/C=US/ST=VA/L=SomeCity/O=MyCompany/OU=MyDivision/CN=rdpgw" && \
    openssl x509 -req -days 365 -in server.csr -signkey key.pem -out server.pem

# clone repository
ARG CACHEBUST
RUN git clone https://github.com/Threebents/rdpgw.git /app

# change to app directory and check go.mod
WORKDIR /app
RUN cat go.mod
RUN go version
RUN go mod download -x

# build rdpgw (without CGO)
RUN CGO_ENABLED=0 GOOS=linux go build -trimpath -tags '' -ldflags '' -o '/opt/rdpgw/rdpgw' ./cmd/rdpgw

# build rdpgw-auth (with CGO)
RUN CGO_ENABLED=1 GOOS=linux go build -trimpath -tags '' -ldflags '' -o '/opt/rdpgw/rdpgw-auth' ./cmd/auth

# set permissions
RUN chmod +x /opt/rdpgw/rdpgw && \
    chmod +x /opt/rdpgw/rdpgw-auth && \
    chmod u+s /opt/rdpgw/rdpgw-auth

FROM alpine:latest

RUN apk --no-cache add linux-pam musl

# make tempdir in case filestore is used
ADD tmp.tar /

COPY --chown=0 rdpgw-pam /etc/pam.d/rdpgw

USER 1001
COPY --chown=1001 run.sh run.sh
COPY --chown=1001 --from=builder /opt/rdpgw /opt/rdpgw
COPY --chown=1001 --from=builder /etc/passwd /etc/passwd
COPY --chown=1001 --from=builder /etc/ssl/certs /etc/ssl/certs

USER 0

WORKDIR /opt/rdpgw
ENTRYPOINT ["/bin/sh", "/run.sh"]
