Local Mitigation (LoM)
=============================================

# Goals
1. Ability to detect anomalies at runtime locally within switch.
2. Ability to mitigate the detected anomalies at the point of detection from within the switch.
3. Ability to run required safety checks before mitigation
4. Publish detected anomalies and results of mitigations via gNMI upon SUBSCRIBE.
5. Describe every publish data via schema.
6. Log every published data as a backup source

# Problems to solve. 
## What we have today
1. Today SONiC reports events via events channel & syslog.
2. The external tools monitor published events/logs, correlate with additional data to identify occurrence of an anomaly.
3. The tools reports the anomaly
4. The tools that watch for anomaly attempts to mitigate, which often requires the tool to run command(s) inside the device.
5. The amomaly is directed for manual mitigation if the tool would fail to mitigate.

## Problems
1. Latency</br>
   - The logs/events takes many hops to reach a storage that can be watched
   - The tools handles thousands of switches and hence manages finite resources to manage multiple
   - The tools/components on the path have their own disjoint maintenance windows
   - Environmental factors</br>
   All the above leads to solid latency that can run in minutes in average. Even in the best/happy path, it can take many seconds.
   
2. Reliability</br>
   More the entities in the path from switch publishing an event to mitigation, the reliability goes down as it is the sum of reliability of each component in the path. </br>
   With multiple discrete components shared by multiple different, event the planned maintenance can't be synchronized which can have a say on reliability.
   
3. Reaching the device</br>
   Most of the mitigation actions require to run one or more commands inside the switch.</br>
   A sample could be `sudo config bgp shutdown neighbor <IP>`</br>
   This requires tool to get into the device via ssh.</br>
   If the switch is either not reachable via Mgmt network or has internal issue that blocks user log-in, the auto-mitigation is not possible.
   This anomaly has to be deferred to manual action.
   Any manual action can take in the order of hours/days to mitigate.

4. Hardware abstraction
   A network is likely made of switches from multiple vendors.</br>
   Each vendor have their own logs/events they publish which external tools use for anonaly detection.</br>
   Each vendor have their own CLI command for a single action, like BGP shutdown, which the tools use for mitigation.</br>
   The detection tool has to map to different code/implementation per vendor.
   The mitigation tool has to use an abstraction layer, which takes in the high level command and translate to appropriate CLI command per vendor.
   
5. OS update handling
   Switch OS from any vendor evolves constantly to add features, deprecate some feature, bug fixes, improvements, ...</br>
   Some updates could affect the events/logs published out, which could invalidate/fail the external anomaly detection.</br>
   Some updates may demand different set of commands for a mitigation, which can silently/transparently invalidate the existing mitigation action.</br>
   
6. Knowledge sharing</br>
   A switch may carry new events or deprecate old.</br>
   If unless this knowledge sharing is made into a strict process, this can leave a big gap between anomaly detection/mitigation FW and switches.
   
7. Too bulky
   When a switch upgrades its OS, it takes years before all switches in the network is upgraded to it. While this train is in motion, new trains could be created.</br>
   The tool is *required* to handle all.
   This implies that the tool never upgrades on OS upgrade, but adds one more "else if" clause.
   As trains takes years, there is no process to revisit the tool to purge handlers for versions which no more exist in Network.
   
8. And more ...

## How we solve
Detection & mitigations run **locally** within the switch.
1. Define a containerized service that watches for anomalies
   - Watch locally published events
   - Watch file system 
   - Watch DB entities
   - Watch counters
   - Run some commands periodically to assess switch states
   - ...
  
2. Service correlates events and config to detect anomalies
3. Service runs required mitigations preceded by safety-checks for each anomaly.
4. Detection, mitigation & safety check runs are published as events
   -- While mitigation in progress, publish heartbeats to affirm that mitigation is in progress
6. The service self validates via nightly tests for all its anomalies and the follow-up actions as safety-checks & mitigations
7. The service provides external access to internal/built-in detection/mitigation/safety-check actions

## Gains
1. TTD ~= 0  (Time To Detect)
2. TTM ~= 0  (Time To Mitigate)
3. Switch specific - Just handles this HW only 
4. Evolves wuth OS upgrade -- Nightly ensures integrity by failing for any change needed.
5. The external tools can use built-in detection/mitigation/safety-check actions when/where the LoM service is not functioning.
   - This devoids the need for HW and/or OS level abstractions
