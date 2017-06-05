# iib-mq-chgUserid
This project displays how to write a flow and configure MQ to have IIB use different IDs against an MQ queue.
## IIB flow
The IIB flow consists of an HTTPInput node, a Compute node to set the user id for the queue and an MQOutput node to write to the queue.  
The flow uses an MQ client connection to connect to a remote queue manager.  The following 2 properties on the Advanced
tab of the MQOutput node must be changed:
    1. Message context - Set Identity
    2. Alternate user authority - checked.

NOTE: the MQ user identifier can be no longer than 12 characters.
## MQ
The following MQ objects are required:
    1. a remote queue manager (TEST)
    2. a TCP listener port on the queue manager (1414) 
    3. a SVRCONN channel for the client connection (IIB.SVRCONN)
    4. a userid for the flow to connect to MQ (iib).  I am assuming that this userid on the remote queue manager box is 
not a privileged ID and has minimal access as defined below.  A group is needed for the userid against which authorization will be done (iibg). 
    5. a userid that queue access will be checked against.  In my case, I have hard-coded this to *test* but this could be
calculated in the Compute node from some data passed in on the HTTP call.  This userid should also be in a group against which 
the authorization will be done (testg)
    6. a queue to access (T2). 

The following MQ commands are required:
    1. a channel (IIB.SVRCONN) where the MCAUSER is set to '*NOACCESS'.
    2. some kind of authentication.  Here I have used a simplified mechanism based on an AddressMap CHLAUTH rule.
        - SET CHLAUTH('IIB.SVRCONN') TYPE(ADDRESSMAP) ADDRESS('192.168.122.211') USERSRC(MAP) MCAUSER('iib') ACTION(ADD)

NOTE: you could force a userid and password to be specified on the MQOutput node by changing the CONNAUTH attribute of the queue manager or
changing the CHLAUTH to require a password ( CHCKCLNT(REQUIRED) ).  If you did this then you would need to specify a security identity like this:
        - mqsisetdbparms TEST -n mq::iibMqIdent -u test2 -p passw0rd
        
and then specifying iibMqIdent on the security identity field of the MQOutput node.
    3. the group iibg needs access to the queue manager and the ability to set and use an alternate userid.  You might need double quotes around the 
group name on Linux:
        - setmqaut -m TEST -t qmgr -g iibg -all
        - setmqaut -m TEST -t qmgr -g iibg +setall +altusr +connect +inq
    4. the group iibg needs access to set the context on the queue:
        - setmqaut -m TEST -n T2 -t q -g iibg -remove
        - setmqaut -m TEST -n T2 -t q -g iibg +setall
    5. the group testg needs put access to the queue T2
        - setmqaut -m TEST -n T2 -t q -g testg -remove
        - setmqaut -m TEST -n T2 -t q -g testg +put