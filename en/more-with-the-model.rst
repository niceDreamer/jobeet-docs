NOT CONVERTED TO SYMFONY2 YET
=============================

Want to help out?
Fork it on `Github <https://github.com/sftuts/jobeet-docs>`_

Day 6: More with the Model
==========================

Yesterday was great. You learned how to create pretty URLs and how
to use the symfony framework to automate a lot of things for you.

Today, we will enhance the Jobeet website by tweaking the code here
and there. In the process, you will learn more about all the
features we have introduced during the first five days of this
tutorial.

The Propel Criteria Object -------------------------- The Doctrine
Query Object -------------------------

From the second day's requirements:

"When a user comes to the Jobeet website, she sees a list of active
jobs."

But as of now, all jobs are displayed, whether they are active or
not:

::

    <?php
    // apps/frontend/modules/job/actions/actions.class.php
    class jobActions extends sfActions
    {
      public function executeIndex(sfWebRequest $request)
      {

$this->jobeet\_jobs = JobeetJobPeer::doSelect(new Criteria());
$this->jobeet\_jobs = Doctrine::getTable('JobeetJob')
->createQuery('a') ->execute(); }

::

      // ...
    }

An active job is one that was posted less than 30 days ago. The
``doSelect()`` method takes a
``Criteria|Propel Criteria`` object that describes the
database request to execute. In the code above, an empty
``Criteria`` is passed, which means that all the records are
retrieved from the database. An active job is one that was posted
less than 30 days ago. The ``~Doctrine_Query~::execute()`` method
will make a request to the database. In the code above, we are not
specifying any where condition which means that all the records are
retrieved from the database.

Let's change it to only select active jobs:

::

    <?php
    public function executeIndex(sfWebRequest $request)
    {

$criteria = new Criteria();
$criteria->add(JobeetJobPeer::CREATED\_AT, ➥ time() - 86400 \* 30,
Criteria::GREATER\_THAN);

::

      $this->jobeet_jobs = JobeetJobPeer::doSelect($criteria);

$q = Doctrine\_Query::create() ->from('JobeetJob j')
->where('j.created\_at > ?', ➥ date('Y-m-d H:i:s', time() - 86400
\* 30));

::

      $this->jobeet_jobs = $q->execute();

}

The ``Criteria::add()`` method adds a ``WHERE`` clause to the
generated SQL. Here, we restrict the criteria to only select jobs
that are no older than 30 days. The ``add()`` method accepts a lot
of different comparison operators; here are the most common ones:


-  ``Criteria::EQUAL``
-  ``Criteria::NOT_EQUAL``
-  ``Criteria::GREATER_THAN``, ``Criteria::GREATER_EQUAL``
-  ``Criteria::LESS_THAN``, ``Criteria::LESS_EQUAL``
-  ``Criteria::LIKE``, ``Criteria::NOT_LIKE``
-  ``Criteria::CUSTOM``
-  ``Criteria::IN``, ``Criteria::NOT_IN``
-  ``Criteria::ISNULL``, ``Criteria::ISNOTNULL``
-  ``Criteria::CURRENT_DATE``, ``Criteria::CURRENT_TIME``,
   ``Criteria::CURRENT_TIMESTAMP``

Debugging ##ORM## generated SQL
-------------------------------

As you don't write the SQL statements by hand, ##ORM## will take
care of the differences between database engines and will generate
SQL statements optimized for the database engine you choose during
day 3. But sometimes, it is of great help to see the SQL generated
by ##ORM##; for instance, to debug a query that
does not work as expected. In the ``dev``
environment, symfony logs these queries
(along with much more) in the ``log/`` directory. There is one log
file for every combination of an application and an environment.
The file we are looking for is named ``frontend_dev.log``:

::

    # log/frontend_dev.log

