# Hoverfly with -plain-http-tunneling and nginx proxy_pass Bug

## Steps to Reproduce with Local Docker

### Preparations

Build locally a dockerized Hoverfly, one from commit

https://github.com/SpectoLabs/hoverfly/commit/9a0eee5f5dfd41e4597dc0725010048e94187794

as `my-hoverfly:9a0eee5` and another from the latest commit

https://github.com/SpectoLabs/hoverfly/commit/dd42a5b462d1fdbdac2b8dc2bf90a8d767a4caf3

as `my-hoverfly:dd42a5b`. Last step fails for both.

### 1. Create a Network

```
docker network create --driver bridge hoverflybug
```

### 2. Run a Teapot Nginx that responds 418

```
docker run --name teapot -v `dirname $PWD/.`/teapot.nginx.conf:/etc/nginx/nginx.conf:ro --network hoverflybug -p 8502:8502 -d nginx nginx-debug -g 'daemon off;'
```

### 3. Run a MiTM Nginx with proxy_pass to Teapot

```
docker run --name mitm -v `dirname $PWD/.`/mitm.nginx.conf:/etc/nginx/nginx.conf:ro --network hoverflybug -p 8501:8501 -d nginx nginx-debug -g 'daemon off;'
```

### 4. Run Hoverfly With `-plain-http-tunneling` and "headers" Request Matcher

```
docker run --name hoverfly -v `dirname $PWD/.`/simulation.json:/etc/hoverfly/simulation.json:ro --network hoverflybug -p 8888:8888 -p 8500:8500 -d my-hoverfly:dd42a5b -dest mitm -spy -import /etc/hoverfly/simulation.json -plain-http-tunneling -log-level debug
```

or the older version:

```
docker run --name hoverfly -v `dirname $PWD/.`/simulation.json:/etc/hoverfly/simulation.json:ro --network hoverflybug -p 8888:8888 -p 8500:8500 -d my-hoverfly:9a0eee5 -dest mitm -spy -import /etc/hoverfly/simulation.json -plain-http-tunneling -log-level debug
```

### 5. Execute curl In-container

#### PUT with payload and Hoverfly => Does not work

```
docker run --rm --network hoverflybug curlimages/curl:latest -XPUT http://mitm:8501/123 -vvv --proxytunnel --proxy http://hoverfly:8500 -d 'test'
```

With the old commit `9a0eee5` we get 400 from nginx due to issue https://github.com/SpectoLabs/hoverfly/issues/944 and with the commit `dd42a5b` the connection gets stuck.

NOTE: curl's `--proxytunnel` is the best way to emulate how Netty actually does HTTP Connect, and this exact bug is present now in Netty and Hoverfly.

### 6. Tear down

```
docker rm -f hoverfly && docker rm -f mitm && docker rm -f teapot && docker network rm hoverflybug
```
