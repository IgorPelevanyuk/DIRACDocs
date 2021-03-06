-------------------------
Request Management System
-------------------------

:author:  Krzysztof Daniel Ciba <Krzysztof.Ciba@NOSPAMgmail.com>
:date:    Fri, 28th May 2013
:version: v6r9


System Overview
---------------

The Request Management System (RMS) is designed for management of simple operations that are performed 
asynchronously on behalf of users - owners of the requests. The RMS is used for multiple purposes: failure 
recovery (failover system), data management tasks and some others. It is designed as an open system easily 
extendible for new types of operations.  

Architecture and functionality
------------------------------

The core of the Request Management System is a `ReqDB` database which holds requests records together with 
all related data: operations that have to be performed in the defined order and possibly a set of files
attached. All avaiable and useful queries to the `ReqDB` are exposed to the request client (`ReqClient`) 
through `ReqManager` service.

.. image:: ../../../_static/Systems/RMS/ReqDBSchema.png
   :alt: ReqDB schema.
   :align: center 

Each table in the `ReqDB` has a corresponding class in the new API fully supporting CRUD operations. Each table column 
is exposed as a property in the related class. 

.. image:: ../../../_static/Systems/RMS/RequestZoo.png
   :alt: Request, Operation and File API.
   :align: center 

The record class is instrumented with the internal observer for its children (a `Request` instance is observing 
states for all defined `Operations`, each `Operation` is observing 
states of all its `Files`) and built in state machine, which automatizes state propagation:

 * state machine for `Request`

   .. image:: ../../../_static/Systems/RMS/RequestSTM.png
      :alt: State machine for Request.
      :align: center 

 * state machine for `Operation`

   .. image:: ../../../_static/Systems/RMS/OperationSTM.png
      :alt: State machine for operation.
      :align: center 

 * state machine for `File`

   .. image:: ../../../_static/Systems/RMS/FileSTM.png
      :alt: State machine for File.
      :align: center 

User is allowed to change only `File` statuses and in case of specific `Operation`'s types - `Operation` statuses, 
as `Request` builtin observers will automatically propagete and update statues of parent objects.


CRUD
----

Create
^^^^^^

Construction of a new request is quite simple, one has to create a new `Request` instance::

  >>> from DIRAC.RequestManagementSystem.Client.Request import Request
  >>> from DIRAC.RequestManagementSystem.Client.Operation import Operation
  >>> from DIRAC.RequestManagementSystem.Client.File import File
  >>> request = Request() # # create Request instance
  >>> request.RequestName = "foobarbaz"
  >>> operation = Operation() # # create new operation 
  >>> operation.Type = "ReplicateAndRegister"
  >>> operation.TargetSE = [ "CERN-USER", "PIC-USER" ]
  >>> opFile = File() # #  create File instance
  >>> opFile.LFN = "/foo/bar/baz" # # and fill some bits 
  >>> opFile.Checksum = "123456"
  >>> opFile.ChecksumType = "adler32"
  >>> operation.addFile( opFile ) # # add File to Operation
  >>> request.addOperation( operation ) # # add Operation to Request

Invoking `Request.addOperation` method will enqueue operation to the end of operatins list in the request. If you need 
to modify execution order, you can use `Request.insertBefore` or `Request.insertAfter` methods. 
Please notice there is no limit of `Operations` per `Request`, but it is not recommended to keep over there  
more than a few. In case of `Files` in a single `Operation` the limit is set to one hundred, which seems to 
be a reasonable number. In case you think this is not enough (or too much), please patch the code 
(look for `MAX_FILES` in `Opereation` class).     
 
The `Request` and `Operation` classes are behaving as any iterable python object, i.e. you can loop over operations 
in the request using::

  >>> for op in request: print op.Type
  ReplicateAndRegister
  >>> for opFile in operation: print opFile.LFN, opFile.Status, opFile.Checksum
  /foo/bar/baz Waiting 123456 
  >>> len( request ) # # number of operations 
  1
  >>> len( operation ) # # number of files in operation   
  1
  >>> request[0].Type # # __getitem__, there is also __setitem__ and __delitem__ defined for Request and Operation
  'ReplicateAndRegister'
  >>> operation in request # # __contains__ in Request
  True
  >>> opFile in operation # # __contains__ in Operation
  True

