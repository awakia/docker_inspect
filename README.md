# Dockerfile sample to inspect

## Before you start

In your mac, create 1MB and 1GB dummy files:

```
mkfile 1m 1m.dummy
mkfile 1g 1g.dummy
```

## Experiment

```
time docker build -t experiment .
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
