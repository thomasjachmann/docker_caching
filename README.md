# Docker Caching

I have a problem with docker's caching for ```ADD``` instructions. This is current with Docker version 1.1.2, build d84a070.


## Background

I'm building a docker image for a rails app and wanted to use the standard
strategy to prevent unnecessary ```bundle install```s by ```ADD```ing the
Gemfile first, then bundling and adding the rest afterwards. The first two steps
should be cached as long as the Gemfile isn't changed.


## Gotcha

There's an additional gotcha: I can't just clone the repository and issue
```docker build``` because the repository is rather large and contains files
that shouldn't be included in the docker image. Packaging them and sending them
to the docker server is not an option, so I wrote a build script that exports
the current HEAD and builds the docker image from there.  # Docker Caching 


## How to reproduce the problem

Just call:

```bash
./build
```

On the first run, this produces:

```
Sending build context to Docker daemon  7.68 kB
Sending build context to Docker daemon
Step 0 : from busybox
 ---> a9eb17255234
Step 1 : ADD export/hard_work /app/hard_work
 ---> 17b4f579f0a7
Removing intermediate container 4244c46f94b7
Step 2 : RUN echo ">>>>> I'm doing intensive work based on hard_work now! <<<<<"
 ---> Running in 566189d272bc
>>>>> I'm doing intensive work based on hard_work now! <<<<<
 ---> 5aab022d522c
Removing intermediate container 566189d272bc
Step 3 : ADD export /app
 ---> d2dc250d9e25
Removing intermediate container 740c5e3f2170
Step 4 : RUN echo ">>>>> The rest was added and invalidated the cache. <<<<<"
 ---> Running in bfbf4bfb8e73
>>>>> The rest was added and invalidated the cache. <<<<<
 ---> 95c1e0ca6d14
Removing intermediate container bfbf4bfb8e73 Successfully built 95c1e0ca6d14
```

On following runs, this produces:

```
Sending build context to Docker daemon  7.68 kB
Sending build context to Docker daemon
Step 0 : from busybox
 ---> a9eb17255234
Step 1 : ADD export/hard_work /app/hard_work
 ---> Using cache
 ---> 17b4f579f0a7
Step 2 : RUN echo ">>>>> I'm doing intensive work based on hard_work now! <<<<<"
 ---> Using cache
 ---> 5aab022d522c
Step 3 : ADD export /app
 ---> dea611a3647c
Removing intermediate container 0a39e87df8cc
Step 4 : RUN echo ">>>>> The rest was added and invalidated the cache. <<<<<"
 ---> Running in 48c27ac56603
>>>>> The rest was added and invalidated the cache. <<<<<
 ---> b1427ce029d9
Removing intermediate container 48c27ac56603
Successfully built b1427ce029d9
```

The cache is being used for adding the single file, the intensive work isn't
done, as expected.

But after creating an empty commit with ```git commit --allow-empty -m empty```,
this is the result of a ```./build```:

```
Sending build context to Docker daemon  7.68 kB
Sending build context to Docker daemon
Step 0 : from busybox
 ---> a9eb17255234
Step 1 : ADD export/hard_work /app/hard_work
 ---> f507c437b23e
Removing intermediate container 8a113eca2c91
Step 2 : RUN echo ">>>>> I'm doing intensive work based on hard_work now! <<<<<"
 ---> Running in cf225987ee9e
>>>>> I'm doing intensive work based on hard_work now! <<<<<
 ---> 74eb4b4b90be
Removing intermediate container cf225987ee9e
Step 3 : ADD export /app
 ---> 44249a1b0e6a
Removing intermediate container d167acf92d8b
Step 4 : RUN echo ">>>>> The rest was added and invalidated the cache. <<<<<"
 ---> Running in a8234ebd6b4c
>>>>> The rest was added and invalidated the cache. <<<<<
 ---> 01ef2ec776be
Removing intermediate container a8234ebd6b4c
Successfully built 01ef2ec776be
```

```hard_work``` is ```ADD```ed again although it hasn't changed. Nothing really
has changed, just an empty commit has been added that shouldn't even result in
any changes in the exported version.

What am I doing wrong? How can I get ```ADD``` to use the cache when the file's
contents don't change? I thought docker used tar to create a checksum of
directories ```ADD```ed so it would even be able to cache line 6 in my ```Dockerfile```.
