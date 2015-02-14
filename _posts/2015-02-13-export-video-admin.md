---
layout: post
title:  "Django export video prototype. How it works? - API in diagrams. Admin tricks"
date:   2015-02-13 17:44:03
сategory: kickstarter
tags: kickstarter python package working django-mtr-sync diagram api django
image: '/img/2015/02/14/howitworks.png'
---

**Only 24 hours to go! Thank you for your support!**

We have first working prototype with a simple API and export function.
For testing we use `Person` model with prepopulated data. All actions you can see on the video below:

<p>
<div class="video-wrapper">
<iframe src='https://www.youtube.com/embed/L4ti1qERSLs' frameborder='0' allowfullscreen></iframe>
</div>
</p>

It's easy to export even from admin app, we use it only for fallback and testing.

Project github repo: [https://github.com/mtrgroup/django-mtr-sync][github]

<!--more-->

## How it works?

![How it works]({{ '/img/2015/02/14/howitworks.png' | prepend: site.baseurl }})

In this diagram you can see how export action works.
On the first step `Settings` passed to the manager using `Celery` and then all prepared data
passed to `Processor`. To understand the process displayed in diagram you can read description of blocks bellow:

- `make_from_params` - `Settings` model class method to create settings new instance from `**kwargs` or
if it has `id` key fetched from database. We use this method to reduce data size when transfer to the Celery task
- `export_data` used in Manager to construct queryset and filter queryset for `Processor`
- `prepare_queryset` - used in settings to create queryset from `main_model`
- `prepare_data` - maps fields from queryset to the fields in settings
- `make_processor` used to create `Processor` instance from the settings
- `export_data` used in Processor to write data to file and generate Report about operation using signals

## Admin tricks

First of all **we don't recommend** to use `oreder_with_respect_to` in `Meta` when you want to manually set order in admin.
Because an `_order` field is not regular field and you don't want to spend your time writing a wrapper for it in the form or a similar functionality.

## First trick. How to pass parent object to inline?

To do this just rewrite `get_formset` method to redefine `formfield_callback`. Then you can pop `parent object` in
`formfield_for_dbfield`. Then create a field by calling `parent class real method` and now you can do whatever you want
with the current field.

{% highlight python %}
class FieldInline(admin.TabularInline):
    model = Field
    extra = 0

    def get_formset(self, request, obj=None, **kwargs):
        """Pass parent object to inline form"""

        kwargs['formfield_callback'] = partial(
            self.formfield_for_dbfield, request=request, obj=obj)
        return super(FieldInline, self).get_formset(request, obj, **kwargs)

    def formfield_for_dbfield(self, db_field, **kwargs):
        """Replace inline attribute field to selectbox with choices"""

        parent = kwargs.pop('obj')

        field = super(
            FieldInline, self).formfield_for_dbfield(db_field, **kwargs)

        if db_field.name == 'attribute':
            field = forms.ChoiceField(
                label=field.label,
                choices=parent.model_attributes())

        return field
{% endhighlight %}

## Second trick. How to generate choices based on current object attribute?

We use custom `ModelForm` and `self.field['field']._queryset` to get
current settings attribute with using `self.initial` data. When `settings` value was set we can populate
`attribute` with dynamic choices.

{% highlight python %}
class FieldForm(forms.ModelForm):
    def __init__(self, *args, **kwargs):
        """Replace default attribute field to selectbox with choices"""

        super(FieldForm, self).__init__(*args, **kwargs)

        settings = self.initial.get('settings', None)
        if settings:
            for av_settings in self.fields['settings']._queryset:
                if av_settings.id == settings:
                    settings = av_settings
                    break

            self.fields['attribute'] = forms.ChoiceField(
                label=self.fields['attribute'].label,
                choices=settings.model_attributes())

    class Meta:
        exclude = []
        model = Field
{% endhighlight %}

## Third trick. How to hide inline objects when parent object isn't created yet?
Just rewrite `get_inline_instances` method with checking if object exists and return list of inlines.

{% highlight python %}
class SettingsAdmin(admin.ModelAdmin):
    list_display = (
        'name', 'action', 'main_model',
        'processor', 'created_at', 'updated_at'
    )
    date_hierarchy = 'created_at'
    inlines = (FieldInline,)
    actions = ['run']

    def get_inline_instances(self, request, obj=None):
        """Show inlines only in saved models"""

        if obj:
            inlines = super(
                SettingsAdmin, self).get_inline_instances(request, obj)
            return inlines
        else:
            return []

    def run(self, request, queryset):
        for settings in queryset:
            export_data.apply_async(args=[{'id': settings.id}])

            self.message_user(
                request,
                _('mtr.sync:Data synchronization started in background.'))

    run.short_description = _('mtr.sync:Sync data')
{% endhighlight %}

###Any questions? Write it on [Kickstarter][kickstarter] or in comments below!

###Thanks for your attention!

[kickstarter]: https://www.kickstarter.com/projects/1625615835/django-opensource-improved-import-export-package
[github]: https://github.com/mtrgroup/django-mtr-sync