---
layout: post
title: Moving to Vim
category: vim
tags: vim sublimetext tools text-editors
description: Moving from Sublime Text to Vim
feature-img: images/vimrc-background.png
comments: true
---

Just over six months ago, I set myself a challenge to use [Vim](http://www.vim.org/) as my primary text editor for 30 days. I succeeded, and haven't looked back since.

## Introduction

I'm always looking for ways to improve my development environment and tooling. I spend so much time in front of the computer I get satisfaction when that time is spent well. In the past few years I've improved my setup with things like [tmux](https://tmux.github.io/) for managing my window sessions or [Alfred](https://www.alfredapp.com) for quick mouse-free workflows.

With tools like these, you spend a lot less time using the mouse. It made me appreciate how much more productive this is, and wanted to it a step further and try Vim for a while.

## Why Vim?

For the past few years I've been using [Sublime Text](http://www.sublimetext.com/). It's a great editor and I'm pretty quick at navigating and manipulating code in it.  However, I have seen other developers using Vim to impressively tailor code snippets with just a few keys, never once touching the mouse. I also felt I was a good as I was going to get with Sublime Text, apart from discovering a new plug-in I was not going to get much faster. I had reached the ceiling, something I had heard never really happens with Vim.

Another huge benefit of Vim, is that it's available everywhere. If you are on a server and need to edit some config file, you can be pretty sure Vim is available - the same cannot be said for GUI editors like Sublime.


## 30 Days of Vim

I have toyed with Vi and Vim in the past, but, like a lot of people it went like this:

* open the file
* type some text, not realising I'm in 'Normal' mode
* re-writing the garbled text I've just entered
* hit `[command + s]`
* remember the previous command does nothing and finally enter `:wq` - because thats what Google said to do last time...

More recently, I've tried to use Vim as my main editor but after quickly getting frustrated I've always swapped back to Sublime Text.

Having read [many articles](http://www.viemu.com/a-why-vi-vim.html) and seen numerous talks such the one below, I decided I needed to give Vim a proper chance. I planned to use Vim as my primary editor for 30 days, and if I didn't feel it was worth the effort by the end of that I'd stick with what I know and leave Vim be.

<iframe width="560" height="315" src="https://www.youtube.com/embed/SkdrYWhh-8s" frameborder="0" allowfullscreen></iframe>

## How I went about it

Vim definitely takes some time to get used to. It's more of a 'learning wall' than a 'learning curve'.

![Editor Learning Curves]({{ site.baseurl }}/images/vim-learning-curve.jpg)

Thoughbot wrote an [interesting blog post]( https://robots.thoughtbot.com/the-vim-learning-curve-is-a-myth ) saying the vim learning curve is a myth. I disagree with that statement, but I do agree that if you stick at it for a few days it doesn't take long to overcome the curve and become productive.

### Getting the basics down

I started off with a weekend of [vimtutor]( http://linuxcommand.org/man_pages/vimtutor1.html ) and getting some of the basics down with Derek Wyatt's [great screencasts](http://derekwyatt.org/vim/tutorials/novice/#Welcome). If his enthusiasm can't get you excited about Vim, then I don't know what can.

I also started off with an empty `.vimrc` file. I think it's definitely worth your with learning the basics and then extending that knowledge by adding shortcuts and plugins as you go.

### Transition from Sublime Text

Once I had the basics, I was ready to get back to work and do some Rails development in Vim. I started off with some core plugins to get going and make the transition from Sublime Text easier

  * [nerdtree](https://github.com/scrooloose/nerdtree) - NERD tree as a file tree explorer
  * [CtrlP](https://github.com/ctrlpvim/ctrlp.vim) - A fuzzy file finder, one of the things I liked most in Sublime Text
  * [syntastic](https://github.com/scrooloose/syntastic) - For syntax checking
  * [vim-rails](https://github.com/tpope/vim-rails]) - A plugin to make Rails development much easier in Vim
  * [tcomment](https://github.com/tomtom/tcomment_vim) - Easily comment/uncomment blocks of code

### Searching

It took me a little while to get a decent search setup for Vim.

For file searching I went with `Ag`, also known as [the silver searcher](https://github.com/ggreer/the_silver_searcher). And set it up using this [blog post](https://robots.thoughtbot.com/faster-grepping-in-vim) from Thoughtbot. [Ag](https://github.com/ggreer/the_silver_searcher) is a really nice, lightning fast code-searching tool, which is great to have on the command line and even better within Vim.

For in-file searching I used [incsearch](https://github.com/haya14busa/incsearch.vim) which provides nicer visual feedback and defaults than the out-of-the box search in my opinion.

### Macvim

Vim works really well in the terminal, but I switched over to the GUI version, [Macvim](https://github.com/b4winckler/macvim) after a few days. I feel it makes the transition a little easier. Simple things like hitting `[command + s]` to save, and `[command + tab]` to switch between my editor and my console are ingrained into my muscle memory, having them still work with Macvim makes life a little easier.


### Copy and paste

Copy and paste doesn't work like you'd expect it to in a standard editor. Text is 'yanked to' and pasted from registers. I constantly found myself copying some text, then deleting a word and pasting what I expected was the text I copied but instead was the text I had deleted. The deletion over-wrote the value in the default register. I used [vim-easyclip](https://github.com/svermeulen/vim-easyclip) to provide a copy and paste work flow that was more in line with what I was used to.

### Disable Arrow Keys

Navigating with just the keyboard is huge time saver. However, when learning Vim the tendency is often to replace the mouse with arrow keys. However, this is still an anti-pattern. The idea behind Vim's key mappings, are that the keys you use the most should be closest. Navigation is something you're constantly doing so the navigation is done with keys on the home row `h`, `j`, `k` and `l`. If you use the arrow keys, you're moving our hand off the home row and defeating the purpose of having these keys close.

For me this was hard to stop doing, my hand would be hitting the keys before I even realized it. I ended up [disabling the arrow keys](https://github.com/nburkley/dotfiles/commit/65180005e1f43671bd38253a9c85e038c385fbee) to prevent myself continuing to use them. 



## How the 30 days went

<blockquote class="twitter-tweet" lang="en">
  <p lang="en" dir="ltr">
    Finally starting one of my New Years resolutions, use <a href="https://twitter.com/hashtag/vim?src=hash">#vim</a> full-time for a while. I&#39;ve had more productive days...
  </p>&mdash; Niall (@niallburkley) <a href="https://twitter.com/niallburkley/status/588027045206188033">April 14, 2015</a>
</blockquote>

The first few days were slow. Not only was I trying to figure out what code I was writing, but I was trying to figure out _how_ to write it. At times I found it hard to think straight because I didn't know where everything was - not where everything was on screen, but in my head. In Sublime Text I knew how to indent text, swap panes or search and replace with just a few short cuts, in Vim I still had to check cheat sheets or Google. It felt like working at a really cluttered desk.

After a week I started to get in to the swing of things. There were still some things that slowed me down, but when I did things right, the 'vim way', it was really satisfying.

<blockquote class="twitter-tweet" lang="en">
  <p lang="en" dir="ltr">Day 7 of my 30 days of <a href="https://twitter.com/hashtag/vim?src=hash">#vim</a>. Finally getting a balance between <a href="https://twitter.com/hashtag/nerdtree?src=hash">#nerdtree</a> and command-T. My mouse is starting to gather dust.
  </p>&mdash; Niall (@niallburkley) <a href="https://twitter.com/niallburkley/status/590207244475752448">April 20, 2015</a>
</blockquote>

I won't say I as faster working with Vim than Sublime Text after two weeks, but using Vim was no longer slowing down my workflow. And what's more, I was really enjoying it. Day by day I was finding things that were slowing me down and fixing them with a plug-in or a shortcut. I was already convinced Vim was for me.

By the end of the 30 days, I was totally converted. Editing code wasn't just faster and more efficient, it was more fun. I could really customize the editor to work just like I wanted, and have it fit into my development workflow perfectly. It was no longer just another off-the-shelf tool in my work flow, it was a fully customized part of my development.

<blockquote class="twitter-tweet" lang="en">
  <p lang="en" dir="ltr">Day 30 of 30 days of <a href="https://twitter.com/hashtag/vim?src=hash">#vim</a>!&#10;&#10;I&#39;m converted, I was after just 2 weeks. I&#39;m quicker, more efficient, having more fun &amp; have loads more to learn
  </p>&mdash; Niall (@niallburkley) <a href="https://twitter.com/niallburkley/status/600738713941999616">May 19, 2015</a>
</blockquote>


It's interesting to look back and see the highs and lows of the [30 days on twitter](https://twitter.com/niallburkley/timelines/665915070174597120).


## Six months on

I'm still using Vim six months on. I'm pretty sure I'll be using it for a long time to come. I've learnt a lot in the last few months but I know I still have much more to learn. I still do a lot of tweaking of [my dotfiles](https://github.com/nburkley/dotfiles), trying to tune things as I work.

There's some great resources out there to help continuing learning:

* [usevim](http://usevim.com/) is a great blog with lots of nuggets of Vim advice
* [Practical Vim](https://pragprog.com/book/dnvim2/practical-vim-second-edition) is great book on Vim by Dew Neil
* [Vimcasts](http://vimcasts.org/), also by Drew Neil are a great collection of vim screencasts
* [/r/vim](https://www.reddit.com/r/vim), the vim sub-reddit is quite active and has lots of good information

It's also really helpful to take a look at [other people's dotfiles](https://github.com/search?utf8=%E2%9C%93&q=vimrc) to see how they're using Vim, what their setup is and what plugings they're using.

All in all, I'm delighted I switched to Vim. It's nice to log onto a machine and know that my editor of choice will always be there.

<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