Once the request is ready, you can insert it to the `ReqDB`::

  >>> from DIRAC.RequestManagementSystem.Client.ReqClient import ReqClient
  >>> rc = ReqClient() # # create client
  >>> rc.putRequest( request ) # # put request to ReqDB

Read
^^^^

Reading request back can be done using two methods defined in the `ReqClient`:

  * for reading::

  >>> from DIRAC.RequestManagementSystem.Client.ReqClient import ReqClient
  >>> rc = ReqClient() # # create client
  >>> rc.peekRequest( "foobarbaz" ) # # get request from ReqDB for reading

  * for execution (request status on DB side will flip to 'Assigned')::

  >>> from DIRAC.RequestManagementSystem.Client.ReqClient import ReqClient
  >>> rc = ReqClient() # # create client
  >>> rc.getRequest( "foobarbaz" ) # # get request from ReqDB for execution

If you don't specify request name in `ReqClient.getRequest` or `ReqClient.peekRequest`, the one with "Waiting" 
status and the oldest `Request.LastUpdate` value will be chosen. 


Update
^^^^^^

Updating the request can be done by using methods that modify operation list::

  >>> del request[0] # # remove 1st operation using __delitem__
  >>> request[0] = Operation() # # overwrite 1st operation using __setitem__
  >>> request.addOperation( Operation() ) # # add new operation
  >>> request.insertBefore( Operation(), request[0] ) # # insert new operation at head
  >>> request.insertAfter( Operation(), request[0] ) # # insert new opration after 1st 
  
To make those changes persistent you should of course put modified and say dirty request back 
to the `ReqDB` using `ReqClient.putRequest`.  


Delete
^^^^^^

Nothing special here, just execute `ReqClient.deleteRequest( requestName )` to remove whole request from `ReqDB`. 


Request validation
------------------

The validation of a new Request that is about to enter the system for execution is checked at two levels:

  * low-level: each property in `Request`, `Operation` and `File` classes is instrumeted to check if value provided 
    to its setter has a meaningful type and value::

  >>> opFile.LFN = 1
  Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "DIRAC/RequestManagementSystem/private/Record.py", line 52, in __setattr__
    object.__setattr__( self, name, value )
  File "DIRAC/RequestManagementSystem/Client/File.py", line 137, in LFN
    raise TypeError( "LFN has to be a string!" )
  TypeError: LFN has to be a string!
  >>> operation.SubmitTime = False
  Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "DIRAC/RequestManagementSystem/private/Record.py", line 52, in __setattr__
    object.__setattr__( self, name, value )
  File "DIRAC/RequestManagementSystem/Client/Operation.py", line 370, in SubmitTime
    raise TypeError( "SubmitTime should be a datetime.datetime!" )
  TypeError: SubmitTime should be a datetime.datetime!


  * high-level: additionally there is also a request validator helper class (`RequestValidator` or its global 
    instance `gRequestValidator`) - a gatekeeper checking if request 
    is properly defined. The validator is blocking insertion of a new record to the `ReqDB` in case of missing or 
    malformed attrubutes and returning `S_ERROR` describing the reason for rejection, i.e.::

  >>> from DIRAC.RequestManagementSystem.private.RequestValidator import gRequestValidator
  >>> from DIRAC.RequestManagementSystem.Client.Request import Request
  >>> invalid = Request()
  >>> gRequestValidator.validate( invalid )
  {'Message': 'RequestName not set', 'OK': False}
  >>> invalid.RequestName = "foobarbaz"
  >>> gRequestValidator.validate( invalid )
  {'Message': "Operations not present in request 'foobarbaz'", 'OK': False}
  >>> from DIRAC.RequestManagementSystem.Client.Operation import Operation
  >>> invalid.addOperation( Operation() )
  {'OK': True, 'Value': ''}
  >>> gRequestValidator.validate( invalid )
  {'Message': "Operation #0 in request 'foobarbaz' hasn't got Type set", 'OK': False}
  >>> invalid[0].Type = "ForwardDISET"
  >>> gRequestValidator.validate( invalid )
  {'Message': "Operation #0 of type 'ForwardDISET' is missing Arguments attribute.", 'OK': False}


