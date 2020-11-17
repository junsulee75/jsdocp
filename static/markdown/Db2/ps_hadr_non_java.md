
Objectives
====


You configured DB2 pureScale on server side and now, may wonder how to handle connection rerouting in failure scenarios like below.   

- A member down on pureScale primary cluster.  
  Of course, you won't expect to takeover the connection to standby but just to takeover to other live members in the same cluster.
- Whole primary cluster is down. THat means application cannot access any of members there.  
  In this case, connections should be routed to the standby cluster assuming DBA did HADR takeover in advance accordingly.  
- Then while applications run on new primary pureScale cluster (original standby), it should be able to take over to another member within the same cluster when it fails too.  
- Now, you recovered the original primary cluster and let it join HADR as standby and make the HADR status as 'PEER'.  
  In most cases, DBAs are likely to bring the HADR primary back to original cluster.  
  Then, the connections should be able to come back to original cluster once hadr takeover is issued.

How does that sound ? Complicated ? 
Still not sure how to achieve that exactly even after studying related properties from many documents ?    
Hard to get examples even searching with Google ?  Just want to get a handy configuration that has been tested at least in real case examples ?   

Here, let me share hands on examples.    
I am not saying the configuration here is the best practice but at least can be referred as baseline experiences helping you to proceed your own test on your demand.  


This page explains the scenario with non java application and client affinity scenario, not WLB (workload balancing).   
Of course, WLB is supported very well with non java application, however, I think the more popular and realistic scenarios are using a designated member with non java application such as running a remote CLP based batch work etc.   

Then, let's test it out.   

Test systems
=====
SERVER
---
* Primary pureScale cluster
"DB2 v11.5.0.0", "special_39945"
```
jsps1 - member 0
jsps2 - member 1
jsps3 - CF 128
jsps4 - CF 129
```

* Standy pureScale cluster
"DB2 v11.5.0.0", "special_39945"
```
shps011 - member 0
shps012 - member 1
shps013 - CF 128
shps014 - CF 129
```

* INSTANCE : `db2inst1`,  DB : `HADRDB`, port : `50000`  

CLIENT
---
* pureScale non java client
  DB2 v11.1.2.2
```
lamps1.fyre.ibm.com 
```

TEST
===

HADR configuration review before starting tests
---

* jsps1 - member 0 

```
db2inst1@jsps1:/home/db2inst1 $ db2pd -db hadrdb -hadr

Database Member 0 -- Database HADRDB -- Active -- Up 14 days 19:56:36 -- Date 2020-10-04-20.19.19.449161

                            HADR_ROLE = PRIMARY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 1
                        LOG_STREAM_ID = 0
                           HADR_STATE = PEER
                           HADR_FLAGS = TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = jsps1
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 0
                  STANDBY_MEMBER_HOST = shps011
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
                  ...
```

* jsps2 - member 1

```
Database Member 1 -- Database HADRDB -- Active -- Up 14 days 20:41:41 -- Date 2020-10-04-21.00.48.459548

                            HADR_ROLE = PRIMARY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 1
                        LOG_STREAM_ID = 1
                           HADR_STATE = PEER
                           HADR_FLAGS = TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = jsps2
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 1
                  STANDBY_MEMBER_HOST = shps011
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
                  ...
```

* shps011 - member 0 (Replaying standby)
```
Database Member 0 -- Database HADRDB -- Standby -- Up 6 days 14:45:01 -- Date 2020-10-04-21.02.43.851049

                            HADR_ROLE = STANDBY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 0
                        LOG_STREAM_ID = 0
                           HADR_STATE = PEER
                           HADR_FLAGS = TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = jsps1
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 0
                  STANDBY_MEMBER_HOST = shps011
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
                  ...

                            HADR_ROLE = STANDBY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 0
                        LOG_STREAM_ID = 1
                           HADR_STATE = PEER
                           HADR_FLAGS = TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = jsps2
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 1
                  STANDBY_MEMBER_HOST = shps011
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
                  ...
```

