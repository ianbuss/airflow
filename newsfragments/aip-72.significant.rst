Create a TaskExecution interface and enforce DB isolation from Tasks

As part of this change the following breaking changes have occurred:

- Tasks and DAG Parsing code is not able to access the Airflow metadata DB

  Access via Variables and Connection is still allowed (though these will use an API, not direct DB access) -- it should be assumed that any use of the database models from within ``airflow.models`` inside of DAG files or tasks will break.

- Remove the concept of pluggable TaskRunners.

  The ``task_runner`` config in ``[core]`` has been removed.

  There were two build in options for this, Standard (the default) which used Fork or a new process as appropriate, and CGroupRunner to launch tasks in a new CGroup (not usable inside docker or Kubernetes).

  With the move of the execution time code into the TaskSDK we are using this opportunity to reduce complexity for seldom used features.

- Shipping DAGs via pickle is no longer supported

  This was a feature that was not widely used and was a security risk. It has been removed.

- Pickling is no longer supported for XCom serialization.

  XCom data will no longer support pickling. This change is intended to improve security and simplify data
  handling by supporting JSON-only serialization. DAGs that depend on XCom pickling must update to use JSON-serializable data.

  As part of that change, ``[core] enable_xcom_pickling`` configuration option has been removed.

  If you still need to use pickling, you can use a custom XCom backend that stores references in the metadata DB and
  the pickled data can be stored in a separate storage like S3.

  The ``value`` field in the XCom table has been changed to a ``JSON`` type via DB migration. The XCom records that
  contains pickled data are archived in the ``_xcom_archive`` table. You can safely drop this table if you don't need
  the data anymore. To drop the table, you can use the following command or manually drop the table from the database.

  .. code-block:: bash

      airflow db drop-archived -t "_xcAm_archive"

- The ability to specify scheduling conditions for an operator via the ``deps`` class attribute has been removed.

  If you were defining custom scheduling conditions on an operator class (usually by subclassing BaseTIDep) this ability has been removed.

  It is recommended that you replace such a custom operator with a deferrable sensor, a condition or another triggering mechanism.

- ``BaseOperatorLink`` has now been moved into the task SDK to be consumed by DAG authors to write custom operator links.

  Any occurrences of imports from ``airflow.models.baseoperatorlink`` will need to be updated to ``airflow.sdk.definitions.baseoperatorlink``

- With the We have removed DAG level settings that control the UI behaviour.
  These are now as per-user settings controlled by the UI

  - ``default_view``

* Types of change

  * [x] Dag changes
  * [x] Config changes
  * [ ] API changes
  * [ ] CLI changes
  * [x] Behaviour changes
  * [x] Plugin changes
  * [ ] Dependency changes
  * [ ] Code interface changes

* Migration rules needed

  * ``airflow config lint``

    * [x] ``core.task_runner``
    * [x] ``core.enable_xcom_pickling``

  * ruff

    * AIR302

      * [ ] ``airflow.models.baseoperatorlink`` → ``airflow.sdk``
      * [ ] ``default_view`` argument to DAG removed