A word of caution has to be clearly stated over here: both low- and high-level validation is not checking if 
actual value provided during `Request` definition makes sense, i.e. if you put to the `Operation.TargetSE` unknown 
name of target storage element from the validation point of view your request will be OK, but  it will 
miserably fail during exection.    

Request execution
-----------------

The execution of the all possible requests is done in only one agent: `RequestExecutingAgent` using special set 
of handlers derived from `OperationHandlerBase` helper class. What is different from the previous attempt is 
the way the request is treated: the agent will try to execute request as a whole in one go, while in the old RMS  
there was several different agents in place, each trying to execute one sub-request type. This approach was
a horrible complication for maintain request's state machine. 

.. image:: ../../../_static/Systems/RMS/RequestExecution.png
   :alt: Treating of Request in the RequestExecutionAgent.
   :align: center 

The `RequestExecutingAgent` is using `ProcessPool` utility to create slave workers (subprocesses running `RequestTask`) 
desingnated to execute requests read from `ReqDB`. Each worker is processing request execution using following steps:
 
  * downloading and setting up request's owner proxy
  * loop over waiting operations in the request
  * creating on-demand and executing specific operation handler 
  * if operation status is not updated after treatment inside the handler, worker jumps out the loop 
    otherwise tries to pick up next waiting `Operation` 
    
Outside the main execution loop worker is checking request status and depending of its value finalizes request 
and puts it back to the ReqDB.

Extending
---------

At the moment of writing following operation types are supported:

  * DataManagement (under DMS/Agent/RequestOperations):

    - `PhysicalRemoval`
    - `PutAndRegister` 
    - `RegisterFile`
    - `RemoveFile`
    - `RemoveReplica`
    - `ReplicateAndRegister`
    - `ReTransfer`

  * RequestManagement (under RMS/Agent/RequestOperation)

    - `ForwardDISET`

This of course does not cover all possible needs for a specific VO, hence all developers are encouraged to create and keep
new operation handlers in VO spin-off projects. Definition of a new operation type should be easy withing context of 
the new RequestManagementSystem. All you need to do is to put in place operation handler (inherited from `OperationHandlerBase`) and/or
extend `RequestValidator` to cope with the new type. The handler should be a functor and should override two methods: 
constructor (__init__) and () operator ( __call__)::

    """ KillParrot operation handler """
    from DIRAC import gMonitor
    from DIRAC.RequestManagementSystem.private.OperationHandlerBase import OperationHandlerBase 
    import random

    class KillParrot( OperationHandlerBase ):
      """ operation handler for 'KillParrot' operation type

      see OperationHandlerBase for list of methods and DIRAC tools exposed 

      please notice that all CS options defined for this handler will 
      be exposed there as read-only properties

      """
      def __init__( self, request = None, csPath = None ):
        """ constructor -- DO NOT CHANGE its arguments list """
        # # AND ALWAYS call BASE class constructor (or it won't work at all)
        OperationHandlerBase.__init__(self, request, csPath )
        # # put there something more if you need, i.e. gMonitor registration
        gMonitor.registerActivity( "ParrotsDead", ... )
        gMonitor.registerActivity( "ParrotsAlive", ... )

      def __call__( self ):
        """ this has to be defined and should return S_OK/S_ERROR """
        self.log.info( "log is here" )
        # # and some higher level tools like ReplicaManager
        self.replicaManager().doSomething()
        # # request is there as a member 
        self.request 
        # # ...as well as Operation with type set to Parrot
        self.operation 
        # # do something with parrot 
        if random.random() > 0.5:
          self.log.error( "Parrot is still alive" )
          self.operation.Error = "It's only sleeping"
          self.operation.Status = "Failed"
          gMonitor.addMark( "ParrotsAlive" , 1 )
        else:
          self.log.info( "Parrot is stone dead")
          self.operation.Status = "Done"     
          gMonitor.addMark( "ParrotsDead", 1)
        # # return S_OK/S_ERROR (always!!!)
        return S_OK()
        
