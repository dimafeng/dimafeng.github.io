---
layout: devops
title:  "MongoDB backup within docker container"
categories: docker, bash
---
Here's a pet project I'm working on - It is a simple blog engine written in java 8 that uses mongodb as a primary database. I was pretty excited about docker when I heard about it for the first time. Recently, I wrote a [small article](/notes/2015/05/22/docker1/) describing docker essentials - it's a post showing basic fundamental commands that you can use when working with docker.

Today I'm going to share the scripts I wrote to make and restore backups of a mongodb database. I was surprised that I didn't find anything similar on google so I hope somebody finds these scripts helpful.

## Backup

{% highlight bash %}
rm -rf /tmp/mongodump && mkdir /tmp/mongodump
docker run -it --rm --link mongo:mongo -v /tmp/mongodump:/tmp mongo bash -c 'mongodump -v --host $MONGO_PORT_27017_TCP_ADDR:$MONGO_PORT_27017_TCP_PORT --db '$1' --out=/tmp'
tar -cvf $2 -C /tmp/mongodump *
rm -rf /tmp/mongodump
{% endhighlight %}

If you name your container something other than `mongo`, you need to change the **link** parameter from `mongo:mongo` to `[name]:mongo`. Also, if you don't use the default port (27017), you'll need to modify the variables `$MONGO_PORT_27017_TCP_ADDR` and `$MONGO_PORT_27017_TCP_PORT`.

You can now use it as follows:

{% highlight bash %}
./mongo-backup.sh [database_name] ~/backup.tar
{% endhighlight %}

Now `backup.tar` will contain all your collections packed in a `tar` file.

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

This script has two possible modes: tar restoring and collection restoring.

The first mode looks like this:

{% highlight bash %}
./mongo-restore.sh ~/backup.tar [database_name]
{% endhighlight %}

The second one may look like this:

{% highlight bash %}
./mongo-restore.sh ~/users.json [database_name]
{% endhighlight %}

---

If you want to help me improve my scripts, you can send me a pull request to [this repository](https://github.com/dimafeng/dimafeng-examples/tree/master/scripts).