6. With no external components involved, not only latency is improved, it improves the reliability too.
7. The detection/mitigation service is **now distributed**.</br>
   Instead of tools creating instances to scale across thousands of switches, the work is now being delegated to every switch to do this using its own resources.
8. The Dev adding/updating a feature, now gets the **ownership** to create & mitigate a new anomaly or update an existing one.</br> 
   In other words now anonmaly detection/mitigation resides with the rightful owner, the switch's Dev/SME.
   
# A sample use case
DHCP relay discard detected and mitigation is to restart the DHCP service, if safety-check(s) would pass.

```
//Anomaly published
{
    "anomalies-dhcp-relay:dhcp-relay-discard": {
        "timestamp": "2022-11-01T17:48:02.35189Z",
        "instance-id": "062445d7-87a0-4600-b482-1921053d103d",
        "anomaly-instance-id": "062445d7-87a0-4600-b482-1921053d103d",
        "anomaly-key": "Ethernet0",
        "ifname": "Ethernet0",
        "service-name": "dhcp-relay",
        "action-type": "anomaly",
        "mitigation-state": "init"
    }
}

// heartbeat for this anomaly which is under processing for local mitigation
// Duplicate of the above except for timestamp & mitigation-state
// Two heartbeats for an anomaly instance will only differ by timestamp.
{
    "anomalies-dhcp-relay:dhcp-relay-discard": {
        "timestamp": "2022-11-01T17:50:02.35189Z",
        "instance-id": "062445d7-87a0-4600-b482-1921053d103d",
        "anomaly-instance-id": "062445d7-87a0-4600-b482-1921053d103d",
        "anomaly-key": "Ethernet0",
        "ifname": "Ethernet0",
        "service-name": "dhcp-relay",
        "action-type": "anomaly",
        "mitigation-state": "in-progress"
    }
}

// Safety-check that completed for this anomaly
{
    "safetychecks-service-restart:service-restart": {
        "timestamp": "2022-11-01T17:51:02.35189Z",
        "instance-id": "c8156f6c-b220-4cd0-bc0b-b803aee3217d",
        "anomaly-instance-id": "062445d7-87a0-4600-b482-1921053d103d",
        "anomaly-key": "Ethernet0",
        "anomaly-data": "<JSON string of o/p from preceding actions, here just anomaly>"
        "service-name": "dhcp-relay",
        "action-type": "safety-check",
        "result-code": 0,
        "result-str": "Safety-check for dhcp-relay service restart succeeded"
    }
}

// Mitigation that is completed for this anomaly
{
    "mitigations-service-restart:service-restart": {
        "timestamp": "2022-11-01T17:51:52.35189Z",
        "instance-id": "01acf729-f03e-40cc-8b9e-cfff07b43ab6",
        "anomaly-instance-id": "062445d7-87a0-4600-b482-1921053d103d",
        "anomaly-key": "Ethernet0",
        "anomaly-data": "<JSON string of from preceding actions, here anomaly & safety-check>"
        "service-name": "dhcp-relay",
        "action-type": "mitigation",
        "result-code": 0,
        "result-str": "Mitigation for dhcp-relay service restart succeeded"
    }
}

{
    "anomalies-dhcp-relay:dhcp-relay-discard": {
        "timestamp": "2022-11-01T17:48:02.35189Z",
        "instance-id": "062445d7-87a0-4600-b482-1921053d103d",
        "anomaly-instance-id": "062445d7-87a0-4600-b482-1921053d103d",
        "anomaly-key": "Ethernet0",
        "anomaly-data": "<JSON string of from preceding actions, here initial-anomaly, safety-check & mitigation>"
        "ifname": "Ethernet0",
        "service-name": "dhcp-relay",
        "action-type": "anomaly",
        "mitigation-state": "complete",
        "result-code": 0,
        "result-str": "Mitigation for dhcp-relay service restart succeeded"
    }
}
```


# Requirements
## Action definitions
1. Define an action via YANG schema
2. The definition includes
   - Type of action as detection/safety-check/mitigation
   - Input & Output parameters associated with the action
   - The i/p parameters sets its source as configuration or o/p parameters from other actions
   - The o/p parameters from other actions are expressed via leafref.
   - The configurable i/p parameters are defaulted
   - The definition also specifies configurable entities that are common across all actions

## Actions visibility
1. Every completion of a run of an action reflects as follows
   - Data is published via events channel
   - Data is logged via syslog with specific prefix.
   - last N actions' o/p are cached in STATE-DB for anyone to query/watch.
  
