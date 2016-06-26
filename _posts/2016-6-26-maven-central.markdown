---
layout: post
title:  "Publishing of your artifact to maven central"
categories: maven central, gradle
---

Recently, I finished my pet-project and wanted to publish it to maven central. As I found out later,
publishing is not a trivial thing. In this post, I'm going to show you all steps which you need to
go through to publish your artifact.

[My project is a scala wrapper][1] for a very useful library [testcontainers-java][2]. As you may assume, it's written in scala, but all the following is applicable for java/groovy/whatever language project if it's built by gradle.

## Preparation

First of all, you need to request a group id reservation. To do so, you need to sign up in [Sonatype's JIRA][3] and create an issue where you'll have to ask to approve your group id. [This][4] is how I did that.

## GPG Keys

The next step is a creation of a gpg key. You need to install some additional utilities, on macos it looks like that
`brew install gpg gpg2`.

Now, tet's create a key.
{% highlight shell %}
gpg --gen-key
{% endhighlight %}

Pick default answers to the first two questions. Then enter two years as an expiration period. And fill out personal information.

Now you can list all your keys:
{% highlight shell %}
gpg2 --list-secret-keys
{% endhighlight %}
Copy a keyid of your key and distribute it as follows:

{% highlight shell %}
gpg2 --keyserver hkp://pool.sks-keyservers.net --send-keys [keyid]
{% endhighlight %}
A bit more details on this step are [here][5].

## Gradle script

Now we can touch the project. Here are changes which should be made to your `build.gradle`:

{% highlight shell %}
...
apply plugin: 'maven'
apply plugin: 'signing'

task sourcesJar(type: Jar) {
    classifier = 'sources'
    from sourceSets.main.allSource
}

task javadocJar(type: Jar) {
    classifier = 'javadoc'
    from scaladoc
}

artifacts {
    archives javadocJar, sourcesJar
}

signing {
    sign configurations.archives
}

uploadArchives {
    repositories {
        mavenDeployer {
            beforeDeployment { MavenDeployment deployment -> signing.signPom(deployment) }

            repository(url: "https://oss.sonatype.org/service/local/staging/deploy/maven2/") {
                authentication(userName: ossrhUsername, password: ossrhPassword)
            }

            snapshotRepository(url: "https://oss.sonatype.org/content/repositories/snapshots/") {
                authentication(userName: ossrhUsername, password: ossrhPassword)
            }

            pom.project {
                name 'testcontainers-scala'
                packaging 'jar'
                description 'Scala wrapper for testcontainers-java'
                url 'https://github.com/dimafeng/testcontainers-scala'

                scm {
                    connection 'scm:git:git@github.com:dimafeng/testcontainers-scala.git'
                    developerConnection 'scm:git:git@github.com:dimafeng/testcontainers-scala.git'
                    url 'git@github.com:dimafeng/testcontainers-scala.git'
                }

                licenses {
                    license {
                        name 'The MIT License (MIT)'
                        url 'https://opensource.org/licenses/MIT'
                    }
                }

                developers {
                    developer {
                        name 'Your name'
                        email 'your email'
                    }
                }
            }
        }
    }
}
{% endhighlight %}

This code allows publishing of releases and snapshots to *oss.sonatype.org*. You just need to adjust `pom.project` section to match your project information.

More information on it is [here][6].

## Publishing

It seems it's time to deploy your artifact. Make sure your work with non-snapshot version and then execute:

{% highlight shell %}
./gradlew uploadArchives
{% endhighlight %}

Once it's done, you need to promote your release. To do so, you have to go to [this page][7]. In the table with repositories, you'll see one that looks like your group id but without dots. Go in there and then click 'Close' in the top menu. Refresh the view in a few seconds and go to the 'Activity' tab - if you did everything correctly on the previous steps, you'll see no errors and a button 'Release' will be enabled (it's located to the right of 'Close' button), click on it and that's pretty much it.

Enjoy having your own artifact in maven cental!


[1]: https://github.com/dimafeng/testcontainers-scala
[2]: https://github.com/testcontainers/testcontainers-java
[3]: https://issues.sonatype.org/projects/OSSRH
[4]: https://issues.sonatype.org/browse/OSSRH-23192
[5]: http://central.sonatype.org/pages/working-with-pgp-signatures.html
[6]: http://central.sonatype.org/pages/gradle.html
[7]: https://oss.sonatype.org/#stagingRepositories
