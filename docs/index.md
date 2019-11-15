<h1 align="center">
    <strong>μMongo: sync/async ODM</strong>
</h1>
<p align="center">
<a href="https://pypi.python.org/pypi/umongo" target="_blank">
    <img src="https://img.shields.io/pypi/v/umongo.svg" alt="Package version">
</a>
<a href="https://travis-ci.org/Scille/umongo" target="_blank">
    <img src="https://img.shields.io/travis/Scille/umongo/master.svg" alt="Build Status">
</a>
<a href="http://umongo.readthedocs.org/en/latest/?badge=latest" target="_blank">
    <img src="https://readthedocs.org/projects/umongo/badge/?version=latest" alt="Documentation Status">
</a>
<a href="https://coveralls.io/github/Scille/umongo?branch=master" target="_blank">
    <img src="https://coveralls.io/repos/github/Scille/umongo/badge.svg?branch=master" alt="Code coverage">
</a>
</p>

---

μMongo is a Python MongoDB ODM. Its inception comes from two needs:

1. The lack of async ODM
2. The difficulty in doing document (un)serialization with existing ODMs.

From this point, μMongo made a few design choices:

- Stay close to the standards MongoDB driver to keep the same API when possible,
  i.e., use ``find({"field": "value"})`` like usual but retrieve your data nicely OO wrapped!
- Work with multiple drivers ([PyMongo](https://api.mongodb.org/python/current/){:target="_blank"}, [TxMongo](https://txmongo.readthedocs.org/en/latest/){:target="_blank"}, [motor_asyncio](https://motor.readthedocs.org/en/stable/){:target="_blank"} and [mongomock](https://github.com/vmalloc/mongomock){:target="_blank"} for the moment)
- Tight integration with [Marshmallow](http://marshmallow.readthedocs.org){:target="_blank"} serialization library to easily
  dump and load your data with the outside world
- i18n integration to localize validation error messages
- Free software: MIT license
- Test with 90%+ coverage ;-)

## Quick example

```Python
from datetime import datetime
from pymongo import MongoClient
from umongo import Instance, Document, fields, validate

db = MongoClient().test
instance = Instance(db)

@instance.register
class User(Document):
    email = fields.EmailField(required=True, unique=True)
    birthday = fields.DateTimeField(validate=validate.Range(min=datetime(1900, 1, 1)))
    friends = fields.ListField(fields.ReferenceField("User"))

    class Meta:
        collection = db.user

# Make sure that unique indexes are created
User.ensure_indexes()

goku = User(email='goku@sayen.com', birthday=datetime(1984, 11, 20))
goku.commit()
vegeta = User(email='vegeta@over9000.com', friends=[goku])
vegeta.commit()

vegeta.friends
# <object umongo.data_objects.List([<object umongo.dal.pymongo.PyMongoReference(document=User, pk=ObjectId('5717568613adf27be6363f78'))>])>

vegeta.dump()
# {id': '570ddb311d41c89cabceeddc', 'email': 'vegeta@over9000.com', friends': ['570ddb2a1d41c89cabceeddb']}

User.find_one({"email": 'goku@sayen.com'})
# <object Document __main__.User({'id': ObjectId('570ddb2a1d41c89cabceeddb'), 'friends': <object umongo.data_objects.List([])>,
#                                 'email': 'goku@sayen.com', 'birthday': datetime.datetime(1984, 11, 20, 0, 0)})>

```

## Get it now

```bash
$ pip install umongo           # This installs umongo with pymongo
$ pip install my-mongo-driver  # Other MongoDB drivers must be installed manually
```

Or to get it along with the MongoDB driver you're planing to use::

```bash
$ pip install umongo[motor]
$ pip install umongo[txmongo]
$ pip install umongo[mongomock]
```
