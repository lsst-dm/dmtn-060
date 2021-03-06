
.. Add content here.

Outline
=======

#. Rucio
#. FTS3
#. IRODS
#. Globus
#. Grid Community Toolkit


Rucio
=====

Introduction
------------

Rucio is a project that provides services and associated libraries for allowing scientific 
collaborations to manage large volumes of data spread across facilities 
at multiple institutions and organisations. 
Rucio has been developed by the ATLAS experiment.
Rucio is used to manage accounts, files, datasets and distributed storage systems.  
The DDM system manages LHC data within a heterogeneous distributed environment, and has demonstrated very large scale data managemnent: 1 billion file replicas and 255 PB across 130 sites. 

In our first examination we consider the installation, configuration, and testing of the main Rucio service and supporting database, Rucio Storage Elements (RSEs),  and Rucio client utilities. In particular we are interested to discover if there are inordinate challenges and/or barriers in installing and testing software such as Rucio from the ATLAS project outside of that ecosystem and within our prototype infrastructure. 

Testbed within NCSA LSST vSphere Environment
--------------------------------------------

The Rucio testing is performed within a testbed of VMs in the NCSA LSST vSphere environment::

    lsst-dbb-rucio.ncsa.illinois.edu
    lsst-dbb-mq.ncsa.illinois.edu
    lsst-dbb-fts1.ncsa.illinois.edu
    lsst-dbb-fts2.ncsa.illinois.edu

The installations of Rucio and associated services are set up and maintained via Puppet modules
coordinated with NCSA admins. As such the examination goes beyond an individual's sandbox 
and possibly one-off installations, and considers issues of systematic installation and 
maintenance (e.g., automatic updates, firewalls, etc). 


Rucio Service and Daemons 
-------------------------

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

Rucio Storage Elements 
----------------------

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

We can use the :command:`rucio-admin` and :command:`rucio` command line clients to administer
and perform listings related to with the RSE::

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


RSE with gsiftp protocol
------------------------

In order to demonstrate a more generally usable, accessible Rucio storage element, we consider 
an instance  based on gsiftp protocol. This RSE will support, for example, upload and download
from various remote hosts (i.e., clients other than the main Rucio server).  To achieve this 
we install a 
`globus-gridftp-server <https://opensciencegrid.github.io/docs/data/gridftp/>`_
as available from the 
`Open Science Grid <https://www.opensciencegrid.org/>`_
`yum repositories <https://opensciencegrid.github.io/docs/common/yum/>`_ 
utlizing the testbed machine :file:`lsst-dbb-fts1.ncsa.illinois.edu` as the installation host.
It is efficient to install the OSG service as it has ready support 
for the Adler32 checksum algorithm that Rucio utilizes::

        # uberftp lsst-dbb-fts1.ncsa.illinois.edu
        220 lsst-dbb-fts1.ncsa.illinois.edu GridFTP Server 11.8 (gcc64, 1477634051-85) [Globus Toolkit 6.0] ready.
        230 User daues logged in.
        UberFTP (2.8)> quote cksm Adler32 0 -1 /tmp/hello_world
        213 3880065f

The RSE can be defined within the Rucio system using some brief scripting that loads
entries into appropriate database tables::

        #!/usr/bin/env python
        
        from rucio.core.rse import add_protocol
        from rucio.core.rse import add_rse
        from rucio.core.rse import add_rse_attribute
        from rucio.core.rse import list_rses
        
        dbb_gsiftp = {  'hostname': 'lsst-dbb-fts1.ncsa.illinois.edu',
                        'scheme':   'gsiftp',
                        'port':   '2811',
                        'prefix': '/usr/local/data',
                        'impl': 'rucio.rse.protocols.gfal.Default',
                        'domains': {"lan": {"read": 1, "write": 1, "delete": 1},
                                    "wan": {"read": 1, "write": 1, "delete": 1} } }
        add_rse('NCSA_DATADISK')
        add_protocol('NCSA_DATADISK', parameter=dbb_gsiftp)

We can upload a file :file:`Calib05.fits` into the Rucio system, targeting this RSE and a Dataset user.root:upload5Cal::


        # rucio  upload --rse NCSA_DATADISK  --summary  user.root:upload5Cal Calib05.fits
        2017-09-04 20:30:27,542 WARNING user.root:upload5Cal cannot be distinguished from scope:datasetname. Skipping it.
        2017-09-04 20:30:28,332 INFO    Dataset successfully created
        2017-09-04 20:30:28,935 INFO    Adding replicas in Rucio catalog
        2017-09-04 20:30:29,683 INFO    Replicas successfully added
        2017-09-04 20:30:29,964 INFO    File user.root:Calib05.fits successfully uploaded on the storage
        2017-09-04 20:30:29,964 23024   INFO    File user.root:Calib05.fits successfully uploaded on the storage
 