* shps012 - member 1 (Expected as it's not hadr standby replay member for now. )   
```
db2inst1@shps012:/home/db2inst1 $ db2pd -db hadrdb -hadr

Database HADRDB not activated on database member 1 or this database name cannot be found in the local database directory.

Option -hadr requires -db <database> or -alldbs option and active database.
```

CLIENT CONFIGURATION
--
This is the main interest in this page.   
While there had been multiple example configurations that work successfully, I introduce one example that I think simpler that others.  

First of all, someone may have the perception that it may be necessary to configure node and database catalog on the client side such as using the following commands.   
```
db2 catalog tcpip node psnode remote jsps1 server 50000 remote_instance db2inst1
db2 catalog database hadrdb at node psnode
```   
That is incorrect.  
We don't need those steps but we will only use  `~/sqllib/cfg/db2dsdriver.cfg`.   
However, having the catalog node/db does not matter either that will not hurt the test results on this page.      

To utilize relevant host names, added the below on `/etc/hosts` on the client host.   

```
172.16.165.63 jsps1.fyre.ibm.com jsps1 # priamry pureScale member 0
172.16.167.133 jsps2.fyre.ibm.com jsps2 # primary pureScale member 1

172.16.230.93 shps011.fyre.ibm.com shps011  #standby pureScale member 0
172.16.36.121 shps012.fyre.ibm.com shps012  #standby pureScale member 1
```

Then all I need is configuring `~/sqllib/cfg/db2dsdriver.cfg` like below.     
The key combination is `alternateserverlist` and `alternategroup` for routing the connection across the clusters.      

```xml
db2inst1@lamps1 ~ $ cat ~/sqllib/cfg/db2dsdriver.cfg
<configuration>
  <dsncollection>
     <dsn alias="HADRDB" name="HADRDB" host="jsps1" port="50000"/>
  </dsncollection>
  <databases>
    <database name="HADRDB" host="jsps1" port="50000">
      <parameter name="keepAliveTimeOut" value="10"/>
      <parameter name="MemberConnectTimeout" value="2"/>
      <acr>
        <parameter name="enableACR" value="true"/>
        <parameter name="maxAcrRetries" value="30"/>
        <parameter name="acrRetryInterval" value="3"/>
        <parameter name="enableAlternateServerListFirstConnect" value="true"/>
        <parameter name="enableSeamlessAcr" value="true"/>
        <alternateserverlist>
          <server name="m0" hostname="jsps1" port="50000"/>
          <server name="m1" hostname="jsps2" port="50000"/>
        </alternateserverlist>
        <alternategroup>
         <parameter name="enableAlternateGroupSeamlessAcr" value="true"/>
         <database name="HADRDB" host="shps011" port="50000"/>
        </alternategroup>
      </acr>
    </database>
  </databases>
</configuration>
```

Here, main points are having member hosts in the primary cluster in the following section.   

```xml
...
        <alternateserverlist>
          <server name="m0" hostname="jsps1" port="50000"/>
          <server name="m1" hostname="jsps2" port="50000"/>
        </alternateserverlist>
...
```

And the putting replay standby host within `<alternategroup>` section.   

```xml
...
        <alternategroup>
         <parameter name="enableAlternateGroupSeamlessAcr" value="true"/>
         <database name="HADRDB" host="shps011" port="50000"/>
        </alternategroup>
...
```

The command will validate the semantics of the configuration file.  
```
db2inst1@lamps1 ~ $ db2cli validate

===============================================================================
Client information for the current copy:
===============================================================================

Client Package Type       : IBM DB2 Connect Server
Client Version (level/bit): DB2 v11.1.2.2 (s1706091900/64-bit)
Client Platform           : Linux/X8664
Install/Instance Path     : /opt/ibm/db2/V11.1
DB2DSDRIVER_CFG_PATH value: <not-set>
db2dsdriver.cfg Path      : /home/db2inst1/sqllib/cfg/db2dsdriver.cfg
DB2CLIINIPATH value       : <not-set>
db2cli.ini Path           : /home/db2inst1/sqllib/cfg/db2cli.ini
db2diag.log Path          : /home/db2inst1/sqllib/db2dump/db2diag.log

===============================================================================
db2dsdriver.cfg schema validation for the entire file:
===============================================================================

Success: The schema validation completed successfully without any errors.

===============================================================================
The validation is completed.
===============================================================================
```


SERVER CONFIGURATION
---
Basically nothing to configure for the client reroute.    
Someone may wonder if `alternate server` is necessary to configure on the server directory list from his experience doing with traditional single based HADR system.   
Again, no.   All configuration will be on the client side.   
I will make it sure there is no `alternate server` on both primary and standby again for this test.   


```
db2inst1@jsps1:/home/db2inst1/work $ db2 update alternate server for database HADRDB using  hostname NULL port NULL
DB20000I  The UPDATE ALTERNATE SERVER FOR DATABASE command completed 
successfully.
DB21056W  Directory changes may not be effective until the directory cache is 
refreshed.


db2inst1@jsps1:/home/db2inst1/work $ db2 list db directory

 System Database Directory

 Number of entries in the directory = 2

Database 1 entry:

 Database alias                       = HADRDB
 Database name                        = HADRDB
 Local database directory             = /db2sd_20200603021640/db2inst1/dbpath
 Database release level               = 15.00
 Comment                              =
 Directory entry type                 = Indirect
 Catalog database partition number    = 0
 Alternate server hostname            =
 Alternate server port number         =
```

```
 db2inst1@shps011:/home/db2inst1 $ db2 update alternate server for database HADRDB using  hostname NULL port NULL
DB20000I  The UPDATE ALTERNATE SERVER FOR DATABASE command completed 
successfully.
DB21056W  Directory changes may not be effective until the directory cache is 
refreshed.
db2inst1@shps011:/home/db2inst1 $ db2 list db directory

 System Database Directory

 Number of entries in the directory = 2

Database 1 entry:

 Database alias                       = HADRDB
 Database name                        = HADRDB
 Local database directory             = /db2sd_20200905013958/db2inst1
 Database release level               = 15.00
 Comment                              =
 Directory entry type                 = Indirect
 Catalog database partition number    = 0
 Alternate server hostname            =
 Alternate server port number         =
```

CLIENT TEST APPLICATION
--------

Used a simple shell script to connect to the database and run the SQL showing the current connected server hostname continuosly.   

```Shell
db2inst1@lamps1 ~/test/non_java_hadr $ cat non_java_test.sh
#!/bin/ksh

time db2 connect to hadrdb user db2inst1  using xxxxx 

while true 
do
echo =================
date
time db2 "select varchar(host_name,20) current_hostname from sysibmadm.env_sys_info"

sleep 5
done


db2 terminate


```

Running the client test shell script.   
```
db2inst1@lamps1 ~/test/non_java_hadr $ sh ./non_java_test.sh

   Database Connection Information

 Database server        = DB2/LINUXX8664 11.5.0.0
 SQL authorization ID   = DB2INST1
 Local database alias   = HADRDB


real	0m1.150s
user	0m0.020s
sys	0m0.022s

=================
Mon Oct  5 16:09:55 PDT 2020

CURRENT_HOSTNAME    
--------------------
jsps1.fyre.ibm.com  

  1 record(s) selected.


real	0m2.259s
user	0m0.016s
sys	0m0.020s

=================
Mon Oct  5 16:10:02 PDT 2020

CURRENT_HOSTNAME    
--------------------
jsps1.fyre.ibm.com  

  1 record(s) selected.


real	0m2.046s
user	0m0.017s
sys	0m0.019s
...
```

![Preprocessing](static/img/nonjava_pshadr/nonjava_pshadr_01_normal_initial.png)

Test scenario #01 : pureScale hadr primary member 0 stop
---


- ACTION 
```
db2inst1@jsps1:/home/db2inst1 $ date ; db2stop member 0 force ; date
Mon Oct  5 16:20:37 PDT 2020
10/05/2020 16:21:00     0   0   SQL1064N  DB2STOP processing was successful.
SQL1064N  DB2STOP processing was successful.
Mon Oct  5 16:21:00 PDT 2020

```

- RESULT : The connection is rerouted to othere member 1 within  the primary cluster.  
![Preprocessing](static/img/nonjava_pshadr/nonjava_pshadr_02_member0_down.png)

```
...

=================
Mon Oct  5 16:20:42 PDT 2020
SQL30108N  A connection failed in an automatic client reroute environment. The 
transaction was rolled back. Host name or IP address: "jsps2.fyre.ibm.com". 
Service name or port number: "50000". Reason code: "1". Connection failure 
code: "2". Underlying error: "*".  SQLSTATE=08506

real	0m9.009s
user	0m0.011s
sys	0m0.025s

=================
Mon Oct  5 16:20:56 PDT 2020

CURRENT_HOSTNAME    
--------------------
jsps2.fyre.ibm.com  

  1 record(s) selected.


real	0m1.207s
user	0m0.011s
sys	0m0.024s

=================
Mon Oct  5 16:21:02 PDT 2020

CURRENT_HOSTNAME    
--------------------
jsps2.fyre.ibm.com  

  1 record(s) selected.


real	0m1.045s
user	0m0.016s
sys	0m0.022s

...

```



Test scenario #02 : pureScale hadr primary member 1 stop
---
This is for testing if the connection comes back to member 0 when member 1 is down.  

- ACTION : start member 0 again and stop member 1 where the connection is connected currently.    

```
db2inst1@jsps1:/home/db2inst1 $ db2start member 0
10/05/2020 16:27:04     0   0   SQL1063N  DB2START processing was successful.
SQL1063N  DB2START processing was successful.
db2inst1@jsps1:/home/db2inst1 $ 
db2inst1@jsps1:/home/db2inst1 $ date ; db2stop member 1 force ; date
Mon Oct  5 16:27:20 PDT 2020
10/05/2020 16:27:46     1   0   SQL1064N  DB2STOP processing was successful.
SQL1064N  DB2STOP processing was successful.
Mon Oct  5 16:27:46 PDT 2020
```


- RESULT : Take over back to member 0.   
![Preprocessing](static/img/nonjava_pshadr/nonjava_pshadr_03_member1_down.png)

```
...

=================
Mon Oct  5 16:27:13 PDT 2020

CURRENT_HOSTNAME    
--------------------
jsps2.fyre.ibm.com  

  1 record(s) selected.


real	0m2.065s
user	0m0.018s
sys	0m0.025s

=================
Mon Oct  5 16:27:20 PDT 2020
SQL30108N  A connection failed in an automatic client reroute environment. The 
transaction was rolled back. Host name or IP address: "jsps1.fyre.ibm.com". 
Service name or port number: "50000". Reason code: "1". Connection failure 
code: "2". Underlying error: "104".  SQLSTATE=08506

real	0m10.207s
user	0m0.018s
sys	0m0.035s

=================
Mon Oct  5 16:27:36 PDT 2020

CURRENT_HOSTNAME    
--------------------
jsps1.fyre.ibm.com  

  1 record(s) selected.


real	0m1.241s
user	0m0.016s
sys	0m0.021s

=================
Mon Oct  5 16:27:42 PDT 2020

CURRENT_HOSTNAME    
--------------------
jsps1.fyre.ibm.com  

  1 record(s) selected.


real	0m1.043s
user	0m0.017s
sys	0m0.018s

...
```

Test scenario #03 : HADR takeover     
---
This is for testing if the connection is routed to standby cluster when the connection can't be made to any members in original primary cluster.    
Considering the connection will be available on the standby(new primary) only after HADR takeover is done, testing the scenario with HADR takeover.    

- ACTION : Run HADR takeover on replay standby.   

```
db2inst1@shps011:/home/db2inst1 $ date ;  db2 takeover hadr on db hadrdb ; date
Mon Oct  5 16:34:36 PDT 2020
DB20000I  The TAKEOVER HADR ON DATABASE command completed successfully.
Mon Oct  5 16:34:49 PDT 2020
```


- RESULT  : Connection is routed to new pureScale primary HADR cluster.    

![Preprocessing](static/img/nonjava_pshadr/nonjava_pshadr_04_primary_down.png) 

```
...
=================
Mon Oct  5 16:34:33 PDT 2020

CURRENT_HOSTNAME    
--------------------
jsps1.fyre.ibm.com  

  1 record(s) selected.


real	0m1.043s
user	0m0.018s
sys	0m0.021s

=================
Mon Oct  5 16:34:39 PDT 2020
SQL30108N  A connection failed in an automatic client reroute environment. The 
transaction was rolled back. Host name or IP address: "shps011.fyre.ibm.com". 
Service name or port number: "50000". Reason code: "3". Connection failure 
code: "2". Underlying error: "*".  SQLSTATE=08506

real	0m11.052s
user	0m0.015s
sys	0m0.019s

=================
Mon Oct  5 16:34:55 PDT 2020

CURRENT_HOSTNAME    
--------------------
shps011.fyre.ibm.com

  1 record(s) selected.


real	0m2.208s
user	0m0.014s
sys	0m0.026s
...
```

Test scenario #04 : member 0 stop on new primary cluster (old standby)
---
This is the test if takeover within the new primary cluster.   

Before the test, we will have to start primary hadr on other member 1.    

```
db2inst1@shps011:/home/db2inst1 $ db2pd -db hadrdb -hadr

Database Member 0 -- Database HADRDB -- Active -- Up 0 days 08:00:26 -- Date 2020-10-05-16.38.44.434537

                            HADR_ROLE = PRIMARY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 1
                        LOG_STREAM_ID = 0
                           HADR_STATE = PEER
                           HADR_FLAGS = STANDBY_REPLAY_NOT_ON_PREFERRED TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = shps011
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 0
                  STANDBY_MEMBER_HOST = jsps1
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
...
                            HADR_ROLE = PRIMARY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 1
                        LOG_STREAM_ID = 1
                           HADR_STATE = REMOTE_CATCHUP
                           HADR_FLAGS = ASSISTED_REMOTE_CATCHUP STANDBY_REPLAY_NOT_ON_PREFERRED TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = shps011
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 0
                  STANDBY_MEMBER_HOST = jsps1
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
...

db2inst1@shps011:/home/db2inst1 $ ssh shps012
Last login: Sun Oct  4 21:04:03 2020 from 172.16.230.93
db2inst1@shps012:/home/db2inst1 $ db2pd -db hadrdb -hadr

Database HADRDB not activated on database member 1 or this database name cannot be found in the local database directory.

Option -hadr requires -db <database> or -alldbs option and active database.
db2inst1@shps012:/home/db2inst1 $ db2 start hadr on db hadrdb as primary
DB20000I  The START HADR ON DATABASE command completed successfully.

db2inst1@shps012:/home/db2inst1 $ db2pd -db hadrdb -hadr

Database Member 1 -- Database HADRDB -- Active -- Up 0 days 00:00:33 -- Date 2020-10-05-16.41.00.200258

                            HADR_ROLE = PRIMARY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 1
                        LOG_STREAM_ID = 1
                           HADR_STATE = PEER
                           HADR_FLAGS = STANDBY_REPLAY_NOT_ON_PREFERRED TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = shps012
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 1
                  STANDBY_MEMBER_HOST = jsps1
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
...

db2inst1@shps012:/home/db2inst1 $ exit
Connection to shps012 closed.
db2inst1@shps011:/home/db2inst1 $ db2pd -db hadrdb -hadr

Database Member 0 -- Database HADRDB -- Active -- Up 0 days 08:03:40 -- Date 2020-10-05-16.41.58.689594

                            HADR_ROLE = PRIMARY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 1
                        LOG_STREAM_ID = 0
                           HADR_STATE = PEER
                           HADR_FLAGS = STANDBY_REPLAY_NOT_ON_PREFERRED TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = shps011
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 0
                  STANDBY_MEMBER_HOST = jsps1
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
...

```

Now, `jsps1` is replay standby host.   
```
db2inst1@jsps1:/home/db2inst1 $ db2pd -db hadrdb -hadr

Database Member 0 -- Database HADRDB -- Standby -- Up 0 days 00:07:45 -- Date 2020-10-05-16.42.31.978529

                            HADR_ROLE = STANDBY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 0
                        LOG_STREAM_ID = 0
                           HADR_STATE = PEER
                           HADR_FLAGS = STANDBY_REPLAY_NOT_ON_PREFERRED TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = shps011
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 0
                  STANDBY_MEMBER_HOST = jsps1
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED

...

                            HADR_ROLE = STANDBY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 0
                        LOG_STREAM_ID = 1
                           HADR_STATE = PEER
                           HADR_FLAGS = STANDBY_REPLAY_NOT_ON_PREFERRED TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = shps012
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 1
                  STANDBY_MEMBER_HOST = jsps1
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED

```

- ACTION : Stop member 0  where the current connection is being made.  

```
db2inst1@shps011:/home/db2inst1 $ date ; db2stop force member 0 ; date
Mon Oct  5 16:46:07 PDT 2020
10/05/2020 16:46:29     0   0   SQL1064N  DB2STOP processing was successful.
SQL1064N  DB2STOP processing was successful.
Mon Oct  5 16:46:29 PDT 2020
```


- Result : Connection is successfully rerouted to other alive member 1.    

![Preprocessing](static/img/nonjava_pshadr/nonjava_pshadr_05_standby_member0_down.png)

```
...
=================
Mon Oct  5 16:46:05 PDT 2020

CURRENT_HOSTNAME    
--------------------
shps011.fyre.ibm.com

  1 record(s) selected.


real	0m2.045s
user	0m0.015s
sys	0m0.021s

=================
Mon Oct  5 16:46:12 PDT 2020
SQL30108N  A connection failed in an automatic client reroute environment. The 
transaction was rolled back. Host name or IP address: "shps012.fyre.ibm.com". 
Service name or port number: "50000". Reason code: "1". Connection failure 
code: "2". Underlying error: "*".  SQLSTATE=08506

real	0m0.099s
user	0m0.013s
sys	0m0.021s

=================
Mon Oct  5 16:46:17 PDT 2020

CURRENT_HOSTNAME    
--------------------
SQL0952N  Processing was cancelled due to an interrupt.  SQLSTATE=57014

real	0m1.275s
user	0m0.013s
sys	0m0.021s

=================
Mon Oct  5 16:46:24 PDT 2020

CURRENT_HOSTNAME    
--------------------
shps012.fyre.ibm.com

  1 record(s) selected.


real	0m1.041s
user	0m0.011s
sys	0m0.022s

=================
Mon Oct  5 16:46:30 PDT 2020

CURRENT_HOSTNAME    
--------------------
shps012.fyre.ibm.com

  1 record(s) selected.


real	0m1.048s
user	0m0.019s
sys	0m0.024s

...
```



Test scenario #05 : member 1 stop on new primary cluster (old standby)
---
This is the same test done in original primary cluster.   
Basically, we want to see the behavior remains the same on the new primary cluster under the client configuration.    

- ACTION   
```
db2inst1@shps011:/home/db2inst1 $ db2start member 0
10/05/2020 16:49:38     0   0   SQL1063N  DB2START processing was successful.
SQL1063N  DB2START processing was successful.
db2inst1@shps011:/home/db2inst1 $ 
db2inst1@shps011:/home/db2inst1 $ date ; db2stop member 1 force ; date
Mon Oct  5 16:49:53 PDT 2020
10/05/2020 16:50:22     1   0   SQL1064N  DB2STOP processing was successful.
SQL1064N  DB2STOP processing was successful.
Mon Oct  5 16:50:22 PDT 2020
```

- RESULT  : The connection is routed to other member 0 successfully.   

![Preprocessing](static/img/nonjava_pshadr/nonjava_pshadr_06_standby_member1_down.png)

```
...
=================
Mon Oct  5 16:49:51 PDT 2020

CURRENT_HOSTNAME    
--------------------
shps012.fyre.ibm.com

  1 record(s) selected.


real	0m2.045s
user	0m0.015s
sys	0m0.021s

=================
Mon Oct  5 16:49:58 PDT 2020
SQL30108N  A connection failed in an automatic client reroute environment. The 
transaction was rolled back. Host name or IP address: "shps011.fyre.ibm.com". 
Service name or port number: "50000". Reason code: "1". Connection failure 
code: "2". Underlying error: "*".  SQLSTATE=08506

real	0m8.713s
user	0m0.021s
sys	0m0.021s

=================
Mon Oct  5 16:50:12 PDT 2020

CURRENT_HOSTNAME    
--------------------
shps011.fyre.ibm.com

  1 record(s) selected.


real	0m1.199s
user	0m0.014s
sys	0m0.019s

=================
Mon Oct  5 16:50:18 PDT 2020

CURRENT_HOSTNAME    
--------------------
shps011.fyre.ibm.com

  1 record(s) selected.


real	0m1.039s
user	0m0.018s
sys	0m0.016s
...
```


Test scenario #06 : Taking HADR primary back to original cluster  
---
This is the final test doing HADR takeover back to the original cluster.    
In real system, this will happen when the original cluster comes back to normal status and DBA wants to move the primary back.   

- ACTION : Running HADR takeover on the current reply standby.  
```
db2inst1@jsps1:/home/db2inst1 $ date; db2 takeover hadr on db hadrdb ; date
Mon Oct  5 16:55:14 PDT 2020
DB20000I  The TAKEOVER HADR ON DATABASE command completed successfully.
Mon Oct  5 16:55:29 PDT 2020
``` 

- RESULT : The connections is rerouted to original cluster.  

![Preprocessing](static/img/nonjava_pshadr/nonjava_pshadr_07_hadr_takeover_back.png)

```
...
=================
Mon Oct  5 16:55:08 PDT 2020

CURRENT_HOSTNAME    
--------------------
shps011.fyre.ibm.com

  1 record(s) selected.


real	0m1.044s
user	0m0.016s
sys	0m0.023s

=================
Mon Oct  5 16:55:14 PDT 2020
SQL30108N  A connection failed in an automatic client reroute environment. The 
transaction was rolled back. Host name or IP address: "jsps1.fyre.ibm.com". 
Service name or port number: "50000". Reason code: "3". Connection failure 
code: "2". Underlying error: "104".  SQLSTATE=08506

real	0m20.891s
user	0m0.012s
sys	0m0.020s

=================
Mon Oct  5 16:55:40 PDT 2020

CURRENT_HOSTNAME    
--------------------
jsps1.fyre.ibm.com  

  1 record(s) selected.


real	0m1.052s
user	0m0.013s
sys	0m0.019s

=================
Mon Oct  5 16:55:46 PDT 2020

CURRENT_HOSTNAME    
--------------------
jsps1.fyre.ibm.com  

  1 record(s) selected.


real	0m1.040s
user	0m0.018s
sys	0m0.015s
...
``` 


Start HADR primary on other member
---

```
db2inst1@jsps1:/home/db2inst1 $ db2pd -db hadrdb -hadr

Database Member 0 -- Database HADRDB -- Active -- Up 0 days 00:23:32 -- Date 2020-10-05-16.58.18.687972

                            HADR_ROLE = PRIMARY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 1
                        LOG_STREAM_ID = 0
                           HADR_STATE = PEER
                           HADR_FLAGS = STANDBY_REPLAY_NOT_ON_PREFERRED TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = jsps1
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 0
                  STANDBY_MEMBER_HOST = shps011
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
...
                            HADR_ROLE = PRIMARY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 1
                        LOG_STREAM_ID = 1
                           HADR_STATE = REMOTE_CATCHUP
                           HADR_FLAGS = ASSISTED_REMOTE_CATCHUP STANDBY_REPLAY_NOT_ON_PREFERRED TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = jsps1
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 0
                  STANDBY_MEMBER_HOST = shps011
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
...
```

I didn't start member 1 after the TEST SCENARIO #2 and that doesn't matter for this test.  By the way, starting it again.   

```
db2inst1@jsps1:/home/db2inst1 $ ssh jsps2
Last login: Mon Oct  5 15:54:20 2020 from 172.16.165.63

db2inst1@jsps2:/home/db2inst1 $ db2pd -db hadr -hadr
Unable to attach to database manager on member 1.
Please ensure the following are true:
   - db2start has been run for the member.
   - db2pd is being run on the same physical machine as the member.
   - DB2NODE environment variable setting is correct for the member
     or db2pd -mem setting is correct for the member.
Another possibility of this failure is the Virtual Address Space Randomization is currently enabled on this system.


db2inst1@jsps2:/home/db2inst1 $ db2_ps
 
Node 0
     UID        PID       PPID    C     STIME     TTY     TIME CMD
    root       6462          1    0     16:26       ? 00:00:00 db2wdog 0 [db2inst1]
db2inst1       6465       6462    7     16:26       ? 00:02:19 db2sysc 0
    root       6502       6462    0     16:26       ? 00:00:09 db2ckpwd 0
    root       6503       6462    0     16:26       ? 00:00:09 db2ckpwd 0
    root       6504       6462    0     16:26       ? 00:00:09 db2ckpwd 0
db2inst1       6524       6462    0     16:26       ? 00:00:00 db2vend (PD Vendor Process - 1) 0
db2fenc1       7276       6462    0     16:28       ? 00:00:00 db2fmp ( ,0,0,0,0,0,0,00000000,0,0,0000000000000000,0000000000000000,00000000,00000000,00000000,00000000,00000000,00000000,0000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000002fe296000,0000000000000000,0000000000000000,1,0,0,,,,,a89c10,14,1e014,2,0,1,0000000000061fc0,0x240000000,0x240000000,1600000,1e800b,2,2718096
db2fenc1       7325       6462    0     16:28       ? 00:00:00 db2fmp ( ,1,0,0,0,0,0,00000000,0,0,0000000000000000,0000000000000000,00000000,00000000,00000000,00000000,00000000,00000000,0000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000002fe296000,0000000000000000,0000000000000000,1,0,0,,,,,a89c10,14,1e014,2,0,1,0000000000081fc0,0x240000000,0x240000000,1600000,1e800b,2,2730097
db2inst1      12438       6462    0     16:42       ? 00:00:01 db2acd 0 ,0,0,0,1,0,0,00000000,0,0,0000000000000000,0000000000000000,00000000,00000000,00000000,00000000,00000000,00000000,0000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000000,00000002fe296000,0000000000000000,0000000000000000,1,0,0,,,,,a89c10,14,1e014,2,0,1,0000000000041fc0,0x240000000,0x240000000,1600000,1e800b,2,3450087
jsps1.fyre.ibm.com: db2nps ## completed ok
 
Node 1
     UID        PID       PPID    C     STIME     TTY     TIME CMD
jsps2.fyre.ibm.com: db2nps ## completed ok

db2inst1@jsps2:/home/db2inst1 $ db2 start hadr on db hadrdb as primary
SQL1032N  No start database manager command was issued.

db2inst1@jsps2:/home/db2inst1 $ db2instance -list
ID	  TYPE	           STATE		HOME_HOST		CURRENT_HOST		ALERT	PARTITION_NUMBER	LOGICAL_PORT	NETNAME
--	  ----	           -----		---------		------------		-----	----------------	------------	-------
0	MEMBER	         STARTED		    jsps1		       jsps1		   NO	               0	           0	  jsps1
1	MEMBER	         STOPPED		    jsps2		       jsps2		   NO	               0	           0	  jsps2
128	CF	            PEER		    jsps3		       jsps3		   NO	               -	           0	  jsps3
129	CF	         PRIMARY		    jsps4		       jsps4		   NO	               -	           0	  jsps4

HOSTNAME		   STATE		INSTANCE_STOPPED	ALERT
--------		   -----		----------------	-----
   jsps4		  ACTIVE		              NO	   NO
   jsps3		  ACTIVE		              NO	   NO
   jsps2		  ACTIVE		              NO	   NO
   jsps1		  ACTIVE		              NO	   NO


db2inst1@jsps2:/home/db2inst1 $ db2start member 1
10/05/2020 17:00:32     1   0   SQL1063N  DB2START processing was successful.
SQL1063N  DB2START processing was successful.

db2inst1@jsps2:/home/db2inst1 $ db2 start hadr on db hadrdb as primary 
DB20000I  The START HADR ON DATABASE command completed successfully.

db2inst1@jsps2:/home/db2inst1 $ db2pd -db hadrdb -hadr

Database Member 1 -- Database HADRDB -- Active -- Up 0 days 00:00:16 -- Date 2020-10-05-17.01.45.387102

                            HADR_ROLE = PRIMARY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 1
                        LOG_STREAM_ID = 1
                           HADR_STATE = PEER
                           HADR_FLAGS = STANDBY_REPLAY_NOT_ON_PREFERRED TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = jsps2
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 1
                  STANDBY_MEMBER_HOST = shps011
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
...


db2inst1@jsps2:/home/db2inst1 $ exit
Connection to jsps2 closed.

db2inst1@jsps1:/home/db2inst1 $ db2pd -db hadrdb -hadr

Database Member 0 -- Database HADRDB -- Active -- Up 0 days 00:27:16 -- Date 2020-10-05-17.02.02.866327

                            HADR_ROLE = PRIMARY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 1
                        LOG_STREAM_ID = 0
                           HADR_STATE = PEER
                           HADR_FLAGS = STANDBY_REPLAY_NOT_ON_PREFERRED TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = jsps1
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 0
                  STANDBY_MEMBER_HOST = shps011
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
...

```