2. Any action can be explicitly requested via CONFIG-DB
   - The request can be placed with all data and a time limit.
   - Only one action can be requested at a time.
   - A request will be acted upon by LoM Only once.
   - Result of the run will be published/logged/cached as above with flag indicating manual invoke.
   - The request is always placed with an max lifetime set as seconds since epoch to expire.
   
## Actions control
1. Actions can be disabled via global or action-specific switch
2. Dry run of actions are possible with mimic turned on. Here LoM will not make any state change to switch's control/data plane.
3. Service is controlled via FEATURE flag

## LoM service
1. Runs all the anomaly actions concurrently or invoked periodically as per requirements of the action.
2. Runs sequence of bound safety-checks & mitigations per running config.
3. Supports plugin model for action implementation.
4. Supports multiple languages as Python, Go & C/C++ for action plugin implementation.
5. Runs a basic engine/action model where back engine runs as separate process that manages external processes that run the plugins.
6. I/F between Engine & Plugin processes are defined via C lib, which can be invoked from supported languages *directly*, devoding the need for SWIG abstraction.
7. A plugin process may manage multiple actions.
8. Runs only one mitigation sequence at any one time.
9. Anomaly detections go stale if not acted upon set time limit and get aborted.
10. Ensures every action is invoked with list of outputs from preceding actions in the chain.
11. Comes with built-in config for all needed. Hence self sufficient to run with no external configuration.
12. Service honors all configuration settings.
13. Running config = Built-in config + config updates
14. The running config is carried over upon image upgrade.
15. A CLI for running config is extendeded to LoM service.

