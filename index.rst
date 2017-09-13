..
  Technote content.

  See https://developer.lsst.io/docs/rst_styleguide.html
  for a guide to reStructuredText writing.

  Do not put the title, authors or other metadata in this document;
  those are automatically added.

  Use the following syntax for sections:

  Sections
  ========

  and

  Subsections
  -----------

  and

  Subsubsections
  ^^^^^^^^^^^^^^

  To add images, add the image file (png, svg or jpeg preferred) to the
  _static/ directory. The reST syntax for adding the image is

  .. figure:: /_static/filename.ext
     :name: fig-label

     Caption text.

   Run: ``make html`` and ``open _build/html/index.html`` to preview your work.
   See the README at https://github.com/lsst-sqre/lsst-technote-bootstrap or
   this repo's README for more info.

   Feel free to delete this instructional comment.

:tocdepth: 1

.. Please do not modify tocdepth; will be fixed when a new Sphinx theme is shipped.

.. sectnum::

.. Add content below. Do not include the document title.

.. Add content here.


Introduction
============

From the ATLAS project documentation, the Rucio project is the new version of ATLAS Distributed Data Management (DDM) system services. Rucio is used to manage accounts, files, datasets and distributed storage systems.  The DDM system manages LHC data within a heterogeneous distributed environment, and has demonstrated very large scale data managemnent: 1 billion file replicas and 255 PB across 130 sites. 

In our first examination we consider the installation, configuration, and testing of the main Rucio service and supporting database, Rucio Storage Elements (RSEs),  and Rucio client utilities. In particular we are interested to discover if there are inordinate challenges and/or barriers in installing and testing software such as Rucio from the ATLAS project outside of that ecosystem and within our prototype infrastructure. 

Testbed within NCSA vSphere Environment
=======================================

The Rucio testing is performed within a testbed of VMs in our vSphere environment::

    lsst-dbb-rucio.ncsa.illinois.edu
    lsst-dbb-mq.ncsa.illinois.edu
    lsst-dbb-fts1.ncsa.illinois.edu
    lsst-dbb-fts2.ncsa.illinois.edu

The installations of Rucio and associated services are set up and maintained via Puppet modules
coordinated with NCSA admins. As such the examination goes beyond an individual's sandbox 
and possibly one-off installations, and considers issues of systematic installation and 
maintenance (e.g., automatic updates, firewalls, etc). 


Rucio Service and Daemons 
=================================

The installation of the main Rucio service entails the :command:`yum install` of several dependencies, while
Rucio itself can proceed via :command:`pip install rucio`. The version of the software that we currently use is
rucio 1.12.6. 


The main rucio service runs as a webapp in httpd via wsgi, 
and there are numerous Rucio daemons that are run separately and concurrently to add
functionality and features.  
Configuration of the service and various daemons presents some challenge, though starting with 
a core set  and extending incrementally seems to work best.
A template httpd conf file for :file:`/etc/httpd/conf.d/rucio.conf` and a template service/daemon configuration file for 
:file:`/opt/rucio/etc/rucio.cfg` are included with the Rucio installation.

The initial set of Rucio daemons that we start with is::

    rucio-transmogrifier
    rucio-judge-cleaner
    rucio-judge-evaluator
    rucio-judge-injector
    rucio-judge-repairer
    rucio-undertaker
    rucio-necromancer
    rucio-hermes
    rucio-kronos
    rucio-abacus-rse
    rucio-abacus-account

For authentication purposes with associated services, the main Rucio server possesses a host certificate
:file:`/etc/grid-security/hostcert.pem` (i.e., an X509 cert/key pair  :file:`hostcert.pem/hostkey.pem`).
For this step we obtain necessary host certificates from UIUC Technology Services, acquiring certificates
signed by the InCommon IGTF Certificate Authority. This choice is a 
reasonably good fit as it is in common use at NCSA (e.g., Blue Waters)
and is also included within the bundle of CAs present in the :file:`osg-ca-certs` package available from the Open Science Grid (OSG) 
:command:`yum` repositories apparently in use with OSG/ATLAS/Rucio on the US side.

Rucio relies on a supporting database. In the main ATLAS production
installation this is an Oracle database; in our initial investigation and prototyping
mariadb/mysql is used (MariaDB-server 10.0.32).  The setup of the database tables makes use of 
a utility :file:`reset_database.py` available in the Rucio installation, and additional
template configuration can be retrieved from :file:`https://gitlab.cern.ch/rucio01/rucio.git`.

Following the :file:`rucio` database initialization, a :command:`root` account exists, and
authenticating as the root user is prerequisite to further interaction with Rucio.
In our setup I configured use of a personal X509 certificate/key pair from the OSG, though several
other options are possible (username and password, a GSS/Kerberos token).

Having authenticated as the root user, one can make accounts, and scopes corresponding
to accounts (the scope serves to partition the namespace).  One can create additional accounts of 
type SERVICE (like a production account of an experiment), GROUP (such as a team within a project), and 
ordinary USER accounts (suitable for an individual). We will initially work with the root account 
and USER accounts. 

[More to follow.]



.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
