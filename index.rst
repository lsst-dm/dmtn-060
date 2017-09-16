
.. Add content here.

#. :ref:`rucio-intro`
#. :ref:`rucio-testbed`
#. :ref:`rucio-service`
#. :ref:`rucio-rse1`


.. _rucio-intro:

Introduction
============

From the ATLAS project documentation, the Rucio project is the new version of ATLAS Distributed Data Management (DDM) system services. Rucio is used to manage accounts, files, datasets and distributed storage systems.  The DDM system manages LHC data within a heterogeneous distributed environment, and has demonstrated very large scale data managemnent: 1 billion file replicas and 255 PB across 130 sites. 

In our first examination we consider the installation, configuration, and testing of the main Rucio service and supporting database, Rucio Storage Elements (RSEs),  and Rucio client utilities. In particular we are interested to discover if there are inordinate challenges and/or barriers in installing and testing software such as Rucio from the ATLAS project outside of that ecosystem and within our prototype infrastructure. 

.. _rucio-testbed:

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


.. _rucio-service:

Rucio Service and Daemons 
=========================

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

.. _rucio-rse1:

Rucio Storage Elements 
======================

A Rucio Storage Element (RSE) is the construct by which Rucio addresses storage space. An RSE has 
a unique identifier, a name (as a string), a storage type (DISK/TAPE), and a set of meta attributes such
as a list of supported protocols, e.g., file, https, gsiftp, xrootd, srm, etc. 
The definition of the protocol has a :command:`prefix` that, for example, may specify the root directory
of the relevant storage area.  

RSE with Posix protocol
-----------------------

We start with a simple case defining an RSE with file/posix protocol on the main 
:file:`lsst-dbb-rucio.ncsa.illinois.edu` server itself. 
The definition of the RSE with file/posix protocol may be configured 
within Rucio database tables by using some brief scripting::


        #!/usr/bin/env python

        from rucio.core.rse import add_protocol
        from rucio.core.rse import add_rse
        from rucio.core.rse import add_rse_attribute
        from rucio.core.distance import add_distance
        from rucio.core.rse import list_rses

        dbb_posix = {   'hostname': 'lsst-dbb-rucio.ncsa.illinois.edu',
                        'scheme':   'file',
                        'prefix': '/usr/local/prefix',
                        'impl': 'rucio.rse.protocols.posix.Default',
                        'domains': {"lan": {"read": 1, "write": 1, "delete": 1},
                                    "wan": {"read": 1, "write": 1, "delete": 1} } }

        add_rse('MOCK1')
        add_rse_attribute('MOCK1', 'istape', False)

        add_protocol('MOCK1', parameter=dbb_posix)

We can use the :command:`rucio-admin` and :command:`rucio` command line clients to interact with the RSE::

        # echo $RUCIO_ACCOUNT
        root

        #  rucio-admin account set-limits root MOCK1  3000000000000
        Set account limit for account root on RSE MOCK1: 3.000 TB

        # rucio list-rses
        MOCK1

        # rucio list-rse-attributes MOCK1
        MOCK1: True
        istape: False

This RSE can be used to illustrate the upload of files to be managed by Rucio. 
Rucio has the concepts of Datasets (a named set of files) and 
Containers (a named set of Datasets or Containers) to serve as grouping/container constructs.
With the following :command:`upload` , a file :file:`Raw01.fits` is stored within
a dataset :file:`Dataset1` in the MOCK1 RSE::    


        #  rucio  upload  --rse MOCK1 --scope user.root user.root:Dataset1  Raw01.fits
        2017-09-16 13:27:06,857 WARNING user.root:Dataset1 cannot be distinguished from scope:datasetname. Skipping it.
        2017-09-16 13:27:08,215 INFO    Dataset successfully created
        2017-09-16 13:27:08,949 INFO    Adding replicas in Rucio catalog
        2017-09-16 13:27:09,667 INFO    Replicas successfully added
        2017-09-16 13:27:09,682 INFO    File user.root:Raw01.fits successfully uploaded on the storage
        2017-09-16 13:27:09,682 11718   INFO    File user.root:Raw01.fits successfully uploaded on the storage
        2017-09-16 13:27:11,850 INFO    Will update the file replicas states
        2017-09-16 13:27:11,850 11718   INFO    Will update the file replicas states
        2017-09-16 13:27:11,869 INFO    File replicas states successfully updated
        2017-09-16 13:27:11,869 11718   INFO    File replicas states successfully updated

The contents of the newly created :file:`Dataset1` can be listed. The file :file:`Raw01.fits` within
the Rucio system has a unique global identifier GUID::

        #  rucio   list-files   user.root:Dataset1
        Enter PEM pass phrase:
        +----------------------+--------------------------------------+-------------+------------+----------+
        | SCOPE:NAME           | GUID                                 | ADLER32     | FILESIZE   | EVENTS   |
        |----------------------+--------------------------------------+-------------+------------+----------|
        | user.root:Raw01.fits | 086FAAFA-2539-4096-9E4B-61D00D0C77E2 | ad:7a91116a | 3.187 kB   |          |
        +----------------------+--------------------------------------+-------------+------------+----------+
        Total files : 1
        Total size : 3.187 kB

While listing of files is independent of a particular RSE or location, one can also explictly list all
of the existing fie replicas for a regsitered file::


        #  rucio   list-file-replicas   user.root:Dataset1
        +-----------+------------+------------+-----------+----------------------------------------------------------------------------------------------+
        | SCOPE     | NAME       | FILESIZE   | ADLER32   | RSE: REPLICA                                                                                 |
        |-----------+------------+------------+-----------+----------------------------------------------------------------------------------------------|
        | user.root | Raw01.fits | 3.187 kB   | 7a91116a  | MOCK1: file://lsst-dbb-rucio.ncsa.illinois.edu:0/usr/local/prefix/user/root/06/00/Raw01.fits |
        +-----------+------------+------------+-----------+----------------------------------------------------------------------------------------------+



.. .. rubric:: References

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
