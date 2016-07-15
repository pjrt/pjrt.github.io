---
title: KeyChords in Xmonad
tags: [Xmonad, Keychords, Haskell, Vim]
layout: post
---

As a heavy Vim user, key chords feel pretty natural to me. With the exceptions
of terminals (for some reason vi-mode just doesn't feel natural to me) I always
try to replicate them in all the tools I use: to me chords (ie: a series of key
strokes) are just easier and less stressful than the Emacs way (ie:
multi-modifier-key combos).

XMonad is another tool I use heavily, and up until recently, I've been using it
the Emacs way. After using XMonad for years, I finally decided to see if maybe
there was a way to have chords in XMonad. A Google search lead me to [this][1]
StackOverflow answer which in term lead me to [XMonad.Actions.Submap][2].

Bingo! This looks promising! And the SO answer even gave an example of how
to build a long chord. So with that knowledge, I set to try to build a
function to simplify the ability to build these chords.

Introducing [xmonad-keychords][x-keys-repo], a module to build chords in
XMonad.

### Usage

Using this module is pretty simple, simply import it:

{% highlight haskell %}
import XMonad.Actions.KeyChord (keychords)
{% endhighlight %}

And use it in your key-mappings like so:

{% highlight haskell %}
  , ((modm, xK_semicolon), keychords
      [ ([(0, xK_i), (0, xK_k)], action1)
      , ([(0, xK_i), (0, xK_w)], action2)
      , ([(0, xK_j), (0, xK_j)], action3)
      , ([(0, xK_s), (0, xK_a), (shiftMask, xK_t)], action4)
      ])
{% endhighlight %}

Now I can do `mod-; i k` to invoke `action1` (`w` for `action2`). We could make
it use `Shift` instead of `modm` but then you end up never being able to use
`Shift-;` (aka `:`) for nothing else (Vim included). So `mod-;` is a nice
compromise.

### Installation

I've opened a pull request against [xmonad-contrib][pr], once that's in you
can simply install xmonad-contrib and import the module. In the mean time,
it is kind of annoying to install.

You need to download and copy the `Data` directory [here][path-tree], and the
`XMonad` directory [here][xkey-mod] into your `~/.xmonad/lib` (make the `lib`
directory if you don't have it). Your final `.xmonad` should look like this:

{% highlight bash %}
$ ls ~/.xmonad/lib
   Data/  XMonad/
{% endhighlight %}

Now you can just import the module as above and use it.

[1]: http://stackoverflow.com/a/32028957
[2]: http://xmonad.org/xmonad-docs/xmonad-contrib/XMonad-Actions-Submap.html
[x-keys-repo]: https://github.com/pjrt/xmonad-keychord
[pr]: https://github.com/xmonad/xmonad-contrib/pull/65
