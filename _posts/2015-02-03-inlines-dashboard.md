---
layout: post
title:  "Django inline import and export explanation. Creating main model and dashboard page"
date:   2015-02-03 20:47:56
сategory: kickstarter
tags: kickstarter python package working django-mtr-sync
image: '/img/2015/02/03/structure.png'
---

Project already includes partial inline integration as "Mapping fields feature" but we need to add search field in import and export settings pages to set id of object (parent object). In mockup, ​you can see "Choose main model to import" it used to determine main model and search field for choosing concreete object, shortcut for this will be link from inlines.



> Big thanks to **Russell Keith-Magee, DSF President** and member of the Django core team for helping us to clarify django license compliance requirements.

> Also huge thanks and kudos to **Heather Kohler** for $250! It's awesome!

So other models and fields will be automatically added to this model and other "Choose model" select boxes will show list of models that belongs to main model, so you now can import or export all data to inline or related models (OneToOne, ManyToMany, ForeignKey).

For convenience we will add in `stacked` and `tabular` inlines link to quick import or export so in the settings form
main model and search object form will be automatically filled.

![Inline mode]({{ '/img/2015/02/03/inline.png' | prepend: site.baseurl}})

## Work in progress

For now, we created basic main model and empty dashboard page wich integrates in standart admin page.
If you wish to create your own Django standalone app or refactor existing project, all usefull things we will place
in separate post, for now we have only information about how we doing it live. Thanks for your patience!

###What we have done

- Created simple main model `LogEntry` and managers, for reporting about operations
  {% highlight python %}
@python_2_unicode_compatible
class LogEntry(models.Model):
"""Stores reports about imported and exported operations and link to files
"""

ERROR = 0
RUNNING = 1
SUCCESS = 2

STATUS_CHOICES = (
    (ERROR, _('mtr.sync:Error')),
    (RUNNING, _('mtr.sync:Running')),
    (SUCCESS, _('mtr.sync:Success'))
)

EXPORT = 0
IMPORT = 1

ACTION_CHOICES = (
    (EXPORT, _('mtr.sync:Export')),
    (IMPORT, _('mtr.sync:Import'))
)

buffer_file = models.FileField(
    _('mtr.sync:file'), upload_to=FILE_PATH)
status = models.PositiveSmallIntegerField(
    _('mtr.sync:status'), choices=STATUS_CHOICES)
action = models.PositiveSmallIntegerField(
    _('mtr.sync:action'), choices=ACTION_CHOICES, db_index=True)

started_at = models.DateTimeField(
    _('mtr.sync:started at'), auto_now_add=True)
completed_at = models.DateTimeField(
    _('mtr.sync:completed at'), null=True, blank=True)
updated_at = models.DateTimeField(
    _('mtr.sync:updated at'), auto_now=True)

export_objects = ExportManager()
import_objects = ImportManager()
running_objects = RunningManager()

class Meta:
    verbose_name = _('mtr.sync:log entry')
    verbose_name_plural = _('mtr.sync:log entries')

    get_latest_by = '-started_at'

def __str__(self):
    return '{} - {}'.format(self.started_at, self.get_status_display())

  {% endhighlight %}
- Using prefixed gettext names we prevent collisions in translations `path.to.app:text to translate` and in english
translation just delete prefix to translate, in future updates we will write simple script for it
- Export and Import managers for query shortcuts
- `get_absolute_url` for downloading imported or exported file
- Standalone settings with `getattr_with_prefix` and `SYNC_SETTINGS_PREFIX` to avoid settings name collisions
- Settings for custom path generation, for example if you want to create relation with `LogEntry` and generate name
using reverse relation model attribute for filepath
- Simple model integration into `admin.site`
- Added `dashboard.html` template and `dashboard` view with context data and `staff_member_required`
  {% highlight python %}
@staff_member_required
def dashboard(request):
    context = {
        'last_imported': LogEntry.import_objects.all()[:10],
        'last_exported': LogEntry.export_objects.all()[:10]
    }

    return render(request, 'mtr/sync/dashboard.html', context)
  {% endhighlight %}

###Simple admin integration
![Empty dashboard page]({{ '/img/2015/02/03/admin.png' | prepend: site.baseurl}})

###What will be done in next days

- Integrate `South` migrations for `Django 1.6`
- Add `Fabic` file for rutine tasks such as `makemessages`, `runserver`, `clean *.pyo files` and others things
- List log entries in dashboard and write integration test for it
- Add template and view for **import uploading**
- Design error messages handling for user-friendly experience
- Create `Celery` basic configuration and test simple task

**If you have any questions just write to us and we answer as soon as possible!**

**We are on [Kickstarter][kickstarter]! Thanks for your support!**

[kickstarter]: https://www.kickstarter.com/projects/1625615835/django-opensource-improved-import-export-package