---
layout: post
title: After
date:   2015-03-11 21:47:56
сategory: kickstarter
tags: kickstarter python package working django-mtr-sync firstday django
image: '/img/2015/02/20/video.png'
---

After 5 weeks of work we can introduce working package. We freezed main features and started testing.

## What package ca
- Import (only creating), export data
- Processor API for supporting other formats
- Uses Celery for background tasks and for processing large volumes of data
- Creates reports about importing and exporting operations
- Auto discovering models for use in package
- Data range settings (start, end cells in table), for example, if you need to import data where there is a header with logo or any other unnecessary information
- Field settings:
  - maps model fields with data fields
  - adds and removes fields (ability to skip fields on import)
  - related models(fields, supports only ForeignKey) import-export by choosing main model
- Supports: CSV(native python3), XLS (using: xlwt-future, xlrd), XLSX (using: openpyxl optimized writer, reader mode, for fast processing of large volumes of data)
- Saves import, export settings for the processing of data from various sources and for simplicity
- Integration with standart django admin app
- Supports: Django 1.7 Python 2.7, 3.3+ (Django 1.6 will be added soon)

Thanks for your support!

Project on github: [https://github.com/mtrgroup/django-mtr-sync](https://github.com/mtrgroup/django-mtr-sync)