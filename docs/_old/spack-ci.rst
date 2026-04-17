.. ## Copyright (c) 2019-2021, Lawrence Livermore National Security, LLC and
.. ## other RADIUSS Project Developers. See the top-level COPYRIGHT file for details.
.. ##
.. ## SPDX-License-Identifier: (MIT)

.. _spack-ci:

====================
Using Spack Pipeline
====================

Spack provides a feature to generate a GitLab pipeline that will build all the
specs of a given environment in a GitLab pipeline. This feature is documented
by `Spack <https://spack.readthedocs.io/en/latest/pipelines.html>`_ and was
originally introduced to run on cloud resource.

We intend to illustrate the use of Spack pipelines on an HPC cluster, and not
for deployment, but rather for testing.


Using a unique instance of Spack
================================

One of the first special feature that appears in our situation is that we don't
have Spack already on the system, and we don't want to clone it in each job.

Requirements
------------

#. We want to use a single instance of Spack across all the CI jobs, for
   performance reasons.

#. We don't want two independent pipelines to use the same instance of Spack,
   to avoid locks and conflicts on the Spack version used.

#. We don't want to download Spack history again if we already have it.

Implementation
--------------

In terms of implementation, we can start with a script to get Spack.

--------

.. raw:: html

   <details>
   <summary><a>A script to get Spack</a></summary>

.. literalinclude:: ../../.gitlab/get-spack
   :start-after: [get-spack--]
   :end-before: [--get-spack]

This code is called in CI by a dedicated job:

.. literalinclude:: ../../.gitlab-ci.yml
   :start-after: [get-spack--]
   :end-before: [--get-spack]

.. note::
   We define the script outside of the CI. This is a good practice for testing
   and sharing the CI script outside CI.

.. raw:: html

   </details>

--------

But the most critical part is how to share the location of Spack with the child
pipeline.

We first create a global variable for the path, made "unique" by using the
commit ID:

.. literalinclude:: ../../.gitlab-ci.yml
   :start-after: [create-unique-path--]
   :end-before: [--create-unique-path]

Then we propagate it to the child pipelines in the trigger job:

.. literalinclude:: ../../.gitlab/generate.yml
   :start-after: [send-variable-child--]
   :end-before: [--send-variable-child]

The important thing to note here is that we need to change the variable name to
pass it to the child pipeline. This has been
`reported to GitLab <https://gitlab.com/gitlab-org/gitlab/-/issues/213729>`_.

Managing the GPG key
--------------------

One of the first blocking point while attempting to share a single Spack
instance will be GPG key management.

In the general case each build jobs will register the mirror key, which will
result in deadlocks in our case. We can instead register the key ahead of time.

We will produce a GPG key using an instance of Spack, and then create a back-up
location, so that we can use it for any spack instance we create in the CI
context. This is already detailed in `spack mirror/build cache tutorial
<https://spack-tutorial.readthedocs.io/en/latest/tutorial_binary_cache.html>`_.

.. code-block:: bash

  $ spack gpg create "My Name" "<my.email@my.domain.com>"
  gpg: key 070F30B4CDF253A9 marked as ultimately trusted
  gpg: directory '/home/spack/spack/opt/spack/gpg/openpgp-revocs.d' created
  gpg: revocation certificate stored as '/home/spack/spack/opt/spack/gpg/openpgp-revocs.d/ABC74A4103675F76561A607E070F30B4CDF253A9.rev'
  $ mkdir ~/private_gpg_backup
  $ cp ~/spack/opt/spack/gpg/*.gpg ~/private_gpg_backup
  $ cp ~/spack/opt/spack/gpg/pubring.* ~/mirror

Then we simply need to point the variable ``SPACK_GNUPGHOME`` to the location
where the key is stored. Spack won't try to register the key if this variable
is already set.

--------

.. raw:: html

   <details>
   <summary><a>More about this issue.</a></summary>
   <br>
   Spack pipeline feature was designed to work with jobs in containers,
   isolated from one another. We are breaking this assumption by using a unique
   Spack instance on the shared file system.
   <br>
   <br>
   Each build jobs will attempt to register a GPG key concurrently, but using
   the same Spack instance. One of the job will create a lock that will cause
   all the others to fail.

.. raw:: html

   </details>

--------


Only generate one pipeline per stage
------------------------------------

Because we are using a single instance of Spack, we want to avoid race
conditions that could cause locks.

In practice it forced us to isolate each job that generates a pipeline in a
separate stage. That is because of potential locks when concretizing two specs
at the same time.

--------

.. raw:: html

   <details>
   <summary><a>More about this issue.</a></summary>
   <br>
   We generate one pipeline per machine. The file system, where the unique
   instance of Spack lives, is shared among the machines. This is why we need
   to sequentially generate the pipelines. We may instead choose to have two
   Spack instances in the future.

.. literalinclude:: ../../.gitlab-ci.yml
   :start-after: [one-generate-per-stage--]
   :end-before: [--one-generate-per-stage]

.. raw:: html

   </details>

--------


Configuring the environment
===========================

There are several things to think of when configuring a Spack environment for
CI on a cluster.

Isolate spack configuration
---------------------------

We need to make sure the environment (in particular the configuration files) is
isolated. In practice it implies to workaround the user scope of configuration,
as we want our pipeline to behave in a deterministic manner whoever triggers
it.

The most robust way to do so is to make sure that the environment is the only
place where the configuration is defined, and override everything else.

.. note::
   We use ``include`` to keep things clear and split the config in separate
   files, but everything could be in a single ``spack.yaml`` file.

For example, we use the double colon override syntax in the ``packages``
section:

.. literalinclude:: ../../spack-environments/radiuss/toss_3_x86_64_ib/packages.yaml
   :start-after: [config-override--]
   :end-before: [--config-override]

The same needs to be applied to the ``compilers`` section.

We also make sure to move any cache or stage directory to the Spack root dir,
making them specific to that instance by design:

.. literalinclude:: ../../spack-environments/radiuss/spack.yaml
   :start-after: [config-override--]
   :end-before: [--config-override]

.. note::
   We do not use the ``::`` syntax on the ``config`` section. That is because
   we assume that it will not be affected as much by the user scope. However,
   note that we use it on the ``build_stage`` subsection, since it is a list
   that would otherwise consist in the merge of all the scopes.

Performance optimization
------------------------

Several ways of improvement we are exploring:

* At the moment, each pipeline starts with a clone of Spack. Even if we do a
  shallow clone, this takes between 2 and 8 minutes in our observations.

* The working directory is in the ``workspace`` filesystem, which is slow. We
  do not need persistence of our temporary files, so we could renounce to it
  and work in the node shared memory ``/dev/shm``. Our first experiments
  suggests that this would greatly improve the performances.

Shared configuration files
--------------------------

We are planning to share the configuration files (``compilers.yaml`` and
``packages.yaml`` for the most part) in
`another open-source repo <https://github.com/LLNL/radiuss-spack-configs>`_.

This will help ensure consistency in out testing across LLNL open-source
projects. This is already in use in `RAJA <https://github.com/LLNL/RAJA>`_,
`Umpire <https://github.com/LLNL/Umpire>`_ and `CHAI
<https://github.com/LLNL/CHAI>`_. Projects could still add their own
configurations.