Once the new handler is ready you should also update config section 
for the `RequestExecutingAgent`::

    RequestExecutingAgent {
      OperationHandlers {
         # # Operation.Type
         KillParrot {
           # # add Location for new handler w.r.t. PYTHONPATH settings 
           Location = VODIRAC/RequestManagementSystem/Agent/RequestOperations/KillParrot
           ParrotsFoo = True
           ParrotsBaz = 1,2,3
         }
      }
    }    

Please notice that all CS options defined for each handler is exposed in it as read-only property. In the above example
`KillParrot` instance will have boolean `ParrotsFoo` set to `True` and `ParrotsBaz` list set to `[1,2,3]`. You can access 
them in the handler code using `self.ParrotsFoo` and `self.ParrotsBaz`, nothing special, except you can only read their values.
Any write attempt will raise `AttributeError` bailing out from request execution chain. 

From now on you can put the new request to the `ReqDB`::

  >>> request = Request()
  >>> operation = Operation()
  >>> operation.Type = "KillParrot"
  >>> request.addOperation( operation )
  >>> reqClient.putRequest( request )

and your brand new request with a new operation type would be eventually picked up and executed by the agent.

Installation
------------

1. Login to host, install `ReqDB`::

  dirac-install-db ReqDB

2. Install `ReqProxyHandler`::

  dirac-install-service RequestManagement/ReqProxy

Modify CS by adding::

  Systems {
    RequestManagement {
      URLs {
       ReqProxyURLs = dips://<hostA>:9191/RequestManagement/RequestProxy
      } 
    } 
  }

You need at least one of these - they are backing up new requests in case the `ReqManagerHandler` is down. Full description can be found in `ReqManager and ReqProxies`_.

3. Install `ReqManagerHandler`::

  dirac-install-service RequestManagement/ReqManager

4. Install `CleanReqDBAgent`::

  dirac-install-agent RequestManagement/CleanReqDBAgent

5. Install `RequestExecutingAgent`::

  dirac-install-agent RequestManagement/RequestExecutingAgent

If one `RequestExecutingAgent` is not enough (and this is a working horse replacing `DISETForwadingAgent`, 
`TransferAgent`, `RemovalAgent` and `RegistrationAgent`), clone it several times.

1. If VO is using FTS system, install `FTSDB`::

  dirac-install-db FTSDB

2. Stop `DataManagement/TransferDBMonitor` service and install `FTSManagerHandler`::

  runsvctrl d runit/DataManagement/TransferDBMonitor
  dirac-install-service DataManagement/FTSManager

3. Configure FTS sites using command `dirac-dms-add-ftssite`::

  dirac-dms-add-ftssite SITENAME FTSSERVERURL

In case of LHCb VO::

  dirac-admin-add-ftssite CERN.ch https://fts22-t0-export.cern.ch:8443/glite-data-transfer-fts/services/FileTransfer 50
  dirac-admin-add-ftssite CNAF.it https://fts.cr.cnaf.infn.it:8443/glite-data-transfer-fts/services/FileTransfer 50
  dirac-admin-add-ftssite PIC.es https://fts.pic.es:8443/glite-data-transfer-fts/services/FileTransfer 50
  dirac-admin-add-ftssite RAL.uk https://lcgfts.gridpp.rl.ac.uk:8443/glite-data-transfer-fts/services/FileTransfer 50
  dirac-admin-add-ftssite SARA.nl https://fts.grid.sara.nl:8443/glite-data-transfer-fts/services/FileTransfer 50
  dirac-admin-add-ftssite NIKHEF.nl https://fts.grid.sara.nl:8443/glite-data-transfer-fts/services/FileTransfer 50
  dirac-admin-add-ftssite GRIDKA.de https://fts-fzk.gridka.de:8443/glite-data-transfer-fts/services/FileTransfer 50
  dirac-admin-add-ftssite IN2P3.fr https://cclcgftsprod.in2p3.fr:8443/glite-data-transfer-fts/services/FileTransfer 50
 
4. Install `CleanFTSDBAgent`::

  dirac-install-agent DataManagement/CleanFTSDBAgent


5. Install `FTSAgent`::

  dirac-install-agent DataManagement/FTSAgent

Again, as in case of `RequestExecutingAgent`, if one instance is not enough, you should probably clone it several times.


