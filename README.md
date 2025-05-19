# tui-test




```
V2==================================================
+-------------------+
| Internal User     |
| (Enterprise LAN)  |
+-------------------+
         |
         v
+-------------------------------+
| Infoblox DNS                  |
| - www.internalweb.int.com --> |
|     10.1.10.20                |
| - api.internalweb.int.com --> |
|     10.1.10.20                |
+-------------------------------+
         |
         v
+------------------------+
|   OAG (Access Gateway) |
|  (e.g., 10.1.20.10)    |
+------------------------+
         |
         v
+-------------------+
|    OKTA (SSO)     |
+-------------------+
         |
         v
+-------------------------------------+
|         F5 BIG-IP (LTM)             |
|  +-----------------------------+    |
|  |         VIP_IP              |    |
|  |   10.1.10.20:80             |    |
|  +-----------------------------+    |
+-------------------------------------+
        /                       \
       /                         \
      v                           v
+------------------------------------------------+  +------------------------------------------------+
| OpenShift Cluster A (Active)                   |  | OpenShift Cluster B (Passive)                  |
|  HAProxy Router: 10.1.1.1                      |  |  HAProxy Router: 10.1.1.2                      |
|                                                |  |                                                |
|  +------------------------------+              |  |  +------------------------------+              |
|  |  HAProxy Router Pod(s)       |              |  |  |  HAProxy Router Pod(s)       |              |
|  +------------------------------+              |  |  +------------------------------+              |
|        |           |                            |  |        |           |                            |
|        v           v                            |  |        v           v                            |
|  +-----------+  +-----------+                   |  |  +-----------+  +-----------+                   |
|  |  Route:   |  |  Route:   |                   |  |  |  Route:   |  |  Route:   |                   |
|  | www.      |  | api.      |                   |  |  | www.      |  | api.      |                   |
|  | internal  |  | internal  |                   |  |  | internal  |  | internal  |                   |
|  | web.int.  |  | web.int.  |                   |  |  | web.int.  |  | web.int.  |                   |
|  | com       |  | com       |                   |  |  | com       |  | com       |                   |
|  +-----|-----+  +-----|-----+                   |  |  +-----|-----+  +-----|-----+                   |
|        |             |                          |  |        |             |                          |
|        v             v                          |  |        v             v                          |
|  +-----------+  +-----------+                   |  |  +-----------+  +-----------+                   |
|  | Web App   |  | REST API  |                   |  |  | Web App   |  | REST API  |                   |
|  | Pods      |  | Pods      |                   |  |  | Pods      |  | Pods      |                   |
|  +-----------+  +-----------+                   |  |  +-----------+  +-----------+                   |
+------------------------------------------------+  +------------------------------------------------+
```

1. Initial F5 Command Setup: Manual Failover Only (No Automatic Failover)
To ensure manual failover only (i.e., F5 does NOT automatically fail over to the Passive cluster when Active fails), you must:

Disable health monitors on pool members (so F5 does not automatically mark them down and failover).

Manually enable/disable pool members as needed.

Do not configure auto-failback or failover triggers.


Create Pools and Virtual Servers (for both URLs)
================================================
# Create pool for www.internalweb.int.com
tmsh create ltm pool pool_www_internalweb \
    members add { 10.1.1.1:80 { session user-enabled state user-up } 10.1.1.2:80 { session user-disabled state user-down } }

# Create pool for api.internalweb.int.com
tmsh create ltm pool pool_api_internalweb \
    members add { 10.1.1.1:80 { session user-enabled state user-up } 10.1.1.2:80 { session user-disabled state user-down } }

# Create virtual server for www.internalweb.int.com
tmsh create ltm virtual vs_www_internalweb \
    destination <VIP_IP>:80 \
    pool pool_www_internalweb \
    profiles add { http }

# Create virtual server for api.internalweb.int.com
tmsh create ltm virtual vs_api_internalweb \
    destination <VIP_IP>:80 \
    pool pool_api_internalweb \
    profiles add { http }
(Replace <VIP_IP> with your actual virtual IP address.)



2. Manual Failover: Switch Passive to Active When Active Fails
When the Active cluster (10.1.1.1) fails, manually disable it and enable the Passive (10.1.1.2):

# For www.internalweb.int.com
tmsh modify ltm pool pool_www_internalweb members modify { 10.1.1.1:80 { session user-disabled state user-down } 10.1.1.2:80 { session user-enabled state user-up } }

# For api.internalweb.int.com
tmsh modify ltm pool pool_api_internalweb members modify { 10.1.1.1:80 { session user-disabled state user-down } 10.1.1.2:80 { session user-enabled state user-up } }


3. Revert Back When Failed Cluster Is Online (Restore Active-Passive)
Once 10.1.1.1 is healthy and you wish to revert, enable 10.1.1.1 and disable 10.1.1.2:

