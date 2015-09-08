---
layout: post
title:  "Vim essentials (Part 1)"
categories: vim
---
I've always been curious about how cool programmers use **vim** so efficient. Many screencasts and videos from conferences
show that some programmers prefer vim over real IDEs. I decided to master **vim**, there should be something charming - so many peope can't be wrong. Here is [21 Reasons to Learn Vim][1]. So, let's get it started.

##Vim in 5 sec

The only thing you should now is: hit <kbd>Esc</kbd> and `:q!`

<blockquote class="twitter-tweet" lang="en"><p lang="en" dir="ltr">I&#39;ve been using Vim for about 2 years now, mostly because I can&#39;t figure out how to exit it.</p>&mdash; I Am Devloper (@iamdevloper) <a href="https://twitter.com/iamdevloper/status/435555976687923200">February 17, 2014</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

[More info](http://stackoverflow.com/questions/11828270/how-to-exit-the-vim-editor) about it.

##How to start

I started to poke aroud and realized that there is no one source which can describe how to do something like that:

<p>
<img img="{{ site.url }}/assets/vim/my.png" />
</p>

Ok, we're motivated to do something cool. Even though, vim is a crossplatform solution, this article aims to cover MacOS and iTerm2. I like this stack and use it all time. So, first of all, we need to update default **vim** 7.3 to 7.4. We could use `brew install vim` but for macos, there is a better way - using macvim. **Macvim** is a latest vim compiled with proper flags. It adds a few benifits like system cliapboard support. 

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

This is basic vim configuration. Now we are able to modify it to customize vim view and behaviour to meet our needs. There is a bunch of cool features which you have to turn on in the beginning. To them you just need to add following lines to your `~/.vimrc` file.

* `set mouse=a` allows you to work with vim using your mouse; 
* `set number` shows line numbers;
* `set clipboard=unnamed` helps to work with MacOS cliapborad using `y`;
* A few options to manage indentation:

{% highlight vim %}
set tabstop=4
set expandtab       "Use softtabstop spaces instead of tab characters for indentation
set shiftwidth=4    "Indent by 4 spaces when using >>, <<, == etc.
set softtabstop=4   "Indent by 4 spaces when pressing <TAB>

set autoindent      "Keep indentation from previous line
set smartindent     "Automatically inserts indentation in some cases
set cindent         "Like smartindent, but stricter and more customisable
{% endhighlight %}

## Plugins Installation

Now we can add some plugins to extend standard view and functionality. I found an awesome site with a catalog of all available plugins [http://vimawesome.com/][3]. The plugin installation is really simple. You need to copy the string which looks like `Plugin '***/***'` and paste it to `~/.vimrc` bellow the line with text `"Plugins go here`. After that, you may install all plugins: open vim, and type `:PluginInstall`.

<p>
<img img="{{ site.url }}/assets/vim/plugin-install.gif" />
</p>

Let's start with essentials.

### NERDTree

>The NERD tree allows you to explore your filesystem and to open files and directories. It presents the filesystem to you in the form of a tree which you manipulate with the keyboard and/or mouse. It also allows you to perform simple filesystem operations.

{% highlight vim %}
Plugin 'scrooloose/nerdtree'
{% endhighlight %}

This is essential plugin which hepls to work with project tree.

[Page](https://github.com/scrooloose/nerdtree)

<p>
<img img="{{ site.url }}/assets/vim/nerdtree.gif" />
</p>

**A few tips and trick:**

* To make it more conviniet, you can add `map <F5> :NERDTreeToggle<CR>` to your `~/.vimrc`. It allows you to show NERDTree by hitting <kbd>F5</kbd>;
* Hit `r` to refresh file tree.

### ctrlp.vim

> Full path fuzzy file, buffer, mru, tag, ... finder for Vim.

{% highlight vim %}
Plugin 'kien/ctrlp.vim'
{% endhighlight %}

[Page](https://github.com/kien/ctrlp.vim)

<p>
<img img="{{ site.url }}/assets/vim/crtlp.gif" />
</p>

### Fugitive

This plugin helps you to manage git from vim.

{% highlight vim %}
Plugin 'tpope/vim-fugitive'
{% endhighlight %}

[Page](https://github.com/tpope/vim-fugitive)

### Airline

Airline is a customizible statusbar and tabline. 

<p>
<img src="https://github.com/bling/vim-airline/wiki/screenshots/demo.gif"/>
</p>

The installation process is pretty tricky.
First of all, you need to add the plugin:

{% highlight vim %}
Plugin 'bling/vim-airline'
{% endhighlight %}

Then, add following configuration lines:

{% highlight vim %}
set guifont=Inconsolata\ for\ Powerline:h15
let g:Powerline_symbols = 'fancy'
let g:airline_theme='dark'
let g:airline#extensions#tabline#enabled = 1
let g:airline#extensions#tabline#left_sep = ' '
let g:airline#extensions#tabline#left_alt_sep = '|'
let g:airline_powerline_fonts = 1
{% endhighlight %}

We just did basic configuration of airline. It's almost done, but you'll have font issues. To solve them, you need to checkout [powerline/fonts](https://github.com/powerline/fonts) and run `./install.sh`. The next step is **iTerm2** configuration - go to *Preferences* -> *Profiles* -> *Text* and set **Regular Font** and **Non-ASCII Font** to `Inconsolata-g for Powerline`.

<p>
<img img="{{ site.url }}/assets/vim/iterm-conf.png" />
</p>

## Theme

By default, vim inherits system colours and it doesn't look too awesome. To change it, there are hundreds of themes. I used [vimcolors.com](http://vimcolors.com/) to pick a theme. 

The installation process is simple. Pick a theme and go to its github page. Copy the author name and project name and paste to `~/.vimrc` 

{% highlight vim %}
Plugin 'demorose/up.vim'
{% endhighlight %}

Then, you need to add this line to apply theme:

{% highlight vim %}
colorscheme [theme_name]
{% endhighlight %}

## Summary

In this article, I described all steps which I went through to make a vim view look more than a simple text editor. 
I mentioned only a few plugins out of hundreds, but it's extremely easy to find and install new ones by yourself. The next part of this article is going to collect all
essensial shortcuts and commands which you should know to be productive in vim.

[1]: http://www.jovicailic.org/2014/08/why-vim-21-reasons-to-learn-vim/
[2]: https://github.com/VundleVim/Vundle.vim
[3]: http://vimawesome.com/
