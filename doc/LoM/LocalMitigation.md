SONiC Local Mitigations (LoM) Service
=====================================

# Goals
1. Run a containerized service inside the switch that monitors switch constantly, reports any anomalies and mitigate as needed.
2. Provide a way to add/update an action that does one of the following.
   - Run a anomaly detection.
   - Run a safety check that might be required for mitigation.
   - Run mitigation for detected anomaly.
   - Run any cleanup required upon running few actions.
3. Provide a unified way to describe actions in schema.
4. Provide a unified way to configure the actions.
5. Provide a way to run a set of actions sequentially to detect an anomaly and mitigate.
6. Report run of any action to external clients via SONiC events channel.
7. Provide a way to update any action.

# Problem to solve
Today all detections & mitigations are performed by external services. The problems or short falls we have are 
1. As any detection is based on exported data, the latency involved delays the alerting.
2. Any problem/failure in exporting will result in alerts not being created.
   - There have been cases of missing data exports hence missed alerts.
3. Any inability to access the switch, will block any mitigation action.
4. An OS upgrade could impact external service's ability to detect/mitigate whenexpected bhavior changes.
5. A single instance handling many switches incurs inherent scaling issues.

# Proposal
1. Run a dedicated service that can run all detection & mitigation actions locally within the switch.
   - The detection actions run forever until anomaly is detected
   - When detected, run configured mitigation actions, which are often preceded by safety-check actions
   - The entire set of actions to run upon a detection is pre-configured and explicitly ordered. 
   - There may not be any follow up actions, which means we only detect & publish.
2. An action can be a detection for an anomaly, a safety-check for a purpose, a mitigation action for an anomaly, a cleanup, ... 
3. Actions can vary by logic and/or config.
4. Actions are vetted by nightly tests hence verfied to work on the hosting OS.
5. Actions are plugins, so an update/add can be done via file copy.
6. Actions are defined by Schema and schema specifies configurable knobs.
7. Service is built with static config, which can be overrriden via config-DB
8. Action is implemented by plugins that are versioned.
9. The set of available actions, their version & running configuration is reflected in STATE-DB.
10. The CLI extends dynamically to any action using YANG schema.
11. An action add/update/re-config will not impact control/data plane

# Benefits
1. Running locally, hence monitoring the switch constantly will give the **BEST** possible TTD (_Time to Detect_).
   - Running locally provide access to all data, whereas external service strictly relies on published data _only_, which is often vendor/version agnostic.
2. Local mitigation triggered right upon an anomaly detection, will give the **BEST** possible TTM (_Time To Mitigate_).
   - Local mitigations can have TTM in seconds
3. Running locally improves the performance, availability & efficiency.
4. Running locally enables heartbeats that explicitly indicate Device health.
   - A healthy system may report no anomaly but still sends heartbeats to indicate its good health.
5. Running locally enables actions/service to be vetted in nightly tests.
6. The actions can be customized for this version and indeed tested to work with Simulation.
7. The plugin model allows for granular update of actions to ***only*** affected devices with simple file copy & config update.
   - Allows to fix switches running any older builds
   - Plugin update does not affect control & Data planes
8. The plugins as versioned and switches report the running versions. Rollback to any older version can be via small config update.
9. As the actions are defined in YANG schema, the published reports are structured 
10. Actions run have full visibility via gNMI & STATE-DB.
11. For any new SONiC bug found, actions can be rolled out to affected devices only for detection with or w/o mitigation.
    - Create detection & mitigation action & update nightly tests
    - Identify target releases
    - Kick off auto tests runs on target releases.
    - Deploy the tested actions.
    - Note: When a new image is released it has the latest set of actions.
12. Supports enable/disable of actions via config update.
    

# A Use case:
## Link flap / CRC / Discards
For a link some of the more common issues can be flap, CRC-errors & discards. Here we need a way to watch for it and use an algorithm to identify anomalies. In case of anomaly detected, auto-mitigation is expected to watch the link availability and mitigate by bringing link down. The checks can stop at switch level if availability is > 75%, else additional safety checks are used to evaluate before bringing the link down, including escalating to DRI if SCS would fail.

