---
layout: post
title:  "MongoDB backup within docker container"
categories: docker, bash
---
I'm working on my pet project. This's simple blog engine written in java 8 and using mongo as primery database. I was pretty excited about docker when I heard about it first time. Recentry I've written [small article]({% post_url 2015-05-22-docker1 %}) with docker essentials - it's a post with basic commands which I use every time when I need to do something with docker.

Today I'm going to share my scripts which I wrote to make and restore backup of mongo database. I was surprised that I didn't find anything similar on google. I hope that this could be helpfulf for somebody.

## Backup

{% highlight bash %}
rm -rf /tmp/mongodump && mkdir /tmp/mongodump
docker run -it --rm --link mongo:mongo -v /tmp/mongodump:/tmp mongo bash -c 'mongodump -v --host $MONGO_PORT_27017_TCP_ADDR:$MONGO_PORT_27017_TCP_PORT --db '$1' --out=/tmp'
tar -cvf $2 -C /tmp/mongodump *
rm -rf /tmp/mongodump
{% endhighlight %}

If you named you cantainer rather than `mongo` you need to change **link** parameter from `mongo:mongo` to `[name]:mongo`. Also you need to change variables `$MONGO_PORT_27017_TCP_ADDR` and `$MONGO_PORT_27017_TCP_PORT` in case if don't use default port (27017).

Now you can use it as follows:

{% highlight bash %}
./mongo-backup.sh [database_name] ~/backup.tar
{% endhighlight %}

`backup.tar` will contain all collections packed in `tar`.

## Restoring

{% highlight bash %}
TMP_DIR="/tmp/mongorestore/"
rm -rf $TMP_DIR && mkdir $TMP_DIR
if [[ $1 =~ \.tar$ ]];
then
        #FILENAME=$(echo $1 | sed 's/.*\///')
        FILENAME=$2"/"
        mkdir $TMP_DIR
        echo "Data will be extracted into :"$TMP_DIR
        tar -C $TMP_DIR -xvf $1
else
        FILENAME=$(echo $1 | sed 's/.*\///')
        cp $1 $TMP_DIR$FILENAME
fi

docker run -it --rm --link mongo:mongo -v $TMP_DIR:/tmp mongo bash -c 'mongorestore --drop -v --host $MONGO_PORT_27017_TCP_ADDR:$MONGO_PORT_27017_TCP_PORT --db '$2' /tmp/'$FILENAME
rm -rf $TMP_DIR
{% endhighlight %}

This script has 2 modes: 'restore tar' and 'restore collection'. The first mode looks as follows:

{% highlight bash %}
./mongo-restore.sh ~/backup.tar [database_name]
{% endhighlight %}

The second one may be like this:

{% highlight bash %}
./mongo-restore.sh ~/users.json [database_name]
{% endhighlight %}

---

If you want to improve my scripts and share it with me you can send me pull request to [this repository](https://github.com/dimafeng/dimafeng-examples/tree/master/scripts).