7. Once all requests from old version of system are processed, shutdown and remove agents:: 

  RequestManagement/DISETForwardingAgent
  RequestManagement/RequestCleaningAgent
  DataManagement/TransferAgent
  DataManagement/RegistrationAgent
  DataManagement/RemovalAgent

and services::

  RequestManagement/RequestManager
  RequestManagement/RequestProxy
  DataManagement/TransferDBMonitor

and dbs::

  RequestManagement/RequestDB
  DataManagement/TransferDB

-------------------------
ReqManager and ReqProxies
-------------------------

:author:  Krzysztof Daniel Ciba <Krzysztof.Ciba@NOSPAMgmail.com>
:date:    Fri, 18th Jan 2013
:version: first

Overview
--------

The `ReqManager` service is a handler for `ReqDB` using DISET protocol. It exposes all CRUD operations on requests (creating, reading, 
updating and deleting) plus several helper functions like getting requests/operation attributes, exposing some useful information 
to the web interface/scripts and so on.

The `ReqProxy` is a simple service which start to work only if `ReqManager` is down for some reason and newly created requests cannot be
inserted to the `ReqDB`. In such case the `ReqClient` is sending them  to one of the `ReqProxies`, where
the request is serialised and dumped ot the file in the local file system for further processing. A separate background thread in the 
`ReqProxy` is periodically trying to connect to the `ReqManager`, forwading saved requests to the place they can 
be eventually picked up for execution.  

.. image:: ../../../_static/Systems/RMS/RequestProxy-flow.png
   :alt: Request's forwarding in DIRAC.
   :align: center 

Installation
------------

For the proper request processing there should be only one central instance of the `ReqManager` 
service up and running - prefereably close to the hosts on which request processing agents are running. 

For the `RequestProxies` situation is quite opposite: they should be installed in the several different places 
all over the world, preferably close to the biggest CEs or SEs used by the community. Take the LHCb VO as 
an example, where each of Tier1 is running its own `ReqProxy`. Notice that you have to have at least one `ReqProxy` 
running somewhere for normal operation, preferably not sharing the host used by the `ReqManager` service. 

Example configuration::

  Systems { 
    RequestManagement {
      Services {
        RequestManager {
          LogLevel = INFO
          HandlerPath = DIRAC/RequestManagementSystem/Service/RequestManagerHandler.py
          Port = 9143
          Protocol = dips
          Backend = mysql
          Authorization {
            Default = authenticated
          }
        RequestProxy {
          LogLevel = INFO
          HandlerPath = DIRAC/RequestManagementSystem/Service/RequestProxyHandler.py
          Port = 9161
          Protocol = dips
          Authorization {
            Default = authenticated
          }
        }
      }
      URLs {
        ## the only instance of RequestManagerHandler
        RequestManager = dips://<central>:9143/RequestManagement/RequestManager
        ## comma separated list to all RequestProxyHandlers
        RequestProxyURLs = dips://<hostA>:9161/RequestManagement/RequestProxy, dips://<hostB>:9161/RequestManagement/RequestProxy
      }
    }
  }

 
Don't forget to put correct FQDNs instead of <central>, <hostA>, <hostB> in above example!   


Upgrading from DIRAC v6r5
-------------------------

The DIRAC releases prior to v6r6 were using different model for request forwarding: each CE 
was able to run its own RequestManager (local), preferebly with the file backend (which BTW is obsolete now). 
Requests created by jobs were put to the local RequestDB using local RequestManager URL. A separate locally running 
ZuziaAgent [#]_ was picking them up and sending to the central RequestManager service.

For upgrading from the old to the new forwarding model you should follow this procedure:

* install and configure the RequestProxy service in all the places where ZuziaAgent is running, make sure 
  the port used by this service (9161) is visible to the outside world
* stop ZuziaAgent when no more requests are held in the local RequestDB
* stop local RequestManager
* update DIRAC configuration by adding RequestProxy FQDN to the RequestProxyURLs 
  and removing ZuziaAgent and local RequestManager sections 
  

.. rubric:: Footnotes
 
.. [#] Zuzia is a little Susan in Polish, the given name of a daugther of DIRAC team developer who had left the project a few years ago. 