### Detection:
An action may be written for flap by observing STATE-DB changes in link status and an action to watch counters for CRC and/or discards. These actions are run forever until an anomaly is detected or action/service is disabled. Upon detection, it gets published via SONiC events channel that reaches EventHub via low latency channel. Each action is also written into syslog for Kusto based channel, so we have a fallback. Data publiwshed & logged are as per schema.

### Safety check:
A link down auto mitigation needs availability safety check. Returns success if it is > 75%. Whatever it finds gets published via events and logged as well.

### Mitigation:
Link down  mitigation is invoked which just put admin down for this link. 

### conclusion:
The original detected anomaly is re-published with final result as success or not.
In the scenario, where local check suffice, we detect & mitigate in seconds. Every action is published, so external tools are in sync. Even in the case where local check fails (_<= 75%_), we do publish the anomaly in seconds. This happens irrespective of whether device is pushing data w.r.t counters and/or state consistently or not. We have seen cases, where devices miss to publish data.
NOTE: An ICM is fired. If mitigation is done by LoM, the ICM will be marked "_mitigated_".

### Sequencing & config
Every detection action is tied to corresponding safety check & mitigation actions. This is called sequencing, by adding them in ordered sequence. A sequence progresses only if last executed action succeeds. Any failure aborts the sequence and causes the detected anomaly to be re-published with failure code indicating "_mitigation attempt failed_". This is called binding sequence. The Service comes with built-in actions and associated seqeunces.
Every action could be fine tuned via some configurable knobs. A SONiC CLI or configlet could do.

## Memory utilization
This is a complex probem as memory used by a daemon depends on its load (_e.g. count of peers, routes, route-maps configured_), platform and likely need a proper algorithm to vet it fool-proof as "_is daemon really growing constantly_" vs "_transient blips_". Even the threshold could be dynamic based on load level. This algorithm might need few tweaks & updates as we observe, like storage cluster might need a different algorthm. Hence hard coding this into image will not fit. Actions are small plugins/minions that can be easily updated w/o impact to control/data plane.

### Detection
The action does the monitoring. We can have variations per platform & target OS version. This is vetted in nightly tests for its target platform & OS version. It constantly monitors. It can cache usage in various time points as required by algorithm to do a proper analysis. This detection action is kicked off at the start and run until detected or disabled.

### Safety-check
Asses the daemon(s) that is/are in trouble to verify if its restart will impact Dataplane or not. If not, it returns success, else returns failure, which will abort sequence and no mitigation action is executed. This will also look at the service restart history to ensure that no more than N restarts is attempted in M seconds.

### Mitigation
This may be just service restart or more (_say remove/reset files/data_). The action is executed and its result is published

### Conclusion
- The action is anytime updatable as we see need for tuning.
- This type of detection is better equipped to run locally as it has wealth  of data, like per daemon load level, combined picture and better customized for this OS version.
- Mitigation is in seconds if it would not have data plane impact. The most common daemons that face issues are control plane daemons like LLDP.
- Anomaly is re-published with mitigation result to let exernal services be aware of what LoM did.