# Overview
![image](https://user-images.githubusercontent.com/47282725/205552069-19e7e1d3-5222-4494-af76-7be5f4f1e6cd.png)



# SONiC updates/impacts
## LoM Service/container
1. A new service is added for LoM (Local Mitigation)
2. Service is internally architected as plugin based, where each action (detection/safety-check/mitigation) can be added as plugins.
3. The service is fully self contained for all actions with exceptions of mitigations to be run at host level.
4. The service will be mounted with a host dir, exclusive to this feature as RW.
5. The service will use this for any data it shares with host services like D-Bus and image-upgrade script.
6. The mitigations that need to run at host context uses host's D-Bus service.
7. The D-Bus scripts are copied into host path mounted as RW and host-DBus service is configured to load scripts from here.
8. The service maintains the running config in this dir and used by show command.
9. Any updated plugun scripts could be copied into this dir and the service will reload upon restart/SIGHUP.
10. The image-upgrade script will carry over the contents of this dir during image upgrade to carry over the current state.
11. The container image is created and added as a service/feature managwed by systemd & FEATURE table.
12. This service is integrated with sonic-buildimage via submodule update.
13. The YANG schema will be created along with other SONiC YANG modules.
14. Nightly tests will be added for each action.

## Redis-DB
### CONFIG/STATE-DB
- The service comes with pre-built configuration. 
- This can be tweaked via CONFIG-DB. The CONFIG-DB is used only for incremental updates
  - The schema denotes configurable parameters explicitly
- The current running config is written into STATE-DB. This will be the final running config which is built-in config that is tweaked from incremental updates. Hence the final running/in-use config.
- The current config is also saved into a file in host mounted dir, which will be taken over during image upgrade. This way any updates gets carried over to the new image.
- The CLI commands will be provided to config/show. The pre-existing gNMI support could be extended to the new DB objects transparently.
- Explicit action invocation can be requested via CONFIG-DB with an expiry time.
  - A generic object to invoke any action with required inputs specified with time point of expiry
  - This can be used by DRIs/tools to tap on Switch's ability to run a mitigation or safety-check or to report an anomally manually to trigger the chain of actions per configuration.

[Please refer github repo for final YANG models]

#### ACTIONS-List
This lists all the actions.
The CONFIG-DB entry is created only for tweaking.  Any update is absorbed and internally persisted into running config.
An explicit delete of an update in CONFIG-DB will revert the config back to built-in settings.
Hene CONFIG-DB may have none or a subset of actions only and carries only the diff/tweaks made.
The STATE-DB will have the complete list of actions with current config data in use.

    container LOM_ACTIONS {
        description "List of actions";

        list actions-list {

            key "ACTION-NAME";

            leaf ACTION-NAME {
                type string;
                description "Fully qualified ame of the action as
                    <yang module name>:<container name>";
            }

            leaf action-type {
                type cmn:action-types  // Marks as anomaly / mitigation / safety-check
            }

            leaf action-config {
                type string
                description "
                  Config knobs are custom to Action and hence configured as JSON string.
                  Each action that needs some config defines the same in its schema.
                  The configurable fields are marked with "config true;"
                  The writer honors the schema via the common validation process
                  which is applied to any CONFIG-DB update.";
            }
            leaf disable {
                type boolean;
                default False;
                description "Disable is set as configurable & base-entity in schema.
                     Hence not part of action-config which holds custom entities per
                     action.";
            }
            leaf mimic {
                type boolean;
                default False;
                description "Mimic is set as configurable & base-entity in schema.
                     Hence not part of action-config which holds custom entities per
                     action. 
                     True - Runs the action, data is published but with return code
                     indicating mimicked.";
            }
        }
    }


#### ACTIONS-Bindings
Each anomaly action is bound to one or more actions as safety-checks & mitigations.
Each action comes with built-in binding, which can be tweaked via CONFIG-DB.
The STATE-DB reflects running bindings for all anomaly actions.

    container LOM_ACTION_BINDINGS {
        description "List of bindings";

        list bindings-list {
            key "ACTION-NAME";
            config true;
            
            leaf ACTION-NAME {
                type leafref {
                    path "../../LOM_ACTIONS/ACTION-NAME";
                }
                description "Name of the action" of the action as
                    <yang module name>:<container name>";
            }   
            
            list bindings {
                key "sequence"

                leaf sequence {
                    type uint8;
                    description "Order of running. Lower sequence go before higher";
                }

                leaf ACTION-NAME {
                    type leafref {
                        path "../../LOM_ACTIONS/ACTION-NAME";
                    }
                    description "Name of the action" of the action as
                        <yang module name>:<container name>";
                }
            }
        }
        
#### Global/Service level config
- As above the service comes with the config pre-defined as built-in.
- The CONFIG-DB could be used to tweak the same.
- The STATE-DB will reflect the final values in use.
- The CLI commands will be provided to config/show.

    container LOM_ACTION_GLOBALS {
        description "global settings";
        config true;
        
        leaf disable-anomalies {
            type boolean;
            description "If true, it implies all anomaly detections are turned off.
                This overrides action level enabled switch.";
        }

        leaf disable-mitigations {
            type boolean;
            description "If true, it implies all mitigation actions are turned off.
                This overrides action level enabled switch, when True.";
        }
        
        leaf mimic-mitigations {
            type boolean;
            description "If true, it implies all mitigation actions are mimicked.
                This overrides action level enabled switch, when True.";
        }
        leaf min-recur-duration {
            type uint8;
            default 0;
            description "Minimum duration between two reports for the same anomaly.
                An anomaly, when local mitigation not possible, will take some solid time
                before auto/manual fix is done. During this time, when the anomaly stays
                active, we need periodic alert and this fixes the duration between two
                reports.";
        }
    }   

#### ACTIONS-Cache
Each event published is also cached in STATE-DB wirh a max cap of N
Each event is cached as JSON string with timestamp.
This can be used by tools as a backup when events channel's streaming is broken.

   container LOM_ACTION_CACHE {
        key "instance-id, timestamp"
      
        leaf timestamp {
            type yang:date-and-time;
            description "time of the event";
        }
        leaf instance-id {
            type yang:uuid;
            description "UUID for this instance";
        }
        
        leaf data {
            type string;
            description "JSON String of the published data";
        }
   }
   
#### ACTIONSRequest
Way to request explicit run of an action or report an anomally manually.

   container LOM_ACTION_REQUEST {
         key "instance-id";
                        
        leaf instance-id {
            type yang:uuid;
            description "UUID for this instance";
        }       
        leaf exp-timestamp {
            type yang:date-and-time;
            description "time of the event";
        }
        
        leaf action-data {
            type string;
            description "Initial data to invoked actions
               This data along with instance-id will be published as user
               created anomaly";
        }
        
        leaf binding-name {
            type string;
            description "Name of the anomaly action whose binding
               is executed; In the absence of this go by provided
               list of bindings";
        }
        
        list bindings {
            key "sequence"

            leaf sequence {
                type uint8;
                description "Order of running. Lower sequence go before higher";
            }   

            leaf ACTION-NAME {
                type leafref {
                    path "../../LOM_ACTIONS/ACTION-NAME";
                }   
                description "Name of the action" of the action as
                    <yang module name>:<container name>";
            }   
        }   
   }
              

#### counters
Table name: LOM_COUNTERS

| COUNTER      | Description                        |
| ----------- | ---------------------------------- |
| Detected    | Total count of detected anomalies  |
| mitigated   | Total count of mitigated anomalies |
| failed-safety-checks   | Total count of safety-checks failed  |
| failed-mitigations     | Total count of mitigations failed    |
| Percent-resolved-within-1m	| < count mitigated in less than 1m >/< total mitigated > |
| Percent-resolved-within-2m	| < count mitigated in less than 2m >/< total mitigated > |
| Percent-resolved-within-5m	| < count mitigated in less than 5m >/< total mitigated > |
| Percent-count-mitigated	| < count of mitigated >/< count detected with mitigation configured as enabled > |

## DBUS usage
1. Mitigations that *require* commands running in the host scope are executed via D-BUS
2. Mitigation scripts that are to run via D-Bus are maintained in a dir that is exclusive to LoM service.
3. SONiC-host-service enabled to register scripts from this dir.
4. A mitigation action will invoke required script.
5. Each invocation is protected by a instant key, which is provided to script run time via file and passed in as arg by the caller. This helps protect from inadvertent use by anyone else.

## gNMI commands
1. The existing gNMI command to view events could be used to view all published actions.
2. The new gNMI commands under development (via different project) for config update can be transparently used for any CONFIG-DB updates.
3. The current gNMI commands to view STATE-DB gets transparently extended to the STATE-DB additions.
4. In short this project does not introduce any new gNMI commands.

## Service Update
1. Service update can be divided as 3 parts.
   1. Config update of actions, binding and global changes/tweaks.
   2. Add/update plugins for actions.
   3. Service core update.
   
2. Config update:
   - Possible via gNMI channel being added for any configuration update.
   - CLI will be provided for new objects.
   - Running config is separately maintained by the service.
   - Image upgrade script is tweaked to carry over this running config to new image.
   
3. Plugins code update:
   - Possible via file copy into the switch
   - New plugin files may be copied to the path mapped to container.
   - A service restart/SIGHUP to relevant processes will reload the new file.
   - Image upgrade script will carry over upon image update.
   - Plugin files are versioned and the new image will load the latest, which could be from carried over file or built-in.
   
4. Service update
   - Build a new container image and push to ACR.
   - Container images are versioned as in SONiC.
   - Set the new version as golden version in FEATURE table in CONFIG-DB via CLI/gNMI.
   - A built-in anomaly is triggered when golden version != running version
   - The mitigation action bound to this anomaly is triggered which executes script via D-BUS.
   - The mitigation action script via D-Bus does upgrade as follows.
     1. downloads the new docker image
     2. Load the new docker image
     3. Tag it as the latest.
     4. Restart the LOM service
     5. Run post-checks.
     6. On success, remove old image.
     7. On failure, tag old image as latest and remove new image followed by service restart
   - NOTE: Systemd is configured with 1 min timeout so as to allow any on going mitigation to complete before exiting for a stop/restart.

5. SONiC image update
   - SONiC image is built with this container image, like all other features.
   - Any image upgrade will install the built-in image
   - Based on config, it would self update as needed.
   

# LoM service Actions:
Therea are 3 types of actions as below. Each are independent units of actions bound together via pre-declared/run-time-config.
1. Anomaly detection
2. Safety-checks
3. Mitigations

### Anomaly-detections
  - These actions run as plugins and their only goal is to detect occurrence of an anomaly.
  - The code that runs for detection runs forever or may demand periodic invocation.
  - The code that runs until detection sleeps on signal/event or do a periodic poll on some object, which may be DB-counters/scan a file
  - Returns to the caller upon successful detection or an internal failure.
  - Successful detection reports data as per schema.
  - Nightly tests ensure the data integrity in each OS release.
  - The engine publishes it via events channel and passes it to subsequently invoked safety-checks and mitigations as part of shared context.
  - The config specifies mitigations to run with order specified.
  - A config update can override this.
  - An anomaly can be manually reported via DB to trigger follow up actions or provide a custom set of actions to run.
    -  This is to enable external services/DRI to trigger repair actions for an anomaly explicitly. 
  - Publishing
    - Anomaly is published upon detection with data as per schema.
    - An instance of anomaly detection is uniquely identified by a UUID, called instance-id.
  - Heartbeats:
    - Anomaly is re-published with same instance ID, but different timestamp.
    - Status set to in-progress
  - Completion
    -  Anomaly is re-published with same instance ID, but different timestamp.
    -  Status set to *complete*
    -  Mitigation status code set to 0 for success or appropriate non-zero error code
    -  Mitigation status string set to human readable description of the mitigation status code.
 - For external tools
    - The run of corresponding mitigations may complete / abort.
    - In either case, the result is published.
    - The external service that watches for anomaly events could track its local mitigation progress via subsequent published events for same instance-ID.
    - Watch for heartbeats / completion.

### Mitigation-actions
  - These are independent units of actions that can be used as mitigation by an anomaly.
  - An anomaly may use one or more mitigation actions as ordered.
  - A couple of mitigation actions could run as ordered.
    - E.g., *config-reload* followed by *Service-restart* could be required as full mitigation.
  - Like anomalies bind to mitigations, mitigations bind to safety-checks.
  - The only difference is that for anomalies, mitigations follow its occurrence, whereas for mitigations, the safety-checks precedes the mitigation run.
  - A mitigation action may require some i/p from any of the preceding actions (anomaly/safety-check/mitigation)
    - E.g., service-restart is a mitigation action, and it requires the name of the service as i/p.
    - E.g., config-reload is a mitigation action and it does not need any i/p from the triggering anomaly.
  - Mitigations that require i/p from an action must refer to corresponding parameter in each action's schema.
    - Use leafref from mitigation schema to refer to referring schema.
    - In case of multiple referrers, use union of leafref
    -  Multiple different anomalies may need service-restart and each anomaly should specify its service name as data.
    - The schema validation will help ensure the mitigation's requirements are in-place in every referring schema.
  - Result of every mitigation run is published.
    - Each run is uniquely identified by an UUID as instanceId.
  - The published data includes the instance-ID of the corresponding anomaly.
    - All actions that run for instance of an anomaly, include the instance-id of the anomaly.
    - This can be used to track all instances of all actions for a single instance of an anomaly.
  - Each mitigation action is timed by engine. The action could configure its own timeout over the default timeout used by engine.

### Safety checks
  - These are independent units of actions that can be used as safety-checks by mitigations.
  - Like mitigation its inputs are made to reference other actions that are expected to precede.
  - The data published by safety checks could be referred to by other actions.
  - The scope of the checks could be local or spans neighbor devices in same tier and/or more.
  - Each safety-check action is timed by the engine. The action could configure its own timeout over the default timeout used by engine.
  - The result of each safety-action run is published with an UUID that denotes each instance uniquely.
  - The published data includes the instance-ID of the corresponding anomaly

# Schema
##  Overview:
1. This defines the action as data with configurable entities explicitly marked (*config true;*)
2. All related actions are declared under a single YANG schema module.
3. YANG schema module is versioned.
4. For each action the following are true  
   - Action is defined as a YANG schema container.
   - instanceID - Action has its own instance-id, which is a UUID to globally identify instance of an action.
   - AnomalyInstanceId is the Instance ID of the associated anomaly which triggered this action.
   - Type as anomaly or mitigation or safety-check

5. Each action declares its i/p & o/p parameters as attributes.
6. The i/p attributes are tied to referred schemas via schema reference.
   - When referring multiple schemas use union
7. The o/p attributes are the result of the action.
   - The o/p attributes could be referred to by others.
8. The mitigation & safety-checks have result-code and result-str.
9. The published data per action is per this schema.
10. Refer git repo for the final definitions.


# Events reliability
  - Events channel from Switch to the final consumer often has multiple hops and which increases the probability of loss.
  - An event just published once has higher chance of getting lost.
  - When an anomaly is detected and published, it is followed by heartbeat/safety-check/mitigation publishing.
  - Each publishing carries its data and data of the preceding actions.
  - Hence even if the external tool receives any one of the many, it gets the history of all preceding.
  - When final anomaly-completed is published, it is repeated few times, as N times every 5 seconds.
  - The completed anomaly event also carries the results from preceding safety-checks & mitigation actions.
  - Only when events are missed for an extended period, an anomaly gets missed.
  - Whenever a tool identifies an anomaly and attempt to mitigate, can query the switch cache to assess any actions by LoM.
  - A tool that attempts to do any switch update is expected to disable all actions/mitigations to pause LoM while it is repairing/updating the switch.

# TESTS
  - Each action is covred by unit tests & nightly tests
  - Nightly tests will validate the data against schema
  - All config tweaks & state-DB updates are tested via nightly tests.
  - Events are simulated or mocked as needed.
