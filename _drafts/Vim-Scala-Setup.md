---
title: My Scala setup in Vim
tags: [Vim, Scala, sbt, development]
layout: post
---

It was back in 2013 when I decided to stop using IDEs when doing Scala
development. I have since modified my setup quite a bit to the point where some
co-workers are surprised when I can do a lot of things they can do in IntelliJ
or something else. I've never documented my setup so I decided to finally put on
paper (or rather, html) for other people who might be considering jumping ship.

If you don't care about my life story, [skip down](#my-setup).

# Why did I abandon IDEs?

Now I haven't fully abandoned IDEs, I still use IntelliJ when playing around with
Android development (you have to use and IDE for that) but for everyday
development (in both Scala and Haskell), I've given up on them. The reason are
numerous but they can be better explained with a story, the story of why I HAD
to stop using IntelliJ in order to do my job.

## The evil file

It was back when I used to work for Updater, Inc. Everyone on the team was on
IntelliJ and everything was fine. IntelliJ would freeze up for a second or so
every now and then but that wasn't such a big deal. Everything was fine until
one faithful day I started working on this one file. I don't remember what was
in the file but I do remember that there wasn't any crazy in; just some classes
and some implicits. Yet, for some reason, whenever I opened this file in
IntelliJ, the whole editor would freeze; nothing could be done except force
quit. At first I thought something was up with my PC but turns out the same was
happening to everyone else. To make matters worse this was a file that I would
be working on for the next two weeks or so, so I had to come up with some
solution.

## Vim to the rescue

I already knew how to use Vim at this point in my career (from a previous job
where I had used it for python scripting) so it seemed like the obvious solution;
when editing that evil file, just do it in Vim. This worked great for a while
but it grew annoying having to switch between IntelliJ and Vim constantly. I
started to look for some solutions in Vim to help me: I found CtrlP (which I
have since replaced, more on that later), I found ctags and I found scala-vim.
This worked great, I would edit files in Vim and use sbt for compilation. But it
was still hard since I had to manually search for files and lines that sbt would
complain about; until I found [qbst](https://gist.github.com/aloiscochard/4698501).

With Vim and the tools mentioned I was able to finish the project with an issue,
and then I continued...on to the next project...without switching back. I did
eventually try to switch back and then I found the slowness of everything
(re-indexing, file-search, text-search, etc) to be unbearable; after being
able to jump around the codebase at lightning speed IntelliJ just felt too slow.

Now this isn't a rant against IntelliJ in particular, the reason my Vim
environment was so fast was because I sacrificed a lot of features in order to
gain that speed (a sacrifice that I found worthwhile). If I managed to get my
Vim environment to do the same things as IntelliJ it would probably be just as
sluggish.

After switching to Vim full-time my setup evolved and got more sophisticated
through the years. When I worked at SumAll, Inc (Haskell shop) the same
setup was used with some changes (ghc instead of sbt, hasktags instead of ctags,
etc).

## My setup

But enough history, lets talk about what I do (and you could do) to setup an
efficient Vim environment for Scala.

### vim-scala

This is a no-brainer, Vim doesn't support Scala (syntax, indentation, etc)
by default, so you need a plugin for it. I personally use
[vim-scala](https://github.com/derekwyatt/vim-scala).

### vim-fugitive

Probably one of the best Vim plugins out there; if you are using Vim and git,
then you should have [vim-fugitive](vf). Not only is it a great git wrapper but
it will also help us deal with "project" issues later on.

### ctags

Ctags makes one half of my "IDE". Without ctags you would need to search
your codebase every time you want to jump to a type or class. You OS might
already come with ctags installed, however I recommend [universal-ctags](https://github.com/universal-ctags/ctags).

Vim already understands ctag files so no plugins needed there. However Vim
needs to be told where to find said file. This setting is set by the variable
`ctags` in Vim. To set this variable, simply do `set ctags=...` in your vimrc.

But now we've reached an issue, where do we place our tags file? We could pick
a global location somewhere in $HOME, and that would work, but I prefer [Tim
Pope's approach](put-tim-pope-philosophy-here). With this approach, you place your
tags file in `.git/ctags` and load it from there using [vim-fugitive](vf) (since
it will load files found in `.git/`).


vf: https://github.com/tpope/vim-fugitive
