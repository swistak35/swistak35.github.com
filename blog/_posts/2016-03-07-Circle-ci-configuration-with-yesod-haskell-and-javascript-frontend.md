---
layout: post
title: Circle CI configuration with yesod (haskell) backend and javascript frontend
excerpt_separator: <!--more-->
tags: ['dsp2016']
---

For ["Daj się poznać" contest](http://www.maciejaniserowicz.com/daj-sie-poznac/) I've decided to [make a tool](https://github.com/swistak35/symmetrical-chainsaw) which is close to pinboard and pinterest, but with features I was missing. When [I've chosen frameworks which I'll gonna use](https://twitter.com/andrzejkrzywda/status/645903406742245376), and both of them came with some default tests, I've decided to setup continuous integration using CircleCI. This post describes how to setup CircleCI for repository with two apps inside (so automatic CircleCI language recognition doesn't work) and one of them is haskell, which is not very popular and there aren't too much resources how to do it.

<!--more-->

### Javascript frontend

That part was easy. Asuumption is that your javascript application is located in `frontend` directory and the tests are run by running `npm run test` command.

To run our tests, we needed only one simple command in `circle.yml`:

{% highlight yaml %}
test:
  post:
    - cd frontend && npm run test
{% endhighlight %}

If we want `npm run test` command to work, we need firstly to install our dependencies, which is done by this part of `circle.yml` config:

{% highlight yaml %}
dependencies:
  override:
    - cd frontend && npm install
{% endhighlight %}

This should already work, however we don't want to wait each time to install all node dependencies, but rather we'd like to cache them between builds. This can be achieved by following code:

{% highlight yaml %}
dependencies:
  cache_directories:
    - "frontend/node_modules"
{% endhighlight %}

### Haskell backend

That part is more tricky.

In the beginning I've tried to achieve this by running `cabal install` and `cabal test` commands on test container, however I had some problems with installing all necessary dependencies. As on my development machine I am using [stack](https://github.com/commercialhaskell/stack) to manage dependencies (and more), I was curious that maybe it would be easier to install this tool which is on a higher level than cabal, but probably more straightforward to run.

I've found that [someone already tried to do this](https://github.com/philipmw/yesod-website/blob/master/circle.yml) but it looks that without success. Using part of this repository I've managed to install stack on CircleCI without any difficulties:

{% highlight yaml %}
dependencies:
  cache_directories:
    - "~/.stack"

  override:
    - wget -q -O- https://s3.amazonaws.com/download.fpcomplete.com/ubuntu/fpco.key | sudo apt-key add -
    - echo 'deb http://download.fpcomplete.com/ubuntu/precise stable main' | sudo tee /etc/apt/sources.list.d/fpco.list
    - sudo apt-get update && sudo apt-get install stack -y
{% endhighlight %}

As you can see, I've also added `~/.stack` directory to cache haskell packages.

Then, similarly like in javascript frontend, we want to install dependencies:

{% highlight yaml %}
dependencies:
  override:
    - ...
    - cd backend && stack setup && stack build --test --only-dependencies
{% endhighlight %}

That is the part when things went a little bit weird. Initially, I was using only `stack build`, but it looks that stack didn't compile test dependencies. `stack build --test` on the other hand, was building test dependencies, but it was instantly running all the tests. That's why I end up using `stack build --test --only-dependencies` command.

In the end, we want to run the tests:

{% highlight yaml %}
test:
  post:
    - cd backend && PGUSER=ubuntu PGDATABASE=circle_test stack test
{% endhighlight %}

After that change, it almost works.

### Yesod ignoreEnv vs useEnv

When you firstly use yesod, it has `config/settings.yml` file which keeps [information about database](https://github.com/swistak35/symmetrical-chainsaw/blob/443969ab82eba0f6146c99dbe5511e1205abd436/backend/config/settings.yml#L26-L32):

{% highlight yaml %}
database:
  user:     "_env:PGUSER:symmetrical_chainsaw_user"
  password: "_env:PGPASS:scpasswd"
  host:     "_env:PGHOST:localhost"
  port:     "_env:PGPORT:5432"
  database: "_env:PGDATABASE:symmetrical_chainsaw_db"
  poolsize: "_env:PGPOOLSIZE:10"
{% endhighlight %}

You can probably guess that these `_env:PGUSER` strings are there to allow developer to replace this values by using environment variables. For test environment there's different file, `config/test-settings.yml` in which you can override some values from `config/settings.yml`. So, if you want to replace this values for test container, but you also want to have some reasonable default for your development machine, you can end up with following contents of `config/test-settings.yml` file:

{% highlight yaml %}
database:
  database: "_env:PGDATABASE:symmetrical_chainsaw_db_test"
{% endhighlight %}

Then it should work, right? Nope. At least not in automatically generated version by stack yesod website. By default, for some reason, env variables in `config/test-settings.yml` file don't work. You have to change one argument in `test/TestImport.hs` file. By default it's

{% highlight haskell %}
settings <- loadAppSettings
  ["config/test-settings.yml", "config/settings.yml"]
  []
  ignoreEnv
{% endhighlight %}

and you should change it to

{% highlight haskell %}
settings <- loadAppSettings
  ["config/test-settings.yml", "config/settings.yml"]
  []
  useEnv
{% endhighlight %}

You can also check [relevant commmit](https://github.com/swistak35/symmetrical-chainsaw/commit/56fc35c96663bfbb43bd94806d97b837f31938b7#diff-a0bcd3030b500becb7a7408f9cd5aab6) in Symmetrical Chainsaw project.

### Final config

Finally I've end up with following `circle.yml`:

{% highlight yaml %}
machine:
  ghc:
    version: 7.10.2
  node:
    version: 5.1.0

dependencies:
  cache_directories:
    # Frontend app javascript modules
    - "frontend/node_modules"

    # Backend app haskell modules
    - "~/.stack"

  override:
    # Build haskell dependencies
    - wget -q -O- https://s3.amazonaws.com/download.fpcomplete.com/ubuntu/fpco.key | sudo apt-key add -
    - echo 'deb http://download.fpcomplete.com/ubuntu/precise stable main' | sudo tee /etc/apt/sources.list.d/fpco.list
    - sudo apt-get update && sudo apt-get install stack -y
    - cd backend && stack setup && stack build --test --only-dependencies

    # Build javascript dependencies
    - cd frontend && npm install

test:
  post:
    - cd backend && PGUSER=ubuntu PGDATABASE=circle_test stack test
    - cd frontend && npm run test
{% endhighlight %}

After this, my setup of continuous integration was ready to defend me from my mistakes.



