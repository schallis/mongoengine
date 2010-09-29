==================
Defining documents
==================
In MongoDB, a **document** is roughly equivalent to a **row** in an RDBMS. When
working with relational databases, rows are stored in **tables**, which have a
strict **schema** that the rows follow. MongoDB stores documents in
**collections** rather than tables - the principle difference is that no schema 
is enforced at a database level. 

Defining a document's schema
============================
MongoEngine allows you to define schemata for documents as this helps to reduce
coding errors, and allows for utility methods to be defined on fields which may
be present. 

To define a schema for a document, create a class that inherits from
:class:`~mongoengine.Document`. Fields are specified by adding **field
objects** as class attributes to the document class::

    from mongoengine import *
    import datetime
    
    class Page(Document):
        title = StringField(max_length=200, required=True)
        date_modified = DateTimeField(default=datetime.datetime.now)

Fields
======
By default, fields are not required. To make a field mandatory, set the
:attr:`required` keyword argument of a field to ``True``. Fields also may have
validation constraints available (such as :attr:`max_length` in the example
above). Fields may also take default values, which will be used if a value is
not provided. Default values may optionally be a callable, which will be called
to retrieve the value (such as in the above example). The field types available 
are as follows:

* :class:`~mongoengine.StringField`
* :class:`~mongoengine.URLField`
* :class:`~mongoengine.IntField`
* :class:`~mongoengine.FloatField`
* :class:`~mongoengine.DecimalField`
* :class:`~mongoengine.DateTimeField`
* :class:`~mongoengine.ListField`
* :class:`~mongoengine.DictField`
* :class:`~mongoengine.ObjectIdField`
* :class:`~mongoengine.EmbeddedDocumentField`
* :class:`~mongoengine.ReferenceField`
* :class:`~mongoengine.GenericReferenceField`
* :class:`~mongoengine.BooleanField`
* :class:`~mongoengine.GeoLocationField`
* :class:`~mongoengine.FileField`
* :class:`~mongoengine.EmailField`
* :class:`~mongoengine.SortedListField`
* :class:`~mongoengine.BinaryField`

Field arguments
---------------
Each field type can be customized by keyword arguments.  The following keyword 
arguments can be set on all fields:

:attr:`db_field` (Default: None)
    The MongoDB field name.

:attr:`name` (Default: None)
    The mongoengine field name.

:attr:`required` (Default: False)
    If set to True and the field is not set on the document instance, a
    :class:`~mongoengine.base.ValidationError` will be raised when the document is
    validated.

:attr:`default` (Default: None)
    A value to use when no value is set for this field.

:attr:`unique` (Default: False)
    When True, no documents in the collection will have the same value for this
    field.

:attr:`unique_with` (Default: None)
    A field name (or list of field names) that when taken together with this
    field, will not have two documents in the collection with the same value.

:attr:`primary_key` (Default: False)
    When True, use this field as a primary key for the collection.

:attr:`choices` (Default: None)
    An iterable of choices to which the value of this field should be limited.
    

List fields
-----------
MongoDB allows the storage of lists of items. To add a list of items to a
:class:`~mongoengine.Document`, use the :class:`~mongoengine.ListField` field
type. :class:`~mongoengine.ListField` takes another field object as its first
argument, which specifies which type elements may be stored within the list::

    class Page(Document):
        tags = ListField(StringField(max_length=50))

Embedded documents
------------------
MongoDB has the ability to embed documents within other documents. Schemata may
be defined for these embedded documents, just as they may be for regular
documents. To create an embedded document, just define a document as usual, but
inherit from :class:`~mongoengine.EmbeddedDocument` rather than 
:class:`~mongoengine.Document`::

    class Comment(EmbeddedDocument):
        content = StringField()

To embed the document within another document, use the
:class:`~mongoengine.EmbeddedDocumentField` field type, providing the embedded
document class as the first argument::

    class Page(Document):
        comments = ListField(EmbeddedDocumentField(Comment))

    comment1 = Comment('Good work!')
    comment2 = Comment('Nice article!')
    page = Page(comments=[comment1, comment2])

