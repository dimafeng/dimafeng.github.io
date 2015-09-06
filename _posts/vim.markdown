I've always been curious about how cool programmers use **vim** so efficient. Many screencasts and videos from conferences
show that some programmers prefer vim over real IDEs. I decided to master **vim**, there should be something charming - so many peope can't be wrong. Here is [21 Reasons to Learn Vim][1]. So, let's get it started.

##Vim in 5 sec

The only thing you should now is: hit <kbd>Esc</kbd> and `:q!`

<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr">I&#39;ve been using Vim for about 2 years now, mostly because I can&#39;t figure out how to exit it.</p>&mdash; I Am Devloper (@iamdevloper) <a href="https://twitter.com/iamdevloper/status/435555976687923200">February 17, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

[More info](http://stackoverflow.com/questions/11828270/how-to-exit-the-vim-editor) about it.

##How to start

I started to poke aroud and realized that there is no one source which can describe how to do something like that:

Ok, we're motivated to do something cool. I'm using MacOS and, first of all, we need to update default **vim** 7.3 to 7.4. We could use `brew install vim` but for macos, there is a better way - using macvim. **Macvim** is a latest vim compiled with proper flags. It adds a few benifits like cliapboard sharing. 

{% highlight bash %}
$ brew install macvim
$ alias vim='/usr/local/Cellar/macvim/7.4-74/bin/mvim -v'
{% endhighlight %}

So, all special vim features are controlled by plugin system.
The first step for plugin installation is to pick plugin manager. Vim has bunch of plugin managers, but as I understand there is the most pupular one - [Vundle][2].

The installation procces is pretty easy:

{% highlight bash %}
$ git clone https://github.com/VundleVim/Vundle.vim.git ~/.vim/bundle/Vundle.vim
{% endhighlight %}

Then, you need to create a file `~/.vimrc` with following content:

{% highlight vim %}
set nocompatible              " be iMproved, required
filetype off                  " required

set rtp+=~/.vim/bundle/Vundle.vim
call vundle#begin()
Plugin 'VundleVim/Vundle.vim'

"Plugins go here

call vundle#end()            " required
filetype plugin indent on
{% endhighlight %}

This is basic vim configuration. Then we'll be able to modify it to customize vim view and behaviour to meet our needs.

Now we can add some plugins to extend standard view and functionality. I found an awesome site with a catalog of all available plugins [http://vimawesome.com/][3]. The plugin installation is really simple. You need to copy the string which looks like `Plugin '***/***'` and paste it to `~/.vimrc` bellow the line with text `"Plugins go here`.

Let's start with essentials.

### NERDTree

>The NERD tree allows you to explore your filesystem and to open files and directories. It presents the filesystem to you in the form of a tree which you manipulate with the keyboard and/or mouse. It also allows you to perform simple filesystem operations.

{% highlight vim %}
Plugin 'scrooloose/nerdtree'
{% endhighlight %}

This is essential plugin which hepls to work with project tree.

[NERDTree][img]

To make it more conviniet, you can add `map <F5> :NERDTreeToggle<CR>`

[1]: http://www.jovicailic.org/2014/08/why-vim-21-reasons-to-learn-vim/
[2]: https://github.com/VundleVim/Vundle.vim
[3]: http://vimawesome.com/
