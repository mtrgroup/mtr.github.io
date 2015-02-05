---
layout: post
title:  "Work started, $135 of $500 funded, basic structure of package, mkvirtualenv helper"
date:   2015-02-02 13:47:56
сategory: kickstarter
tags: kickstarter python package working django-mtr-sync firstday django
image: '/img/2015/02/02/structure.png'
---

We are very happy to report that after first day on the Kickstarter, the project has already collected $135! We are very pleased that we have found a like-minded people. Now we started developing the package and added a project template to github.

During the development we will share with you our ideas that we use in our development.
We will communicate our development process that will help you in developing and publishing your own packages.

**Our project is here [https://www.kickstarter.com/projects/1625615835/django-opensource-improved-import-export-package][kickstarter]**

<!--more-->

##Basic project structure

![Project structure]({{ '/img/2015/02/02/structure.png' | prepend: site.baseurl}})

The `mtr_sync` folder is our main package app and the `tests` folder is where test project is located, where we will write tests. There are many ways of testing applications but the most common is to create separate project for standalone app and test it. This approach uses django itself and other large packages.

For convenience we created bash script, `.env` that creates a new virtualenv if it
doesn't exist and installs all packages from requirements.txt. Of course it olud be customizeble.

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

To use it you need to install `virtualenvwrapper` globally by `sudo pip install virtualenvwrapper`
We will post more details about environment configuration soon.

We are starting to converting mockups in to the architecture of app. Then we will create models, views, templates for main features.
Of course, you will be able to read about our progress in the next posts.

Thanks for your support!

[kickstarter]: https://www.kickstarter.com/projects/1625615835/django-opensource-improved-import-export-package