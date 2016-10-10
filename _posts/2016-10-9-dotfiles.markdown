---
layout: devops
title:  "How to manage your dotfiles"
categories: linux, osx, dotfiles, vim
---

A few months ago I wrote a [post with Linux commands cheat sheet][1]. But then I started thinking that I don't need to remember all those commands if I have
a collection of scripts and aliases which I can use and transfer from one machine to another with ease. Moreover, if had such repository I would store scripts which make freshly installed OS X/MacOs a fully configured development environment.
To achieve this goal, people come up with
an idea of storing their dotfiles and other useful scripts in some (semi-)standard way on github.

## What are dotfiles and (semi-)standard way

* **dotfiles** are files whose names start with a dot. In this post, we're talking about dotfiles located in `~/`. Most of them are configurations of different parts of your system.

* **(semi-)standard** is the way these files are stored on github and there's no one standard way to do it. There is a site [dotfiles.github.io][2] where all good practices are collected on. Basically, you need to structure your git repo in a way that is easy to install and easy to update when some changes pushed to upstream.

In most cases, you need to clone the dotfiles repo to `~/dotfiles` and then run an install script. The install script usually backs up old files and creates symbolic links to those from the cloned repo. Besides, a dotfiles repo may contain scripts for installing additional utils and applications and it can make a fresh system a ready-to-go system in minutes.

<div style="text-align: center;">
<img style="width: 400px;" src="/assets/dotfiles.jpg" />
</div>


## Which dotfiles to work with

Now we can loot at some files which you can modify to configure system for your needs. I have [my own dotfiles][3] (they're not quite done yet, but I'm working on it) you can use it as an idea. The intention of having your own dotfiles (and not using existing ones) was to understand how it all works and why you need each line in your configs. Basically, my dotfiles is **a compilation of all popular dotfiles repositories of which parts I found useful with several modifications**.

### install.sh

This is an entry point to the dotfiles repo. Currently, my script is split into three parts: config linking, common parts installation and OS specific application installation. The linking step just creates symbolic links to files from the repo. So you can pull new changes and they automatically will be applied. To make work on my dotfiles more smooth, I use names without dots but create symlinks with dots.

{% highlight bash %}
install.sh link
{% endhighlight %}

Then you can install common application and scripts. Currently, it contains:

* [oh-my-zsh][6] and theme

>Oh-My-Zsh is an open source, community-driven framework for managing your ZSH configuration. It comes bundled with a ton of helpful functions, helpers, plugins, themes, and a few things that make you shout...

* Some python packages

{% highlight bash %}
install.sh install_commons
{% endhighlight %}

The OS specific application installation, in my case, uses brew to install all essential applications. You can review and modify the list of applications in `brew.sh` and then run:

{% highlight bash %}
install.sh install_app_mac
{% endhighlight %}

### .bash_profile

This is the first file to be loaded when a bash shell is starting. Common practice is to have less code here but load all that you need from separate files grouped by purpose. In my case, I load `.aliases` and `.functions`.

### .zshrc

This is an entry point to the zsh configuration that loads all others configurations. Since I'm using **oh-my-zsh** this configuration is from the standard **oh-my-zsh** distribution with a few changes. You can easily go through the file and uncomment lines you need.
In my case, I made the following changes:

* Theme `ZSH_THEME="dracula"` - this is nice looking dracula theme (downloaded during `install.sh install`)
* Plugins `plugins=(docker gradle)` - here you can specify which plugins should be run.

Also, this config script loads `.bash_profile` mentioned above, so all aliases and functions will be working in both **bash** and **zsh**.

### .aliases

All aliases are stored here. The quick overview of several popular repositories gave a basic pattern of this file. Usually, people make their own shortcuts to most used paths in the everyday routine:

{% highlight bash %}
alias d="cd ~/Documents/Dropbox"
alias dl="cd ~/Downloads"
alias dt="cd ~/Desktop"
alias dev="cd ~/Documents/dev"
alias g="git"
alias h="history"
{% endhighlight %}

### .functions

This file contains functions that can be used as regular command line applications. For the convenience, I organized them in the way it could be easily transformed into a documentation.

{% highlight bash %}
function mkd() # Create a new directory and enter it
{
	mkdir -p "$@" && cd "$_";
}
{% endhighlight %}

Then I added an alias `alias dhelp="grep "^function" ~/dotfiles/functions | sed 's/function/+/g' | sed 's/() #/ - /g'"` and now it works as follows:

<pre>
$ dhelp
+ calc -  Simple calculator
+ mkd -  Create a new directory and enter it
+ cdf -  Change working directory to the top-most Finder window location
+ fs -  Determine size of a file or total size of a directory
+ dataurl -  Create a data URL from a file
+ server -  Start an HTTP server from a directory, optionally specifying the port
+ gz -  Compare original and gzipped file size
+ json -  Usage: `json '{"foo":42}'` or `echo '{"foo":42}' | json`
+ digga -  Run `dig` and display the most useful info
+ escape -  UTF-8-encode a string of Unicode symbols
+ unidecode -  Decode \x{ABCD}-style Unicode escape sequences
+ codepoint -  Get a character’s Unicode code point
+ a -  `a` with no arguments opens the current directory in Atom Editor, otherwise opens the given location
+ v -  `v` with no arguments opens the current directory in Vim, otherwise opens the given location
+ o - `o` with no arguments opens the current directory, otherwise opens the given location
+ tre -  `tre` is a shorthand for `tree` with hidden files and color enabled
+ grr -  Git: Revert changes & Remove all untracked
+ gr -  Git: Revert changes in specific file
+ gacp -  Git: add all, commit and push. [-m "comment"]
</pre>

### .gitconfig

If you work with multiple git accounts (e.g. you have github account and git account at your work) here's the trick - if you have the following lines in `.gitconfig`

{% highlight ini %}
[user]
	name = Dmitry Fedosov
	email = "(none)"
{% endhighlight %}

you'll be asked to specify local email for each repo.


### .osx

OS X/MacOs has a built-in command line tool for configuration - defaults. To get an idea which settings to tune, [here's
a script][7] that interactively asks you what to change. You can go through it and copy settings into your config.

### .vimrc

<div style="text-align: center;">
<img src="https://draculatheme.com/assets/img/screenshots/vim.png" />
</div>

I wrote a [post some time ago][4] about **vim** configuration. I haven't changed too much since that time.
I just configured nice looking theme [Dracula][5]. It's installed from Vundle, so you just need to run `:PluginInstall` in your vim.


[1]: /2016/03/27/linux-commands/
[2]: https://dotfiles.github.io/
[3]: https://github.com/dimafeng/dotfiles
[4]: /2015/09/13/vim1/
[5]: https://draculatheme.com/vim/
[6]: http://ohmyz.sh/
[7]: https://gist.github.com/brandonb927/3195465/