Dec 6 15:47:12 symfony [debug] {sfPropelLogger} exec: SET NAMES
'utf8' Dec 6 15:47:12 symfony [debug] {sfPropelLogger} prepare:
SELECT ➥ jobeet\_job.ID, jobeet\_job.CATEGORY\_ID,
jobeet\_job.TYPE, ➥ jobeet\_job.COMPANY, jobeet\_job.LOGO,
jobeet\_job.URL, jobeet\_job.POSITION, ➥ jobeet\_job.LOCATION,
jobeet\_job.DESCRIPTION, jobeet\_job.HOW\_TO\_APPLY, ➥
jobeet\_job.TOKEN, jobeet\_job.IS\_PUBLIC, jobeet\_job.CREATED\_AT,
➥ jobeet\_job.UPDATED\_AT FROM ``jobeet_job`` WHERE
jobeet\_job.CREATED\_AT>:p1 Dec 6 15:47:12 symfony [debug]
{sfPropelLogger} Binding '2008-11-06 15:47:12' ➥ at position :p1 w/
PDO type PDO::PARAM\_STR

You can see for yourself that Propel has generated a where clause
for the ``created_at`` column
(``WHERE jobeet_job.CREATED_AT > :p1``).

    **NOTE** The ``:p1`` string in the query indicates that Propel
    generates prepared statements. The actual value of ``:p1``
    ('``2008-11-06 15:47:12``' in the example above) is passed during
    the execution of the query and properly escaped by the database
    engine. The use of prepared statements dramatically reduces your
    exposure to
    `SQL injection <http://en.wikipedia.org/wiki/Sql_injection>`_
    attacks. Dec 04 13:58:33 symfony [info] {sfDoctrineLogger}
    executeQuery : SELECT j.id AS j\_\_id,
    j.category*id AS j*\_category*id, j.type AS j**type, j.company AS j**company, j.logo AS j**logo, j.url AS j**url, j.position AS j**position, j.location AS j**location, j.description AS j*\_description,
    j.how\_to*apply AS j*\_how\_to*apply, j.token AS j*\_token,
    j.is*public AS j*\_is\_public,
    j.is*activated AS j*\_is*activated, j.email AS j*\_email,
    j.expires*at AS j*\_expires\_at, j.created*at AS j*\_created\_at,
    j.updated*at AS j*\_updated\_at FROM jobeet\_job j WHERE
    j.created\_at > ? (2008-11-08 01:13:35)


You can see for yourself that Doctrine has a where clause for the
``created_at`` column (``WHERE j.created_at > ?``).

    **NOTE** The ``?`` string in the query indicates that Doctrine
    generates prepared statements. The actual value of ``?``
    ('2008-11-08 01:13:35' in the example above) is passed during the
    execution of the query and properly escaped by the database engine.
    The use of prepared statements dramatically reduces your exposure
    to `SQL injection <http://en.wikipedia.org/wiki/Sql_injection>`_
    attacks.


This is good, but it's a bit annoying to have to switch between the
browser, the IDE, and the log file every time you need to test a
change. Thanks to the symfony web debug toolbar, all the
information you need is also available within the comfort of your
browser:

.. figure:: http://www.symfony-project.org/images/jobeet/1_4/06/web_debug_sql.png
   :alt: SQL statements in the web debug toolbar
   
   SQL statements in the web debug toolbar

Object Serialization
-------------------------------

Even if the above code works, it is far from perfect as it does not
take into account some requirements from day 2:

"A user can come back to re-activate or extend the validity of the
job ad for an extra 30 days..."

But as the above code only relies on the ``created_at`` value, and
because this column stores the creation date, we cannot satisfy the
above requirement.

But if you remember the database schema we have described during
day 3, we also have defined an ``expires_at`` column. Currently, if
this value is not set in fixture file, it remains always empty. But
when a job is created, it can be automatically set to 30 days after
the current date.

When you need to do something automatically before a ##ORM## object
is serialized to the database, you can override the ``save()``
method of the model class:


.. raw:: html

   <?php
       // lib/model/JobeetJob.php
       class JobeetJob extends BaseJobeetJob
       {
         public function save(PropelPDO $con = null)
         {
           if ($this->
   
isNew() && !$this->getExpiresAt()) { $now = $this->getCreatedAt() ?
$this->getCreatedAt('U') : time(); :math:`$this->setExpiresAt($`now
+ 86400 \* 30); }

::

        return parent::save($con);
      }
    
      // ...
    }


