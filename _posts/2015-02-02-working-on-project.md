---
layout: post
title:  "Work started, 135$ of 500$ funded, basic structure of package, mkvirtualenv helper"
date:   2015-02-02 18:47:56
сategory: kickstarter
tags: kickstarter python package working django-mtr-sync oneday
image: '/img/02-02/structure.png'
---

And so after the first not full day on Kickstarter at the time of writing, the project has already collected $135!
We are very pleased that we have found a like-minded people. Now we start developing the package and added a project template to github.

During the development we will share with you my small but useful ideas that we use ourselves, and as you can
follow the stages of development that will give you an idea how to develop and publish your own real packages.

###Basic project structure

![Project structure]({{ '/img/02-02/structure.png' | prepend: site.baseurl}})

So, the `mtr_sync` folder is our main package app and the `tests` folder is where test project is located, where we will write tests. There are many of approaches testing applications but the most comfortable is to create separate project for standalone app and test it. This approach uses django itself and other large packages.

For convenience we created bash script, `.env` that creates new virtualenv if it
don't exist and installs all packages from requirements.txt, of course you can customize it as you wish.

{% highlight bash %}
# activate or create virtualenv for project

VERSION=$1

if [[ -z "$VIRTUAL_ENV" ]]; then

source /usr/local/bin/virtualenvwrapper.sh

fi

if [[ -z "$1" ]]; then

VERSION=3

fi

ENV_NAME="django-mtr-sync-$VERSION"
PYTHON_BIN="/usr/bin/python$VERSION"

if [ -n "$(lsvirtualenv | grep $ENV_NAME)" ]; then
    workon $ENV_NAME
else
    mkvirtualenv $ENV_NAME -p $PYTHON_BIN
    pip install -r requirements.txt
fi
{% endhighlight %}

For using it you need to install `virtualenvwrapper` globally by `sudo pip install virtualenvwrapper`
More details about environment configuration we will post soon.

So now, we can start to convert mockups in to the architecture of app. Then we create models, views, templates for main features.
Of course, you can read all about this in next posts.

Thanks for your support!