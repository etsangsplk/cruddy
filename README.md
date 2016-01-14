# cruddy

[![Circle CI](https://circleci.com/gh/cloudnative/cruddy.svg?style=svg)](https://circleci.com/gh/cloudnative/cruddy)

A simple CRUD wrapper around Amazon DynamoDB.

## Installation

```
$ pip install cruddy
```

## Getting Started

The first thing to do is to create a CRUD handler for your DynamoDB table.  The
constructor for the CRUD class takes a number of parameters to help configure
the handler for your application.  The full list of parameters are:

* table_name - name of the backing DynamoDB table (required)
* profile_name - name of the AWS credential profile to use when creating the
  boto3 Session
* region_name - name of the AWS region to use when creating the boto3 Session
* prototype - a dictionary that describes the prototypical object stored in
  your table (see below)
* supported_ops - a list of operations supported by the CRUD handler
  (choices are list, get, create, update, delete, query)
* encrypted_attributes - a list of tuples where the first item in the tuple is
  the name of the attribute that should be encrypted and the second
  item in the tuple is the KMS master key ID to use for
  encrypting/decrypting the value.
* debug - if not False this will cause the raw_response to be left
  in the response dictionary

### Prototypes

A prototype is a description of the prototypical item in your table.  It's
kind of like a template for the item.  A prototype can be used to describe what
attributes are in the item, which are required or optional, and the type of
value that is associated with the attribute.  In addition, there are special
values you can use that allow a small range of calculated values in your item.

If you don't specify a prototype, cruddy will store whatever values are in the
item with no validation or insertion of calculated values.

Let's look at a few examples using prototypes.

```
{
  'id': '',
  'created_at': 1,
  'foo': 1
}
```

This prototype says that your item must have an ``id`` attribute whose value is
of type ``str``, a ``created_at`` attribute whose value is of type ``int``, and
a ``foo`` attribute whose value is also an ``int``.  Your item may contain
other items as well (this is not a schema) but it must contain these attribute
name/value pairs.  If the item you pass into the ``create`` method does not
contain these attributes cruddy will create the necessary attributes and will
initialize the value to what ever value you have specified.


#### Calculated Values

The above example assumes that you are going to generate the ``id`` and
``created_at`` values in your application code.  You may, however, prefer to
have cruddy handle that for you.  In that case, you can make use of cruddy's
calcuated value tokens.

```
{
  'id': 'on-create:<uuid>',
  'created_at': 'on-create:<timestamp>'
}
```

Now, when you create a new item you could supply one without an ``id`` or
``created_at`` value and cruddy will calculate these values for you.  If those
attributes already exist in the item, cruddy will not overwrite them.  Note
that the calulated values are specified as ``on-create``.  This is called a
``trigger`` and indicates when the calculation will be performed.

If you wanted to also have a timestamp to indicate when an item has been
modified (i.e. created or updated) you could do this.

```
{
  'id': 'on-create:<uuid>',
  'created_at': 'on-create:<timestamp>',
  'modified_at': 'on-update:<timestamp>'
}
```

The currently supported calculated value types are:

* **<uuid>** to generate a string representation of a Type4 UUID
* **<timestamp>** to generate an integer timestamp generated by
  ``int(time.time()*1000)``

The currently supported triggers for calculated values are:

* **on-create** will be applied when the item is created
* **on-update** will be applied when the item is created or updated

### Configuring your CRUD handler

An easy way to configure your CRUD handler is to gather all of the parameters
together in a dictionary and then pass that dictionary to the class
constructor.

```
import cruddy

params = {
    'profile_name': 'foobar',
    'region_name': 'us-west-2',
    'table_name': 'fiebaz',
    'prototype': {'id': '<on-create:uuid>',
                  'created_at': '<on-create:timestamp>',
                  'modified_at': '<on-update:timestamp>'}
}

crud = cruddy.CRUD(**params)
```

Once you have your handler, you can start to use it.

```
item = {'name': 'the dude', 'email': 'the@dude.com', 'twitter': 'thedude'}
response = crud.create(item)
```

The response returned from all CRUD operations is a Python object with the
following attributes.

* **data** is the actual data returned from the CRUD operation (if successful)
* **status** is the status of the response and is either ``success`` or
``error``
* **metadata** is metadata from the underlying DynamoDB API call
* **error_type** will be the type of error, if ``status != 'success'``
* **error_code** will be the code of error, if ``status != 'success'``
* **error_type** will be the full error message, if ``status != 'success'``
* **raw_response** will contain the full response from DynamoDB if the CRUD
handler is in ``debug`` mode.
* **is_successful** a simple short-cut, equivalent to ``status == 'success'``

You can convert the CRUDResponse object into a standard Python dictionary using
the ``flatten`` method

```
>>> response = crud.create(...)
>>> response.flatten()
{'data': {'created_at': 1452109758363,
  'name': 'the dude',
  'email': 'the@dude.com',
  'twitter': 'thedude',
  'id': 'a6ac0fd7-cdde-4170-a1a9-30e139c44897',
  'modified_at': 1452109758363},
 'error_code': None,
 'error_message': None,
 'error_type': None,
 'metadata': {'HTTPStatusCode': 200,
  'RequestId': 'LBBFLMIAVOKR8LOTK7SRGFO4Q3VV4KQNSO5AEMVJF66Q9ASUAAJG'},
 'raw_response': None,
 'status': 'success'}
 >>>
 ```
 
## CRUD operations

The CRUD object supports the following operations.  Note that depending on the
value of the ``supported_operations`` parameter passed to the constructor, some
of these methods may return an ``UnsupportedOperation`` error type.

### list()

Returns a list of items in the database.  Encrypted attributes are not
decrypted when listing items.

### get(*id*, *decrypt=False*)

Returns the item corresponding to ``id``.  If the ``decrypt`` param is not
False (the default) any encrypted attributes in the item will be decrypted
before the item is returned.  If not, the encrypted attributes will contain the
encrypted value.

### create(*item*)

Creates a new item.  You pass in an item containing initial values.  Any
attribute names defined in ``prototype`` that are missing from the item will be
added using the default value defined in ``prototype``.

### update(*item*)

Updates the item based on the current values of the dictionary passed in.

### delete(*id*)

Deletes the item corresponding to ``id``.

## Beyond CRUD

The following operations extend beyond the basic CRUD functions but are
included because of they are quite useful.

### query(*query*)

Cruddy provides a limited but useful interface to query GSI indexes in DynamoDB
with the following limitations (hopefully some of these will be expanded or
eliminated in the future.

* The GSI must be configured with a only HASH and not a RANGE.
* The only operation supported in the query is equality

To use the ``query`` operation you must pass in a query string of this form:

    <attribute_name>=<value>

As stated above, the only operation currently supported is equality (=) but
other operations will be added over time.  Also, the ``attribute_name`` must be
an attribute which is configured as the ``HASH`` of a GSI in the DynamoDB
table.  If all of the above conditions are met, the ``query`` operation will
return a list (possibly empty) of all items matching the query and the
``status`` of the response will be ``success``.  Otherwise, the ``status`` will
be ``error`` and the ``error_type`` and ``error_message`` will provide further
information about the error.

### increment_counter(*item*, *counter_name*, [*increment*])

Atomically increments a counter attribute in the item.  You must specify the
name of the attribute as ``counter_name`` and, optionally, the ``increment``
which defaults to ``1``.

## Using the handler interface

In addition to the methods described above, cruddy also provides a generic
handler interface.  This is mainly useful when you want to wrap a cruddy
handler in a Lambda function and then call that Lambda function to access the
CRUD capabilities.

To call the handler, you simply put all necessary parameters into a Python
dictionary and then call the handler with that dict.

```
params = {
    'operation': 'create',
    'item': {'foo': 'bar', 'fie': 'baz'}
}
response = crud.handler(**params)
```

So, you could define a Lambda function like this:

```
import logging
import json

import cruddy

LOG = logging.getLogger()
LOG.setLevel(logging.INFO)

config = json.load(open('config.json'))
crud = cruddy.CRUD(**config)


def handler(event, context):
    LOG.info(event)
    response = crud.handler(**event)
    return response.flatten()
```

Where ``config.json`` looks like this:

```
{
    "region_name": "us-west-2",
    "table_name": "foobar",
    "prototype": {"id": "<on-create:uuid>",
                  "created_at": "<on-create:timestamp>",
                  "modified_at": "<on-update:timestamp>",
                  "foo": 1,
                  "bar": ""},
}
```

If you uploaded this function (and config file) to AWS Lambda you could then
invoke the handler like this.

```
import json

import boto3

session = boto3.Session()
lambda_client = session.client('lambda')

params = {'operation': 'create', 'item': {'fie': 'baz'}}
response = lambda_client.invoke(
    FunctionName='myfunction',
    InvocationType='RequestResponse',
    Payload=json.dumps(params))
cruddy_response = json.load(response['Payload'])
```

The variable ``cruddy_response`` would now contain the response structure
returned by cruddy, flattened into a Python dictionary.

## The cruddy CLI

cruddy also offers a CLI that allows you to access your DynamoDB table or
Lambda-based handler via a simple command line interface.  It supports all
operations supported by cruddy.

### Using the cruddy CLI with a DynamoDB table

To use cruddy to directly interact with a DynamoDB table, you need to place the
configuration information for your cruddy handler in a JSON file.  So, from our
previous example if we created a file called ``fiebaz.json`` like this:

```
{
    "profile_name": "foobar",
    "region_name": "us-west-2",
    "table_name": "fiebaz",
    "prototype": {"id": "<on-create:uuid>",
                  "created_at": "<on-create:timestamp>",
                  "modified_at": "<on-update:timestamp>"}
}
```

We could then reference this when using the cruddy CLI:

```
$ cruddy --config-file fiebaz.json list
[
  {<a listing of all items in fiebaz>}
  ...
]
```

Use the ``--help`` for more information on how to use the cruddy CLI.

### Using the cruddy CLI with a Lambda handler

All of the operations of the CLI work exactly the same whether you are using it
with a DynamoDB table directly or through a Lambda controller.  The only
difference is that rather than referencing a config file containing info about
the table and other parameters needed to create the cruddy CRUD handler, you
simply tell the CLI about the Lambda function.

```
$ cruddy --lambda-fn fiebaz list
[
  {<a listing of all items in fiebaz>}
  ...
]
```

where ``fiebaz`` is the name of your Lambda handler.

