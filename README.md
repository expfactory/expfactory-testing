# Expfactory Testing

This repository will help you to test your experiment setup (or other) within
an experiment factory container. We will basically be building a container with
some version of expfactory installed (one you will clone locally and bind to the container
to install) and then include one or more experiments (from the library or folders in
this repository.) Here we have provided an example task in experiments, the
stroop task exported from labjs.

## 1. Clone Expfactory
The first step is to have a local version of expfactory for use. Likely you will have
forked the repository, and checked out a new branch to work with.

```bash
git clone git@github.com:expfactory/expfactory.git
```

## 2. Add Experiments
If desired, add experiment folders to the present working directory. You can skip this
step and just use identifiers from [the library](https://expfactory.github.io/experiments), but
likely you have an experiment that you are working on that you want to test!

```bash
# stroop-task is a subfolder in this $PWD
EXPS="/data/stroop-task"

$ echo $EXPS
/data/stroop-task
```

If you wanted to add an experiment from the library (meaning no subfolder here), you could add it's unique id here, but without `/data`.

```bash
EXPS="/data/stroop-task test-task"
```

## 3. Generate your Dockerfile
Now we will use the expfactory builder to generate a Dockerfile. Let's run the command to generate the Dockerfile. Notice how we are binding `$PWD` to `/data`? This is going to be where our experiment is installed from, and where the Dockerfile is written to.

```bash
docker run -v $PWD/:/data vanessa/expfactory-builder build ${EXPS}
Expfactory Version: 3.13
local experiment /data/stroop-task found, validating...
LOG Recipe written to /data/Dockerfile
WARNING 1 local installs detected: build is not reproducible without experiment folders

To build, cd to directory with Dockerfile and:
              docker build --no-cache -t expfactory/experiments .
```

Now we have a Dockerfile and startscript. Great! You might need to tweak permissions, since the folder was bound (and edited as root user from the container)

```bash
sudo chown -R $USER $PWD
```

## 4. Customize the Dockerfile
Here is where we need to customize the Dockerfile. Note that you can also start with the one provided in this example repository as an example.  Here are the files we have in the $PWD, it's important that expfactory is here, along with your experiment subfolder

```bash
$ ls
Dockerfile  stroop-task  expfactory  README.md  startscript.sh
```

The reason we are doing this is so we can build the container and add the entire expfactory software we are working on from this local version. Now let's edit the Dockerfile to do this. Open it up!

See this line?

```bash
RUN git clone -b master https://www.github.com/expfactory/expfactory
```

This would normally clone and install expfactory from Github. We want to add the code to be used locally, so instead lets remove this line, and replace it with:

```
ADD expfactory/ /opt/expfactory
```

You are also free to replace the original command with a branch that you are pushing to (and working on)
but this requires an extra step of updating it first.

## 5. Make changes
Make all the changes you need to the expfactory folder - this is what will be added and installed to your container! You want to do this before building. Once you have your changes, then you are ready to build (and run) the container!

## 5. Build the development container
Now build your experiment container! Name it something simple, since you won't need to push to Docker Hub or maintain a namespace.

```bash
docker build -t expfactory-test .
...
Removing intermediate container 586afc7b8843
 ---> c577080944e2
Step 29/29 : EXPOSE 80
 ---> Running in 2652b4651576
Removing intermediate container 2652b4651576
 ---> 573148107121
Successfully built 573148107121
Successfully tagged expfactory-test:latest
```

This will go through through (and finish) the install routine! 

## 6. Start the Container
The easiest thing to do now is to run the container in detached, and shell inside.

```bash
docker run -d --name stroop -p 80:80 expfactory-test start
```

You should be able to open your browser now to see the server running. You can look at logs from the outside:

```bash
$ docker logs stroop
Database set as filesystem
Starting Web Server

 * Starting nginx nginx
   ...done.
==> /scif/logs/gunicorn-access.log <==

==> /scif/logs/gunicorn.log <==
[2018-07-15 22:27:27 +0000] [1] [INFO] Starting gunicorn 19.9.0
[2018-07-15 22:27:27 +0000] [1] [INFO] Listening at: http://0.0.0.0:5000 (1)
[2018-07-15 22:27:27 +0000] [1] [INFO] Using worker: sync
[2018-07-15 22:27:27 +0000] [35] [INFO] Booting worker with pid: 35
```

or leave it hanging on the console:

```bash
$ docker logs -f stroop
Database set as filesystem
Starting Web Server

 * Starting nginx nginx
   ...done.
==> /scif/logs/gunicorn-access.log <==

==> /scif/logs/gunicorn.log <==
[2018-07-15 22:27:27 +0000] [1] [INFO] Starting gunicorn 19.9.0
[2018-07-15 22:27:27 +0000] [1] [INFO] Listening at: http://0.0.0.0:5000 (1)
[2018-07-15 22:27:27 +0000] [1] [INFO] Using worker: sync
[2018-07-15 22:27:27 +0000] [35] [INFO] Booting worker with pid: 35
WARNING No user experiments selected, providing all 1
[2018-07-15 22:29:59,777] INFO in general: New session [subid] expfactory/3460e8e5-0415-474d-8961-9c8e37e348d0
[2018-07-15 22:29:59,786] INFO in utils: [router] None --> stroop-task [subid] expfactory/3460e8e5-0415-474d-8961-9c8e37e348d0 [user] You
[2018-07-15 22:30:03,681] DEBUG in main: Next experiment is stroop-task
[2018-07-15 22:30:03,681] INFO in utils: [router] stroop-task --> stroop-task [subid] expfactory/3460e8e5-0415-474d-8961-9c8e37e348d0 [user] You
[2018-07-15 22:30:03,682] DEBUG in utils: Redirecting to /experiments/stroop-task
[2018-07-15 22:30:03,702] DEBUG in utils: Rendering experiments/experiment.html
```

or use `--tail=30` to just show some number of lines at the end.

```bash
$ docker logs -f --tail=30 stroop
```

and I also like to shell inside to look at logs:

```bash
docker exec -it stroop bash
cat /scif/logs/expfactory.log
```

## 7. Debug Inside
In my case I clicked through the experiment to debug, and right clicked in the browser to see
the javascript console and debug a request. I then wanted to shell inside (in a different terminal) to see
if the files I had pickled were there (meaning that it hit the server).

```bash
docker exec -it stroop bash
ls /opt/expfactory
```

They weren't, which meant that the CORS settings were preventing the POST, not something happening on the server (e.g., the response never gets there!)
