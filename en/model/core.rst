************
Model Basics
************

.. php:class:: Model

    Model class improves how you interact with the structured data. Extend
    this class (or :php:class:`SQL_Model`) to define the structure of your
    own models, then use object instances to interact with individual records.

Introduction
============

In Software Computing most of the time you deal with structured data, like
this table below:

+----+--------+---------+--------+
| id | name   | surname | gender |
+====+========+=========+========+
| 1  | John   | Smith   | m      |
+----+--------+---------+--------+
| 2  | Betty  | Shark   | f      |
+----+--------+---------+--------+
| 3  | Alfred | Hitch   | m      |
+----+--------+---------+--------+

Data of this table can be stored in SQL or Non-SQL engine, 
however some of the common characteristics are pertained e.g.:

- Data is stored as individual records (row or schema)
- All the records have identical or nearly similar structure
- Each record has unique identifier (id)
- Each column has type

Different data engines would introduce a different ways to interract, for example
if you wish change Betty's surname to "Hitch", you would do it differently::

    SQL: update person set surname='Hitch' where id=1;

    MongoDB: ....

Concept of models proposes a common format for any data source, which looks
like this::

    $person->load(1)->set('surname', 'Hitch')->save();


Defining the Model
==================
Similarly to how you need to create a database structure before you store data
in SQL, you would also need to define a model in Agile Toolkit before you can
use it.

.. note:: While there are automated ways to convert model into SQL and back, I
    recommend you to initially do this manually. Later in this documentation
    I'll give my reasons as to why.

Model in Agile Toolkit is object of a "Model" class. To create your own model
you need to create your own class by extending Model::

    class Model_Person extends Model {
        public $table='person';
        function init() {
            parent::init();

            $this->addField('name');
            $this->addField('email');

            $this->setSource('MongoDB');
        }
    }

After you do that, you can add 'Model_Person' as you normally do it with
any other object in Agile Toolkit::

    $person = $this->add('Model_Person');

Note that you also need to properly set the $table property on your model.

.. php:attr:: table

    Contains name of table, session key, collection or file, depending on
    data controller you are using.

Core Concepts
=============

Before continuing with explanation of all the other features and capabilities of the
model in Agile Toolkit, you would need to understand some of the core concepts
and terminology used by the framework:


Record Loading
--------------
If you have seen ORM in other frameworks, you might be familiar with the
principle of havign a separate objects for each record. Agile Toolkit uses a
different principle of record loading.

A single Model Object may have record "loaded" inside itself at any given time.
The record data can then be "unloaded" and the same object can be re-used
for another record.

.. php:method:: tryLoad

    Ask the controller data to load the model by a given $id

.. php:method:: load

    Like tryLoad method but if the record not found, an exception is thrown

.. php:method:: unload

    Forget loaded data

.. php:method:: loaded

    Returns true if the model is loaded

Here are a few examples of loading and unloading data::

    $person = $this->add('Model_Person');

    echo $person->loaded();  // false, not loaded


    $person->load(1);
    echo $person->loaded();  // true now

    $person->load(2);     // no need to unload
    echo $person->loaded();  // still true

    $person->unload();
    echo $person->loaded();  // false now


    $person->tryLoad(12313123); // no such record
    echo $person->loaded();  // will be false


    $person->load(12313123); // generates an exception

You will see a common pattern in Agile Toolkit pages, where
models are loaded with the data passed through the GET parameters::

    $this->person  = $this->add('Model_Person')->load($_GET['id']);

If the specified ID passed here is not found in the database, then
exceptionis generated and API handles that.

Generical models
----------------
While a general rule says that all your business models needs to be defined
as classes extending from Model or SQL_Model, you can , however, have a
generic model defined like this::

    $m = $this->add('Model', ['table'=>'person']);
    $m->addField('name');
    $m->setSource('Array', ['John', 'Peter', 'Joe']);

The short notation demonstrated here is good if you are simply willing to
test model functionality and do not require comprehensive model definition.


Accessing and Changing field values
-----------------------------------
Model contains the information loaded from the Data Source and
there are several ways to access it.

.. php:method:: get($name = null)

    Get the value of a model field. If field $name is not specified, then
    returns associative hash containing all field data

.. php:method:: set($name, [$value])

    Set value of fields. If only a single argument is specified
    and it is a hash, will use keys as property names and set values
    accordingly.