# For www.internalweb.int.com
tmsh modify ltm pool pool_www_internalweb members modify { 10.1.1.1:80 { session user-enabled state user-up } 10.1.1.2:80 { session user-disabled state user-down } }

# For api.internalweb.int.com
tmsh modify ltm pool pool_api_internalweb members modify { 10.1.1.1:80 { session user-enabled state user-up } 10.1.1.2:80 { session user-disabled state user-down } }


4. Recommended F5 Setup for Manual Failover Only
Do not assign health monitors to the pools.
This prevents F5 from automatically marking nodes down and failing over.

Do not use priority groups or auto-failback.

Manually control pool member state using session user-enabled/user-disabled and state user-up/user-down.

Document and script the change process for operations teams.

References
This approach is based on disabling/enabling pool members for manual traffic control, as recommended for scenarios where failover must be operator-driven and not automatic


Summary:

Initial setup: Only 10.1.1.1 enabled, 10.1.1.2 disabled.

Failover: Manually disable 10.1.1.1 and enable 10.1.1.2.

Revert: Manually enable 10.1.1.1 and disable 10.1.1.2.

No health monitors or auto-failover mechanisms should be configured if you want full manual control.


Clarifying Priority Groups and Auto-Failback
Priority Groups: In F5 pools, priority groups are used to define which pool members are "preferred" (higher priority) and which are backup (lower priority). If you use priority groups with health monitors, F5 will automatically fail over to the backup when the preferred is down.

Auto-Failback: This is a device-level (traffic group) feature. If enabled, when the preferred device comes back online, F5 automatically moves the traffic group back to it. If disabled, you must manually move the traffic group back.

For your requirement (manual failover only):

Do not use priority groups in pools (just enable/disable pool members manually).

Do not enable auto-failback at the traffic group/device level.









```
V1==================================================
+-------------------+
| Internal User     |
| (Enterprise LAN)  |
+-------------------+
         |
         v
+-------------------------------+
| Infoblox DNS                  |
| - www.internalweb.int.com --> |
|     10.1.10.20                |
| - api.internalweb.int.com --> |
|     10.1.10.20                |
+-------------------------------+
         |
         v
+-------------------+      +-------------------+      +------------------+
|   www.internalweb.int.com |    OKTA (SSO)     | ---> |   OAG (Gateway)  |
|   api.internalweb.int.com |                   |      +------------------+
+-------------------+      +-------------------+              |
         |                                                 v
         +---------------------------------------------+
                                                   |
                                         +-------------------------------------+
                                         |          F5 BIG-IP (LTM)           |
                                         |  +-----------------------------+   |
                                         |  |         VIP_IP              |   |
                                         |  |   10.1.10.20:80             |   |
                                         |  +-----------------------------+   |
                                         +-------------------------------------+
                                          /                                \
                                         /                                  \
                                        v                                    v
        +------------------------------------------------+  +------------------------------------------------+
        | OpenShift Cluster A (Active)                   |  | OpenShift Cluster B (Passive)                  |
        |  HAProxy Router: 10.1.1.1                      |  |  HAProxy Router: 10.1.1.2                      |
        |                                                |  |                                                |
        |  +------------------------------+              |  |  +------------------------------+              |
        |  |  HAProxy Router Pod(s)       |              |  |  |  HAProxy Router Pod(s)       |              |
        |  +------------------------------+              |  |  +------------------------------+              |
        |        |           |                            |  |        |           |                            |
        |        v           v                            |  |        v           v                            |
        |  +-----------+  +-----------+                   |  |  +-----------+  +-----------+                   |
        |  |  Route:   |  |  Route:   |                   |  |  |  Route:   |  |  Route:   |                   |
        |  | www.      |  | api.      |                   |  |  | www.      |  | api.      |                   |
        |  | internal  |  | internal  |                   |  |  | internal  |  | internal  |                   |
        |  | web.int.  |  | web.int.  |                   |  |  | web.int.  |  | web.int.  |                   |
        |  | com       |  | com       |                   |  |  | com       |  | com       |                   |
        |  +-----|-----+  +-----|-----+                   |  |  +-----|-----+  +-----|-----+                   |
        |        |             |                          |  |        |             |                          |
        |        v             v                          |  |        v             v                          |
        |  +-----------+  +-----------+                   |  |  +-----------+  +-----------+                   |
        |  | Web App   |  | REST API  |                   |  |  | Web App   |  | REST API  |                   |
        |  | Pods      |  | Pods      |                   |  |  | Pods      |  | Pods      |                   |
        |  +-----------+  +-----------+                   |  |  +-----------+  +-----------+                   |
        +------------------------------------------------+  +------------------------------------------------+
```
