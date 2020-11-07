
Objectives
====

You configured DB2 pureScale on server side and now, may wonder how to handle connection rerouting in failure scenarios like below.   

- A member down on pureScale primary cluster.  
  Of course, you won't expect to takeover the connection to standby but just to takeover to other live members in the same cluster.
- Whole primary cluster is down. That means application cannot access any of members there.  
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


This page explains the scenario with java application. For non java applicatino, refer to this page.    
[Non java application connection reroute with pureScale HADR environment](ps_hadr_non_java.md)

From non java scenario, `enableAlternateGroupSeamlessAcr` was used but the property cannot be used for java application.   

[Client reroute properties for non java and java clients](https://www.ibm.com/support/knowledgecenter/en/SSEPGG_11.5.0/com.ibm.swg.im.dbclient.config.doc/doc/c0059619.html)

So we need to try something else.     

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


HADR configuration review before starting tests
---

* jsps1 - member 0 

```
db2inst1@jsps1:/home/db2inst1 $ db2pd -db hadrdb -hadr

Database Member 0 -- Database HADRDB -- Active -- Up 7 days 16:42:41 -- Date 2020-11-02-17.34.52.823699

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
Database Member 1 -- Database HADRDB -- Active -- Up 7 days 15:16:05 -- Date 2020-11-02-17.35.22.679477

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
Database Member 0 -- Database HADRDB -- Standby -- Up 0 days 00:00:19 -- Date 2020-11-02-17.30.48.279561

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

TEST SCENARIOS
====

Here are two main test scenarios depending on the client reroute mode on the primary cluster being configured with `WLB` or `affinity`.    

01 WLB ( Workload balancing) 
---


CLIENT CONFIGURATION
--

Testing with the DB2 JCC driver that is downloaded from the targe DB2 server version `V11.5`.    
However, the result would be same with jcc driver files from older DB2 release V11.1 or V10.5.
```
$ ls -tlr *.jar
-r--r--r-- 1 db2inst1 db2iadm1 6550443 Nov  2 20:31 db2jcc4.jar

$ java -cp db2jcc4.jar com.ibm.db2.jcc.DB2Jcc -version
IBM Data Server Driver for JDBC and SQLJ 4.26.14
```

And on the remote client host, there are hosts entries for related DB2 servers.   
```
$ grep fyre /etc/hosts
172.16.165.63 jsps1.fyre.ibm.com jsps1 # priamry pureScale member 0
172.16.167.133 jsps2.fyre.ibm.com jsps2 # primary pureScale member 1
172.16.230.93 shps011.fyre.ibm.com shps011  #standby pureScale member 0
172.16.36.121 shps012.fyre.ibm.com shps012  #standby pureScale member 1
```

SERVER CONFIGURATION
---
There is nothing to configure especially for JDBC connection.   
DB directory information is default on both primary and standby. 


```
db2inst1@jsps1:/home/db2inst1 $ db2 list db directory

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

01-0 WLB within primary cluster  
--------
Firstly, let's forget about HADR at the moment and just see how WLB works within a pureScale cluster.      
Keep in mind that WLB configuration itself enables client reroute within a pureScale cluster and connection will be made to a member depending on server list that has server workload information and priority.    
The server list informatin will be sent from DB2 server to client JVM(Java virtual machine) ever some seconds and that is cached within the java process to be referred for connection attempts.   

Here, I used a simple standalone java to connect to the database and run the SQL showing the current connected server hostname continuosly as well as server list information as the reference.        
Necessary jcc properties are implemented in the program.    

```java
$ cat TestWLB.java

import java.sql.*;
import java.io.*;
import java.util.Properties;

public class TestWLB{


	public static void main(String[] args){
		TestWLB DB2Client = new TestWLB();
		DB2Client.JDBCTest ();
	}

	public void JDBCTest() {

		Connection con = null;
		Statement stmt1 = null;
		Statement stmt2 = null;
		Statement stmt3 = null;
		ResultSet   rs1 = null;
		ResultSet   rs2 = null;
		ResultSet   rs3 = null;
		String   query = null;

		String URL = "jdbc:db2://jsps1.fyre.ibm.com:50000/hadrdb:enableSysplexWLB=true;"; // Just WLB only for rerouting within member, not considering HADR for now.
		String userid = "db2inst1";
		String passwd = "xxxxxxxx";

		try {
			System.out.println("============= Test starts !! ");  	
			// connect
			System.out.println("=====before connect !");
			Class.forName("com.ibm.db2.jcc.DB2Driver");
			con = DriverManager.getConnection(URL, userid, passwd);
			System.out.println("=====Initial Connection successful !");
			stmt1 = con.createStatement();
			stmt2 = con.createStatement();
			stmt3 = con.createStatement();
			String strResult1="";
			String strResult2="";
			String strResult3="";
	
			// Endless loop for running SQL query. We are not going to close collection for the test purpose.
			while (true) {
				//promptEnterKey();  // uncomment this when you want to pause each step.

				rs1 = stmt1.executeQuery("SELECT CURRENT TIMESTAMP FROM SYSIBM.SYSDUMMY1");
				while(rs1.next()){
					strResult1 = rs1.getString(1);
				}
				rs2 = stmt2.executeQuery("SELECT HOST_NAME FROM TABLE(SYSPROC.ENV_GET_SYS_INFO())");
				while(rs2.next()){	
					strResult2 = rs2.getString(1);
				}
				System.out.print("Time:hostname => |" + strResult1 + "|" + strResult2 + "| , ");

				// print serverlist for reference.
				// PRIORITY : Describes the relative capacity of a member to handle work. The higher the value, the more work a client should drive to that member.
				rs3 = stmt3.executeQuery("SELECT HOSTNAME, PRIORITY FROM TABLE (MON_GET_SERVERLIST (-1))");
				System.out.print("ServerList (host:priority) => ");
				while(rs3.next()){
					System.out.print(rs3.getString("HOSTNAME") + "|" + rs3.getString("PRIORITY") + "|");
				}
				System.out.println(" ");

				rs1.close();
				rs2.close();
				rs3.close();

				Thread.sleep(1000);
			}
		}catch(Exception e){
			e.printStackTrace();
		}

		System.out.println("If you see this message, it means it passed the loops. So that is strange and not expected ");
					
	}

	public void promptEnterKey(){
		System.out.println("Press \"ENTER\" to continue...");
		try {
			System.in.read();
		} catch (IOException e) {
			e.printStackTrace();
		}
	}

}

``` 

For the test, 2 terminals will be used.      

On terninal 1, starting the java test program and watch outputs.   
On terminal 2, stop and start DB2 members with scenarios.  

Terminal 1 :  The program started and following up transactions connect to mmeber 0 or member 1 depending on server list at the time.    
This is expected using WLB.    

![Preprocessing](static/img/java_pshadr/java_pshadr_01_normal_initial.png) 

```
db2inst1@lamps1 ~/test/java_hadr $ java -cp ./db2jcc4.jar:. TestWLB
============= Test starts !! 
=====before connect !
=====Initial Connection successful !
Time:hostname => |2020-11-03 17:57:37.72334|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|85|jsps2.fyre.ibm.com|100| 
Time:hostname => |2020-11-03 17:57:38.814174|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|85| 
Time:hostname => |2020-11-03 17:57:39.836064|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|85|jsps2.fyre.ibm.com|100| 
Time:hostname => |2020-11-03 17:57:40.847468|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|85|jsps2.fyre.ibm.com|100| 
Time:hostname => |2020-11-03 17:57:41.859717|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|85|jsps2.fyre.ibm.com|100| 
Time:hostname => |2020-11-03 17:57:42.86903|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|85|jsps2.fyre.ibm.com|100| 
Time:hostname => |2020-11-03 17:57:43.8801|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|85|jsps2.fyre.ibm.com|100| 
Time:hostname => |2020-11-03 17:57:44.895976|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|85|jsps2.fyre.ibm.com|100| 
Time:hostname => |2020-11-03 17:57:45.912232|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|85|jsps2.fyre.ibm.com|100| 

...
```

Terminal 2 : stop member 0
```
$ db2stop member 0 force
```

![Preprocessing](static/img/java_pshadr/java_pshadr_02_member0_down.png) 

Terminal 1 : Then it only connects to remained member 1 showing the dead member 0 priority in server list as zero(0) which means connection will not goes to the member 0.   

```
...
Time:hostname => |2020-11-03 17:59:20.732948|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|87| 
Time:hostname => |2020-11-03 17:59:21.742736|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|87| 
Time:hostname => |2020-11-03 17:59:22.757284|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|87| 
Time:hostname => |2020-11-03 17:59:23.76659|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|87| 
Time:hostname => |2020-11-03 17:59:24.779722|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|87| 
Time:hostname => |2020-11-03 17:59:25.790813|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|87| 
Time:hostname => |2020-11-03 17:59:26.80491|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|87| 

Time:hostname => |2020-11-03 17:59:28.0327|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|0| 
Time:hostname => |2020-11-03 17:59:29.063408|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|0| 
Time:hostname => |2020-11-03 17:59:30.073085|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|0| 
Time:hostname => |2020-11-03 17:59:31.083718|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|0| 
Time:hostname => |2020-11-03 17:59:32.095828|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|0| 
Time:hostname => |2020-11-03 17:59:33.106439|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|0| 

...
```

Terminal 2 : start member 0 again

```
$ db2start member 0
```

![Preprocessing](static/img/java_pshadr/java_pshadr_01_normal_initial.png) 

Terminal 1 :  server list priority shows non zero for member 0 and connections are made again to member 0 too.   

```
...
Time:hostname => |2020-11-03 18:01:46.550294|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|0| 

Time:hostname => |2020-11-03 18:01:47.558225|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|100| 
Time:hostname => |2020-11-03 18:01:48.629353|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|52| 
Time:hostname => |2020-11-03 18:02:04.196024|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|52| 
Time:hostname => |2020-11-03 18:02:05.384217|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|52| 
Time:hostname => |2020-11-03 18:02:06.408891|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|52| 
...
```

Terminal 2 : stop member 1
```
$ db2stop mebmer 1 force
```
![Preprocessing](static/img/java_pshadr/java_pshadr_03_member1_down.png) 

Terminal 1 : Then it only connects to remained member 0 showing the dead member 1 priority in server list as zero(0) which means connection will not goes to the member 1.  
```
...
Time:hostname => |2020-11-03 18:02:59.412545|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|67| 
Time:hostname => |2020-11-03 18:03:00.422648|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|67| 
Time:hostname => |2020-11-03 18:03:01.433165|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|67| 
Time:hostname => |2020-11-03 18:03:02.443295|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|44|jsps1.fyre.ibm.com|100| 
Time:hostname => |2020-11-03 18:03:03.552886|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|67| 
Time:hostname => |2020-11-03 18:03:04.56364|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|60|jsps1.fyre.ibm.com|100| 
Time:hostname => |2020-11-03 18:03:05.589984|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|60|jsps1.fyre.ibm.com|100| 

Time:hostname => |2020-11-03 18:03:06.761869|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 
Time:hostname => |2020-11-03 18:03:07.813209|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 
Time:hostname => |2020-11-03 18:03:08.822591|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 
Time:hostname => |2020-11-03 18:03:09.836127|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 
Time:hostname => |2020-11-03 18:03:10.847661|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 
Time:hostname => |2020-11-03 18:03:11.864379|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 
Time:hostname => |2020-11-03 18:03:12.877604|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 

...
```

Terminal 2 : start member 1 again
```
$ db2start member 0
```
![Preprocessing](static/img/java_pshadr/java_pshadr_01_normal_initial.png) 

Terminal 1 : server list priority shows non zero for member 0 and connections can be made to both members.    

```
...
Time:hostname => |2020-11-03 18:04:01.688454|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 
Time:hostname => |2020-11-03 18:04:02.703664|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 
Time:hostname => |2020-11-03 18:04:03.7142|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 
Time:hostname => |2020-11-03 18:04:04.724871|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 

Time:hostname => |2020-11-03 18:04:05.808484|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|60| 
Time:hostname => |2020-11-03 18:04:06.846462|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|60| 
Time:hostname => |2020-11-03 18:04:07.864445|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|60| 
Time:hostname => |2020-11-03 18:04:08.875236|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|60| 
Time:hostname => |2020-11-03 18:04:09.885257|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|60| 
Time:hostname => |2020-11-03 18:04:10.894416|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|60|jsps1.fyre.ibm.com|100| 
Time:hostname => |2020-11-03 18:04:21.923957|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|60|
...
```


01-1 CLIENT REROUTE across HADR clusters when using WLB    
-------

So far, with regard to down situation on any member, we observed connection reroute works perfectly within the primary pureScale cluster with just only one jcc property `enableSysplexWLB=true` enabling WLB.     

For next, we will see how to make the connection to be rerouted to HADR standby cluster when connections cannot be made to the current cluster.     
The possible situation could be HADR takeover as part of normal planned operation or due to outage of whole members in the cluster.     

For that purpose, we will bring some more properties below.     

First of all, we still need WLB parameter as basis.   
`enableSysplexWLB=true`      
And use few parameter having prefix `alternateGroup` like below.      
For `alternateGroupServerName`,  use the replay standby hostname.     
```
alternateGroupDatabaseName=HADRDB
alternateGroupServerName=shps011.fyre.ibm.com
alternateGroupPortNumber=50000
```

And set parameters for routing connections seamlessly.     
```
enableSeamlessFailover=true
enableAlternateGroupSeamlessACR=true
```

Also give retry count and interval before routing to new cluster.     
This needs to be considered and find the best value from tests and business application demands.    
In most cases, it's unwanted to reoute right away in a situation the connection problem is temporoary for some seconds.      
Also the situation that needs pureScale HADR takeover should be rare.     

```
maxRetriesForClientReroute=4
retryIntervalForClientReroute=3
```

So the standalone test java programe can be modified like below.     
```java
...
                // Just WLB only for rerouting within cluster, not considering HADR for now.
                //String URL = "jdbc:db2://jsps1.fyre.ibm.com:50000/hadrdb:enableSysplexWLB=true;";
                // pureScale HADR
                String URL = "jdbc:db2://jsps1.fyre.ibm.com:50000/hadrdb:enableSysplexWLB=true;alternateGroupDatabaseName=HADRDB;alternateGroupServerName=shps011.fyre.ibm.com;alternateGroupPortNumber=50000;enableSeamlessFailover=true;enableAlternateGroupSeamlessACR=true;maxRetriesForClientReroute=4;retryIntervalForClientReroute=3;";
...
```

With this, the scenarios within primary pureScale cluster will work as same as we have seem in the previous section `01-0 WLB within primary cluster`.      

so we continue test from HADR takeover scenario.


Terminal 2 : While connection attempts continue with the test java program,  run `takeover hadr' from the replay standby host.       

```
db2inst1@shps011:/home/db2inst1 $ date;db2 takeover hadr on db hadrdb ;date
Fri Nov  6 14:45:57 PST 2020
DB20000I  The TAKEOVER HADR ON DATABASE command completed successfully.
Fri Nov  6 14:46:15 PST 2020
```

![Preprocessing](static/img/java_pshadr/java_pshadr_04_primary_down.png) 

Termianl 2 : Connection is routed to standby pureScale cluster and new server list is made on it.      
```
...
Time:hostname => |2020-11-06 14:45:49.387264|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|52| 
Time:hostname => |2020-11-06 14:45:50.395039|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|52| 
Time:hostname => |2020-11-06 14:45:51.403811|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|52| 
Time:hostname => |2020-11-06 14:45:52.409257|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|52| 
Time:hostname => |2020-11-06 14:45:53.418118|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|52| 
Time:hostname => |2020-11-06 14:45:54.515282|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|53|jsps2.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:45:55.522186|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|49| 
Time:hostname => |2020-11-06 14:45:56.54369|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|49| 
Time:hostname => |2020-11-06 14:45:57.554647|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|100|jsps1.fyre.ibm.com|49| 


Time:hostname => |2020-11-06 14:46:20.848229|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|43|shps011.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:46:39.428918|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|43| 
Time:hostname => |2020-11-06 14:46:40.581182|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|43| 
Time:hostname => |2020-11-06 14:46:41.587007|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|43| 
Time:hostname => |2020-11-06 14:46:42.59249|shps012.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|43| 
Time:hostname => |2020-11-06 14:46:43.601349|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|43| 
Time:hostname => |2020-11-06 14:46:44.607586|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|43| 
...
```

`db2pd -hadr` output after `takeover hadr`.    

```
db2inst1@jsps1:/home/db2inst1/ $ db2pd -hadr -db hadrdb

Database Member 0 -- Database HADRDB -- Standby -- Up 0 days 00:02:44 -- Date 2020-11-06-14.48.57.770539

                            HADR_ROLE = STANDBY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 0
                        LOG_STREAM_ID = 0
                           HADR_STATE = PEER
                           HADR_FLAGS = TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = shps011
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 0
                  STANDBY_MEMBER_HOST = jsps1
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
             HADR_CONNECT_STATUS_TIME = 11/06/2020 14:46:36.089145 (1604702796)
...
                            HADR_ROLE = STANDBY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 0
                        LOG_STREAM_ID = 1
                           HADR_STATE = PEER
                           HADR_FLAGS = TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = shps012
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 1
                  STANDBY_MEMBER_HOST = jsps1
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
             HADR_CONNECT_STATUS_TIME = 11/06/2020 14:46:37.975801 (1604702797)
...
```

```
db2inst1@shps011:/home/db2inst1 $ db2pd -hadr -db hadrdb

Database Member 0 -- Database HADRDB -- Active -- Up 3 days 21:19:34 -- Date 2020-11-06-14.50.03.548070

                            HADR_ROLE = PRIMARY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 1
                        LOG_STREAM_ID = 0
                           HADR_STATE = PEER
                           HADR_FLAGS = TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = shps011
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 0
                  STANDBY_MEMBER_HOST = jsps1
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
             HADR_CONNECT_STATUS_TIME = 11/06/2020 14:46:36.083745 (1604702796)
...
```

```
db2inst1@shps012:/home/db2inst1 $ db2pd -hadr -db HADRDB

Database Member 1 -- Database HADRDB -- Active -- Up 0 days 00:02:27 -- Date 2020-11-06-14.48.39.109596

                            HADR_ROLE = PRIMARY
                          REPLAY_TYPE = PHYSICAL
                        HADR_SYNCMODE = ASYNC
                           STANDBY_ID = 1
                        LOG_STREAM_ID = 1
                           HADR_STATE = PEER
                           HADR_FLAGS = TCP_PROTOCOL
                  PRIMARY_MEMBER_HOST = shps012
                     PRIMARY_INSTANCE = db2inst1
                       PRIMARY_MEMBER = 1
                  STANDBY_MEMBER_HOST = jsps1
                     STANDBY_INSTANCE = db2inst1
                       STANDBY_MEMBER = 0
                  HADR_CONNECT_STATUS = CONNECTED
             HADR_CONNECT_STATUS_TIME = 11/06/2020 14:46:30.046008 (1604702790)
             ...
            
```

Terminal 2 : Stop member 0
```
db2inst1@shps011:/home/db2inst1 $ db2stop member 0 force
11/06/2020 14:51:31     0   0   SQL1064N  DB2STOP processing was successful.
SQL1064N  DB2STOP processing was successful.
```

![Preprocessing](static/img/java_pshadr/java_pshadr_05_standby_member0_down.png)


Terminal 1 : Then it only connects to remained member 1 showing the dead member 0 priority in server list as zero(0) which means connection will not goes to the member 0.   

```
...
Time:hostname => |2020-11-06 14:50:50.947681|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:50:59.949147|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|71|shps012.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:51:00.966526|shps011.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:50:54.049052|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|100| 

Time:hostname => |2020-11-06 14:50:55.134242|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:50:56.152771|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:50:57.158816|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:50:58.16464|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:50:59.171689|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|0|
...
```

Terminal 2 : Start member 0.
```
db2inst1@shps011:/home/db2inst1 $ db2start member 0 
11/06/2020 14:52:40     0   0   SQL1063N  DB2START processing was successful.
SQL1063N  DB2START processing was successful.
```

Terminal 1 : server list priority shows non zero for member 0 and connections are made again to member 0 too.  
```
...
Time:hostname => |2020-11-06 14:52:31.490621|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:52:32.497588|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:52:33.503294|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|0| 

Time:hostname => |2020-11-06 14:52:34.573587|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:52:35.602657|shps011.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:52:55.489939|shps012.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|87| 
Time:hostname => |2020-11-06 14:52:56.518074|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|87| 
...
```

Terminal 2 : Stop member 1 
```
db2inst1@shps011:/home/db2inst1 $ db2stop member 1 force
11/06/2020 14:53:56     1   0   SQL1064N  DB2STOP processing was successful.
SQL1064N  DB2STOP processing was successful.
```

![Preprocessing](static/img/java_pshadr/java_pshadr_06_standby_member1_down.png)

Terminal 1 : Then it only connects to remained member 0 showing the dead member 1 priority in server list as zero(0) which means connection will not goes to the member 1.    
```
...
Time:hostname => |2020-11-06 14:53:22.990754|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:53:23.997438|shps011.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:53:25.006447|shps011.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:53:26.012938|shps012.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|87|shps012.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:53:34.98181|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|87|shps012.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:53:36.054505|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:53:37.07134|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:53:38.077053|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:53:39.082254|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:53:40.08851|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|0| 
...
```

Terminal 2 : start member 1    
```
db2inst1@shps011:/home/db2inst1 $ db2start member 1
11/06/2020 14:54:47     1   0   SQL1063N  DB2START processing was successful.
```

Terminal 1 : server list priority shows non zero for member 0 and connections can be made to both members.    

```
...
Time:hostname => |2020-11-06 14:54:51.888156|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:54:52.895854|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:54:53.901963|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|0| 

Time:hostname => |2020-11-06 14:54:54.953892|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|82|shps011.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:54:56.130232|shps012.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:55:05.072881|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:54:58.154757|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|82|shps011.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:54:59.161054|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|82|shps011.fyre.ibm.com|100| 
...
```

Terminal 2 : Run `takeover hadr' back from the current replay standby host.  

```
db2inst1@jsps1:/home/db2inst1/test/TS003788476 $ date;db2 takeover hadr on db hadrdb ;date
Fri Nov  6 14:57:03 PST 2020
DB20000I  The TAKEOVER HADR ON DATABASE command completed successfully.
Fri Nov  6 14:57:22 PST 2020
```

![Preprocessing](static/img/java_pshadr/java_pshadr_07_hadr_takeover_back.png)

Terminal 1 : Connection is routed to new primary pureScale cluster and new server list is made on it.  
```
Time:hostname => |2020-11-06 14:56:57.46272|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|78| 
Time:hostname => |2020-11-06 14:56:58.467245|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|78| 
Time:hostname => |2020-11-06 14:56:59.474563|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|78| 
Time:hostname => |2020-11-06 14:57:00.48085|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|78| 
Time:hostname => |2020-11-06 14:56:53.562099|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|67|shps011.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:56:54.63134|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|71|shps011.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:56:55.647386|shps012.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|78| 
Time:hostname => |2020-11-06 14:57:04.579173|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|100|shps012.fyre.ibm.com|78| 

Time:hostname => |2020-11-06 14:57:34.532523|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|74|jsps1.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:57:49.843535|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|74|jsps1.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:57:50.852778|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:57:51.876712|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|74|jsps1.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:57:52.901753|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:57:53.909858|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:57:54.918328|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:57:55.926666|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|0| 
Time:hostname => |2020-11-06 14:57:56.974122|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|74|jsps1.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:57:57.980071|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|74|jsps1.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:57:58.987252|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|74|jsps1.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:58:00.00264|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps2.fyre.ibm.com|74|jsps1.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 14:58:01.012019|jsps2.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|100|jsps2.fyre.ibm.com|78| 
```

`db2pd -hadr` output after `takeover hadr` back to orignial primary clsuter.    
```
...
db2inst1@jsps1:/home/db2inst1 $ db2pd -hadr -db hadrdb

Database Member 0 -- Database HADRDB -- Active -- Up 0 days 00:12:31 -- Date 2020-11-06-14.58.44.001684

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
             HADR_CONNECT_STATUS_TIME = 11/06/2020 14:57:28.774733 (1604703448)
             ...
```

```
db2inst1@jsps2:/home/db2inst1 $ db2pd -hadr -db hadrdb

Database Member 1 -- Database HADRDB -- Active -- Up 0 days 00:02:00 -- Date 2020-11-06-14.59.34.826078

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
             HADR_CONNECT_STATUS_TIME = 11/06/2020 14:57:45.248529 (1604703465)
             ...
```

```
db2inst1@shps011:/home/db2inst1 $ db2pd -hadr -db hadrdb

Database Member 0 -- Database HADRDB -- Standby -- Up 0 days 00:01:57 -- Date 2020-11-06-14.59.13.614294

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
             HADR_CONNECT_STATUS_TIME = 11/06/2020 14:57:28.772902 (1604703448)
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
             HADR_CONNECT_STATUS_TIME = 11/06/2020 14:57:45.249316 (1604703465)
             ...
```

01-2 CLIENT REROUTE across HADR clusters using WLB when creating new connection.    
------
So far, we have seen the combinatino of WLB and the `alternateGroup` prefix properties covers all the situation sceanrios in pureScale HADR environment for existing connection reroute.     
Then what if there is situation for new incoming connection scenarios such as Web Application server(WAS) restart.    

For the test, we will takeover HADR in advance and start the java test program to see if the initial connection can be made to the current primary pureScale cluster.  

Termianl 2 : Takever HADR to standby     
```
db2inst1@shps011:/home/db2inst1 $ date;db2 takeover hadr on db hadrdb
Fri Nov  6 15:18:37 PST 2020
DB20000I  The TAKEOVER HADR ON DATABASE command completed successfully.
```

Terminal 1 : Initial connection is successfully routed to the current primary cluster.    
```
db2inst1@lamps1 ~/test/java_hadr $ date;java -cp ./db2jcc4.jar:. TestWLB
Fri Nov  6 15:23:23 PST 2020
============= Test starts !! 
=====before connect !
=====Initial Connection successful !
Time:hostname => |2020-11-06 15:23:25.591765|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|50| 
Time:hostname => |2020-11-06 15:23:26.638385|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|50| 
Time:hostname => |2020-11-06 15:23:35.57678|shps011.fyre.ibm.com| , ServerList (host:priority) => shps011.fyre.ibm.com|50|shps012.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 15:23:36.599757|shps011.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|50| 
Time:hostname => |2020-11-06 15:23:29.681492|shps012.fyre.ibm.com| , ServerList (host:priority) => shps012.fyre.ibm.com|100|shps011.fyre.ibm.com|50| 
...
```

NOTE 
---

There is old document with the same topic, however, part of the contents are incorrect.   
https://www.ibm.com/developerworks/data/library/techarticle/dm-1509hadr-purescale/index.html

For example, the document suggests to use WLB with `clientRerouteAlternateServerName` parameter. 

```java
...
 // pureScale HADR : WLB + clientRerouteAlternateServerName
                String URL = "jdbc:db2://jsps1.fyre.ibm.com:50000/hadrdb:enableSysplexWLB=true;clientRerouteAlternateServerName=shps011.fyre.ibm.com;clientRerouteAlternatePortNumber=50000;maxRetriesForClientReroute=4;retryIntervalForClientReroute=3;";
```

But the test shows it fails to route to new primary cluster after `HADR takeover`.    
We have discussed with one of author of the technote and agreed that the suggeestion is wrong.    
```
...
Time:hostname => |2020-11-06 15:32:50.194944|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|54|jsps2.fyre.ibm.com|100| 
Time:hostname => |2020-11-06 15:32:51.21627|jsps1.fyre.ibm.com| , ServerList (host:priority) => jsps1.fyre.ibm.com|54|jsps2.fyre.ibm.com|100| 

com.ibm.db2.jcc.am.DisconnectNonTransientConnectionException: [jcc][t4][2030][11211][4.26.14] A communication error occurred during operations on the connection's underlying socket, socket input stream, 
or socket output stream.  Error location: Reply.fill() - insufficient data (-1).  Message: Insufficient data. ERRORCODE=-4499, SQLSTATE=08001
...
```

CONCLUSION
====
To cover all above scenarios including initial connection rerouting, use WLB and `alternateGroup` prefix properties combination.   
That is only possible way to satisfy all the general outage scenario and rerouting accordingly in pureScale HADR environment.







