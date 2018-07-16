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

## 8. Debug in the Browser
This is where the richest source of debugging is going to originate, because the browser carries all the secrets of your requests and responses, and you just need to know where to look. First, let's review what we are debugging. We have an experiment that is POSTing data to the server, the `/save` endpoint, and
it uses a different method ([fetch](https://davidwalsh.name/fetch)) instead of ([ajax](https://www.w3schools.com/xml/ajax_xmlhttprequest_send.asp)) and it's not working. Specifically, it returns a 400 error:

```javascript
POST http://127.0.0.1/save 400 (BAD REQUEST)
``` 

Ruhroh! 

### Step 1. Hypothesize what might be wrong

A 400 error is typically indicative of a bad request. Here are some reasons this might happen:

#### Errors coming from the view
It could be the case that the POST is reaching the view in the server, and something about it isn't liked, so the server view returns the 400. This could be any of the following:

 1. Some data is posted, the view doesn't like it, and returns the error.
 2. There is some authentication issue, but instead of a 403 or other, a 400 was returned as a more general "sorry this is a bad request."
 3. The expected format of the request is incorrect.

The nice thing about this kind of error is that you can debug it by adding a few lines to the view. In my case, I did the following:

 1. I added some printing to server logs so I could confirm that the view was hit (or not).
 2. I added some lines to save variables to a local file, so I could load. This is useful to then be able to interactively shell into the container, find the data files, load them, and then step through the code until you hit an error.

#### Errors from the Flask server
One level up from the view is the server, which in this case is uwsgi with Flask. The errors we are going to see here related to issues with CORS, or checks for cross request forgery. If this is the error, then we wouldn't see any evidence of the view getting hit, because it wouldn't get there, but we still might get a bad request.

#### Errors from the web Server
One level up from Flask is nginx, which would typically return a 500 error if it was forwarding along an issue from the application (e.g., some code has a typo).

Based on the error that I saw, I hypothesized that we were dealing with something related to the first or second. The next step would be to figure out the extent to which we are hitting the application.


### Step 2. Determine Level of Investigation
I next did as I mentioned previously - I added a bunch of saves and logging to the view that was being hit (and erroring) to determine if we were reaching it, period. The quick answer was that we were not - there was no indication that the application view was being touched. This was very good, because I knew that it was an issue with the Flask server, and specifically, I hypothesized it was an issue with CORS and correctly passing the csrf token header. This could be due to one of several issues.

**Is Jquery influencing it?**
The first (original) method of posting with Ajax had an extra dependency of JQuery, and to maintain support for these experiments I had kept jquery defined as a script. This particular view had the library defined, and also added an `ajaxSetup` to set the header in advance. For those interested, I do this so that any experiment can have the csrf token header added without needing to customize it for the experiment factory apriori. However, if it's the case that I'm not allowed to ask for the csrf token twice (once with the ajaxSetup, and then again via the fetch) the token wouldn't be passed, and we would get an error. To test this, I removed the extra jquery and ajaxSetup steps, and confirmed that the error happened regardless. Later I was also able to confirm that the same csrf token is passed via both methods, one just works and the other one doesn't. So the presence of absence of jquery, or the extra ajaxSetup didn't seem to be a variable.

**Is it the format of the data or headers?**
It's important to have an understanding of how the data is being POSTed to the server, and if possible, to have a comparison between a working example and a non working example. In my case, I was lucky to have an example of a POST (with Ajax) that my colleague had put together using the same experiment view that worked:

```bash
$.ajax({
  type: "POST",
  url: '/save',
  data: { "data": study.options.datastore.exportJson() },
  dataType: "application/json",
  success: function() { console.log('success') },
  error: function(err) { console.log('error', err) }
});
```

I would first try this, for a sanity check. To do this, you can just right click on the browser window, click inspect, and then go to the "Console" tab. You can type JavaScript into here, and whatever functions are defined on the page are available to you! I'd be in major trouble if something that was previously reported working was no longer working. Thankfully, it worked! At this point I knew that I had cornered the problem - we would be able to look at the complete record for a working vs. not working POST, from the same exact view. We've reached the goal of step 2, because the level of investigation is here from the browser. Let's jump into our next step to discuss how to do this.

### Step 3. Investigate 
With a working and non-working example, we can look more closely at a few things:

 - requests from the browser to the server
 - responses from the server back to the browser
 - the data being sent

We then compared this working POST to the one that is being done with fetch to hopefully figure out the issue. Here is an example, first the broken request:

![image](https://user-images.githubusercontent.com/814322/42739205-6cdfa162-8847-11e8-9f11-42059b33da78.png)

and here is the working one, with ajax:

![image](https://user-images.githubusercontent.com/814322/42739195-48631ae4-8847-11e8-9462-87cac679eb54.png)

We think that the answer lies here, and the way to debug this is to make changes, and test incrementally until something works.  The continued 400 response (bad request) tells us that there is still a difference between what this test stroop-task is posting internally with fetch, and what the ajax does. This suggests that something is missing so the server will accept it period.

### So What Happened?
After almost 100 [back and forths](https://github.com/FelixHenninger/lab.js/issues/18), we got it! It was an awesome span of work in under a few days, and it was both fun and fulfilling to tackle solving the culmination of many small pieces into one final product. If you want an early preview of our work, check out the <a href="https://expfactory.github.io/expfactory/integration-labjs" target="_blank">LabJS integration page here</a>, and you can expect some beautiful LabJS based documentation to come in the following weeks.