To complement the example below, I'll also use :php:meth:`Field::defaultValue`
inside field definition. In this example, I'm using generic class for the model,
instead of extending it and creating a separate model::


    $m = $this->add('Model', ['table'=>'person']);
    $m->addField('name');
    $m->addField('age')->type('int')->defaultValue(18);
    $m->setSource('Array', ['John', 'Peter', 'Joe']);

    $m->load(1);
    echo $m->get('name');
    $m->set('age', 25);

    var_dump($m->get());   // outputs [ id=1, name=Peter, age=25 ]

You can also use model as Array, instead of set / get use square brackets::

    $m['age'] = 25;
    echo $m['name'];

.. note:: You can't use ``$m['age']++`` due to some PHP limitation.


.. _model dataset:

The Dataset
-----------
In a traditional database design, the underlying database engine would
group all the data into tables even if the data belongs to different
users. For example, your system might contain list of Users and each User
could have multiple Orders. User must only be able to see orders
by that user and not other orders. Similarly he should be able to modify
only records which are available to him.

A classical problem which often occurs in software design is when you
show user his own orders in a list on a page using this SQL query::

    select * from `order` where user_id = ?

Each order would contain "cancel" button pointing to the delete page and
passing ``id`` parameter. The delete page would contain the following
query::

    delete from `order` where id = ?

The problem here is that if User A known ID of order owned by another
user, he can easily cancel that order.

Agile Toolkit ORM framework allows you to entirely avoid this problem
by changing the way how you think while develop your application. In
Agile Toolkit you do not operate with "table order" instead of operate
with model::

    $orders->load($_GET['id'])->delete();

What's important here is that $order is a model with a limited data-set.

Data Set is a collections of records which model is allowed to load, update
or delete. When developing an app, you must operate with the objects
which already limit the data-set. Here is one example how to do this::


    class Model_MyOrder extends Model_Order {
        function init(){
            parent::init();

            $this->addCondition('user_id', $this->app->auth->model->id);
        }
    }


    // And then
    $orders = $this->add('Model_MyOrders');

The same model object must be used for both displaying the list and executing
delete operations to make sure all the conditions applied properly.

There is however a better way to deal with conditions, which is explained
in the next section.

Relation Traversal
------------------
When you define model, you can specify how they relate to other models.
There are 2 types of basic relations: ``hasOne`` and ``hasMany``::

    $user->hasMany('Order');

    $order->hasOne('User');


Note that Agile Toolkit will automatically add Model_ in front of Order / User
parameter.

``hasOne`` adds a new field in the current model corresponding of 2 parts: $table of
related model and "_id". You can access the ID field at $order->get('user_id');

``hasMany`` does not create any extra fields in your model.

You can traverse thereference by using method ref()

.. php:method:: ref

    Traverses reference of relation

The following example is an alternative approach to the problem described in
the last chapter:

- User hasMany Order
- User can only access their orders
- User can cancel only his own orders

To upgrade the logic, you'll need to add this line to definition of user model::

    $this->hasMany('Order');

Then you will be able to access all orders of the user without any extra
classes or conditions::

    $user = $this->app->auth->model;
    $order = $user->ref('Order');

    $order->load($_GET['id'])->delete();

When traversing into hasMany the model must be loaded and the current id value
will automatically be applied as a condition on Order model dataset.


.. todo:: deep taversal

Data Source
-----------

.. php:method:: setSource

    Associate model with a specified data source. The controller could be
    either a string (postfix for ``Controller_Data_..``) or a class. One data
    controller may be used with multiple models.
    If the $table argument is not specified then :php:attr:`Model::table`
    will be used to find out name of the table / collection

Each model may have a source set. The source is set like this::

    $model->setSource('Session');

    or

    $model->setSource('Array', $arr);

    or

    $model->setSource('MongoDB', 'mycollection');

The first argument here is a name of ":php:class:`Controller_Data`" - a special class
which will control loading and saving of data.

.. note:: If you are extending from SQL_Model you do not need to specify
    a data source - it will work with your current database connection. In the
    future versions of Agile Toolkit Modal_SQL will transition in favor of
    setSource('SQL').

The second argument is optional and if it's specified it will override
:php:attr:`Model::table` of the model. The type of this argument
can vary from driver to driver.

.. todo:: write about lazy write (dirtiness)


Below are table comparing different drivers and showing how the meaning of table
and condition change.

