Local Mitigation (LoM)
=============================================

# Goals
1. Ability to detect anomalies at runtime.
2. Ability to mitigate the deducted anomalies
3. Ability to run required safety checks before mitigation
4. Publish deducted anomalies and results of mitigations via gNMI upon SUBSCRIBE.
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
   
   
# SONiC updates/impacts
## LoM Service/container
1. A new service is added for LoM (Local Mitigation)
2. Service is defined as plugin based, where each action (detection/safety-check/mitigation) can be added as plugins.
3. Plugins are independent entities that are defined via schema
4. Plugins/actions that depend on other preceding actions put their binding via schema.
5. Service is data driven, that it binds actions based on config.
6. All actions that follow an detection is provided a common context that carries data o/p from each preceding action.
7. A subset/all actions can be enabled/disabled via config.
8. The container adds another service/feature as systemd managed.
9. The container will use a private dir /usr/share/sonic/LoM as RW. This dir will be mounted only to LoM container, hence exclusive.
10. The private dir will be used for any runtime update and image upgrade script will carry over for maintianing the updates.

## Redis-DB
### STATE-DB
This holds the current status of actions and their bindings.</br>
[Please refer github repo for final YAANG models]

#### ACTIONS-List
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

            leaf enabled {
                type boolean;
                description "True if enabled";
            }
        }
    }


#### ACTIONS-Bindings
    container LOM_ACTION_BINDINGS {
        description "List of bindings";

        list bindings-list {
            key "ACTION-NAME";
            
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
        
#### Global switches
A way to turn off all.

    container LOM_ACTION_GLOBALS {
        description "global settings"

        leaf disable-anomalies {
            type boolean;
            description "If true, it implies all anomaly detections are turned off.
                This overrides action level enabled switch.";
        }

        leaf disable-mitigations {
            type boolean;
            description "If true, it implies all mitigation actions are turned off.
                This overrides action level enabled switch.";
        }   
    }   
    
#### counters
Table name: LOM_COUNTERS

| COUNTE      | Description                        |
| ----------- | ---------------------------------- |
| Detected    | Total count of detected anomalies  |
| mitigated   | Total count of mitigated anomalies |
| failed-safety-checks   | Total count of safety-checks failed  |
| failed-mitigations     | Total count of mitigations failed    |
| min-TTM                | Lowest time taken to mitigate        |
| Average-TTM            | Average time taken in last N mitigations |
| Percent-resolved-within-1m	| <count mitigated in less than 1m>/<total mitigated> |
| Percent-resolved-within-2m	| <count mitigated in less than 2m>/<total mitigated> |
| Percent-resolved-within-5m	| <count mitigated in less than 5m>/<total mitigated> |
| Percent-count-mitigated	<count of mitigated>/<count detected with configured mitigation> |
