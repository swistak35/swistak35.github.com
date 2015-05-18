---
layout: post
title: Change terminal colorscheme depending on hour of a day
description: One of my colleagues some time ago stated a problem - can we automatically change colorscheme of the terminal based on whether it's day or night? It should solve a problem that we don't like to look at white background in the middle of the night or dark background when sitting in the garden.
---

One of my colleagues some time ago stated a problem - **can we automatically change colorscheme of the terminal based on whether it's day or night?** It should solve a problem that we don't like to look at white background in the middle of the night or dark background when sitting in the garden.

My answer would be: **Yes, but... I have zsh and Konsole (KDE default terminal). My solution depends on both, but I'm pretty sure it could be done also on bash or other shells (fish anyone?). However, I have no clue about other terminal emulators than Konsole. I also use Yakuake terminal emulator, which use the same engine as Konsole. Thus, my solution works for Yakuake too.**

## konsoleprofile command

The first thing is to find themes which we would like to use. In Konsole, we have built-in about a dozen of themes, including Solarized and Solarized Light. I will pick these two - Solarized for night and Solarized Light for day colorscheme.

Firstly we would like to switch a theme at all from our command line. And that's the place we can see the power of Konsole. There's a special command: `konsoleprofile`.
**We can use it to change options of the currently used konsole tab from command line.**
And yes, one of the features is changing currently used colorscheme.

The syntax for changing colorscheme for Solarized theme is:
{% highlight ruby %}
konsoleprofile colors=Solarized
{% endhighlight %}

There are few quirks however. If you have localized Konsole like me, these themes are named "Nasłonecznione" (Solarized) and "Słoneczne światło" (Solarized Light).
**However, only English colorschemes' names work.** How to find how this particular theme is named in English? No idea. I found it by accident, because Yakuake isn't localized so it has English names in settings.

**Second quirk is - don't use spaces.** For example, use:
{% highlight ruby %}
konsoleprofile colors=SolarizedLight
{% endhighlight %}
even if theme is named "Solarized Light". Basically, you have to change name of the theme to camelCase fashion, so "Black on White" colorscheme becomes "BlackOnWhite".

Now we can write simple shell function, which changes colorscheme to Solarized if hour is past 21 or before 6.

{% highlight ruby %}
function switch-theme-night() {
  konsoleprofile colors=Solarized
}

function switch-theme-day() {
  konsoleprofile colors=SolarizedLight
}

function switch_term_colors() {
  if [[ ($(date +%H) -gt 21) || ($(date +%H) -lt 6) ]]
  then
    switch-theme-night
  else
    switch-theme-day
  fi
}
{% endhighlight %}

## zsh hooks

Now, how to use this function to change colorschemes automatically? ZSH have concept called "hooks".
**For example, we can make a "preexec" hook, which will make zsh run some function every time you type a command.** It's "automated enough" for me.

Type this into your `~/.zshrc` file:

{% highlight ruby %}
autoload -U add-zsh-hook
add-zsh-hook preexec switch_term_colors
{% endhighlight %}

And it should be working now!

## tmux

There's one more quirk if you're using tmux. Does it interest you how this `konsoleprofile` communicates with your current terminal?
As far as I have found konsole intercepts commands called in shell and if it's calling konsoleprofile it call some specific code.
In the end `konsoleprofile` code is in `konsole` sources and `konsoleprofile` command you have in your system is just a placeholder (you can check it by `cat /usr/bin/konsoleprofile`).

**This cause some problems, when we have terminal multiplexer on top of our shell.** Thanksfully someone already have found solution for that and posted it on [AskUbuntu.com](http://askubuntu.com/questions/416572/switch-profile-in-konsole-from-command-line).

**In the end, the code (already customized to work in Tmux) I've put in my `~/.zshrc` looks like this:**

{% highlight ruby %}

switch-term-color() {
  arg="${1:-colors=Solarized}"
  if [[ -z "$TMUX" ]]
  then
    konsoleprofile "$arg"
  else
    printf '\033Ptmux;\033\033]50;%s\007\033\\' "$arg"
  fi
}

switch-theme-night() {
  switch-term-color "colors=Solarized"
}

switch-theme-day() {
  switch-term-color "colors=SolarizedLight"
}

function switch_term_colors() {
  if [[ ($(date +%H) -gt 21) || ($(date +%H) -lt 6) ]]
  then
    switch-theme-night
  else
    switch-theme-day
  fi
}

autoload -U add-zsh-hook
add-zsh-hook preexec switch_term_colors

{% endhighlight %}

I hope it'll work for you too.
