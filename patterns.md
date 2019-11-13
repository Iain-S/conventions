# Architecture patterns

Django:

- [Serialize template context](#serialize-template-context)

Application:

- [Layered approach](#layered-approach)
- [Domain file structure conventions](#domain-file-structure-conventions)

## Django

### Serialize template context

Don't pass any "rich" objects into the template context. Instead, serialize
domain objects to Python primitives beforehand.

#### Why?

- Having rich objects in the template is an enabler for undesirable behaviour:

  1. It encourages adding application-logic-rich methods to models, just so that
     functionality is available in templates (see [Why your Django models are fat](https://codeinthehole.com/lists/why-your-models-are-fat/)). 
     This is particularly prevalent for list views.

  2. It makes it easy to trigger hundreds of SQL queries for a single HTTP
     request, especially in a list view where the template loops over a
     `Queryset` and calls methods on each model instance. Serializing in the
     view ensures all SQL queries are performed before template rendering.

- It makes it easy to convert a Django HTML view into a JSON API view as the
  view is already serializing domain objects into JSON -- you just need to remove
  the template-rendering part. This dovetails in with our long-term strategy of
  embracing more React apps backed by JSON APIs for user interfaces.

- It makes it easier to cache data as all the database querying is done in one
  place.

- It makes it easier to compute global or queryset-level variables in one place.

#### How? 

Use `Serializer` classes from the Django-Rest-Framework to serialize rich
objects into dictionaries before rendering templates.

Example for a list-view using an embedded serializer.

```python
from django.views import generic
from rest_framework import serializers

class Teams(generic.ListView):
    template_name = "teams.html"
    model = models.Team
    context_object_name = "teams"

    def get_context_data(self, **kwargs):
        context = super().get_context_data(**kwargs)

        # Serialize the queryset and remove object_list from the ctx so the 
        # Queryset instance is not included in the template context.
        context[self.context_object_name] = self.get_serialized_queryset(context.pop('object_list'))

        return context

    def get_serialized_queryset(self, queryset):
        return self.TemplateSerializer(instance=queryset, many=True).data

    class TemplateSerializer(serializers.Serializer):
        # Obj attributes just need same name
        pk = serializers.IntegerField()
        name = serializers.CharField()

        # Read fields off related objects using `source`
        leader_pk = serializers.IntegerField(source='leader.pk')
        leader_name = serializers.CharField(source='leader.get_full_name')

        # Call a model method using `source`
        num_users = serializers.CharField(source='get_num_users')

        # Call a method on this serializer (looks for method prefixed `get_`)
        call_weight = serializers.SerializerMethodField()

        # Use format=None to get datetime instances instead of strings
        created_at = serializers.DateTimeField(format=None)

        def get_call_weight(self, obj):
            pass
```

You don't have to embed the serializer class - reusable serializers can also be
used.

When serializing a collection of different domain objects (eg for a detail or
template view), pass them to the serializer as a dict:

```python
from django.views import generic
from rest_framework import serializers
from . import forms

class SwitchBrand(generic.FormView):
    template_name = "switch-brand.html"
    form_class = forms.Confirm

    def dispatch(self, request, *args, **kwargs):
        self.account = shortcuts.get_object_or_404(models.Account, number=kwargs['account_number'])
        self.target_brand = shortcuts.get_object_or_404(data_models.Brand, code=kwargs['brand_code'])
        return super().dispatch(request, *args, **kwargs)

    class _TemplateContext(serializers.Serializer):
        account_number = serializers.CharField(source='account.number')
        current_brand_name = serializers.CharField(source='account.portfolio.brand.name')
        target_brand_name = serializers.CharField(source='target_brand.name')
        electricity_agreements = serializers.SerializerMethodField()
        gas_agreements = serializers.SerializerMethodField()

        def get_electricity_agreements(self, obj):
            agreements = obj['account'].get_active_electricity_agreements()
            return [
                {
                    'mpan': agreement.meter_point.mpan,
                    'tariff_code': agreement.tariff.code,
                } for agreement in agreements
            ]

        def get_gas_agreements(self, obj):
            agreements = obj['account'].get_active_gas_agreements()
            return [
                {
                    'mprn': agreement.meter_point.mprn,
                    'tariff_code': agreement.tariff.code,
                } for agreement in agreements
            ]

    def get_context_data(self, **kwargs):
        ctx = super().get_context_data(**kwargs)

        # Pass the account and target brand to the serializer in a dict.
        extra = self._TemplateContext(
            instance=dict(account=self.account, target_brand=self.target_brand)).data
        ctx.update(extra)

        return ctx

    def form_valid(self, form):
        ...
```
The key point for using `Serializer`s is that the data being serialized must be
passed as a single object. Hence when there's a collection of different objects,
we pass a dictionary containing them all.


## Application

### Layered approach

An application should be split into architectural layers, each with their own role, 
each ignorant of the layers that call into them.

Here are our current layers, listed in order with the outer-most layer first.

- `interfaces` - where functionality for websites, command-line applications,
  Celery tasks live. This covers anywhere where some external event (eg a HTTP
  request) triggers a query or an action. No application logic should live in
  this layer, just the translation of the interface event into a application- or
  domain-layer call.

- `application` - where the "use cases" of the application live. Use-cases are
  the actions that the application supports (eg "submit a meter reading",
  "register a new account", "process a move-out"). This layer should
  contain functionality for orchestrating each use-case journey. It can call into the domain
  layer for re-usable domain functionality that doesn't belong to one particular
  use-case.

- `domain` - where re-useable business logic lives. Functionality in the domain layer
  should be agnostic of which application use-case is calling it. 

- `data` - where data models (eg Django models) live. Models should not contain
  any business logic and should be kept very thin.

Related reading:

- [How to structure Django projects](https://www.jamesbeith.co.uk/blog/how-to-structure-django-projects/) by James Beith

### Domain layer file-structure conventions

In a layered approach, the fattest layer will be the domain layer. To ensure
it remains discoverable (ie it's easy to for developers to find functionality
they are looking for), prefer these conventions.

- Package functionality by domain category/subcategory;
- House read-only functionality in a module/package called `queries`.
- House write functionality in a module/package called `operations`.

For example:

```
octoenergy/
    domain/
        $category/
            $subcategory/
                queries.py
                operations.py
                ...
```

Or, for when there's a lot of queries/operations to house:

```
octoenergy/
    domain/
        $category/
            $subcategory/
                queries/
                    __init__.py  # import all "public" objects into here
                    topic1.py
                    topic2.py
                    ...
                operations/
                    __init__.py  # import all "public" objects into here
                    topic1.py
                    topic2.py
                    ...
```

This means a developer whose code needs to ask a question about the
`$subcategory` part of the domain (eg "where did this account last submit a
meter-reading?") knows where to look to find an answer.

This isn't a strict requirement - if there's a better naming convention for your
part of the domain then use it. 

Think of these modules as the _public_ API of the `$subcategory` of the domain.

### Application layer file-structure conventions

The application layer houses orchestration logic for application use-cases. This
involves providing a single entry-point for interface layers to call. 

```
octoenergy/
    application/
        usecases/
            $category/
                $usecase_name/
                    __init__.py  # import all "public" objects into here
                    _component1.py
                    _component2.py
                    ...
```

Eg:

```
octoenergy/
    application/
        usecases/
            comms/
                annual_statements/
                    __init__.py    # import all "public" objects into here
                    _trigger.py    # responsible for spawning Celery tasks
                    _documents.py  # responsible for building PDF documents
                    ...
```