.. raw:: html

   <?php
       // lib/model/doctrine/JobeetJob.class.php
       class JobeetJob extends BaseJobeetJob
       {
         public function save(Doctrine_Connection $conn = null)
         {
           if ($this->
   
isNew() && !$this->getExpiresAt()) { $now = $this->getCreatedAt() ?
$this->getDateTimeObject('created\_at')->format('U') : time();
$this->setExpiresAt(date('Y-m-d H:i:s', $now + 86400 \* 30)); }

::

        return parent::save($conn);
      }
    
      // ...
    }

The ``isNew()`` method returns ``true`` when the object has not
been serialized yet in the database, and ``false`` otherwise.

Now, let's change the action to use the ``expires_at`` column
instead of the ``created_at`` one to select the active jobs:

::

    <?php
    public function executeIndex(sfWebRequest $request)
    {

$criteria = new Criteria();
$criteria->add(JobeetJobPeer::EXPIRES\_AT, time(),
Criteria::GREATER\_THAN);

::

      $this->jobeet_jobs = JobeetJobPeer::doSelect($criteria);

$q = Doctrine\_Query::create() ->from('JobeetJob j')
->where('j.expires\_at > ?', date('Y-m-d H:i:s', time()));

::

      $this->jobeet_jobs = $q->execute();

}

We restrict the query to only select jobs with the ``expires_at``
date in the future.

More with Fixtures
------------------

Refreshing the Jobeet homepage in your browser won't change
anything as the jobs in the database have been posted just a few
days ago. Let's change the fixtures to add a job that is already
expired:

[yml] # data/fixtures/020\_jobs.yml JobeetJob: # other jobs

::

      expired_job:
        category_id:  programming
        company:      Sensio Labs
        position:     Web Developer
        location:     Paris, France
        description:  |
          Lorem ipsum dolor sit amet, consectetur
          adipisicing elit.
        how_to_apply: Send your resume to lorem.ipsum [at] dolor.sit
        is_public:    true
        is_activated: true
        created_at:   2005-12-01
        token:        job_expired
        email:        job@example.com

[yml] # data/fixtures/jobs.yml JobeetJob: # other jobs

::

      expired_job:
        JobeetCategory: programming
        company:        Sensio Labs
        position:       Web Developer
        location:       Paris, France
        description:    Lorem ipsum dolor sit amet, consectetur adipisicing elit.
        how_to_apply:   Send your resume to lorem.ipsum [at] dolor.sit
        is_public:      true
        is_activated:   true
        created_at:     '2005-12-01 00:00:00'
        token:          job_expired
        email:          job@example.com

    **NOTE** Be careful when you copy and paste code in a
    fixture file to not break the indentation. The
    ``expired_job`` must only have two spaces before it.


As you can see in the job we have added in the fixture file, the
``created_at`` column value can be defined even if it is
automatically filled by ##ORM##. The defined value will override
the default one. Reload the fixtures and refresh your browser to
ensure that the old job does not show up:

::

    $ php symfony propel:data-load

You can also execute the following query to make sure that the
``expires_at`` column is automatically filled by the ``save()``
method, based on the ``created_at`` value:

::

    SELECT `position`, `created_at`, `expires_at` FROM `jobeet_job`;

Custom Configuration
--------------------

In the ``JobeetJob::save()`` method, we have hardcoded the number
of days for the job to expire. It would have been better to make
the 30 days configurable. The symfony framework provides a built-in
configuration file for application specific
settings file.
This YAML file can contain any setting you want:

::

    [yml]
    # apps/frontend/config/app.yml
    all:
      active_days: 30

In the application, these settings are available through the global
``sfConfig`` class:

::

    <?php
    sfConfig::get('app_active_days')

The setting has been prefixed by ``app_`` because the ``sfConfig``
class also provides access to symfony settings as we will see later
on.

Let's update the code to take this new setting into account:


.. raw:: html

   <?php
       public function save(PropelPDO $con = null)
       {
         if ($this->
   
isNew() && !$this->getExpiresAt()) { $now = $this->getCreatedAt() ?
$this->getCreatedAt('U') : time(); :math:`$this->setExpiresAt($`now
+ 86400 \* ➥ sfConfig::get('app\_active\_days')); }

::

      return parent::save($con);
    }