Dictionary Fields
-----------------
Often, an embedded document may be used instead of a dictionary -- generally
this is recommended as dictionaries don't support validation or custom field
types. However, sometimes you will not know the structure of what you want to
store; in this situation a :class:`~mongoengine.DictField` is appropriate::
    
    class SurveyResponse(Document):
        date = DateTimeField()
        user = ReferenceField(User)
        answers = DictField()

    survey_response = SurveyResponse(date=datetime.now(), user=request.user)
    response_form = ResponseForm(request.POST)
    survey_response.answers = response_form.cleaned_data()   
    survey_response.save()

Reference fields
----------------
References may be stored to other documents in the database using the
:class:`~mongoengine.ReferenceField`. Pass in another document class as the
first argument to the constructor, then simply assign document objects to the
field::
    
    class User(Document):
        name = StringField()

    class Page(Document):
        content = StringField()
        author = ReferenceField(User)

    john = User(name="John Smith")
    john.save()

    post = Page(content="Test Page")
    post.author = john
    post.save()

The :class:`User` object is automatically turned into a reference behind the
scenes, and dereferenced when the :class:`Page` object is retrieved.

To add a :class:`~mongoengine.ReferenceField` that references the document
being defined, use the string ``'self'`` in place of the document class as the
argument to :class:`~mongoengine.ReferenceField`'s constructor. To reference a
document that has not yet been defined, use the name of the undefined document
as the constructor's argument::

    class Employee(Document):
        name = StringField()
        boss = ReferenceField('self')
        profile_page = ReferenceField('ProfilePage')

    class ProfilePage(Document):
        content = StringField()

Generic reference fields
''''''''''''''''''''''''
A second kind of reference field also exists,
:class:`~mongoengine.GenericReferenceField`. This allows you to reference any
kind of :class:`~mongoengine.Document`, and hence doesn't take a 
:class:`~mongoengine.Document` subclass as a constructor argument::

    class Link(Document):
        url = StringField()
        
    class Post(Document):
        title = StringField()
        
    class Bookmark(Document):
        bookmark_object = GenericReferenceField()

    link = Link(url='http://hmarr.com/mongoengine/')
    link.save()

    post = Post(title='Using MongoEngine')
    post.save()

    Bookmark(bookmark_object=link).save()
    Bookmark(bookmark_object=post).save()

.. note::
   Using :class:`~mongoengine.GenericReferenceField`\ s is slightly less
   efficient than the standard :class:`~mongoengine.ReferenceField`\ s, so if
   you will only be referencing one document type, prefer the standard 
   :class:`~mongoengine.ReferenceField`.

Uniqueness constraints
----------------------
MongoEngine allows you to specify that a field should be unique across a
collection by providing ``unique=True`` to a :class:`~mongoengine.Field`\ 's
constructor. If you try to save a document that has the same value for a unique
field as a document that is already in the database, a 
:class:`~mongoengine.OperationError` will be raised. You may also specify
multi-field uniqueness constraints by using :attr:`unique_with`, which may be
either a single field name, or a list or tuple of field names::

    class User(Document):
        username = StringField(unique=True)
        first_name = StringField()
        last_name = StringField(unique_with='first_name')

Skipping Document validation on save
------------------------------------
You can also skip the whole document validation process by setting 
``validate=False`` when caling the :meth:`~mongoengine.document.Document.save` 
method::

    class Recipient(Document):
        name = StringField()
        email = EmailField()
    
    recipient = Recipient(name='admin', email='root@localhost')
    recipient.save()               # will raise a ValidationError while
    recipient.save(validate=False) # won't

Document collections
====================
Document classes that inherit **directly** from :class:`~mongoengine.Document`
will have their own **collection** in the database. The name of the collection
is by default the name of the class, coverted to lowercase (so in the example
above, the collection would be called `page`). If you need to change the name
of the collection (e.g. to use MongoEngine with an existing database), then
create a class dictionary attribute called :attr:`meta` on your document, and
set :attr:`collection` to the name of the collection that you want your
document class to use::

    class Page(Document):
        title = StringField(max_length=200, required=True)
        meta = {'collection': 'cmsPage'}