+-------------------------+-------------------+--------------------------------------------------+
| Driver                  | Table             | Condition                                        |
+=========================+===================+==================================================+
| SQL + Database/Schema   | Table Name        | set of "where" conditions joined by AND clause   |
+-------------------------+-------------------+--------------------------------------------------+
| Memcache                | Key Prefix        | Sub-prefix                                       |
+-------------------------+-------------------+--------------------------------------------------+
| MongoDB                 | Collection Name   | Conditions                                       |
+-------------------------+-------------------+--------------------------------------------------+
| Redis + Object Type     | Object name       | Prefix                                           |
+-------------------------+-------------------+--------------------------------------------------+


Relational Model
----------------

A significant segment of the database implementations are so called
RDBMS - Relational Database Management Systems. Notable for their
flexibility in data querying they utilize a standardized query language
- SQL. Agile Toolkit takes advantage of the powerful features of RDBMS
(joining, sub-selects, expressions) and has a significantly enhanced
model class to work directly with the database through DSQL.

To take advantage of those features you must use :php:class:`SQL_Model`.
This class extends Model but adds features of a typical ANSI SQL directly
into model. Refer to the documentation of :php:class:`SQL_Model`

You can still use generic Model with SQL driver, such as SQLite, but
both use slightly different implementations. As of version 4.3 I recommend
using SQL_Model as it is much more tested and optimized.

Using Caching
-------------

A single model can have several caches associated with it. For example a
relational model may have Session cache.

When loading model by id with associated cache - the first attempt is made to
load the model from the cache directly. If model is not found in
cache(s), the primary source is used as a fall-back.

When saving model data, it will be also saved into all the associated
caches.

The same data controller class can be used as either primary source or
as a cache.

How to write Model Code
-----------------------
Model is an essential part of your application containing business logic.
You must refrain from using any of the following from inside your model:

- GET and POST arguments, which are exclusive to app running in Web environment
- UI objects or pages, which may not be there in CLI application.
- Always document and think about use cases when model data is loaded / unloaded.
- Think about transactions and commits.

There is a special rule for relying on authenitcation data. In this documentation
I have given example for MyOrders model, which display orders of the currently
logged-in user. This is a valid usage pattern, but you must use it in a separate
class.

Another example is if your application have a system-wide filter. You might want
to create Model_FilteredOrder which would automatically apply conditions from
the global filter, but you should not do that inside the base model.

With those basic requirements in mind, you can now create methods inside
your model class to wrap up some business logic.




There


Model data and methods
~~~~~~~~~~~~~~~~~~~~~~

In a typical ORM implementation, model data is stored in model
properties while reserving all the property names beginning with
underscore. Agile Toolkit stores model data as array in a single
property called "data". To access the data you can use ``set()``,
``get()`` or array-access (square brackets) format.

Before you can access the data, however, you must define some fields.
Below is a typical implementation of a model in Agile Toolkit. Please
note that model is defined using PHP language and it's always defined as
a class.

::

    class Model_User extends Model {
        function init(){
            parent::init();
            $this->addField('name');
            $this->addField('surname');

            $this->addField('daily_salary');
            $this->addField('due_payment');
        }
        function goToWork(){
            $this['due_payment'] = $this['due_payment']
                +$this['daily_salary'];
            return $this;
        }
        function paySalary(){
            echo "Paying ".$this['name']." amount of ".
                $this['due_payment'];
        }
    }

    $m=$this->add('Model_User');
    $m['name']='John';$m['daily_salary']=150;

    for($day=1;$day<7;$day++) $m->goToWork()
    $m->paySalary();

As you see in the example, model User's model combines definition of the
fields with the methods to perform business operations with the model.
When you design model methods, it's important that you follow these
guidelines:

-  Never assume presence of UI.
-  Avoid addressing "owner" object.
-  Keep object hierarchy in mind. Extend "User" model to create
   "Manager" model.
-  All field names must be unique

By following these guidelines, you can design a model which can work
with magnitude of data sources.

Loading and Saving models
~~~~~~~~~~~~~~~~~~~~~~~~~

You can save your model data to a primary source driver or load data if
you know the "id" of the record. The "id" is not necessarily a number,
but it uniquely defines a data within source / table.

Let's extend our user model by adding "Session" source.