The physical file name (PFN) of the replica in this RSE can be displayed with::

        # rucio list-file-replicas  user.root:upload5Cal
        +-----------+--------------+------------+-----------+----------------------------------------------------------------------------------------------------------+
        | SCOPE     | NAME         | FILESIZE   | ADLER32   | RSE: REPLICA                                                                                             |
        |-----------+--------------+------------+-----------+----------------------------------------------------------------------------------------------------------|
        | user.root | Calib05.fits | 5.825 kB   | cbf8eff3  | NCSA_DATADISK: gsiftp://lsst-dbb-fts1.ncsa.illinois.edu:2811/usr/local/data/user/root/ab/38/Calib05.fits |
        +-----------+--------------+------------+-----------+----------------------------------------------------------------------------------------------------------+

Rucio Client Installation
-------------------------

While the main Rucio server :file:`lsst-dbb-rucio.ncsa.illinois.edu` runs the central web application and numerous Rucio daemons
(and the supporting database in our testbed), we would like to verify that a system serving as a Rucio client can function 
with relatively lightweight installation and configuration. Utilizing a fairly minimal CentOS7 instance within the NCSA Nebula OpenStack,
we observe that clients can be set up with a few steps::

        # yum -y install gcc python-devel openssl-devel
        # yum -y install epel-release
        # yum -y install python-pip
        # pip install --upgrade pip
        # pip install rucio-clients

        # yum -y install gfal2-devel gfal2-util gfal2-all

The `gfal2 <https://dmc.web.cern.ch/projects/gfal-2/home>`_ (Grid File Access Library) of CERN
is a library that is described to provide an abstraction layer over grid storage system complexity.
It simplifies file operations within a distributed environment by hiding complexity behind a simple posix API.
The installation of :file:`gfal2-all` will install a collection of plugins that can handle
a variety of transfer protocols that one may wish to use within Rucio client operations
(e.g., upload, download, etc)::

        gfal2-plugin-dcap
        gfal2-plugin-file
        gfal2-plugin-gridftp
        gfal2-plugin-http
        gfal2-plugin-lfc7
        gfal2-plugin-rfio
        gfal2-plugin-sftp
        gfal2-plugin-srm
        gfal2-plugin-xrootd

In principle only the plugins for protocols to be used on a given client are required, though
they are bundled conveniently within :file:`gfal2-all`.  As such a system can undergo setup 
and configuration to serve as a Rucio client in a quick, lightweight manner. An example download
from our Nebula instance proceeds::

        # rucio download user.root:Flat01.fits
        2017-09-07 04:44:02,863 INFO    Using rucio downloader...
        2017-09-07 04:44:02,863 INFO    Thread 1/1 : Starting the download of user.root:Flat01.fits
        2017-09-07 04:44:03,641 INFO    Thread 1/1 : File user.root:Flat01.fits trying from NCSA_DATADISK
        2017-09-07 04:44:03,972 INFO    Thread 1/1 : File user.root:Flat01.fits successfully downloaded from NCSA_DATADISK
        2017-09-07 04:44:04,798 INFO    Thread 1/1 : File user.root:Flat01.fits successfully downloaded. 4.807 kB in 1.93 seconds = 0.0 MBps
        ----------------------------------
        Download summary
        ----------------------------------------
        DID user.root:Flat01.fits
        Total files :                                 1
        Downloaded files :                            1
        Files already found locally :                 0
        Files that cannot be downloaded :             0


Current View
------------

We find that the ATLAS project had made very good progress in packaging and organizing the software 
for other projects to use (e.g., software available by standard means, 
copious Read The Docs style documentation, etc) and we were also able to obtain assistance 
from teams in the community in the U.S. that are using Rucio (thanks to University of Chicago).   
A full measure of suitability of Rucio for LSST DM is dependent on the creation of 
well-defined data backbone use cases (in progress). We also continue the examination of
Rucio and its software ecosystem by taking a detailed look at FTS3 service and file transfer. 


FTS3
====