Capped collections
------------------
A :class:`~mongoengine.Document` may use a **Capped Collection** by specifying
:attr:`max_documents` and :attr:`max_size` in the :attr:`meta` dictionary.
:attr:`max_documents` is the maximum number of documents that is allowed to be
stored in the collection, and :attr:`max_size` is the maximum size of the
collection in bytes. If :attr:`max_size` is not specified and
:attr:`max_documents` is, :attr:`max_size` defaults to 10000000 bytes (10MB).
The following example shows a :class:`Log` document that will be limited to 
1000 entries and 2MB of disk space::

    class Log(Document):
        ip_address = StringField()
        meta = {'max_documents': 1000, 'max_size': 2000000}

Indexes
=======
You can specify indexes on collections to make querying faster. This is done
by creating a list of index specifications called :attr:`indexes` in the
:attr:`~mongoengine.Document.meta` dictionary, where an index specification may
either be a single field name, or a tuple containing multiple field names. A
direction may be specified on fields by prefixing the field name with a **+**
or a **-** sign. Note that direction only matters on multi-field indexes. ::

    class Page(Document):
        title = StringField()
        rating = StringField()
        meta = {
            'indexes': ['title', ('title', '-rating')]
        }
        
Ordering
========
A default ordering can be specified for your
:class:`~mongoengine.queryset.QuerySet` using the :attr:`ordering` attribute of
:attr:`~mongoengine.Document.meta`.  Ordering will be applied when the
:class:`~mongoengine.queryset.QuerySet` is created, and can be overridden by
subsequent calls to :meth:`~mongoengine.queryset.QuerySet.order_by`. ::

    from datetime import datetime

    class BlogPost(Document):
        title = StringField()
        published_date = DateTimeField()

        meta = {
            'ordering': ['-published_date']
        }

    blog_post_1 = BlogPost(title="Blog Post #1")
    blog_post_1.published_date = datetime(2010, 1, 5, 0, 0 ,0)

    blog_post_2 = BlogPost(title="Blog Post #2") 
    blog_post_2.published_date = datetime(2010, 1, 6, 0, 0 ,0)

    blog_post_3 = BlogPost(title="Blog Post #3")
    blog_post_3.published_date = datetime(2010, 1, 7, 0, 0 ,0)

    blog_post_1.save()
    blog_post_2.save()
    blog_post_3.save()

    # get the "first" BlogPost using default ordering
    # from BlogPost.meta.ordering
    latest_post = BlogPost.objects.first() 
    assert latest_post.title == "Blog Post #3"

    # override default ordering, order BlogPosts by "published_date"
    first_post = BlogPost.objects.order_by("+published_date").first()
    assert first_post.title == "Blog Post #1"

Document inheritance
====================
To create a specialised type of a :class:`~mongoengine.Document` you have
defined, you may subclass it and add any extra fields or methods you may need.
As this is new class is not a direct subclass of
:class:`~mongoengine.Document`, it will not be stored in its own collection; it
will use the same collection as its superclass uses. This allows for more
convenient and efficient retrieval of related documents::

    # Stored in a collection named 'page'
    class Page(Document):
        title = StringField(max_length=200, required=True)

    # Also stored in the collection named 'page'
    class DatedPage(Page):
        date = DateTimeField()

Working with existing data
--------------------------
To enable correct retrieval of documents involved in this kind of heirarchy,
two extra attributes are stored on each document in the database: :attr:`_cls`
and :attr:`_types`. These are hidden from the user through the MongoEngine
interface, but may not be present if you are trying to use MongoEngine with 
an existing database. For this reason, you may disable this inheritance
mechansim, removing the dependency of :attr:`_cls` and :attr:`_types`, enabling
you to work with existing databases. To disable inheritance on a document
class, set :attr:`allow_inheritance` to ``False`` in the :attr:`meta`
dictionary::

    # Will work with data in an existing collection named 'cmsPage'
    class Page(Document):
        title = StringField(max_length=200, required=True)
        meta = {
            'collection': 'cmsPage',
            'allow_inheritance': False,
        }
