# Hoverfly and nginx proxy_pass Bug

## Steps to reproduce with local docker

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

### 4. Run Hoverfly

```
docker run --name hoverfly -v `dirname $PWD/.`/simulation.json:/etc/hoverfly/simulation.json:ro --network hoverflybug -p 8888:8888 -p 8500:8500 -d spectolabs/hoverfly:latest -dest mitm -spy -import /etc/hoverfly/simulation.json -log-level debug
```

### 5. Execute curl In-container


#### PUT with payload, no Hoverfly => Works

```
docker run --rm --network hoverflybug curlimages/curl:latest -XPUT http://mitm:8501/123 -vvv -d 'test'
```

#### PUT without payload and Hoverfly => Works

```
docker run --rm --network hoverflybug curlimages/curl:latest -XPUT http://mitm:8501/123 -vvv --proxy http://hoverfly:8500
```

#### PUT with payload and Hoverfly => Does not work, nginx gives 400

```
docker run --rm --network hoverflybug curlimages/curl:latest -XPUT http://mitm:8501/123 -vvv --proxy http://hoverfly:8500 -d 'test'
```

### 6. Tear down

```
docker rm -f hoverfly && docker rm -f mitm && docker rm -f teapot && docker network rm hoverflybug
```
