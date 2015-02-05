---
layout: post
title:  "Theming (template prefixing). Lambda settings variables, testing. Fabfile for all annoying tasks"
date:   2015-02-05 17:00:56
сategory: kickstarter
tags: kickstarter python package working django-mtr-sync fabric theming settings django
image: '/img/2015/02/05/tests.png'
---

**Greetings and Salutations!**

**With your help, we were able to incredibly fast achieve our first goal of $500!
We feel that we are on the right track and the sky is the limit!**

In today’s post we will discuss:

- Theming (prefixed template) helper
- Must have tool `fabric` and rutine tasks.
- Small tip about using the correct application configuration to avoid
  forks and reach maximum convenience and customization.

<!--more-->

## Theming

It becomes obvious that we need to accommodate a package for the various admin apps.
We found very simple and elegant way out, using the setting as a prefix (folders) with templates.
That is how it looks like:

{% highlight python %}
def themed(template):
    """Changing template themes by setting THEME_PATH"""

    return os.path.join('mtr', 'sync', THEME_PATH(), template)

def render_to(template, *args, **kwargs):
    """Shortuct for rendering templates,
    creates functions that returns decorator for view"""

    decorator_kwargs = kwargs

    # outer decorator
    def decorator(f):

        # inner decorator
        @wraps(f)
        def wrapper(request, *args, **kwargs):
            response = f(request, *args, **kwargs)
            if isinstance(response, dict):
                new_template = template
                if decorator_kwargs.pop('themed', True):
                    new_template = themed(template)

                return render(request, new_template, **decorator_kwargs)
            else:
                return response
        return wrapper

    return decorator
{% endhighlight %}

In our code we use `os.path.join` for combining `theme folder` and the `path to the template`.
To change theme just set another folder in the `MTR_SYNC_THEME_PATH` and copy the files in new directory.
For convenience, we use a decorator `render_to` which by default uses a `themed` function.

> To pass params in the decorator just write a function
which will return a decorator. (Example, below)

Now in `views` we can simply write `@render_to('template_name.html'):

{% highlight python %}
from django.contrib.admin.views.decorators import staff_member_required

from .models import LogEntry
from .helpers import render_to


@staff_member_required
@render_to('dashboard.html')
def dashboard(request):
    context = {
        'last_imported': LogEntry.import_objects.all()[:10],
        'last_exported': LogEntry.export_objects.all()[:10]
    }

    return context
{% endhighlight %}

## Settings

For convenience and flexibility of standalone application we use `settings.py` inside module

{% highlight python %}
import os

from django.conf import settings

# custom prefix for avoiding name colissions
PREFIX = getattr(settings, 'MTR_SYNC_SETTINGS_PREFIX', 'MTR_SYNC')


def getattr_with_prefix(name, default):
    """Shortcut for getting settings attribute with prefix"""

    return lambda: getattr(settings, '{}_{}'.format(PREFIX, name), default)


def get_buffer_file_path(instance, filename):
    """Generate file path for report"""

    return os.path.join(
        settings.MEDIA_ROOT, 'sync',
        instance.get_action_display().lower(), filename)

# used to generate custom file path
FILE_PATH = getattr_with_prefix('FILE_PATH', get_buffer_file_path)

# theme path
THEME_PATH = getattr_with_prefix('THEME_PATH', 'default')

{% endhighlight %}
In function `getattr_with_prefix` we use `lambda`. This done to distinguish a local(current app settings) and external(third-party) app settings when settings are changed in test you can get a new value, and it allows you. If we simply assign a default value in `getattr_with_prefix` then when you change the settings in the test you would get a previous value instead of a new.

{% highlight python %}
import os

from django.test import TestCase

from mtr.sync.settings import PREFIX, THEME_PATH
from mtr.sync.helpers import themed


class ThemedTest(TestCase):

    def setUp(self):
        self.template_name = 'test_template.html'
        self.theme_name = 'testtheme'

    def test_changing_theme_by_settings(self):
        """Test changing to the new theme and then fallback to default"""

        new_settings = {
            '{}_{}'.format(PREFIX, 'THEME_PATH'): self.theme_name
        }

        new_theme_path = os.path.join(
            'mtr', 'sync', self.theme_name, self.template_name)
        default_theme_path = os.path.join(
            'mtr', 'sync', THEME_PATH(), self.template_name)

        with self.settings(**new_settings):
            self.assertEquals(themed(self.template_name), new_theme_path)

        self.assertEquals(themed(self.template_name), default_theme_path)
{% endhighlight %}

![Tests]({{ '/img/2015/02/05/tests.png' | prepend: site.baseurl}})

## Fabric

We use this tool to optimize our development process. Using a simple API for writing tasks, you save yourself from time consuming routine operations.
For example, we have the following tasks in our project:

{% highlight python %}
from fabric.api import local, task, settings, hide

APPS = ['mtr.sync']


@task
def clear():
    """Delete unnecessary and cached files"""

    local("find . -name '~*'' -or -name '*.pyo' -or -name '*.pyc' "
        "-or -name 'Thubms.db' | xargs -I {} rm -v '{}'")


@task
def test():
    """Test listed apps"""

    with settings(hide('warnings'), warn_only=True):
        test_apps = ' '.join(map(lambda app: '{}.tests'.format(app), APPS))
        local("./manage.py test {} --pattern='*.py'".format(test_apps))


@task
def run():
    """Run server"""

    local("./manage.py runserver")
{% endhighlight %}

### Task descriptions

- **clear**: Sometimes when you rename a file, there is a problem with it's copy. For example a `*.pyc - compiled bytesource` or
`*.pyo - optimized file`, imported instead of the original file. You can have additional files to delete as by appending
`-or -name "*.whatever"` before `| xargs`.

- **test**: For testing all apps, we use simple loop through `APPS` list variable. Functions `hide ('warnings') and warn_only = True` used for
   hidding fabric errors when test fails

- **run**: Just shortcut for manage.py runserver, as alternative for shortcuts you can write simple shell file

Of course, we will add more tasks with `makemessages` and `compilemessages` documentation and package building process.
You can save your time by writing simple task or shortcut for it.

> Unfortunately `fabric` only available for python2, but finishing porting it
to the python3, detail can be found in the issues on [Github][github]

**You can help us on [Kickstarter][kickstarter]! Thanks for your attention!**

[kickstarter]: https://www.kickstarter.com/projects/1625615835/django-opensource-improved-import-export-package
[github]: https://github.com/fabric/fabric/issues/1050