.. raw:: html

   <?php
       public function save(Doctrine_Connection $conn = null)
       {
         if ($this->
   
isNew() && !$this->getExpiresAt()) { $now = $this->getCreatedAt() ?
$this->getDateTimeObject('created\_at')->format('U') : time();
$this->setExpiresAt(date('Y-m-d H:i:s', $now + 86400 \*
sfConfig::get('app\_active\_days'))); }

::

      return parent::save($conn);
    }

The ``app.yml`` configuration file is a great way to
centralize global settings for your
application.

Last, if you need project-wide settings,
just create a new ``app.yml`` file in the ``config`` folder at the
root of your symfony project.

Refactoring
-----------

Although the code we have written works fine, it's not quite right
yet. Can you spot the problem?

The ``Criteria`` code does not belong to the action (the Controller
layer), it belongs to the Model layer. In the MVC model,
the Model defines all the business logic, and the
Controller only calls the Model to retrieve data from it. As the
code returns a collection of jobs, let's move the code to the
``JobeetJobPeer`` class and create a ``getActiveJobs()`` method:
The ``Doctrine_Query`` code does not belong to the action (the
Controller layer), it belongs to the Model layer. In the
MVC model, the Model defines all the ~business
logic\|Business Logic~, and the Controller only calls the Model to
retrieve data from it. As the code returns a collection of jobs,
let's move the code to the ``JobeetJobTable`` class and create a
``getActiveJobs()`` method:


.. raw:: html

   <?php
       // lib/model/JobeetJobPeer.php
       class JobeetJobPeer extends BaseJobeetJobPeer
       {
         static public function getActiveJobs()
         {
           $criteria = new Criteria();
           $criteria->
   
add(self::EXPIRES\_AT, time(), ➥ Criteria::GREATER\_THAN);

::

        return self::doSelect($criteria);
      }
    }


.. raw:: html

   <?php
       // lib/model/doctrine/JobeetJobTable.class.php
       class JobeetJobTable extends Doctrine_Table
       {
         public function getActiveJobs()
         {
           $q = $this->
   
createQuery('j') ->where('j.expires\_at > ?', date('Y-m-d H:i:s',
time()));

::

        return $q->execute();
      }
    }

Now the action code can use this new method to retrieve the active
jobs.

::

    <?php
    public function executeIndex(sfWebRequest $request)
    {

$this->jobeet\_jobs = JobeetJobPeer::getActiveJobs();
$this->jobeet\_jobs = ➥
Doctrine\_Core::getTable('JobeetJob')->getActiveJobs(); }

This refactoring has several benefits over
the previous code:


-  The logic to get the active jobs is now in the Model, where it
   belongs
-  The code in the controller is thinner and much more readable
-  The ``getActiveJobs()`` method is re-usable (for instance in
   another action)
-  The model code is now unit testable

Let's sort the jobs by the ``expires_at`` column:

::

    <?php

static public function getActiveJobs() { $criteria = new
Criteria(); $criteria->add(self::EXPIRES\_AT, time(),
Criteria::GREATER\_THAN);
$criteria->addDescendingOrderByColumn(self::EXPIRES\_AT);

::

      return self::doSelect($criteria);
    }

public function getActiveJobs() { $q = $this->createQuery('j')
->where('j.expires\_at > ?', date('Y-m-d H:i:s', time()))
->orderBy('j.expires\_at DESC');

::

      return $q->execute();
    }

The ``addDescendingOrderByColumn()`` method adds an ``ORDER BY``
clause to the generated SQL (``addAscendingOrderByColumn()`` also
exists). The ``orderBy`` methods sets the ``ORDER BY`` clause to
the generated SQL (``addOrderBy()`` also exists).

Categories on the Homepage
--------------------------

From the second day's requirements:

"The jobs are sorted by category and then by publication date
(newer jobs first)."

Until now, we have not taken the job category into account. From
the requirements, the homepage must display jobs by category.
First, we need to get all categories with at least one active job.

Open the ``JobeetCategoryPeer`` class and add a ``getWithJobs()``
method: Open the ``JobeetCategoryTable`` class and add a
``getWithJobs()`` method:


.. raw:: html

   <?php
       // lib/model/JobeetCategoryPeer.php
       class JobeetCategoryPeer extends BaseJobeetCategoryPeer
       {
         static public function getWithJobs()
         {
           $criteria = new Criteria();
           $criteria->
   
addJoin(self::ID, JobeetJobPeer::CATEGORY\_ID);
$criteria->add(JobeetJobPeer::EXPIRES\_AT, time(),
Criteria::GREATER\_THAN); $criteria->setDistinct();

::

        return self::doSelect($criteria);
      }
    }

The ``Criteria::addJoin()`` method adds a ``JOIN``
clause to the generated SQL. By default, the join condition is
added to the ``WHERE`` clause. You can also change the join
operator by adding a third argument (``Criteria::LEFT_JOIN``,
``Criteria::RIGHT_JOIN``, and ``Criteria::INNER_JOIN``).

.. raw:: html

   <?php
       // lib/model/doctrine/JobeetCategoryTable.class.php
       class JobeetCategoryTable extends Doctrine_Table
       {
         public function getWithJobs()
         {
           $q = $this->
   
createQuery('c') ->leftJoin('c.JobeetJobs j')
->where('j.expires\_at > ?', date('Y-m-d H:i:s', time()));

::

        return $q->execute();
      }
    }

Change the ``index`` action accordingly:

::

    <?php
    // apps/frontend/modules/job/actions/actions.class.php
    public function executeIndex(sfWebRequest $request)
    {

$this->categories = JobeetCategoryPeer::getWithJobs();
$this->categories = ➥
Doctrine\_Core::getTable('JobeetCategory')->getWithJobs(); }

In the template, we need to iterate through all categories and
display the active jobs:

::

    <?php
    // apps/frontend/modules/job/templates/indexSuccess.php
    <?php use_stylesheet('jobs.css') ?>
    
    <div id="jobs">
      <?php foreach ($categories as $category): ?>
        <div class="category_<?php echo Jobeet::slugify($category->getName()) ?>">
          <div class="category">
            <div class="feed">
              <a href="">Feed</a>
            </div>
            <h1><?php echo $category ?></h1>
          </div>
    
          <table class="jobs">
            <?php foreach ($category->getActiveJobs() as $i => $job): ?>
              <tr class="<?php echo fmod($i, 2) ? 'even' : 'odd' ?>">
                <td class="location">
                  <?php echo $job->getLocation() ?>
                </td>
                <td class="position">
                  <?php echo link_to($job->getPosition(), 'job_show_user', $job) ?>
                </td>
                <td class="company">
                  <?php echo $job->getCompany() ?>
                </td>
              </tr>
            <?php endforeach; ?>
          </table>
        </div>
      <?php endforeach; ?>
    </div>

    **NOTE** To display the category name in the template, we have used
    ``echo $category``. Does this sound weird? ``$category`` is an
    object, how can ``echo`` magically display the category name? The
    answer was given during day 3 when we have defined the magic
    ``__toString()`` method for all the model classes.


For this to work, we need to add the ``getActiveJobs()`` method to
the ``JobeetCategory`` class that returns the active jobs for the
category object:

::

    <?php
    // lib/model/JobeetCategory.php
    public function getActiveJobs()
    {
      $criteria = new Criteria();
      $criteria->add(JobeetJobPeer::CATEGORY_ID, $this->getId());
    
      return JobeetJobPeer::getActiveJobs($criteria);
    }

In the ``add()`` call, we have omitted the third argument as
``Criteria::EQUAL`` is the default value.

The ``JobeetCategory::getActiveJobs()`` method uses the
``JobeetJobPeer::getActiveJobs()`` method to retrieve the active
jobs for the given category.

When calling the ``JobeetJobPeer::getActiveJobs()``, we want to
restrict the condition even more by providing a category. Instead
of passing the category object, we have decided to pass a
``Criteria`` object as this is the best way to encapsulate a
generic condition.

The ``getActiveJobs()`` needs to merge this ``Criteria`` argument
with its own criteria. As the ``Criteria`` is an object, this is
quite simple:

::

    <?php
    // lib/model/JobeetJobPeer.php
    static public function getActiveJobs(Criteria $criteria = null)
    {
      if (is_null($criteria))
      {
        $criteria = new Criteria();
      }
    
      $criteria->add(JobeetJobPeer::EXPIRES_AT, time(),
       ➥ Criteria::GREATER_THAN);
      $criteria->addDescendingOrderByColumn(self::EXPIRES_AT);
    
      return self::doSelect($criteria);
    }

For this to work, we need to add the ``getActiveJobs()`` method to
the ``JobeetCategory`` class:

::

    <?php
    // lib/model/doctrine/JobeetCategory.class.php
    public function getActiveJobs()
    {
      $q = Doctrine_Query::create()
        ->from('JobeetJob j')
        ->where('j.category_id = ?', $this->getId());
    
      return Doctrine_Core::getTable('JobeetJob')->getActiveJobs($q);
    }

The ``JobeetCategory::getActiveJobs()`` method uses the
``Doctrine_Core::getTable('JobeetJob')->getActiveJobs()`` method to
retrieve the active jobs for the given category.

When calling the
``Doctrine_Core::getTable('JobeetJob')->getActiveJobs()``, we want
to restrict the condition even more by providing a category.
Instead of passing the category object, we have decided to pass a
``Doctrine_Query`` object as this is the best way to encapsulate a
generic condition.

The ``getActiveJobs()`` needs to merge this ``Doctrine_Query``
object with its own query. As the ``Doctrine_Query`` is an object,
this is quite simple:

::

    <?php
    // lib/model/doctrine/JobeetJobTable.class.php
    public function getActiveJobs(Doctrine_Query $q = null)
    {
      if (is_null($q))
      {
        $q = Doctrine_Query::create()
          ->from('JobeetJob j');
      }
    
      $q->andWhere('j.expires_at > ?', date('Y-m-d H:i:s', time()))
        ->addOrderBy('j.expires_at DESC');
    
      return $q->execute();
    }

Limit the Results
-----------------

There is still one requirement to implement for the homepage job
list:

"For each category, the list only shows the first 10 jobs and a
link allows to list all the jobs for a given category."

That's simple enough to add to the ``getActiveJobs()`` method:


.. raw:: html

   <?php
       // lib/model/JobeetCategory.php
       public function getActiveJobs($max = 10)
       {
         $criteria = new Criteria();
         $criteria->
   
add(JobeetJobPeer::CATEGORY\_ID, $this->getId());
:math:`$criteria->setLimit($`max);

::

      return JobeetJobPeer::getActiveJobs($criteria);
    }


.. raw:: html

   <?php
       // lib/model/doctrine/JobeetCategory.class.php
       public function getActiveJobs($max = 10)
       {
         $q = Doctrine_Query::create()
           ->
   
from('JobeetJob j') ->where('j.category\_id = ?',
:math:`$this->getId()) ->limit($`max);

::

      return Doctrine_Core::getTable('JobeetJob')->getActiveJobs($q);
    }

The appropriate ``LIMIT`` clause is now hard-coded into
the Model, but it is better for this value to be configurable.
Change the template to pass a maximum number of jobs set in
``app.yml``:

::

    <?php
    <!-- apps/frontend/modules/job/templates/indexSuccess.php -->
    <?php foreach ($category->getActiveJobs(sfConfig::get('app_max_jobs_on_homepage')) as $i => $job): ?>

and add a new setting in ``app.yml``:

::

    [yml]
    all:
      active_days:          30
      max_jobs_on_homepage: 10

.. figure:: http://www.symfony-project.org/images/jobeet/1_4/06/homepage.png
   :alt: Homepage sorted by category
   
   Homepage sorted by category

Dynamic Fixtures
----------------

Unless you lower the ``max_jobs_on_homepage`` setting to one, you
won't see any difference. We need to add a bunch of jobs to the
fixture. So, you can copy and paste an
existing job ten or twenty times by hand... but there's a better
way. Duplication is bad, even in fixture files.

symfony to the rescue! YAML files in symfony can contain
PHP code that will be evaluated just before the parsing of the
file. Edit the ``020_jobs.yml`` fixtures file and add the following
code at the end: ``jobs.yml`` fixtures file and add the following
code at the end:

::

    <?php
    # Starts at the beginning of the line (no whitespace before)
    <?php for ($i = 100; $i <= 130; $i++): ?>
      job_<?php echo $i ?>:

category\_id: programming JobeetCategory: programming company:
Company

.. raw:: html

   <?php echo $i."\n" ?>
           
   
position: Web Developer location: Paris, France description: Lorem
ipsum dolor sit amet, consectetur adipisicing elit.
how\_to*apply: \| Send your resume to lorem.ipsum [at] company*

.. raw:: html

   <?php echo $i ?>
   
.sit is\_public: true is*activated: true token: job*

.. raw:: html

   <?php echo $i."\n" ?>
           
   
email: job@example.com

::

    <?php endfor ?>

Be careful, the YAML parser won't like you if you mess up with
Indentation. Keep in mind the following
simple tips when adding PHP code to a YAML file:


-  The ``<?php ?>`` statements must always start the line or be
   embedded in a value.

-  If a ``<?php ?>`` statement ends a line, you need to explicly
   output a new line ("").


You can now reload the fixtures with the ``propel:data-load`` task
and see if only ``10`` jobs are displayed on the homepage for the
``Programming`` category. In the following screenshot, we have
changed the maximum number of jobs to five to make the image
smaller:

.. figure:: http://www.symfony-project.org/images/jobeet/1_4/06/pagination.png
   :alt: Pagination
   
   Pagination

Secure the Job Page
-------------------

When a job expires, even if you know the URL, it must not be
possible to access it anymore. Try the URL for the expired job
(replace the ``id`` with the actual ``id`` in your database -
``SELECT id, token FROM jobeet_job WHERE expires_at < NOW()``):

::

    /frontend_dev.php/job/sensio-labs/paris-france/ID/web-developer-expired

Instead of displaying the job, we need to forward the user to a 404
page. But how can we do this as the job is retrieved automatically
by the route?

By default, the ``sfPropelRoute`` uses the standard
``doSelectOne()`` method to retrieve the object, but you can change
it by providing a ``method_for_criteria`` option in the
route configuration:

::

    [yml]
    # apps/frontend/config/routing.yml
    job_show_user:
      url:     /job/:company_slug/:location_slug/:id/:position_slug
      class:   sfPropelRoute
      options:
        model: JobeetJob
        type:  object

method\_for\_criteria: doSelectActive method\_for\_query:
retrieveActiveJob param: { module: job, action: show }
requirements: id: + sf\_method: [GET]

The ``doSelectActive()`` method will receive the ``Criteria``
object built by the route:

::

    <?php
    // lib/model/JobeetJobPeer.php
    class JobeetJobPeer extends BaseJobeetJobPeer
    {
      static public function doSelectActive(Criteria $criteria)
      {
        $criteria->add(JobeetJobPeer::EXPIRES_AT, time(),
         ➥ Criteria::GREATER_THAN);
    
        return self::doSelectOne($criteria);
      }
    
      // ...
    }

The ``retrieveActiveJob()`` method will receive the
``Doctrine_Query`` object built by the route:

::

    <?php
    // lib/model/doctrine/JobeetJobTable.class.php
    class JobeetJobTable extends Doctrine_Table
    {
      public function retrieveActiveJob(Doctrine_Query $q)
      {
        $q->andWhere('a.expires_at > ?', date('Y-m-d H:i:s', time()));
    
        return $q->fetchOne();
      }
    
      // ...
    }

Now, if you try to get an expired job, you will be forwarded to a
404 page.

.. figure:: http://www.symfony-project.org/images/jobeet/1_4/06/exception.png
   :alt: 404 for expired job
   
   404 for expired job

Link to the Category Page
-------------------------

Now, let's add a link to the category page on the homepage and
create the category page.

But, wait a minute. the hour is not yet over and we haven't worked
that much. So, you have plenty of free time and enough knowledge to
implement this all by yourself! Let's make an exercise of it. Check
back tomorrow for our implementation.

Final Thoughts
--------------

Do work on an implementation on your local Jobeet project. Please,
abuse the online
`API documentation <http://www.symfony-project.org/api/1_4/>`_
and all the free
`documentation <http://www.symfony-project.org/doc/1_4/>`_
available on the symfony website to help you out. Tomorrow, we will
give you the solution on how to implement this feature.

**ORM**


