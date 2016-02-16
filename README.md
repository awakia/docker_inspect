# Dockerfile sample to inspect

## Before you start

In your mac, create 1MB and 1GB dummy files:

```
mkfile 1m 1m.dummy
mkfile 1g 1g.dummy
```

## Experiment

### 1GB vs 1MB without .dockerignore

#### 1GB

Dockerfile:
```
FROM gliderlabs/alpine:3.3
ADD 1g.dummy /tmp
# ADD 1m.dummy /tmp
```

Then execute the command in bash

```
$ time docker build -t experiment .
Sending build context to Docker daemon 1.075 GB
Step 1 : FROM gliderlabs/alpine:3.3
---> 06f3a228f35b
Step 2 : ADD 1g.dummy /tmp
---> Using cache
---> dcda95cda1b8
Successfully built dcda95cda1b8
docker build -t experiment .  13.05s user 2.35s system 46% cpu 33.476 total
```

It takes too long to sending build context...

#### 1MB

Dockerfile:
```
FROM gliderlabs/alpine:3.3
ADD 1g.dummy /tmp
ADD 1m.dummy /tmp
```

```
$ time docker build -t experiment .
Sending build context to Docker daemon 1.075 GB
Step 1 : FROM gliderlabs/alpine:3.3
 ---> 06f3a228f35b
Step 2 : ADD 1m.dummy /tmp
 ---> d92e6bed1fa5
Removing intermediate container 333fcc9496de
Successfully built d92e6bed1fa5
docker build -t experiment .  12.79s user 2.37s system 45% cpu 33.345 total
```

Almost same!

You should use .dockerignore to exclude context
