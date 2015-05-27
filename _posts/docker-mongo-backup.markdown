I'm working on my pet project. This's simple blog engine written in java 8 and using mongo as primery database. I was pretty excited about docker when I heard about it first time. Recentry I've written [small article]({% post_url 2015-05-22-docker1 %}) with docker essentials - it's a post with basic commands which I use every time when I need to do something with docker.

Today I'm going to share my scripts which I wrote to make and restore backup of mongodatabase. I was surprised that I didn't find anything similar on google. I hope that this could be helpfulf for somebody.

## Backup

{% highlight bash %}
rm -rf /tmp/mongodump && mkdir /tmp/mongodump
#cp $1 /tmp/mongorestore/$FILENAME
docker run -it --rm --link mongo:mongo -v /tmp/mongodump:/tmp mongo bash -c 'mongodump -v --host $MONGO_PORT_27017_TCP_ADDR:$MONGO_PORT_27017_TCP_PORT --db '$1' --out=/tmp'
tar -cvf $2 -C /tmp/mongodump *
rm -rf /tmp/mongodump
{% endhighlight %}

Now you can use it as follows:

{% highlight bash %}

{% endhighlight %}

## Restoring

{% highlight bash %}
rm -rf /tmp/mongorestore && mkdir /tmp/mongorestore
FILENAME=$(echo $1 | sed 's/.*\///')
cp $1 /tmp/mongorestore/$FILENAME
docker run -it --rm --link mongo:mongo -v /tmp/mongorestore:/tmp mongo bash -c 'mongorestore --drop -v --host $MONGO_PORT_27017_TCP_ADDR:$MONGO_PORT_27017_TCP_PORT --db '$2' /tmp/'$FILENAME
rm -rf /tmp/mongorestore
{% endhighlight %}
