---
layout: post
title: We have Funded! Import action video. Using test mixins. Work progress
date:   2015-02-20 13:47:56
сategory: kickstarter
tags: kickstarter python package working django-mtr-sync firstday django
image: '/img/2015/02/20/video.png'
---

**Amazing! We have funded 180%!**

First of all we would like to express our graditude:

- **Heather Kohler** - [twitter.com/hkohler](twitter.com/hkohler)
- **Matt Harley** - [mattharley.com](mattharley.com)

for biggest pledges and an ideas for project!

|Many thanks to:||||
|-|-|-|-|
|James H Thompson|Ryan McDevitt|Timothy Fay|Kerry Channing|Patrick Taylor|
|Craig Anderson|Anselm Lingnau|David Burke|Jardar Skage Øhrn|Jack Eccleshall|
|Chris Adams|Blanc|Arezqui Belaid|Tom Christie|Nicholas WL Koh|
|Daniel Greenfeld|urijah|Rogelio|André Luiz Lopes dos Santos|Michael Herman|
|Mikhail Ushanov||||
||||

##Import feature

Finaly we have first working basic import feature. Let's see it in action in our new video below:

<p>
<div class="video-wrapper">
<iframe src="https://www.youtube.com/embed/JOgQCFB-leg" frameborder='0' allowfullscreen></iframe>
</div>
</p>

<!--more-->

Whe we implementing `Processor` for `xlsx` we find issue in `openpyxl` optimized reader mode. We fixed it just by switching to `2.1` branch in project repo. Our `requirements.txt` now contains:

{% highlight text %}
django>=1.6
pytz
git+https://github.com/pashinin/fabric.git@6783d2f7ad7a449c97cd1ee0df9a2dbc29422fce#egg=fabric
ipython
celery
hg+https://bitbucket.org/openpyxl/openpyxl@2.1#egg=openpyxl
xlwt-future
xlrd
odfpy
six
{% endhighlight %}

**Now you can use fork of `fabric` with python3** using git repo `git+https://github.com/pashinin/fabric.git@6783d2f7ad7a449c97cd1ee0df9a2dbc29422fce#egg=fabric`

## Test mixin tutorial

You have 2 or many test case classes with similar functionality and you want to utilize it. Of course you can subclass `TestCase` but this cause to run it seperatly, to solve this we use simple mixin.

Source of mixin in `tests.py`:
{% highlight python %}
from __future__ import unicode_literals

import os
import datetime

from mtr.sync.api import Manager
from mtr.sync.models import Settings


class ProcessorTestMixin(object):
    MODEL = None
    PROCESSOR = None
    MODEL_COUNT = 50

    def setUp(self):
        self.model = self.MODEL

        self.manager = Manager()
        self.manager.register(self.PROCESSOR)

        self.instance = self.model.objects.create(name='test instance',
            surname='test surname', gender='M', security_level=10)
        self.instance.populate(self.MODEL_COUNT)
        self.queryset = self.model.objects.all()

        self.settings = Settings.objects.create(
            action=Settings.EXPORT,
            processor=self.PROCESSOR.__name__, worksheet='test',
            main_model='{}.{}'.format(
                self.model.__module__, self.model.__name__),
            include_header=False)

        self.fields = self.settings.create_default_fields()

    def check_file_existence_and_delete(self, report):
        """Delete report file"""

        self.assertIsNone(os.remove(report.buffer_file.path))

    def check_report_success(self, delete=True):
        """Create report from settings and assert it's successful"""

        report = self.manager.export_data(self.settings)

        # report generated
        self.assertEqual(report.status, report.SUCCESS)
        self.assertEqual(report.action, report.EXPORT)
        self.assertIsInstance(report.completed_at, datetime.datetime)

        # file saved
        self.assertTrue(os.path.exists(report.buffer_file.path))
        self.assertTrue(os.path.getsize(report.buffer_file.path) > 0)

        if delete:
            self.check_file_existence_and_delete(report)

        return report

    def open_report(self, report):
        """Open data file and return worksheet or other data source"""

        raise NotImplementedError

    def test_create_export_file_and_report_generation(self):
        self.check_report_success()

    def test_export_dimension_settings(self):
        self.settings.start_row = 25
        self.settings.start_col = 10
        self.settings.end_col = 12
        self.settings.end_row = 250

        report = self.check_report_success(delete=False)
        worksheet = self.open_report(report)

        start_row = self.settings.start_row - 1
        end_row = self.settings.end_row - 1
        start_col = self.settings.start_col - 1

        fields_limit = self.settings.end_col - self.settings.start_col + 1
        self.fields = self.fields[:fields_limit]

        if self.queryset.count() < end_row:
            end_row = self.queryset.count() + start_row - 1

        self.check_values(
            worksheet, self.queryset.first(), start_row, start_col)
        self.check_values(
            worksheet, self.queryset.get(pk=end_row - start_row + 1),
            end_row, start_col)

        self.check_file_existence_and_delete(report)

    # TODO: add import test with dimensions
{% endhighlight %}

- We simply abstract `open_report` and `check_values` methods for other processors.
- `MODEL`, `PROCESSOR`, `MODEL_COUNT` params for flexibility

Now we can simply write processors tests:

{% highlight python %}
from __future__ import unicode_literals

from django.test import TestCase

from mtr.sync.tests import ProcessorTestMixin
from mtr.sync.api.processors import xls, xlsx

from ...models import Person


class XlsProcessorTest(ProcessorTestMixin, TestCase):
    MODEL = Person
    PROCESSOR = xls.XlsProcessor

    def open_report(self, report):
        workbook = xls.xlrd.open_workbook(report.buffer_file.path)
        worksheet = workbook.sheet_by_name(self.settings.worksheet)

        return worksheet

    def check_values(self, worksheet, instance, row, index_prepend=0):
        for index, field in enumerate(self.fields):
            value = getattr(instance, field.attribute)
            sheet_value = worksheet.cell_value(row, index+index_prepend)

            self.assertEqual(value, sheet_value)


class XlsxProcessorTest(ProcessorTestMixin, TestCase):
    MODEL = Person
    PROCESSOR = xlsx.XlsxProcessor

    def open_report(self, report):
        workbook = xlsx.openpyxl.load_workbook(
            report.buffer_file.path, use_iterators=True)
        worksheet = workbook.get_sheet_by_name(self.settings.worksheet)
        return worksheet

    def get_row_values(self, row, current_row):
        for index, value in enumerate(current_row):
            if index == row and row:
                return value

    def check_values(self, worksheet, instance, row, index_prepend=0):
        row_values = self.get_row_values(row, worksheet.iter_rows())

        for index, field in enumerate(self.fields):
            value = getattr(instance, field.attribute)
            sheet_value = row_values[index + index_prepend].value

            self.assertEqual(value, sheet_value)
{% endhighlight %}

## Work progress

What we have done:
- Auto registering models for settings with filter
- Created basic import and export actions with integrated xls, xlsx formats.
- Added simple tests and python 3 support
- Implemented Processor API
- Created testing app with model (will be extened to more complex with all supported datatypes)
- Added auto field settings
- Custom queryset for settings

What we have sheduled:
- Support for custom fields, filters
- On create, update, delete handlers in import
- Add error reporting
- Auto `select_related` `prefetch_related`

Thanks for your support!

Project on github: [https://github.com/mtrgroup/django-mtr-sync](https://github.com/mtrgroup/django-mtr-sync)