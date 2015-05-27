I'm trying to start my pet project within docker.  

{% highlight bash %}
rm -rf /tmp/mongorestore && mkdir /tmp/mongorestore
FILENAME=$(echo $1 | sed 's/.*\///')
cp $1 /tmp/mongorestore/$FILENAME
docker run -it --rm --link mongo:mongo -v /tmp/mongorestore:/tmp mongo bash -c 'mongorestore --drop -v --host $MONGO_PORT_27017_TCP_ADDR:$MONGO_PORT_27017_TCP_PORT --db '$2' /tmp/'$FILENAME
rm -rf /tmp/mongorestore
{% endhighlight %}

{% highlight bash %}
rm -rf /tmp/mongodump && mkdir /tmp/mongodump
#cp $1 /tmp/mongorestore/$FILENAME
docker run -it --rm --link mongo:mongo -v /tmp/mongodump:/tmp mongo bash -c 'mongodump -v --host $MONGO_PORT_27017_TCP_ADDR:$MONGO_PORT_27017_TCP_PORT --db '$1' --out=/tmp'
tar -cvf $2 -C /tmp/mongodump *
rm -rf /tmp/mongodump
{% endhighlight %}
