---
description: How to authenticate HTTP request with Nginx
---

# Authenticating HTTP request with Nginx

Authgear has an internal endpoint that can authenticate HTTP request.

## Prerequisite

You must follow [this](../deploy-on-your-cloud/local.md) to get Authgear running first!

## Create a simple application server

Below is a very simple application server written in Python that echoes most of the request headers.

```python
from wsgiref.simple_server import make_server


def header_name(key):
    parts = key.split("_")[1:]
    parts = [part.lower() for part in parts]
    return "-".join(parts)


def app(environ, start_response):
    status = '200 OK'
    headers = [('Content-type', 'text/plain; charset=utf-8')]
    start_response(status, headers)

    for key, value in environ.items():
        if key.startswith("HTTP_"):
            name = header_name(key)
            yield ("%s: %s\n" % (name, value)).encode()


with make_server('', 8000, app) as httpd:
    print("listening on port 8000...")
    httpd.serve_forever()
```

Run it with:

```bash
python3 app.py
```

Visit [http://localhost:8000](http://localhost:8000) to verify it is working.

## Make the application server a service in docker-compose.yaml

We have to write a Dockerfile for our application server.

```text
FROM python:3.7-slim
COPY app.py .
EXPOSE 8000
CMD ["python3", "app.py"]
```

We then declare it as a new service in docker-compose.yaml:

```yaml
services:
  # Other services are omitted for brevity
  app:
    build:
      context: .
    ports:
    - "8000:8000"
```

Finally, run it!

```bash
docker-compose up
```

Visit [http://localhost:8000](http://localhost:8000) to verify our application server is working with docker-compose.

## Add Nginx

Copy the following nginx.conf and save it as nginx.conf.

```text
user  nginx;
worker_processes  1;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
    worker_connections  1024;
}


http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    server {
        server_name _;
        listen 80;

        location / {
            proxy_pass http://authgear:3000;
            proxy_set_header Host $host;
        }

        location /app {
            proxy_pass http://app:8000;
            proxy_set_header Host $host;
            auth_request /_auth;

            auth_request_set $x_authgear_session_valid $upstream_http_x_authgear_session_valid;
            auth_request_set $x_authgear_user_id $upstream_http_x_authgear_user_id;
            auth_request_set $x_authgear_user_anonymous $upstream_http_x_authgear_user_anonymous;
            auth_request_set $x_authgear_user_verified $upstream_http_x_authgear_user_verified;
            auth_request_set $x_authgear_session_acr $upstream_http_x_authgear_session_acr;
            auth_request_set $x_authgear_session_amr $upstream_http_x_authgear_session_amr;

            proxy_set_header x-authgear-session-valid $x_authgear_session_valid;
            proxy_set_header x-authgear-user-id $x_authgear_user_id;
            proxy_set_header x-authgear-user-anonymous $x_authgear_user_anonymous;
            proxy_set_header x-authgear-user-verified $x_authgear_user_verified;
            proxy_set_header x-authgear-session-acr $x_authgear_session_acr;
            proxy_set_header x-authgear-session-amr $x_authgear_session_amr;
        }

        location = /_auth {
            internal;
            proxy_pass http://authgear:3001/resolve;
            proxy_pass_request_body off;
            proxy_set_header Content-Length "";
        }
    }
}
```

Add Nginx in docker-compose.yaml:

```yaml
services:
  # Other services are omitted for brevity
  nginx:
    image: nginx:1.18
    volumes:
    - ./nginx.conf:/etc/nginx/nginx.conf
    ports:
    - "8080:80"
```

Visit [http://localhost:8080](http://localhost:8080) to reach Authgear and the application server.

## Verify the request is authenticated

Visit [http://localhost:8080](http://localhost:8080) and login, then visit [http://localhost:8080/app](http://localhost:8080/app).

You should see the following:

```text
x-authgear-session-valid: true
x-authgear-user-id: acdc0d99-f784-43d5-b53d-70f3506dfbf7
x-authgear-user-anonymous: false
```

