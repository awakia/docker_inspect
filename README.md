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
$ time docker build -t experiment .
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


### Experiment 5: ADD from url

Sending context is slow, if so fetching file from network can be a solution

Dockerfile:
```
FROM gliderlabs/alpine:3.3
ADD http://localhost:8080/1g.dummy /tmp
```

After running static file server at localhost:8080, run experiment below

```
$ time docker build -t experiment .
Sending build context to Docker daemon 750.6 kB
Step 1 : FROM gliderlabs/alpine:3.3
---> 06f3a228f35b
Step 2 : ADD http://localhost:8080/1g.dummy /tmp
Downloading [==================================================>] 1.074 GB/1.074 GB
---> d00fc8bac44a
Removing intermediate container 058e58ce0921
Successfully built d00fc8bac44a
docker build -t experiment .  0.46s user 0.17s system 1% cpu 41.694 total
```

Sending context is fast, but it takes so much time after downloading.
Download time is about 5 seconds.

```
[negroni] Started GET /1g.dummy
[negroni] Completed 200 OK in 5.12256144s
```

### Conclusion

Fetch file from network solves "Sending build context" problem, but it cause another problem after downloading file.


### Code Reading 1: Read around "Sending build context to Docker daemon"

#### Find the place "Sending build context to Docker daemon"

```
var body io.Reader = progress.NewProgressReader(ctx, progressOutput, 0, "", "Sending build context to Docker daemon")

```

https://github.com/docker/docker/blob/931e78df1b50e83837bff4966bb62458675e3c32/api/client/build.go#L179



#### progress.NewProgressReader definition

```
func NewProgressReader(in io.ReadCloser, out Output, size int64, id, action string) *Reader {
```

https://github.com/docker/docker/blob/master/pkg/progress/progressreader.go#L19

just output progress. important thing is just `ctx`

#### important operation if build context is localfile

If build context is from localfile, `tar` command is used.

```
ctx, err = archive.TarWithOptions(contextDir, &archive.TarOptions{
	Compression:     archive.Uncompressed,
	ExcludePatterns: excludes,
	IncludeFiles:    includes,
})

```

https://github.com/docker/docker/blob/931e78df1b50e83837bff4966bb62458675e3c32/api/client/build.go#L159-L163

So check time consuming to create tar like below:

```
$ time tar cf 1g.tar 1g.dummy
tar cf 1g.tar 1g.dummy  0.05s user 1.70s system 44% cpu 3.905 total
```

about 4 seconds. First enough.

#### Another place to change ctx

```
ctx = replaceDockerfileTarWrapper(ctx, relDockerfile, cli.trustedReference, &resolvedTags)
```

https://github.com/docker/docker/blob/931e78df1b50e83837bff4966bb62458675e3c32/api/client/build.go#L173


```
func replaceDockerfileTarWrapper(inputTarStream io.ReadCloser, dockerfileName string, translator translatorFunc, resolvedTags *[]*resolvedTag) io.ReadCloser {
	pipeReader, pipeWriter := io.Pipe()
	go func() {
		tarReader := tar.NewReader(inputTarStream)
		tarWriter := tar.NewWriter(pipeWriter)

		defer inputTarStream.Close()

		for {
			hdr, err := tarReader.Next()
			if err == io.EOF {
				// Signals end of archive.
				tarWriter.Close()
				pipeWriter.Close()
				return
			}
			if err != nil {
				pipeWriter.CloseWithError(err)
				return
			}

			var content io.Reader = tarReader
			if hdr.Name == dockerfileName {
				// This entry is the Dockerfile. Since the tar archive was
				// generated from a directory on the local filesystem, the
				// Dockerfile will only appear once in the archive.
				var newDockerfile []byte
				newDockerfile, *resolvedTags, err = rewriteDockerfileFrom(content, translator)
				if err != nil {
					pipeWriter.CloseWithError(err)
					return
				}
				hdr.Size = int64(len(newDockerfile))
				content = bytes.NewBuffer(newDockerfile)
			}

			if err := tarWriter.WriteHeader(hdr); err != nil {
				pipeWriter.CloseWithError(err)
				return
			}

			if _, err := io.Copy(tarWriter, content); err != nil {
				pipeWriter.CloseWithError(err)
				return
			}
		}
	}()

	return pipeReader
}
```

https://github.com/docker/docker/blob/931e78df1b50e83837bff4966bb62458675e3c32/api/client/build.go#L361-L410

These codes are suspicious for slowing down the speed. Need to check later.

#### Sending context to docker deamon

After creating `ctx`, `ctx` is passed to `body`:

```
var body io.Reader = progress.NewProgressReader(ctx, progressOutput, 0, "", "Sending build context to Docker daemon")
```
https://github.com/docker/docker/blob/931e78df1b50e83837bff4966bb62458675e3c32/api/client/build.go#L179

and then, passed to `options.Context`:

```
options := types.ImageBuildOptions{
	Context:        body,
```

https://github.com/docker/docker/blob/931e78df1b50e83837bff4966bb62458675e3c32/api/client/build.go#L212


Then, `ImageBuild` is called

```
response, err := cli.client.ImageBuild(context.Background(), options)
```

https://github.com/docker/docker/blob/931e78df1b50e83837bff4966bb62458675e3c32/api/client/build.go#L235

In `/docker/engine-api/client/image_build.go`

```
func (cli *Client) ImageBuild(ctx context.Context, options types.ImageBuildOptions) (types.ImageBuildResponse, error) {
	...
	serverResp, err := cli.postRaw(ctx, "/build", query, options.Context, headers)
	...
}
```

https://github.com/docker/docker/blob/95d827cda21c690fabf1d577ea0c489ac26c805a/vendor/src/github.com/docker/engine-api/client/image_build.go#L23-L48


And then, in `/docker/engine-api/client/request.go`

```
func (cli *Client) postRaw(ctx context.Context, path string, query url.Values, body io.Reader, headers map[string][]string) (*serverResponse, error) {
	return cli.sendClientRequest(ctx, "POST", path, query, body, headers)
}
```

just send POST request to deamon.