# Design
![image](https://github.com/renukamanavalan/SONiC/assets/47282725/4255824b-8927-4c81-9962-5e04dc783743)


## Core components
### Engine
This is the core controller of the LoM system. Kicks off the actions that are detections, which are also first action in sequences. A sequence may just have one action, if no mitigation actions are available. It communicates with clients via a published lib for defined client i/f. It waits for plugins to register themselves. Upon registration, it sends out request if the registered action is first action in a sequence. So it may send out multiple requests as one per plugin/action for all detection actions.

The detection action can run forever, as it only returns upon anomaly detected, which may never happen on a good system and which is the most common scenario. As this runs forever, to get its state as running healthy or crashed or stuck, the plugin is required to send periodic heartbeat to engine. The engine collects all heartbeats and send out one heartbeat out via SONiC events via gNMI that has the list of all actions that sent heartbeat since last published heartbeat. This heartbeat makes it visible set of detecton actions that are running in the switch, as healthy. Plugins in trouble reported via syslog and periodically re-published until fixed/disabled. During a mitigation sequence, 

When any action returns, engine publishes its response, check the result code and if good invokes the next action in the sequence if any. In case of failure or no more action in sequence, it re-publishes the original action. The non-first action requests are tracked with timeouts and abort the sequence on timeout. A timedout action continues to be tracked until response or disabled. The stale response from a timedout action is still published for record. 

### Plugin Manager
The plugin Manager manages a set of plugins per config. It interfaces the plugins to the engine. It loads & inits the plugins with config. Registers successfully initialized plugins with engine. Any requests engine initiates for a plugin is received by Plugin Manager and route it to appropriate plugin and re-route the plugin's response to engine, including heartbeats raised by long running plugins.

The plugin manager enables to have a simple plugin i/f, where the plugin never calls out, with exception of periodic heartbeat notifications, but gets called in with just 4 simple blocking methods. On any disable/new/update do appropriate de/re-register the plugins with engine. For any mis-behavior by plugins, manager raises syslog periodically until fixed. The plugin manager also ensures the contract set by plugin i/f is strictly adhere to. It runs the set of plugins under a shared process space.

On the other hand, the plugins can be compiled into Plugin Manager statically. This can drastically bring down the built binary size, hence container image size. The plugin Manager can also load plugins from standalone binaries. 

### Supervisord & rsyslogd
The supervisord to start & manage all processes and rsyslogd to send logs out to host.

### Plugins
The plugins are the core workers that runs an action. A plugin == An action. An action is a standalone entity for a *single* purpose, like CRC-detection/link-availability-check/link-down mitigation/... Each plugin is independent and standalone with no dependency or awareness of other workers. The only binding it may have is the i/p data it requires from preceding plugins. This it expresses via schema with leaf-ref to schemas of all possible plugins/actions that could precede. For example, a link safety check references all link detection actions. A plugin is not be aware of other plugins and act pretty independently. So if there is a shared resource, an abstraction may be provided that enables plugins to access transparently w/o awareness of any other plugins. A sample could be SONiC redis DB where concurrent multi-thread / multi-Go-routine access is not allowed via SWSS common.

The plugins are the unit of actions with a simple i/f of 4 APIs only. All 4 APIs are synchronous even in the intances where a request may run for days/weeks/...
**init**  - Called with action's config. Here the plugin absorbs its config, do its initialization and as well may kick off some long running concurrent activities as needed. 
**GetId** - Ability to get ID & version of this plugin.  
**Request** - Run the action's main job, which may be detection, safety-checks & mitigations. This will be invoked with or w/o timeout. 
**Shutdown** - Way to close/de-init the plugin.

Plugins may be compiled statically into Plugin Manager or it could be explicitly loaded from a standalone binary.

Plugins are versioned. The config sets the version to use. This allows for easy rollback via config update.

One may write variations of same plugin meant for different OS versions & Platform. The buildtime picks the right one.

## LoM Config
It has 4 sets of config and each is detailed in YANG schema.
### Globals.conf
As name signifies the global runtime settings, like ports the internal service listens to, running mode as Prod/Debug, ...

### Procs.conf
The set of plugin manager instances to run with unique runtime ID for each. Associate a set of plugins against runtime-ID to indicate the set of plugins an instance need to load & manage. Each plugin is referreby name, version and optionally path if not statically integrated. When adding/updating a plugin, upon copying the new plugin binary, update this conf to trigger Plugin Manager to load the new/updated binary.

### bindings.conf
An detection action is linked to its mitigation actions with preceding safety checks as ordered set of actions. This is called sequence binding. This binding is provided here with plugin/action names in ordered list. A detection with no mitgation may have no sequence, but just one action with no followups.

Ths first action in a sequence is considered as long running detection action.

### Actions.conf
All actions have some shared config like disable, mimic, heartbeat frequency, .... Each action could have action specific configurable attributes. An example could be window size & thresholds for detections with rolling window, a minimum % of availability to succeed for a safety check and more. The shared configurable knobs are listed in base YANG schema shared by all actions' schema. Each action schema provides its proprietary configurable attributes. 

# Safety Traps
- A detection action has min set frequency to report anomaly. In cases where we don't locally mitigate, the mitigation could take between minutes to hours or even days. Till mitigation is done the anomaly is active. Here we ensure that we do repeatedly raise/publish but at the min frequency set.
   - Any misbehaving plugin is disabled and error is periodically reported until plugin is re-registered, which is likely with an update
- Every followup action and the entire action sequence is watched for time taken.
   - On timeout, the sequence processing is aborted and as well reported/published as mitigation aborted. A periodoc log message is raised for that plugin until response or re-registration. Until the action is considered healthy/active, another request is not sent. 
- Honor red button config which can disable one/many/all actions with SLA.
- Mitigation actions support mimic mode, where we can observe what could have happened w/o making any switch update.
  - A mimiced Link Down will report that it was about to bring link xyz down, but did not as it was in mimic mode.
  - These o/ps could be reviewed along with pre-cursor action results before we enable a mitigation.
- The engine Heartbeats will report the active detections currently running. During mitigation, the position in sequence is reported and heartbeats are more frequent during a sequence run.
   - Any diff between expected list of actions and actions listed in Heartbeats could raise an alert.
   - Absence of heartbeats indicate unhealthy LoM
- The entire state of LoM is available in STATE-DB which is polled out by GWS

# SONiC i/f
LoM is another SONiC service and built as part of SONiC image. Its integration is provided below.

## LoM Service
LoM is a systemd maintained containerized service. Like Telemetry, this service is predominantly written in Go with a vision of supporting non-Go plugins too in future. It is built as docker image with all required systemd's service related files and an entry in FEATURE table as enabled.

## CONFIG
LoM service comes with static config created during build time. The LoM publishes the running config into STATE-DB. The CONFIG-DB updates are needed only when tweaks are needed over static config. The CLI tabbing will dynamically extend based on available schema & schema's contents (_configure=True_) helps fill tabbing with action specific available knobs.

### STATE-DB & CONFIG-DB:
State-DB shows running-config. Config-DB + static built-in config == Running config

**Table**: LoM<br>
**Keys**: "Global", "Procs", "binding", "actions" & "Red-button"

#### LoM|Global
```
"HTTP_RPC_PORT": <port number>
"JSON_RPC_PORT": <...>
... 
```

#### LoM|Procs|< Proc ID >:
```
[
   { "action-name":  "<..>", "plugin-version": "<...>", "plugin-filename": <...?> },
   { "action-name":  "<..>", "plugin-version": "<...>", "plugin-filename": <...?> },
   ...
]

```

#### LoM|Binding|< sequence name >:
```
[
   {
      "name": < action name >",
      "sequence": < Index of the sequence. Expect unique. Executed in ascending order >
   },
   ...
]
```

#### LoM|Actions|< action name >
```
"Description": "..."
"Disable": "True",
"HB_Freq": < seconds > 
"Mimic": "False",
< action specific attrs per schema >
```

#### LoM|RedButton
- This is used to disable one/subset/all actions.
- Unlike other configurations, this RedButton config has a ***SLA*** to reach the target device as this could be disabling an action, which is critical that it is disabled within set SLA
- Red Button config is versioned.
- Update is via CONFIG-DB and LoM updates its running value for the same in STATE-DB.
```
{
   "CreateTimestamp": < Timestamp of the create/udpate by external user >
   "updateTiemstamp": < update timestamp in DUT >
   "DisableAll": True/False  # Disable all actions
   "DisableMitigations": True/False # Disables all mitigations
   "ActionsDisabled": [ < list of action disabled action names > ]
   "targetDevices": <Filter by Metadata; OSVersion >= ... Platform in [ ... ] ...> 
}
```

## LoM Visibility / State / Status
### SONiC events
Results of all actions are published w/o any constraints to Events Hub via SONiC gNMI. Even a stale response (_one that arrives after timeout_) is also published. The publish use SONic Events channel. As per Event's channel usage agreement, all publish o/p are per defined schema with backward compatibility.

### State-DB
#### Actions:
Results of last N actions are cached. This helps when events are missed or brief past history is needed.

#### Counters
Stats are maintained as below to cover all actions
- Count of active actions
- Count of active detection actions
- Count of failed actions (enabled but failed to activate)
- Count of disabled actions
- count of mimiced actions
- Total count of anomaly detections made
- Total count of successful action runs
- Total count failed action runs ( Has non-zero result code )
- Total count of successful sequences
- Total count of failed sequences

#### syslog
All published events are also logged by default by SONiC event module

# Actions Update:
One of the core values of LoM is its flexibility and adaptability to take updates at run time w/o impacting control or data plane. LoM is not built as one monolithic piece but as a collaborative union of plugins bind by config. Each plugin is an **independent** worker item for one specific purpose and this enables an update of a Plugin transparent to rest of the system. The plugins are versioned and use config to update to new version or easy rollback to previous.

## Update requirements
- To update/fix bugs in a published plugin
- To add a new plugin
- To update w/o requiring new SONiC image
- To update all affected versions in a fleet.
- To update w/o impacting control/data plane

## Update creation
- Create a script (bash/Python/...) to vet your action logic.
- Create the plugin for same using LoM Dev environment.
- Add this plugin to a test switch, update config needed to add new action and test it out as plugin.
- Write nightly tests to vet it
- An update to existing plugin may follow subset of above steps based on extent of change.
- Create an self installable archive that has this plugin and configlet to apply with validation logic of target.
  - Running this archive as script copies the plugin to destination and updates the config with built-in configlet.
  - The archive may have metadata for target matching, like "OSVersion <= .... && PLATFORM == .... If present install proceeds on match only.
- Save this archive in a local repo.
  - Services that run update download this onto switch and run it.

## Update vetting
- Identify the target OS versions & Platform for this update.
- Auto create test pipelines for all target scenarios.
- The test pipeline will download the archive from a local repo onto DUT and run the script associated to install.
- Run the nightly tests for all supported actions.
  
## Fleet upgrade
- Take help from FUSE
- FUSE need not do any device isolation for update as these updates are transparent to both control & data planes
- An explicit FUSE policy is created with following info
   - Device metadata to match (e.g. `OSVersion >= 20221131.80`) by specifying few filters.
       - This includes current plugin version <= this update's or not present
   - URL of the self-installable Plugin archive.
- FUSE identifies the matching devices in current scope, download the archive and run the script associated with the archive.

## Fleet update status
- The switch has current running versions in STATE-DB.
- FUSE is used for Action(s)/Plugin(s) update.
  - It updates plugin files & CONFIG-DB.
  - LoM takes in the update, load new plugins and reflect the new values in STATE-DB.
  - On any failure, it sets version to an reserved error string and provide detailed error message in logs repeatedly until fixed.
  - FUSE watch for STATE-DB change with timeout to validate the success of the update.

   
# LoM config update
- As mentioned above the service comes with built-in config. Need config updates only when an update is needed. 
- The SONiC DB schema is provided above.
- YANG schema is provided for config/configlet validation
- NDM will be required to do the updates

# Other teams Ask/Support

## SONiC - Core team
1. To review & approve LoM HLD
2. Help with review of all SONiC image changes
3. Help with review SONiC nightly tests
4. Help with Pipeline updae with action-update added and auto-pipeline creation for all target scenarios.
5. Help with internal repo creation for LoM

## FUSE
1. Help with LoM actions update as described above
2. On any SONiC image upgrade, look for possible plugin updates and apply as part of image update
3. Help with Policy creation & management.
4. Help with internal repo where update archive canbe held

## NDM
1. Add a new schema for LoM objects
2. The rest should be transparent as all validations are run inside SONiC

### LoM Global config
This can be a set of Key-val pairs as map<string, string>. This helps LoM add new vars or retire old ones w/o updating NDM schema.
`map<string, string`

#### Sample
```
"ENGINE_HB_INTERVAL_SECS" : "5",
"MAX_SEQ_TIMEOUT_SECS": "10",
"MIN_PERIODIC_LOG_PERIOD_SECS": 30
```

### LoM Procs Config
This associates a Proc instance with a set of plugins to manage.
```
[
   "<proc-id-0>": [
      {
         "action-name": "<name>",
         "plugin-version": "<version string>",
         "plugin-filename": "<Name of the file>"
      },
      ...
  ],
  "<proc-id-1>": [ ... ]
  ...
]
```
#### Sample:
```
[
   "proc-link-detect": [
      { "action-name": "CRC-Detection", "plugin-version": "1.1.0", "plugin-filename": "crc-detect.0.1.9"},  
      { "action-name": "link-flap", "plugin-version": "1.1.5", "plugin-filename": "linkFlap.xyz"}
   ],
   "proc-link-mitigate": [
      { "plugin-name": "link-safety-check", "plugin-version": "1.2.3", "plugin-filename": "link_safety_ch.8"},  
      { "plugin-name": "link-down", "plugin-version": "1.0.3", "plugin-filename": "link_down_9"}
   ]
]
```

### LoM Actions Config
This configures individual actions. There are common set of attributes shared bu all and each action will have proprietary/custom attributes that can widely vary across actions. The schema of an action will list all its proprietary configurable attributes. To better suit this model, this can be defined as generic `map<key, val>` where key can be any string and val is any string too. The list of actions too vary dynamically. Hence the actions are grouped as a list of action objects with each object holding data as `map<key, val>` where both ekey & val are strings
```
[
   "<action name>": { 
      "<key>": "<val>",
      "<key>": "<val>",
      ...
   },
   ...
]
```

#### Sample
```
[
   "crc-detect": {
      "disable": "false",
      "window-size": "100",
      "min-threshold": "1000",
   }, 
   "link-flap": {
      "disable": "false",
      "flap-cnt": "5",
      "flap-duration": "10"
   },
   "safety-check": {
      "disable": "true",
      "min-availability": "75",
      ...
   },
   ...
]
```
      
### LoM Bindings config
This helps bind a set of actions to run sequentially. First action in a sequence is *always* detection. This action will complete only upon detection of an anomaly. When detected, we need to run a sequence of actions like safety checks and mitigation. This binding associates the required safety checks, miitgation actions and possible cleanup as ordered set of actions via this config. A detection action may not have any action to follow up, if mitigation is not available or not added intentionally. LoM identifies an action to be detection action, if it is available in this config and as first/only action of a sequence.

As actions are dynamic and the sequences too, the config is created as list of entities with key and its value is list of actions as strings. Key is the name of this sequence.
```
[
   "<sequence name>": [
      {
         "action-name": "<Name of the action>",
         "sequence-index": "<Index in this sequence - executed as ordered by sequence>"
      },
      { ... }
      ...
   ],
   ...
]
```

#### Sample:
```
[
   "CRC-Sequence": [
      {
         "action-name": "CRC-detect",
         "sequence-index": "0"
      },
      {
         "action-name": "Link-down",
         "sequence-index": "2"
      },
      {
         "action-name": "Link-Availability-check",
         "sequence-index": "1"
      }
   ]
   "Link-flap": [
      {
         "action-name": "link-flap-detector",
         "sequence-index": "0"
      },
      {
         "action-name": "Link-down",
         "sequence-index": "2"
      },
      {
         "action-name": "Link-Availability-check",
         "sequence-index": "1"
      }
   ]
]
```
         
      
## GWS
### Red Button:
1. Watch for Red Button changes in NDM config.
2. A GWS instance is associated with a set of devices
3. For each matching device  in that set, it pushes the RedButton config.
4. RedButton config has a timestamp of update. This can be compared with red-button config in switch as reflected in STATE-DB. If NDM has later update push it, else all good.
5. GWS helps ensure SLA to update the switch upon any Red Button config changes.

### Heartbeat watcher:
1. Listens for heartbeats for all systems running LoM
2. Raise alert on missing heartbeat
3. Analyze heartbeat contents with its config. If expected actions are missing, raise alert.

## Build/Test pipelines
- As acttions update are intended for all released images, we would need to create Test Run pipeline for any release, run the install script (_as FUSE would do_) and run the entire test suite.
- As the target OS & platforms would be few many, we need a way to automate these pipelines creation

## LoM Dashboard
- Shows all devices running LoM
- Shows by plugin
- A view on counters reported by all devices.