+++
date = "2016-01-13T17:21:05Z"
title = "Fuzzing automation with AFL and Jenkins"

+++

# Introduction
Fuzzing is a very effective testing technique for discovering software bugs. The basic principle is simple. Feed your program with mutated/random data, watch how it reacts. Security researchers spend countless hours creating fuzzers, trimming test cases and fuzzing interesting programs, for fun and profit. Fuzz testing is also considered an important part of every Secure Development Lifecycle (SDLC) and should be performed on a regular basis. Is it though? Considering the modern Continuous Delivery environment and the time needed for preparing and running a successful fuzzing test suite, this type of testing is often left as a permanent entry in the team's backlog or ends up as a one-time test.

Fortunately, the arsenal of fuzzing tools has increased over the past few years and their usability keeps getting better and better. A great example of this progress is the [American Fuzz Lop](http://lcamtuf.coredump.cx/afl/) (AFL) fuzzer created by Michał Zalewski.

However, no matter how easy-to-learn a tool is, sometimes it's all about integrating it into an existing workflow. This tutorial explains how to easily prepare fuzzing jobs in Jenkins using AFL with a little help from Docker.

# Choosing the right Jenkins plugin
Although there are numerous Jenkins plugins for Docker integration I decided to go with [CloudBees Docker Custom Build Environment Plugin](https://wiki.jenkins-ci.org/display/JENKINS/CloudBees+Docker+Custom+Build+Environment+Plugin). The reasons where:

1. Compared to [docker-plugin](https://wiki.jenkins-ci.org/display/JENKINS/Docker+Plugin), Cloudbees' plugin follows a more strict Docker workflow. Devs should not get used to SSH connect to their containers and juggle between sudo-enabled users even in development environments.

2. [Docker build step](https://wiki.jenkins-ci.org/display/JENKINS/Docker+build+step+plugin) could have been another option but it lacked some essential features like unix sockets and volume support.

So, before proceeding make sure you have CloudBees Docker Custom Build Environment Plugin installed.

## Preparing your Docker image
In order to make things simple I created an Ubuntu-based Docker image which has AFL pre-installed in `/usr` and all essential environmental variables set (e.g. CXX,CC, AFL_HARDEN) You can find the image [here](https://hub.docker.com/r/zubu/afl-slave/) and the Dockerfile [here](https://github.com/zuBux/afl-slave).

For the purposes of this tutorial I am going to use the lha compression utility as an example fuzzing target.

Open Jenkins homepage in your browser and select "**New Item**" and "**Freestyle project**" option. Name your Item something like lha-fuzz to distinguish it from a normal build (supposing you 're a member of the development team ) Select your prefered Source Code Management software, in our case it's Git. Specify a sub-folder for checkout, because you'll need a consistent way to refer to the repository from the Dockerfile.

![](/img/git_configuration.png)

Now you 'll need to create a Dockerfile to define your build environment. Since the afl-slave image takes care of everything AFL-related, you only need to specify the dependencies and steps to build your source code. In this case, the Dockerfile will look like this:

```
FROM zubu/afl-slave

# Copy lha code and build it
WORKDIR /lha
RUN aclocal
RUN autoheader
RUN automake -a
RUN autoconf
RUN ./configure && make && make clean all

# Command to see check that everything's working
CMD ["afl-fuzz","-i","/testcases","-o","/tmp/output","/lha/src/lha","x","@@","-w /tmp"]
```

Place the Dockerfile in your workspace, in Debian-based distributions that's `/var/lib/jenkins/workspace/<item name>`. Select "**Build inside a Docker container**". Add two volumes, one for your test cases and one for AFL output. Place the Dockerfile inside your project's workspace and specify the path in "path to docker context" field.

![](/img/docker_volumes.png)

# Running your fuzzing job
For demo purposes let's fuzz the `lha` executable. Select "**Add Build Step**" and choose "**Execute Shell**". Add the following:

```bash
afl-fuzz -i /testcases -o /output/$BUILD_NUMBER /lha/src/lha x @@ -w /tmp
```

Don't forget to run `echo core >/proc/sys/kernel/core_pattern` in your Docker host before starting the container. For more information regarding the AFL fuzzer you can visit the official [README](http://lcamtuf.coredump.cx/afl/README.txt).

Save your job and click "**Build Now**".

## Future Work
The process of evaluating your fuzz test results can be quite painful. Since I'm looking into ways of automating the whole process, I plan to add Ben Nagy's [crashwalk](https://github.com/bnagy/crashwalk) in the afl-slave image. Crashwalk automatically performs bug triage and produces readable (and/or parsable) reports for each crashing test case. I was planning on including it in this tutorial but unfortunately GDB does not always play well with containers. **Stay tuned for Part 2!**
