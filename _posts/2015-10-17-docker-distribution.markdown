---
layout: devops
title:  "Thoughts on Docker, docker-compose and life without registry"
categories: docker, mongo db
---
Not long ago, I wrote two articles about docker usage ([one](/2015/05/22/docker1/),
[two](/2015/05/31/docker-mongo-backup/)), now it's time for third one. I've been using docker for several months and
I've got new thoughts on this technology. It's fair to say that docker didn't have rich tooling,
now it's getting better but it's still not ideal. I had problems with container deployment, container data backups
and etc. In this blog post, I'll show you how I use docker for my pet-project.

## Docker registry replacement

The standard way of container distribution is usage of docker registry. This technique looks like a version
control system e.g. svn. It means that you can push the current state of the image to a server and
then a target machine may just pull it from there. Sounds good, but not in case of personal use -
I don't want to have my own docker registry as well as support of this infrastructure. To solve this problem,
we can use image export functionality. You can do it in this way:

{% highlight bash %}
docker save [image name] | gzip > [output file]
{% endhighlight %}

And then you can upload this image to the server. This works pretty good except one thing, this way requres you to copy
all image layers. It means even small change in your application will produce full image re-upload including base image.

## Docker-compose

My pet project is represented by two containers: java app + mongo database. When I put everything into docker
the **docker-compose** hadn't been released and I had to have bash script which describes all steps of containers' start.
But now I don't need to do it anymore.

>Compose is a tool for defining and running multi-container applications with Docker. With Compose, you define a multi-container application in a single file, then spin your application up in a single command which does everything that needs to be done to get it running.

Sounds pretty cool!

I created simple `docker-compose.yml`:

{% highlight bash %}
mongo:
  image: mongo:3.1.9
  mem_limit: 200MB
  volumes_from:
    - data
  restart: always
data:
  image: mongo:3.1.9
  volumes:
    - /data/db
  command: --break-mongo
app:
  image: [app-name]
  container_name: [app-name]
  ports:
    - "80:8080"
  links:
    - mongo:mongo
  restart: always
{% endhighlight %}

As you see, this looks like regular docker commands written in yml format. Here we have:

* **mongo** container with mongo db started with 200mb ram limit;
* **data** data volume which stores all mongo data;
* **app** application linked to container with mongo db.

It's very clear. And now we can start all container at one time by using just one command:

{% highlight bash %}
docker-compose up
{% endhighlight %}

or

{% highlight bash %}
docker-compose start
{% endhighlight %}

## Put it all together

Let's put **docker compose** and **docker save** all together. I created a bash script `build.sh`:

{% highlight bash %}
REMOTE_HOST=$1
IMAGE_NAME="myApp"
docker build --no-cache -t $IMAGE_NAME .
docker save $REMOTE_HOST | gzip > /tmp/img.zip
scp ./build-remote.sh $REMOTE_HOST:~/build-remote.sh
scp ./docker-compose.yml $REMOTE_HOST:~/docker-compose.yml
scp /tmp/img.zip $REMOTE_HOST:/tmp/myApp.zip
ssh $REMOTE_HOST 'chmod +x ~/build-remote.sh && ~/build-remote.sh'
{% endhighlight %}

And `build-remote.sh` looks like:

{% highlight bash %}
docker-compose stop
docker rm myApp
gunzip < /tmp/myApp.zip | docker load
docker-compose start
{% endhighlight %}

I use it as follows:

{% highlight bash %}
build.sh [user]@[server host]
{% endhighlight %}

What it does? It's simple, first of all, we need to build new version of the image and export it into an archive.
Then we need to transfer it to the server as well as additional scripts and `docker-compose.yml`. Now we're ready to
continue working on server. Here we need to remove old version of container and restart everyting. That's it.

## Backing up

Unfortunately, since my previous post, there weren't big changes in this process. There is a
[thread on StackOverFlow](http://stackoverflow.com/questions/18496940/how-to-deal-with-persistent-storage-e-g-databases-in-docker)
about this problem and it's sad. That's why I stay with backing up process described in [this post](/2015/05/31/docker-mongo-backup/).

## Conclusion

Docker is great for big infrastructures when you have several servers and tens of container. For small projects, it's good, but
sometimes it made me think that it's over-engineering - I don't understand why it's still difficult to backup container's
data. Eventhoug the great thing about docker is that it grows very fast. So, it seems docker will be perfect very soon.