FTS3 (main site http://fts.web.cern.ch , docs http://fts3-docs.web.cern.ch/fts3-docs/ )
is the reliable file transfer service that is used by experiments at CERN (ATLAS, CMS, etc) to
transfer data globally within the Worldwide LHC Computing Grid (WLCG). 
As such it is a mature, tested service that transfers petabytes of data per month and 
provides monitoring and summaries of transfers that have been submitted for management.
FTS3 utilizes GFAL2 (https://dmc.web.cern.ch/projects/gfal-2/home) to implement transfers for numerous commonly used protocols with a single API. 

Our evaluation of Rucio and its software ecosystem described in the section above 
has recently continued with a focused examination 
of FTS3, which services the file transfer aspects of data management with Rucio. 
We have installed a running instance of FTS3 at NCSA and started initial testing of transfers. 
We have set up the configuration so that collaborators at CC-IN2P3 can also begin to 
submit file transfer requests to the service. In this way NCSA & CC-IN2P3 will jointly 
test FTS3 for suitability for managing NCSA/CC-IN2P3 file transfers in the future. 
Discussions with the SciTokens effort (Jim Basney, Brian Bockelman, et al) are forthcoming.
In a recent conference SciTokens reported success in utilizing capability-based 
OAuth2 tokens with FTS3 to transfer files 
over https protocol, and we are interested in this capability for the LSST DM context.


IRODS
=====

The integrated Rule-Oriented Data System (iRODS) (https://irods.org) is open source data management software
used by research organizations and government agencies worldwide. iRODS is released
as a production-level distribution aimed at deployment in mission critical environments.

The iRODS Consortium brings together businesses, research organizations, universities,
and government agencies to ensure the sustainability of iRODS.
The Consortium maintains and supports a commercial-grade distribution of iRODS.
The iRODS Consortium fields a team of software developers, application engineers,
and support staff housed at RENCI at the University of North Carolina at Chapel Hill.

In 2015 we examined iRODS for data/file management features (captured in a set of issues
DM-2439, DM-2441, DM-2442, DM-2572, DM-2692). We found interesting the 
core competencies:

* iRODS implements data virtualization, allowing access to distributed storage assets under a unified logical namespace,
* iRODS automates data workflows, with a rule engine that permits any action to be initiated by any trigger on any server or client in the Zone.

The specific scenarios that we set up in testing were 

* implementing a rule for automatic replication to remote sites following upload of data into iRODS
* enforcing a rule to detect and repair file corruption (good copy on one site replacing corrupted copy on another) on a regular cadence 

Globus
=================================

Globus is a leading provider of secure, reliable research data management services.
Amongst the capabilities  it enables are the ability to transfer files reliably,
to share files with others, and to publish data.

We have experience with the Globus cloud service ('globus online'/GO) and Globus Connect
in development scenarios and in first prototypes, e.g., of disaster recovery. 
GO endpoints such a lsst#lsst-xfer and ncsa#Nearline have served as gateways 
to development GPFS file systems and the NearLine tape archive on the Blue Waters system.
In testing we have utilized globus online for file tranfers within Pegasus workflows.
Globus online has been a useful tool for individual users to initiate a bulk file tranfer
'interactively'/through a web interface. Such transfers are launched into the background
and a user can monitor and be updated with email on the status of the transfer. 


Grid Community Toolkit
======================

The Grid Community Forum (GridCF) (https://gridcf.org) is a community
organized to provide support for core grid software.  The GridCF attempts to support
a software stack christened the Grid Community Toolkit (GCT).  The GCT is an
open-source fork of the Globus Toolkit created by the Globus Alliance.
The official git repository for the Grid Community Toolkit is at
https://github.com/gridcf/gct .

The support for open source Globus Toolkit by the Globus Alliance ended as of January 2018. 
We have significant experience in the use of open source Globus toolikit
over a number of years.  This included running globus-gridftp-server's in front of project storage 
elements as well as utilizing pools of globus-gridftp-server's provided at HPC sites 
as NCSA, SDSC, TACC, NERSC, 
FermiGrid, etc. Use of globus-url-copy on the command line allows for customization 
of parameters such as the 
number of threads, tcp windows size, etc., and enables high performance file transfers. 

.. Make in-text citations with: :cite:`bibkey`.

.. .. bibliography:: local.bib lsstbib/books.bib lsstbib/lsst.bib lsstbib/lsst-dm.bib lsstbib/refs.bib lsstbib/refs_ads.bib
..    :encoding: latex+latin
..    :style: lsst_aa
