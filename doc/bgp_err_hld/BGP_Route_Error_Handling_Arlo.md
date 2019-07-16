

  
# BGP Route Install Error Handling
# High Level Design Document
#### Rev 0.1

# Table of Contents
  * [List of Tables](#list-of-tables)
  * [Revision](#revision)
  * [About This Manual](#about-this-manual)
  * [Scope](#scope)
  * [Definition/Abbreviation](#definitionabbreviation)
  
# List of Tables
[Table 1: Abbreviations](#table-1-abbreviations)

# Revision
| Rev |     Date    |       Author       | Change Description                |
|:---:|:-----------:|:------------------:|-----------------------------------|
| 0.1 | 05/07/2019  |   Sudhanshu Kumar  | Initial version                   |

# About this Manual
This document provides information about how to handle the "route add failure in hardware" related errors in BGP in SONIC.
# Scope
This document describes the high level design of BGP route install error handling feature. Implementation for warm reboot and GR for BGP is out of scope for this feature. When route installation fails in hardware due to table full, BGP may retry again when some routes get deleted. This Retry mechanism in BGP for failed routes will not be implemented in this release.

# Definition/Abbreviation
### Table 1: Abbreviations

| **Term**          | ***Meaning***           |
|-------------------|-------------------------|
| BGP               | Border Gateway Protocol |
| GR                | Graceful Restart        |
| SONIC             | Software for Open Networking in the Cloud                        |
| FRR    	        |  FRRouting                       |
| FPM               |  Forwarding Plane Manager        |
| SwSS              |  SONiC Switch State Service      |
# 1 Requirement Overview
 When BGP learns a prefix, it advertises the route to its peers and sends it to the route table manager(Zebra). The routes are installed in the kernel and sent to APP_DB via fpmsyncd. 
The Orchagent reads the route from APP_DB, creates new resources like nexthop or nexthop group Id and installs the route in ASIC_DB. The syncd triggers the appropriate SAI API and the route is installed in hardware. The CRM manages the count of critical resources allocated by orchagent through SAI API.
Due to resource allocations failures in hardware, SAI API calls can fail and these failures should be notified to Zebra and BGP to withdraw failed routes from kernel and BGP peers.

## 1.1 Functional Requirements

 

 1. BGP should withdraw the routes from its peers which have failed to be installed in hardware.
 1. BGP should mark the routes which are not installed in hardware as failed routes in its RIB-IN table.
 1. Zebra should mark the routes which are not successfully installed in hardware as failed routes.
 1. Zebra should withdraw the failed routes from the kernel.

## 1.2 Configuration and Management Requirements
## 1.3 Scalability Requirements
## 1.4 Warm Boot Requirements
   There is no change needed in BGP warm reboot for supporting this feature.
# 2 Functionality
Refer to section 1

## 2.1 Target Deployment Use Cases

## 2.2 Functional Description
Refer to section 1.1

# 3 Design
## 3.1 Overview
Whenever BGP sends a route to Zebra, it appends a 16-bit opaque-id to the route message. It also stores this id in its route node. The 16-bit id consists of the top 4 bits of protocol source and remaining 12 bits of sequence number. Zebra will store the opaque-id per route per protocol source. It will also send the route to fpmsyncd for installation.
fpmsyncd installs the opaque-id along with the route information.
fpmsyncd subscribes to the changes in the ERROR_ROUTE_TABLE entries. Whenever the error status in ERROR_ROUTE_TABLE is updated, fpmsyncd is notified. It then sends a message to Zebra's routing table to take appropriate action.
Zebra should extract the route destination and opaque-id from the message. It should also extract the protocol source from the opaque-id. It should look up the route and match the opaque-id with entries from various protocol sources. Based on a matching source and opaque-id, it should mark it as not installed in hardware. It should also create a route-netlink message to withdraw this state in the kernel. Also, it sends a message to the source protocol for the route (BGP). If the opaque-id is not found, the failure message is dropped at Zebra.
BGP on receiving the message will match the opaque-id. If the opaque-id matches, it marks the route as not installed in hardware and will withdraw the route from its peer. It should mark the route in RIB-IN as not installed in hardware and remove it from the RIB-OUT list, if any. 
For ECMP case, BGP sends route with list of nexthops to Zebra for programming. In fpmsyncd, as per the route table schema, the route is received with a list of nexthops. If the nexthop group programming fails, it is treated as route add failure in BGP.
For ADD-PATH feature, if the route-add failure notification comes to BGP, it will not switch to route of next best rank.

## 3.2 DB Changes

### 3.2.1 CONFIG DB
A new table BGP_ERROR_CFG_TABLE has been introduced in CONFIG DB (refer section 3.7).
### 3.2.2 APP DB
ROUTE_TABLE  inside APP_DB is modified to add 16-bit opaque-id as a non-key attribute.
### 3.2.3 STATE DB
### 3.2.4 ASIC DB
### 3.2.5 COUNTER DB

## 3.3 FRRouting Design
### 3.3.1 Zebra changes
Protocols communicate with Zebra using ZAPI message format. This message format will be enhanced by adding a 16-bit opaque-id. 
Currently, Zebra communicates with fpmsyncd using netlink library which has predefined standard attributes. A new attribute can be added here to send the the opaque-id to fpmsyncd.
Zebra, on receiving the message containing failed route notification, will match the opaque-id along with the route entry. On successful match, it will withdraw the route from kernel. It will also mark the route with flag as "Not installed in hardware" and store the route. It will not send the next best route to fpmsyncd. At this stage, route is present in Zebra. It will also notify the protocol source(BGP) of the route add failure. 

### 3.3.2 BGP changes
When BGP learns a route, it immediately sends its best route to its peers without waiting for notification from the hardware. It also sends the route to Zebra along with opaque-id for installation in hardware. However, the route may or may not be successfully installed in hardware.  On receiving route add failed notification message, BGP will match the oapque-id. On successful match, it will remove the route from RIB-OUT list and place in RIB-IN with a flag marking the route as not installed in hardware. It will also withdraw the route from its peers.
In case user wants to retry the installation of failed routes, he/she can issue the command in Zebra. The command will notify the protocol source. If the source protocol has the route with flag marked as not installed in hardware, it will remove the flag and send the route to its peers.

## 3.4 SwSS Design

### 3.4.1 fpmsyncd changes
fpmsyncd will program the opaque-id as part of ROUTE_TABLE entry inside APP_DB. The ERROR_DB framework will add the opaque-id as a field when they program the entry in ERROR_ROUTE_TABLE. 
A new class is added in fpmsyncd to subscribe to ERROR_ROUTE_TABLE present inside the ERROR_DB.  Subscription to this table is sufficient to handle the errors in route installation.
Currently, fpmsyncd has a TCP socket with Zebra listening on FPM_DEFAULT_PORT. This socket is used by Zebra to send route add/delete related messages to fpmsyncd. We will reuse the same socket to send information back to Zebra. 
fpmsyncd will convert the ERROR_ROUTE_TABLE entry to Zebra common header format and send the message along with the opaque-id. 
On receiving delete message from Zebra, fpmsyncd will also send a delete route message to clean the route from APP_DB so that OrchAgent can process it. If processing this results in a further error, then it silently ignores this.

## 3.5 SyncD

## 3.6 SAI


## 3.7 CLI
### 3.7.1 Data Models
ROUTE_TABLE  schema is modified to add 16-bit opaque-id as a non-key attribute. 
```
;Stores a list of routes
;Status: Mandatory
key           = ROUTE_TABLE:prefix
nexthop       = *prefix, ;IP addresses separated “,” (empty indicates no gateway)
intf          = ifindex? PORT_TABLE.key  ; zero or more separated by “,” (zero indicates no interface)
blackhole     = BIT ; Set to 1 if this route is a blackhole (or null0)
opaque-id     = 16-bit integer
```
A new table is added in CONFIG_DB to enable and disable error_handling feature.
BGP_ERROR_CFG_TABLE
```
key  					= BGP_ERROR_CFG_TABLE:config
```
The above key has field-value pair as {"enable", "true"/"false"} based on configuration.

### 3.7.2 Configuration Commands
A command is provided in SONIC to enable or disable this feature. 

```
root@sonic:~# config bgp error_handling --help
Usage: config bgp error_handling [OPTIONS] COMMAND [ARGS]...

  Handle BGP route install errors

Options:
  --help  Show this message and exit.

Commands:
  disable  Administratively Disable BGP error-handling
  enable   Administratively Enable BGP error handling
  ```
  When the error_handling is disabled, fpmsyncd will not subcribe to any notification from ERROR_ROUTE_TABLE.  By default, the error-handling feature is disabled. 
  
### 3.7.3 Show Commands
```
sonic(config-router-af)# do show bgp ipv4 unicast 
BGP table version is 1, local router ID is 10.1.0.1, vrf id 0
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,# not installed in hardware
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self 
Origin codes:  i - IGP, e - EGP, ? - incomplete

   Network          Next Hop            Metric LocPrf Weight Path
*># 21.21.21.21/32   4.1.1.2                  0             0 101 ?

Displayed  1 routes and 1 total paths  
 ``` 
  
```
sonic(config-router-af)# do show ip route
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR,
       > - selected route, * - FIB route, # - Not installed in hardware

K>* 0.0.0.0/0 [0/0] via 10.59.128.1, eth0, 09:44:37
C>* 4.1.1.0/24 is directly connected, Ethernet4, 00:01:48
C>* 10.1.0.1/32 is directly connected, lo, 09:44:37
C>* 10.59.128.0/20 is directly connected, eth0, 09:44:37
B># 21.21.21.21/32 [20/0] via 4.1.1.2, Ethernet4, 00:00:07
```
A new command has been introduced for seeing the failed routes as follows  
  show {ip | ipv6} route not-installed [prefix/mask]
```
sonic# show ip route not-installed
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR,
       > - selected route, * - FIB route # - not installed in hardware
B> # 22.1.1.1/32 [20/0] via 4.1.1.2, Ethernet4, 00:00:20
B> # 22.1.1.2/32 [20/0] via 4.1.1.2, Ethernet4, 00:00:20
B> # 30.1.1.1/32 [20/0] via 4.1.1.2, Ethernet4, 00:00:20
B> # 30.1.1.2/32 [20/0] via 4.1.1.2, Ethernet4, 00:00:20
B> # 30.1.1.3/32 [20/0] via 4.1.1.2, Ethernet4, 00:00:20
B> # 30.1.1.4/32 [20/0] via 4.1.1.2, Ethernet4, 00:00:20
B> # 30.1.1.5/32 [20/0] via 4.1.1.2, Ethernet4, 00:00:20
B> # 30.1.1.6/32 [20/0] via 4.1.1.2, Ethernet4, 00:00:20
B> # 30.1.1.7/32 [20/0] via 4.1.1.2, Ethernet4, 00:00:20
B> # 30.1.1.8/32 [20/0] via 4.1.1.2, Ethernet4, 00:00:20
```

### 3.7.4 Debug Commands
In order to retry the installation of failed routes from Zebra, a clear command has been provided.  
  clear {ip | ipv6} route {not-installed | <prefix/mask>}
  ```
sonic# clear ip route 
  not-installed  not installed in hardware
sonic# clear ip route not-installed 
  <cr>        
  A.B.C.D/M   ipv4 prefix with mask
  X:X::X:X/M  ipv6 prefix with mask
```
The above command will send route add message for the failed route from Zebra to fpmsyncd. The same message is also sent to BGP so that BGP can advertise the route.


### 3.7.5 REST API Support

# 4 Flow Diagrams
 ![BGP](images/bgp_error_handling_flow1.png "Figure 1: High level module interaction for route install error notification")

__Figure 1: High level module interaction for route install error notification__ 

![BGP](images/bgp_error_handling_flow2.png "Figure 2: Module flow for route add error notification")

__Figure 2: Module flow for route add error notification__

![BGP](images/bgp_error_handling_flow3.png "Figure 3: Module flow for route install success notification")

__Figure 3: Module flow for route add success notification__
 ![BGP](images/bgp_error_handling_flow4.png "Figure 4: Module flow for route delete success/fail notification")

__Figure 4: Module flow for route delete success/fail notification__
 
# 5 Serviceability and Debug


# 6 Warm Boot Support

# 7 Scalability

# 8 Unit Test

The UT testcases are as follows:

|**Test-Case ID**|**Description**|**Status**|**Comments**|  
|----------------|---------------|----------|------------|
1           | Send an   iBGP route from the traffic generator and see that route is learnt in   BGP.                                                                                                                                                                                                                                                           |        | Check   the route in zebra. fpmsyncd should send route to APP_DB. Check APP_DB. Check   that orchagent should send this route to ASIC_DB. Check      that syncd should send the route to ASIC. |
 2           | Send an   eBGP route from the traffic generator and see that route is learnt in   BGP.                                                                                                                                                                                                                                                           |        |                                                                                                                                                                                                |
 3           | Install   a route and check that error status is present in "show ip route"   in zebra.                                                                                                                                                                                                                                                          |        |                                                                                                                                                                                                |
 4           | Install   a route and check that route is present in kernel.                                                                                                                                                                                                                                                                                     |        |                                                                                                                                                                                                |
 5           | Install   a route and check that route is present in APP_DB and ASIC_DB.                                                                                                                                                                                                                                                                         |        |                                                                                                                                                                                                |
 6           | Execute   the command "show bgp ipv4". Check that the error status (installed   in hardware flag) is shown as 0.                                                                                                                                                                                                                                 |        |                                                                                                                                                                                                |
 7           | Check   that routes with installed flag as TRUE is sent to eBGP peers. Also, BGP   ribout list should have this route.                                                                                                                                                                                                                           |        |                                                                                                                                                                                                |
 8           | Check   that routes with installed flag as TRUE is sent to iBGP peers. (Rules for   iBGP and route reflector will apply).                                                                                                                                                                                                                        |        |                                                                                                                                                                                                |
 9           | Send an   iBGP route from the traffic generator and see that route is learnt in BGP.   But this route is not installed in ASIC_DB. Check that error status is   correctly shown in APP_DB.      Please note that route will be present in BGP/Zebra/APP_DB.                                                                                      |        | Check   the route in zebra. fpmsyncd should send route to APP_DB. Check APP_DB. Check   that orchagent should send this route to ASIC_DB. Check      that syncd should send the route to ASIC. |
 10          | Send an   eBGP route from the traffic generator and see that route is learnt in BGP.   But this route is not installed in ASIC_DB. Check that error status is   correctly shown in APP_DB.      Check different kinds of errors like nexthop group-id add failed, nexthop   add failed, route add failed, route table/nexthop table is full etc. |        |                                                                                                                                                                                                |
 11          | Install   a route and check that route is present in zebra, but not in ASIC_DB and   kernel.                                                                                                                                                                                                                                                     |        |                                                                                                                                                                                                |
 12          | Install   a route and check that route is not present in APP_DB and ASIC_DB. (Can be   due to send error from fpmsyncd).                                                                                                                                                                                                                         |        |                                                                                                                                                                                                |
 13          | Execute   the command "show bgp ipv4". Check that the error status (installed   flag) is shown as some non-zero value. Also check the rib failure flag. In   case of error, check if BGP ribout message does not contain this route.                                                                                                             |        |                                                                                                                                                                                                |
 14          | Check   that routes with installed flag as FALSE is not sent to eBGP peers. Also, BGP   ribout list should not have this route. If route has already been sent by   BGP, it should be withdrawn.                                                                                                                                                 |        |                                                                                                                                                                                                |
 15          | Check   that routes with installed flag as FALSE is not sent to iBGP peers. (Rules   for iBGP and route reflector will apply). If route has already been sent by   BGP, it should be withdrawn.                                                                                                                                                  |        |                                                                                                                                                                                                |
 16          | Send a   route. Execute "show bgp ipv4". Check that this command shows the   error status (installed flag).  See   that it shows error.  Also, check the   rib-failure flag.                                                                                                                                                                     |        |                                                                                                                                                                                                |
17          | Send a   route. Execute "show bgp ipv4". Check that this command shows the   error status (installed flag).  See   that it shows error.  Also, check the   rib-failure flag. Now,send the same route again so that it in installed successfully. Check all the flags.                                                                                                                                                                     |        |                                                                                                                                                                                                |
# 9 Internal Design Information
