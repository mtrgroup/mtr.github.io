---
layout: post
title: "Big update! Work progress, new architecture design, roadmap to release"
date:   2015-07-2 15:47:56
сategory: django-mtr-sync
tags: kickstarter python package working django-mtr-sync roadmap progress django
image: '/img/2015/07/02/diagram.png'
---

## What's going on
After mouthes of continuous work, testing and improving package we make a new conception of data processing. The main question is - "Why you not release it yet?" the answer is - "Due to critical architecture changes and unpolished must have features invented in development process.

Docs, docs, docs! The main problem of project for now is lack of quality docs. To solve this problem we will do an `docs weeks` when commits will be only docs, no additional functionality (bug fixes only).

## Work progress, new features implemented

### Inline integration
This is one of the coolest and new features in *market*, and flexibility of settings let you do what-ever you want, without writing bunch of repetitive code.

inline pic

### Action context, before, after, error handlers
Another cool feature is context of action and handlers, for example you need to save data from previous row, or save data from all rows to variable and then do what you want, may be download or sql update for better perfomance. Also if you want to send email notification if error happened you just add error handler or subscribe to standard django signal.

handlers code example

### Transaction management on demand
If don't need transactions - just don't enable it. The best example of action without transaction is to send object it to async worker, for example celery, or rq.

example of transaction management

### Settings sequences for hiding configuration from manager
For real data sync you will need about 3 or 4 action to run, for example create data, download images and recalculate field in some model. To do this you just create settings sequences, to hide settings and simplify data synchronization, also for your managers.


### Custom datasets from different sources


### Simple admin import, export with ordering included


## Where is my dashboard app?
As you can see there are a lot of work around core functionality, after API stabilization we will implement nice dashboard with Bootstrap for easy customization. Of course this app will be nice to use in user cabinet or else where where you need to handle data from user.

## Utilization of functionality
In `mtr.sync` project structure you can find `helpers.py` files, they will be moved to separate package called `mtr.utils`. We think this package will include all annoying django shortcuts, mixins, helpers. There are a lot of similar packages but they don't have flexibility that we want, for example, auto construction of model fields fabric, so simple *write less do more*.

## New architecture

The first step in synchronization was table processing, but when you need to import or export xml, json data API will have a huge differences of base `Processor` class so for now you just register cutsom dataset. We decided to move from raw `list` values to `dicts` with nested structures. It's a more flexible way to manipulate data and process it with action handlers. Also there are many different data sources not only files, so we must unlink file manipulation from processing.

For better understanding you can see diagram below

## Work progress
Realy nice feature is to include progress status when you dealing with long running action

## Roadmap



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
- Integration with standard django admin app
- Supports: Django 1.7 Python 2.7, 3.3+ (Django 1.6 will be added soon)

Thanks for your support!

Project on github: [https://github.com/mtrgroup/django-mtr-sync](https://github.com/mtrgroup/django-mtr-sync)
