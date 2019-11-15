## Concepts

In **μMongo** 3 worlds are considered:

![Data Flow](img/index/data_flow.png)

### Client world

This is the data from outside μMongo, it can be JSON dict from your web framework
(i.g. ``request.get_json()`` with [flask](http://flask.pocoo.org/){:target="_blank"} or
``json.loads(request.raw_post_data)`` in [django](https://www.djangoproject.com/){:target="_blank"}
or it could be regular Python dict with Python-typed data

**JSON dict example**

```Python
{"str_field": "hello world", "int_field": 42, "date_field": "2015-01-01T00:00:00Z"}
```

**Python dict example**

```Python
{"str_field": "hello world", "int_field": 42, "date_field": datetime(2015, 1, 1)}
```

To be integrated into μMongo, those data need to be unserialized. Same thing
to leave μMongo they need to be serialized (under the hood
μMongo uses [marshmallow](http://marshmallow.readthedocs.org/){:target="_blank"} schema).
The unserialization operation is done automatically when instantiating a
:class:`umongo.Document`. The serialization is done when calling
:meth:`umongo.Document.dump` on a document instance.

### Object Oriented world

So what's good about :class:`umongo.Document` ? Well it allows you to work
with your data as Objects and to guarantee there validity against a model.

First let's define a document with few :mod:`umongo.fields`

```Python
@instance.register
class Dog(Document):
    name = fields.StrField(required=True)
    breed = fields.StrField(default="Mongrel")
    birthday = fields.DateTimeField()
```

First don't pay attention to the ``@instance.register``, this is for later ;-)

Note that each field can be customized with special attributes like
``required`` (which is pretty self-explanatory) or ``default`` (if the
field is missing during unserialization it will take this value).

Now we can play back and forth between OO and client worlds

```Python
>>> client_data = {'name': 'Odwin', 'birthday': '2001-09-22T00:00:00Z'}
>>> odwin = Dog(**client_data)
>>> odwin.breed
"Mongrel"

>>> odwin.birthday
datetime.datetime(2001, 9, 22, 0, 0)

>>> odwin.breed = "Labrador"
>>> odwin.dump()
{'birthday': '2001-09-22T00:00:00+00:00', 'breed': 'Labrador', 'name': 'Odwin'}
```

!!! info
      You can access the data as attribute (i.g. ``odwin.name``) or as item (i.g. ``odwin['name']``).
      The latter is specially useful if one of your field names clashes
      with :class:`umongo.Document`'s attributes.

OO world enforces model validation for each modification

```Python
>>> odwin.bad_field = 42
[...]
AttributeError: bad_field

>>> odwin.birthday = "not_a_date"
[...]
ValidationError: "Not a valid datetime."
```
!!! info
    Just one exception: ``required`` attribute is validated at insertion time, we'll talk about that later.

Object orientation means inheritance, of course you can do that

```Python
@instance.register
class Animal(Document):
    breed = fields.StrField()
    birthday = fields.DateTimeField()

    class Meta:
        allow_inheritance = True
        abstract = True

@instance.register
class Dog(Animal):
    name = fields.StrField(required=True)

@instance.register
class Duck(Animal):
    pass
```

Note the ``Meta`` subclass, it is used (along with inherited Meta classes from
parent documents) to configure the document class, you can access this final
config through the ``opts`` attribute.

Here we use this to allow ``Animal`` to be inheritable and to make it abstract.

```Python
>>> Animal.opts
<DocumentOpts(instance=<umongo.frameworks.PyMongoInstance object at 0x7efe7daa9320>, template=<Document template class '__main__.Animal'>, abstract=True, allow_inheritance=True, collection_name=None, is_child=False, base_schema_cls=<class 'umongo.schema.Schema'>, indexes=[], offspring={<Implementation class '__main__.Duck'>, <Implementation class '__main__.Dog'>})>

>>> Dog.opts
<DocumentOpts(instance=<umongo.frameworks.PyMongoInstance object at 0x7efe7daa9320>, template=<Document template class '__main__.Dog'>, abstract=False, allow_inheritance=False, collection_name=dog, is_child=False, base_schema_cls=<class 'umongo.schema.Schema'>, indexes=[], offspring=set())>

>>> class NotAllowedSubDog(Dog): pass
[...]
DocumentDefinitionError: Document <class '__main__.Dog'> doesn't allow inheritance

>>> Animal(breed="Mutant")
[...]
AbstractDocumentError: Cannot instantiate an abstract Document
```

### Mongo world

What the point of a MongoDB ODM without MongoDB ? So here it is !

Mongo world consist of data returned in a format comprehensible by a MongoDB
driver ([pymongo](https://api.mongodb.org/python/current/){target=_blank} for instance).

```Python
>>> odwin.to_mongo()
{'birthday': datetime.datetime(2001, 9, 22, 0, 0), 'name': 'Odwin'}
```

Well it our case the data haven't change much (if any !). Let's consider something more complex:

```Python
@instance.register
class Dog(Document):
    name = fields.StrField(attribute='_id')
```

Here we decided to use the name of the dog as our ``_id`` key, but for
readability we keep it as ``name`` inside our document.

```Python
>>> odwin = Dog(name='Odwin')
>>> odwin.dump()
{'name': 'Odwin'}

>>> odwin.to_mongo()
{'_id': 'Odwin'}

>>> Dog.build_from_mongo({'_id': 'Scruffy'}).dump()
{'name': 'Scruffy'}
```

!!! note
    If no field refers to ``_id`` in the document, a dump-only field ``id``
    will be automatically added:

        >>> class AutoId(Document):
        ...     pass
        >>> AutoId.find_one()
        <object Document __main__.AutoId({'id': ObjectId('5714b9a61d41c8feb01222c8')})>
        
But what about if we what to retrieve the ``_id`` field whatever it name is ?
No problem, use the ``pk`` property:

```Python
>>> odwin.pk
'Odwin'

>>> Duck().pk
None
```

Ok so now we got our data in a way we can insert it to MongoDB through our favorite driver.
In fact most of the time you don't need to use ``to_mongo`` directly.
Instead you can directly ask the document to ``commit`` it changes in database:

```Python
>>> odwin = Dog(name='Odwin', breed='Labrador')
>>> odwin.commit()
```

You get also access to Object Oriented version of your driver methods:

```Python

>>> Dog.find()
<umongo.dal.pymongo.WrappedCursor object at 0x7f169851ba68>

>>> next(Dog.find())
<object Document __main__.Dog({'id': 'Odwin', 'breed': 'Labrador'})>

>>> Dog.find_one({'_id': 'Odwin'})
<object Document __main__.Dog({'id': 'Odwin', 'breed': 'Labrador'})>
```

You can also access the collection used by the document at any time
(for example to do more low-level operations):

```Python
>>> Dog.collection
Collection(Database(MongoClient(host=['localhost:27017'], document_class=dict, tz_aware=False, connect=True), 'test'), 'dog')
```

!!! note
    By default the collection to use is the snake-cased version of the
    document's name (e.g. ``Dog`` => ``dog``, ``HTTPError`` => ``http_error``).
    However, you can configure (remember the ``Meta`` class ?) the collection
    to use for a document with the ``collection_name`` meta attribute.
