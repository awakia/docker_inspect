# Dockerfile sample to inspect

## Before you start

In your mac, create 1MB and 1GB dummy files:

```
mkfile 1m 1m.dummy
mkfile 1g 1g.dummy
```

## Experiments

### Experiment 1: 1GB vs 1MB without .dockerignore

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
# ADD 1g.dummy /tmp
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

#### Conclusion

You should use .dockerignore to exclude context

### Experiment 2: Ignore 1GB vs 1MB

#### Ignore 1GB

.dockerignore
```
1g.dummy
```

```
time docker build -t experiment .
Sending build context to Docker daemon 1.122 MB
Step 1 : FROM gliderlabs/alpine:3.3
 ---> 06f3a228f35b
Step 2 : ADD 1m.dummy /tmp
 ---> Using cache
 ---> d92e6bed1fa5
Successfully built d92e6bed1fa5
docker build -t experiment .  0.19s user 0.04s system 49% cpu 0.451 total
```

So fast!!!

### Ignore 1MB

.dockerignore
```
1m.dummy
```

```
time docker build -t experiment .
Sending build context to Docker daemon 1.074 GB
Step 1 : FROM gliderlabs/alpine:3.3
 ---> 06f3a228f35b
Step 2 : ADD 1g.dummy /tmp
 ---> Using cache
 ---> dcda95cda1b8
Successfully built dcda95cda1b8
docker build -t experiment .  12.80s user 2.19s system 45% cpu 33.031 total
```

It takes 33 seconds.


#### Conclusion

Sending context itself is slow. We need to inspect why its so slow.

### Experiment 3: Check normal copy speed

#### If you copy 1GB file

```
$ time cp 1g.dummy 1g.dummy.copy
cp -i 1g.dummy 1g.dummy.copy  0.00s user 1.07s system 46% cpu 2.304 total
```

It takes less than 3 seconds.
33 seconds is way too slow.

#### Conclusion

Sending build context is doing more than normal copy.

### Experiment 4: Use docker COPY instead of ADD

Dockerfile:
```
FROM gliderlabs/alpine:3.3
COPY 1g.dummy /tmp
```

```
time docker build -t experiment .
Sending build context to Docker daemon 1.074 GB
Step 1 : FROM gliderlabs/alpine:3.3
 ---> 06f3a228f35b
Step 2 : COPY 1g.dummy /tmp
 ---> bb2e26df224a
Removing intermediate container cb2e4edf9242
Successfully built bb2e26df224a
docker build -t experiment .  12.31s user 2.34s system 34% cpu 42.183 total
```

Docker COPY is slower than ADD

references:
- https://github.com/docker/docker/blob/master/docs/reference/builder.md#add
- https://github.com/docker/docker/blob/680cf1c8f4164d7b3e95759a7cc93ce5843e5543/builder/dockerfile/dispatchers.go#L150-L151

> Add the file 'foo' to '/path'. Tarball and Remote URL (git, http) handling
> exist here. If you do not wish to have this automatic handling, use COPY.

Though, ADD is better.

#### Conclusion

ADD is better than COPY. but this is not what I want to resolve. I do not dive into this problem any farther.