::

    class Model_User extends Model {
        public $table='user';
        function init(){
            parent::init();
            $this->setSource('Session');

Once source is set, you can use a number of additional operations:

::

    $m['name']='John';$m['daily_salary']=150;
    $m->save();
    echo $m->id;    // will contain a generated ID

    $m->load($other_id);    // load different record into model

Model objects in Agile Toolkit are not tied in with any particular
record. They can load any (but one) record from the data-set and save
it. A single object can also iterate through the data-set by loading
each individual record.

There are only two properties which are affected when you load model:
"data" and "id". Next example demonstrates how to display list of all
the users and their respective "due\_payment" field:

::

    foreach($m as $row){
        echo "Please pay ".$row['daily_salary']." to ".
            $row['name']."\n";
    }

When iterating, the
:math:`row becomes automatically associated with the "data" property, however if you are willing to change the content of the model, you should use the `\ m
instead:

::

    foreach($m as $row){
        $m->paySalary();
    }

Model's method ``loaded()`` will return true if model have been loaded
with any data from the source and false otherwise.

::

    $m=$this->add('Model_Table');
    $m->loaded();    // false
    $m->load(1);
    $m->loaded();    // true
    $m->unload();
    $m->loaded();    // false


Deleting model data
~~~~~~~~~~~~~~~~~~~

You can delete a single record of data by calling delete() method or you can
remove all data by calling deleteAll(). If you do not pass id to delete()
method, then loaded record will be deleted.

























Features
--------

In Agile Toolkit model class have the following features:

- Defining column structure and types
- Creating one model by extending another
- Loading one row at a time, manipulating and saving it
- Defining custom methods dealing with data
- Iterating through available records (:ref:`model dataset`)
- Callbacks (e.g. afterLoad or beforeSave)
- Reference traversal

Additionally with the help of Data Source capabilities more features
can be available:

- Adding conditions (filters) on models
- Executing actions on all of the Data Set (update all) without iterating
- Defining skip / count (limit) for records
- Storing complex values in model

A relational database managers (RDBMS) or SQL Servers are capable of
more features and Agile Toolkit provides ways to take advantage of those
features without manually writing queries:

- Joining tables
- Using expressions
- Using sub-selects based on model
- Applying action with existing conditions
- Operating with "actual" field subset

Agile Toolkit standard Data Controllers try to provide you with access to
the features of underlying Data Source, however they will not emulate
features lacking in the Database.

- One primary Data Source per model
- Several secondary Data Sources (caches) per model
- Knowledge of Data Source capabilities

Class Structure
---------------

I have already introduced the main class - :php:class:`Model`, which can
operate with any Data Source::

    $m = $this->add('Model', [ 'table' => 'user' ]);
    $m->setSource('SQL');
    $m->addField('name');
    $m->addField('surname');

However this Model implementation may not support all the features of the
Data Source. A more advanced Data Sources will have a dedicated model class
you can use::


    $m = $this->add('SQL_Model', [ 'table' => 'user' ]);
    $m->addField('name');
    $m->addField('surname');
    $j = $m->join('contact_info','user_id');
    $j->addField('address');
    $m->addCondition('gender', 'm');
    $m->addExpression('full_name')->set('concat(name, " ", surname)');

Limitations and Recommendations
-------------------------------

In order to make working with model more predictable, you must remember
that you must follow these rules:

- Each record must have an ``id`` (numeric or alphanumeric)
- Each ID must correspond to hash of values (by fields), where key is (alphanumeric)
- Model should have field defined (and field types/properties)
- One field is a Title Field (normally "name")
- Model can only access items within data-set (matching conditions)
- Model can only create items which will match match data-set conditions


Creating Data Controllers
-------------------------

Data Controllers implement :php:meth:`Model::load` / :php:meth:`Model::save`
method and some other extensions to the model. If you would like to learn
more about Data Controllers, see :php:class:`Controller_Data`. The rest
of this chapter will focus on defining and using models with existing
controllers.

If you are interested in specific data source features, see:

- :php:class:`Controller_Data_Array` - static array access for models
- :php:class:`Controller_Data_Session` - storing data in Session
- :php:class:`Controller_Data_Mongo` - Accessing MongoDB collections
- :php:class:`Controller_Data_SQL` - PDO-based SQL access. See :php:class:`SQL_Model`
- :php:class:`Controller_Data_Memcache` - Memory Cache
- :php:class:`Controller_Data_RESTful` - Accessing remote API through Model

