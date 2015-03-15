Data Layer (Model)
##################

In our software data is arranged into logical tables or collections.
It's sensible to associate data with a set of relevant actions. PHP
objects is are an ideal container for both the data and the set of
methods which can be applied on the data.

Agile Toolkit further extends the basic Model concept to bring more
features into Model object:

.. figure:: ./model.png

    Field Definitions, Behaviour, Hooks, Controller, ID, Data, Relations,
    Conditions



Normally a single model object represents a single row of data, however
this approach is impractical. Agile Toolkit uses a different approach
for the models, where the model can load any row of data or even perform
operations on all data set. If you have prior experience with other data
frameworks such as Doctrine, Active Record or some other ORM, you should
pay attention to how "Model" works in Agile Toolkit.


The Dataset
~~~~~~~~~~~

Models in Agile Toolkit are not limited to SQL, but can in fact work
with many different data drivers. Therefore I will define all the
possible data you can access through one model as a Dataset.

Dataset is determined by 3 things: 1) Driver 2) Table 3) Scope.

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

Here are some examples:

+-------------------------+-----------------------------+---------------------+------------------------------+
| Use Case                | Driver                      | Table               | Condition                    |
+=========================+=============================+=====================+==============================+
| Model\_Admin            | MySQL                       | user                | is\_admin=1, is\_deleted=0   |
+-------------------------+-----------------------------+---------------------+------------------------------+
| Model\_ShoppingBasket   | Controller\_Data\_Session   | basket              |                              |
+-------------------------+-----------------------------+---------------------+------------------------------+
| Model\_BasketItems      | MySQL                       | item, join basket   | basket.user\_id=123          |
+-------------------------+-----------------------------+---------------------+------------------------------+


setSource - Primary Source
~~~~~~~~~~~~~~~~~~~~~~~~~~

A non-relational models can use ``setSource()`` method to associate
themselves with a driver. Driver is an object of class extending
Controller\_Data. Model will route some of the operations to the
controller, such as loading, saving and deleting records.

Model can only have one source and because relational models already
using SQL you cannot specify a different source.

addCache - Caches
~~~~~~~~~~~~~~~~~

A single model can have several caches associated with it. For example a
relational model may have Session cache.

When loading model with associated cache - the first attempt is made to
load the model from the cache directly. If model is not found in
cache(s), the primary source is used as a fall-back.

When saving model data, it will be also saved into all the associated
caches.

The data controllers typically can be used as either primary source or
as a cache.

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
    $m['name']         = 'John';
    $m['daily_salary'] = 150;

    for($day=1; $day<7; $day++) {
        $m->goToWork();
    }
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

        .....

Once source is set, you can use a number of additional operations:

::

    $m['name']='John';
    $m['daily_salary']=150;
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

When iterating, the ``$row`` is same object as ``$m`` but will have
a corresponding row loaded, you can safely execute it's methods.

::

    foreach($m as $row){
        $row->paySalary();
    }

Model's method ``loaded()`` will return true if model have been loaded
with any data from the source and false otherwise.

::

    $m=$this->add('Model_Book');
    $m->loaded();    // false
    $m->load(1);
    $m->loaded();    // true
    $m->unload();
    $m->loaded();    // false

Deleting model data
~~~~~~~~~~~~~~~~~~~

You can delete a single record of data by calling
``delete($id)`` method or you can remove all data by calling deleteAll().
If you do not pass $id
to delete() method, then the currently loaded data will be deleted.


Relational Models (SQL)
-----------------------

Relational Model base class :doc:class:`SQL_Model` extends standard models and
enhances them with various features. Most of the principles described below
also apply on non-relotaional models.


Goal: Simplification of SQL
~~~~~~~~~~~~~~~~~~~~~~~~~~~

As you describe the model, it will behind the scenes build chunks of
SQL queries to perform operations on that model. This information
is stored in the :php:class:`DSQL`
object which can be used for accessing, saving and deleting your model
data. The nature of DSQL which allows it to perform multiple queries
perfectly matches the needs of reusable models. At the same time, the
power and flexibility of DSQL can still be accessed, if you want to
optimize your queries.

In typical ORM design, you must either use their limited features or use
completely different way to address database. For example, you normally
are unable to perform action on multiple records through ORM interface,
and if you wish to do so, you would need to come up with a full query
yourself.

Agile Toolkit models can provide you with a pre-made DSQL object for you
which you can extend and modify.

Goal: Integrity of DataSet
~~~~~~~~~~~~~~~~~~~~~~~~~~

Relational models perfectly applies the concept of data-set to your
database. You can define multiple models which access the same table,
but would have a different set of conditions and therefore would have
different data-sets.

Model designed to access ``is_admin=true`` users will not be able to
load, save or update user which is not admin. Agile Toolkit ensures
that you would not be able to accidentally go
outside of the data-set when you query, update, delete or insert data.


Read more on :ref:`Models`
