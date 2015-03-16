
=====================
API apps (model-less)
=====================

Therms
======

For example all Horizon Dashboard is model-less without DB directly connected.

For usual app we need index view which provide base filtering some actions like a creating, editing and enything else.

In typical application we must define these things

Usual REST app
--------------

Minimal

* Model - Table in horizon world
* View - index view
* Data - if we haven't model we must load data from remote host

Optional

* Table
* Forms
* Actions
* Templates

If we build with horizon contrib we need these components

Minimal

* Model in horizon_contrib world
* View - index view
* Data - ``Manager`` in horizon_contrib world

Optional

* Table
* Forms
* Actions
* other stuff


New approach
============

We prefer new way which is different in one aspect. We moved responsibility for load data into Model class. Every object is responsible for his data.
This means Model has manager and this managers load all related data. In many cases we would like to use another manager methods like a create,delete,update,get etc..

Manager
-------

First we define our manager. It can be anything, but must provide one method called ``all`` for index views.

For this example returns only array with one dictionary which presents data from remote API.

Manager can be located anywhere we recommend in the ``managers.py``, but is not golden rule.

.. code-block:: python

    from horizon_contrib.api import Manager
    
    class ProjectManager(Manager):

        def all(self, *args, **kwargs):
            return [{'id': 1, 'project': 'Horizon', 'description': 'Foo'}]

.. note::

    See Manager class in the code, its simple object based on ``ClientBase`` which has ``request`` method.

Usually we do this

.. code-block:: python

    class ProjectManager(Manager):

        def all(self, *args, **kwargs):
            # call GET -> protocol:host:port/api/projects and returns lis of projects
            return self.request('api/projects')


And onther methods for us these methods can be implemented later or not. Depends only what we need.

.. code-block:: python

    class ProjectManager(object):

        def create(self, *args, **kwargs):
            raise NotImplementedError

        def update(self, *args, **kwargs):
            raise NotImplementedError

        def delete(self, *args, **kwargs):
            raise NotImplementedError

        # and common methods
        def order_by(self, *args, **kwargs):
            raise NotImplementedError

        def filter(self, *args, **kwargs):
            raise NotImplementedError

Now define our model, in this case is simple Project.

Model
-----

.. code-block:: python

    from horizon import forms
    from horizon_contrib.api import models
    
    from .managers import ProjectManager

    class Project(models.APIModel):

        id = models.IntegerField("ID", required=False)
        name = models.CharField("ID", required=False)
        description = models.CharField("ID", required=False, widget=forms.widgets.Textarea)

        objects = ProjectManager()  # connect our manager

        def __unicode__(self):
            return str(self.name)

        def __repr__(self):
            return str(self.name)

        class Meta:
            abstract = True
            verbose_name = "Project"
            verbose_name_plural = "Projects"


Benefits
^^^^^^^^

.. code-block:: python

    from .models import Project

    Project.objects.all()

    [{'id': 1, 'project': 'Horizon', 'description': 'Foo'}]

    new_project = Project(**{'name': 'Foo', 'description': 'Bar'})

    new_project.save()

    # raise NotImplementedError from your manager class, becase ``save`` is proxied to him in default state.

    project = Project.objects.get(id=1)
    project.delete()

Managers
--------

For advance working with managers we simple extends our ``ProjectManager``

.. code-block:: python

    class ProjectManager(object):
        ...
        SCOPE = "projects"

        def get(self, request, id):
            return self.request(
                request,
                '/{0}/{1}/'.format(self.SCOPE, id))

.. note::

    We known API base url from ``settings`` and now provide model endpoint. Benefits from this see below.

Complex model usual has many to many or querysets of objects

.. code-block:: python


For m2m fields we can chaining managers

.. code-block:: python

    from horizon import forms
    from horizon_contrib.api import models
    
    from horizon_contrib.api import Manager
    from .managers import ProjectManager
    
    class CategoryManager(Manager):

        SCOPE = 'project/categories'  # for now we haven`t parent manager

    class Project(models.APIModel):

        id = models.IntegerField("ID", required=False)
        name = models.CharField("ID", required=False)
        description = models.CharField("ID", required=False, widget=forms.widgets.Textarea)

        objects = ProjectManager()  # connect our manager
        categories = CategoryManager()

        class Meta:
            abstract = True
            verbose_name = "Project"
            verbose_name_plural = "Projects"

.. code-block:: python

    Project.categories.all()

Horizon world
=============


Table
-----

Define your table for index view

.. code-block:: python

    from horizon_contrib.tables import ModelTable

    class ProjectTable(ModelTable):

        class Meta:

            model_class = Project

View
----

.. code-block:: python

    from horizon_contrib.tables import PaginatedView

    from .tables import ProjectTable

    class IndexView(PaginatedView):

        table_class = ProjectTable


 View call ``table.get_table_data`` which returns ``model_class.objects.all()`` in default state

.. code-block:: python

    class IndexView().get_data()

    [{'id': 1, 'project': 'Horizon', 'description': 'Foo'}]
    