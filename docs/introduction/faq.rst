.. Copyright (C) 2017-2019 Maciej Delmanowski <drybjed@gmail.com>
.. Copyright (C) 2019      Tasos Alvas <tasos.alvas@qwertyuiopia.com>
.. Copyright (C) 2017-2019 DebOps <https://debops.org/>
.. Copyright (C) 2022      GRNET <https://grnet.gr>
.. SPDX-License-Identifier: GPL-3.0-or-later

.. _faq:

Frequently Asked Questions
==========================

Here you can find answers to commonly asked questions about DebOps.

.. contents::
   :local:
   :depth: 2


Can I use DebOps roles as standalone?
-------------------------------------

Yes! All [#f1]_ of the Ansible roles included in DebOps are designed to be
self-contained - each role usually manages a specific service or functionality,
and doesn't touch anything that it is not supposed to directly. Configuration
dependent on other roles (for example, firewall configuration for a network
service) is passed along to relevant roles using role dependent variables, on
the Ansible playbook level.

If you want, you can use DebOps roles in your own playbooks with completely
different dependent roles (for example replacing the :ref:`debops.ferm` role
with another firewall management Ansible role).

Some dependent roles like :ref:`debops.secret` and
:ref:`debops.ansible_plugins` are "hard dependencies" and without them roles
will not work as expected - check the example playbooks provided in the
documentation to see how the roles are used.


I installed one of DebOps roles via Ansible Galaxy but it doesn't work, why?
----------------------------------------------------------------------------

TL;DR: Install the DebOps monorepo instead of specific roles and configure the
``roles_path`` parameter in :file:`ansible.cfg` config file. See :ref:`DebOps
installation instructions <install>` for details.

Long ago, DebOps roles were published in separate :command:`git` repositories
on Ansible Galaxy, and using for example:

.. code-block:: console

   ansible-galaxy install debops.nginx

worked as you would expect - installed the :ref:`debops.nginx` role in the
specified directory. Around October 2017, DebOps project was consolidated to
a single monorepo and separate :command:`git` repositories were deprecated, but
still available via Ansible Galaxy as before.

About a year later, when Ansible Galaxy team implemented experimental support
for multi-role repositories and :command:`mazer`, all of the old DebOps roles
were removed from Ansible Galaxy and the DebOps monorepo was published instead.
Unfortunately, the old :command:`ansible-galaxy` tool was not updated, and
using it to install specific DebOps roles resulted in a broken state, where
a bunch of ``debops.apt*`` roles and the DebOps monorepo in a subdirectory were
installed. A solution to that was to install the published monorepo with:

.. code-block:: console

   mazer install debops.debops

You would also need to tell Ansible where to look for DebOps roles, by
configuring the ``roles_path`` parameter in the :file:`ansible.cfg`
configuration file (normally the :command:`debops` script does that for you).

Another year passed, and in June 2019 Ansible Galaxy team removed support for
multi-role repositories and implemented Ansible Collections. But before that,
the Mazer team removed support for multi-role repositories from the
:command:`mazer` client, and at some point DebOps monorepo was uninstallable
via Ansible Galaxy.

Since DebOps v2.0.0 release, the project should be fully supported as an
Ansible Collection available on Ansible Galaxy. If you use an older release
installed from Galaxy, you should consider upgrading to the current stable
release. You can read the :ref:`DebOps installation instructions <install>` to
find out more.


Why DebOps doesn't use :command:`ansible-vault` to store passwords?
-------------------------------------------------------------------

DebOps roles automatically generate randomized passwords for different accounts
and services, using the `password lookup plugin`__. To ensure idempotency,
plaintext passwords are stored on the Ansible Controller host in the
:file:`secret/` directory alongside the Ansible inventory.

.. __: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/password_lookup.html

The :command:`ansible-vault` command does not support automatic generation of
random passwords - you would need to `create each one by hand`__, which gets
tedious after the third host you manage. You can still do this if you want,
passwords used by DebOps roles are stored in variables which can be redefined
in the Ansible inventory.

.. __: https://docs.ansible.com/ansible/latest/user_guide/vault.html

The :file:`secret/` directory is used for much more - Certificate Authority
management via :ref:`debops.pki`, passing secure data between hosts, for
example by :ref:`debops.tinc`, among other things. You can read more about it
in the :ref:`debops.secret` role documentation.


Ansible complains about ``"lookup plugin (*_src) not found"``.
--------------------------------------------------------------

DebOps playbooks and roles are supposed to be "read-only" to ensure that future
updates can be easily installed. To allow for more extensive modifications
(custom files, templates and tasks), a set of Ansible lookup plugins was
developed which allows to "inject" custom changes in the roles without
modifying the main files. These custom lookup plugins are not part of the
official Ansible distribution, and are `provided with the DebOps playbooks`__.

.. __: https://github.com/debops/debops/tree/master/ansible/roles/ansible_plugins/lookup_plugins

The error about lookup plugins not being present might show up if you use
DebOps roles separately from the main playbook, for example downloaded through
Ansible Galaxy. In this case the easiest solution is to download the custom
lookup plugins and provide them alongside your playbook, in
:file:`lookup_plugins/` directory; this should allow Ansible to find them and
use them.

The long term plan is to remove the need for the custom lookup plugins - the
roles that use them should be updated so that any changes that require custom
templates or files can be done through normal Ansible functionality.


Roles fail when running ``debops`` with the ``--skip-tags`` flag.
-----------------------------------------------------------------

This is due to the way tags are structured. As a general rule, if you use
``--skip-tags``, you should use tags in the form ``skip::<role_name>`` as
opposed to ``role::<role_name>``.

If the role you want to skip does not have a matching ``skip::<role_name>``
tag, please open an issue or, even better, create a pull request!

See `Issue #444`__ for more information and an example of such a pull
request.

.. __: https://github.com/debops/debops/issues/444


I ran DebOps against a target host and now I can no longer ssh into the host.
-----------------------------------------------------------------------------

First, you obviously need to connect to the host in some other way; e.g. through
the console.

Second, undo what DebOps has done:

* Edit :file:`/etc/ssh/sshd_config` and see if you can fix something. For
  example, if you were logging in with a username and password, you need to set
  ``PasswordAuthentication yes``.  Run ``service ssh reload`` if you make any
  changes.

* Edit :file:`/etc/pam.d/sshd` and comment out this line::

      account  required     pam_access.so nodefgroup accessfile=/etc/security/access-sshd.conf

* Check :file:`/var/log/auth.log` for more hints.

Finally, read the :ref:`debops.sshd` role documentation. It explains how it
works and how you can configure it so that it does what you want.

DebOps is very slow.
--------------------

There are a few things you can do to speed it up.

First, in :file:`.debops.cfg`, enable ssh pipelining:

.. code-block:: ini

   [ansible ssh_connection]
   pipelining = True

After that, execute the :command:`debops project refresh` command to apply the
new options in the :file:`ansible.cfg` configuration file. See the `Ansible
documentation on pipelining`_ for more information.

Second, use as Ansible/DebOps Controller a machine that is "near" (on
the same network as) the controlled machines. The reason for this is due to how
Ansible performs its operations - each "task" is converted to a Python script
which is then sent over SSH to the host and executed there, returning with the
finished results back to the Ansible Controller host. This usually takes
a 100-200ms or so, depending on network speed and available bandwidth. With
small number of tasks this round-trip cost is negligible, but with large
projects like DebOps with thousands of tasks split between multiple roles and
playbooks it can quickly add up.

These two adjustments alone can often halve the time needed for DebOps
to run.

Third, the ``site.yml`` playbook runs everything, but you don't always need
to run everything. Very often you can run only a subset of the
playbooks; so instead of ``debops run site``, you can run this:

.. code-block:: console

   debops run service/core srv app

Read :ref:`playbooks` for more information.

If your controlled machine is already configured, and you make
a change that affects only something very specific (e.g. a configuration
change that concerns only the :ref:`debops.owncloud` role), you can ask to run
only the relevant playbook:

.. code-block:: console

   debops run service/owncloud

The DebOps playbooks make extensive use of tags to enable you to further
narrow this down. This will only run tasks that configure nginx in the
context of ownCloud:

.. code-block:: console

   debops run service/owncloud --tags role::nginx

The :ref:`documentation of the DebOps roles <ansible_roles>` contains,
in the "Getting started" section of each role, an "Ansible tags" section
describing more tags that you can use.

.. _Ansible documentation on pipelining: https://docs.ansible.com/ansible/latest/collections/ansible/builtin/ssh_connection.html#parameter-pipelining


.. rubric:: Footnotes

.. [#f1] Well, almost all; some of the old roles might still mess with stuff
         outside of their scope, but we are working on fixing that. Stay tuned.
