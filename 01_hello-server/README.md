# hello-server

build

```
docker build ./hello-server --tag hello-server:1.0
```

run

```
docker run --rm --detach --publish 8080:8080 --name hello-server hello-server:1.0
```

-> `curl localhost:8080`

stop

```
docker stop hello-server
```
ki
