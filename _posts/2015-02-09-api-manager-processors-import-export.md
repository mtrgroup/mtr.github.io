---
layout: post
title:  "Processor and Manager API for different formats. Using admin app for constructing settings. New models and fab tasks"
date:   2015-02-09 13:44:03
сategory: kickstarter
tags: kickstarter python package working django-mtr-sync processorapi django
image: '/img/2015/02/09/admin.png'
---

The main goal of the package is to create an easy-to-use admin interface for users and convenient
API for the programmers. We decided to implement Processor API for the different formats, so you can
just plug-in your own format and use it. In this post we will discuss:

- `Manager` class to work with `processors`
- More `fabric` tasks
- New app models `Settings`, `Field`, `Filter` and how they related each-other
- Django standart admin integration
- Celery configuration

Project github repo: [https://github.com/mtrgroup/django-mtr-sync][github]

##You can help us on [Kickstarter][kickstarter]!

<!--more-->

## Processors

To manage different data formats we've created a simple `Manager` class to register and unregister `processors`.
To register processor we use decorator `register(self, cls)` that adds `cls` to `processors`
variable of `manager` instance. Also `import_processors` method used to import all processors from
`IMPORT_FROM` settings variable to avoid thread locals.

{% highlight python %}
class Manager(object):
    """Manager for data processors"""

    def __init__(self):
        self.processors = OrderedDict()

    @classmethod
    def create(cls):
        """Create api manager and import processors"""

        manager = cls()
        manager.import_processors()
        return manager

    def register(self, cls):
        """Decorator to append new processor"""

        if self.has_processor(cls):
            raise ProcessorExists('Processor already exists')
        self.processors[cls.__name__] = cls

        return cls

    def unregister(self, cls):
        """Decorator to pop processor"""

        self.processors.pop(cls.__name__, None)

        return cls

    def has_processor(self, cls):
        """Check if processor already exists"""

        return True if self.processors.get(cls.__name__, False) else False

    def import_processors(self):
        """Import modules within IMPORT_FROM paths"""

        for module in IMPORT_FROM():
            try:
                __import__(module)
            except ImportError:
                print('Invalid module {}, unable to import'.format(module))

manager = Manager.create()
{% endhighlight %}

`Processor` base class is not completed yet, but the main idea is to set `formats` variable and
implement `import_data`, `export_data` methods to incapsulate data manipulation and simplify adding new format.

{% highlight python %}
class Processor(object):
    """Base implementation of import and export operations"""

    formats = OrderedDict()

    def __init__(self, filepath, settings):
        self.filepath = filepath
        self.settings = settings

    def export_data(self, queryset):
        """Export data from queryset to file and return path"""

        raise NotImplementedError

    def import_data(self, data, model):
        """Import data to model and return errors if exists"""

        raise NotImplementedError
{% endhighlight %}

## Admin app
For faster testing of base functionality we will use standard django admin app to construct settings.
In dashboard an other pages we use existing styles and mockup of admin app.

![Simple admin integration]({{ '/img/2015/02/09/admin.png' | prepend: site.baseurl}})

## Extended settings
For constructing `settings form` of import and export pages we created `Settings`, 'Filter', 'Field' models and
use it for saving user specified params. `FilterParams` model used to determine order of filters. `Field` order determined by
django `order_with_respect_to` model settings which creates `_order` field and other goodies.
[More info at django docs][https://docs.djangoproject.com/en/1.7/ref/models/options/#order-with-respect-to]

{% highlight python %}
@python_2_unicode_compatible
class Settings(ActionsMixin):
    """Settings for imported and exported files"""

    name = models.CharField(_('mtr.sync:name'), max_length=100)

    start_column = models.CharField(
        _('mtr.sync:start column'), max_length=10, blank=True)
    start_row = models.PositiveIntegerField(
        _('mtr.sync:start row'), null=True, blank=True)

    end_column = models.CharField(
        _('mtr.sync:end column'), max_length=10, blank=True)
    end_row = models.PositiveIntegerField(
        _('mtr.sync:end row'), null=True, blank=True)

    limit_upload_data = models.BooleanField(
        _('mtr.sync:limit upload data'), default=False)

    main_model = models.CharField(
        _('mtr.sync:main model'), max_length=255)
    main_model_id = models.PositiveIntegerField(
        _('mtr.sync:main model object'), null=True, blank=True)

    created_at = models.DateTimeField(
        _('mtr.sync:created at'), auto_now_add=True)
    updated_at = models.DateTimeField(
        _('mtr.sync:updated at'), auto_now=True)

    processor = models.CharField(_('mtr.sync:processor'), max_length=255)

    class Meta:
        verbose_name = _('mtr.sync:settings')
        verbose_name_plural = _('mtr.sync:settings')

        ordering = ('-id',)

    def __str__(self):
        return self.name

@python_2_unicode_compatible
class Filter(models.Model):
    """Filter data using internal template"""

    name = models.CharField(_('mtr.sync:name'), max_length=255)
    description = models.TextField(
        _('mtr.sync:description'), max_length=20000, null=True, blank=True)
    template = models.TextField(_('mtr.sync:template'), max_length=50000)

    class Meta:
        verbose_name = _('mtr.sync:filter')
        verbose_name_plural = _('mtr.sync:filters')

        ordering = ('-id',)

    def __str__(self):
        return self.name


@python_2_unicode_compatible
class Field(models.Model):
    """Data mapping field for Settings"""

    name = models.CharField(_('mtr.sync:name'), max_length=255)
    model = models.CharField(_('mtr.sync:model'), max_length=255)
    column = models.CharField(_('mtr.sync:column'), max_length=255)

    filters = models.ManyToManyField(Filter, through='FilterParams')

    settings = models.ForeignKey(Settings, verbose_name=_('mtr.sync:settings'))

    def ordered_filters(self):
        related = FilterParams.objects \
            .filter(filter_related__in=self.filters.all()).order_by('_order')
        return map(lambda r: r.filter_related, related)

    class Meta:
        verbose_name = _('mtr.sync:field')
        verbose_name_plural = _('mtr.sync:fields')

        order_with_respect_to = 'settings'

    def __str__(self):
        return self.name


class FilterParams(models.Model):
    _order = models.PositiveIntegerField(default=0)

    filter_related = models.ForeignKey(Filter, related_name='filter_params')
    field_related = models.ForeignKey(Field)

    class Meta:
        ordering = ('_order',)
{% endhighlight %}

Notice that `Settings`, `Field` and `Filter` models are implementation of mockup below

![Simple admin integration]({{ '/img/2015/02/09/mockup.png' | prepend: site.baseurl}})

## Celery

Configuration of `celery` is simple, just create `tasks.py` file in `app` and use `@shared_task` decorator
per task. We don't use `@celery_app.task` decorator because we don't have project and don't know
about project app.

`celery.py` at root of the project (we use it for testing `tests dir`)

{% highlight python %}
from __future__ import absolute_import

import os
import sys

from celery import Celery

from django.conf import settings

# set the default Django settings module for the 'celery' program.
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'app.settings')

# insert parent dir to path
sys.path.insert(
    0, os.path.join(os.path.dirname(__file__), os.path.pardir, os.path.pardir))

app = Celery('app')

# Using a string here means the worker will not have to
# pickle the object when using Windows.
app.config_from_object('django.conf:settings')
app.autodiscover_tasks(lambda: settings.INSTALLED_APPS)

{% endhighlight %}

`tasks.py` at `mtr/sync`

{% highlight python %}

from celery import shared_task


@shared_task
def export_data():
    return True


@shared_task
def import_data():
    return True


@shared_task
def check_periodic_export():
    pass


{% endhighlight %}

In the `settings.py` of project we added `BROCKER_BACKEND = True`, `CELERY_ALWAYS_EAGER = True`, `CELERY_EAGER_PROPAGATES_EXCEPTIONS = True` for using celery without starting broker.
But we think that for the best testing results (*near to production environment*) `Redis` will be a better choice to use.

> **BUG WARNING**: if you don't add celery import in `__init__.py` in your project, `@shared_task` will use
default settings not `celery app` that you create

`__init__.py` at `app` (project) directory

{% highlight python %}

from __future__ import absolute_import

# This will make sure the app is always imported when
# Django starts so that shared_task will use this app.
from .celery import app as celery_app

{% endhighlight %}

## Fabric

To speed up working and reduce typing of commands we make this steps:

- Moved file to root, now every time don't need to `cd` to `tests`
- Removed `os.path.join` for paths and used `lcd()` context function
- Added tasks for: `managing locales`, `celery worker`, `pip installation` and `fast migration`

As you can see it's easier to add params to tasks and create new one.

{% highlight python %}
import os

from fabric.api import local, task, settings, hide, lcd

APPS = ['mtr.sync']
PROJECT_DIR = 'tests'


@task
def clear():
    """Delete unnecessary and cached files"""

    local("find . -name '~*' -or -name '*.pyo' -or -name '*.pyc' "
        "-or -name 'Thubms.db' | xargs -I {} rm -v '{}'")


@task
def test():
    """Test listed apps"""

    with settings(hide('warnings'), warn_only=True):
        test_apps = ' '.join(map(lambda app: '{}.tests'.format(app), APPS))
        with lcd(PROJECT_DIR):
            local("./manage.py test {} --pattern='*.py'".format(test_apps))


@task
def run():
    """Run server"""

    with lcd(PROJECT_DIR):
        local('./manage.py runserver')


@task
def celery():
    """Start celery worker"""

    with lcd(PROJECT_DIR):
        local('celery worker -A app')


@task
def locale(action='make', lang='en'):
    """Make messages, and compile messages for listed apps"""

    if action == 'make':
        for app in APPS:
            with lcd(os.path.join(*app.split('.'))):
                local('django-admin.py makemessages -l {}'.format(lang))
    elif action == 'compile':
        for app in APPS:
            with lcd(os.path.join(*app.split('.'))):
                local('django-admin.py compilemessages -l {}'.format(lang))
    else:
        print('Invalid action: {}, available actions: "make"'
            ', "compile"'.format(action))


@task
def install():
    """Install packages for testing"""

    with lcd(PROJECT_DIR):
        local('pip install -r requirements.txt')


@task
def migrate():
    """Make migrations and migrate"""

    with lcd(PROJECT_DIR):
        local('./manage.py makemigrations')
        local('./manage.py migrate')

{% endhighlight %}

###Any questions? Write it on [Kickstarter][kickstarter] or in comments below!

###Thanks for your attention!

[kickstarter]: https://www.kickstarter.com/projects/1625615835/django-opensource-improved-import-export-package
[github]: https://github.com/mtrgroup/django-mtr-sync