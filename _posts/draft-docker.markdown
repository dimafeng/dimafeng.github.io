---
layout: post
title:  "Docker tips"
categories: docker
---
This's not a complete article. It'll be a post with docker tips which I'll append with new tips as they become known for me.

## Docker install

Docker still dosn't have full support in mac os. Of course they have [Boot2Docker][2], but it's docker lauched within a VM. So, I prefer use my own VM running in VirtualBox on another machine. I use ubuntu. Installation on ubuntu linux looks as follows:

{% highlight bash %}
sudo apt-get install docker.io
{% endhighlight %}

If you see error like that:

{% highlight bash %}
FATA[0000] Post http:///var/run/docker.sock/v1.17/images/create?fromImage=mongo%3Alatest: dial unix /var/run/docker.sock: no such file or directory
{% endhighlight %}

you just need to reboot server. 

## Mongo container 

If you want to start mongo within docker container you won't need to write your own Docker file to build image with mongo, you can just download [official docker image][1] from repo.

Pulling the image from repo:
{% highlight bash %}
sudo docker pull mongo
{% endhighlight %}

Running the image:
{% highlight bash %}
sudo docker run --name last-mongo -d mongo
{% endhighlight %}



[1]: https://registry.hub.docker.com/u/library/mongo/
[2]: https://docs.docker.com/installation/mac/