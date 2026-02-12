# Scripts to edit and repair Home Assistant entity and device entries in the relevant HA database tables and registry files.
Specifically, these scripts help to manage Devices and Entities at their source level by allowing complete
flexibility to Rename, Delete, and Move both Entities and Devices along with their underlying registry entries
in `/homeassistant/.storage` and tables in `home-assistant_v2.db`

This allows for cleaner, more flexible, and more complete editing of devices and entities than is possible through the GUI. This also saves manual and error-prone messing with JSON registries and SQL databases.

All routines by default (optionally) create timestamped backups of any changed files or registries before writing 
to them. Also, if working on a live system (as opposed to a copy), the routines by default(optionally) stop 
`ha core` before accessing or changing any files. I also added a fair bit of error checking to prevent db and
registry corruption and to avoid bone-headed mistakes.

To be extra cautious, it's still best to manually shut down core and do a more complete backup yourself. For
example:
```
ha core stop
mkdir -p /backup/backup-$(date +%Y%m%d)
sudo cp -a /config/home-assistant_v2.db /config/*.yaml /config/.storage /backup/backup-$(date +%Y%m%d)
```

**CAUTION:** These routines should only be used by “power users” who know what they are doing, are comfortable
taking risks in manipulating core registry and database code, and want to make changes beyond what is possible with
the non-technical GUI interface

Despite all the error checking, backups, and flow embedded in my code, there are certain to be bugs in my code and
ways to cause damage to your databases that may require you in the worst case to restore from a full backup.

Please look through the code and in particular read the description at the head of each file before using it so 
you know what you are doing. And of course use at your own risk! 
(though bug reports and suggestions are always welcome)

## delete_device+entity
#### Description
Delete devices and/or their corresponding entities completely from homeassistant registries & databases

Useful if you want to:
1. Completely delete existing devices and/or their associated entities to clean up HA from inadvertently added
devices or devices no longer needed that cannot be removed fully or at all using their integration GUI

2. Delete devices and entities that have been partially deleted (and hidden) via the GUI

#### Usage
```
delete_device+entity [-d DEV_NAME | -D DEV_ID | -e ENT_NAME | -E ENT_ID | -M metadata_id ] [-A] [-b BACKUP_DIR] [-x HA_BASE]
  where you need to specify exactly one of: -d, -D, -e, -E, -M
    If -d or -D specified OR -e or -E or -M plus -A specified, then
    the corresponding device and ALL its entities are deleted
        
    If -e or -E or -M specified with no '-A', then ONLY the specified entity is deleted
    
    Note: '-x' is the location of the base homeassistant directory (default: /homeassistant)
```

#### Files affected
- Delete devices ‘.storage’ info from:
```
core.device_registry
```
- Delete entity ‘.storage’ info from:
```
core.entity_registry
core.restore_state
homeassistant.exposed_entities
```
- Delete corresponding entity data from the following home-assistant_v2.db tables:
```
states_meta
states
statistics_meta
statistics
statistics_short_term
state_attributes
```

## rename_device

#### Description
Rename device from OLD_DEVICE_NAME to NEW_DEVICE_NAME and optionally rename name_by_user to NEW_NAME_BY_USER

See also `rename_device+entities`

#### Usage
```
rename_device [-n NEW_NAME_BY_USER] [-b BACKUP_DIR] [-x HA_BASE] OLD_DEVICE_NAME NEW_DEVICE_NAME
  where:
    '-x' is the location of the base homeassistant directory (default: /homeassistant)
```
#### Files affected
```
core.device_registry
```

## rename_entity

#### Description
Rename one or more of the following:
  - Rename entity_id name from OLD_ENTITY_ID to NEW_ENTITY_ID
    (in core.entity_registry, core.restore_state, homeassistant_exposed_entities, lovelace.dashboard_*, yaml files, home-assistant_v2.db)
  - Rename the corresponding 'unique_id' to NEW_UNIQUE_ID (in core.entity_registry)
  - Rename the corresponding 'object_id' to NEW_OBJECT_ID_BASE (in core.entity_registry)
  - Rename the corresponding 'original_name' to NEW_ORIGINAL_NAME (in core.entity_registry)
  - Rename the corresponding 'friendly_name' to NEW_FRIENDLY_NAME (in core.restore_state and if present in HA db)
  - Rename designated entity_id names mentioned anywhere in configurations, automations, scripts [USE carefully]
       See variables: YAML_FILES_BASIC and YAML_FILES_ADDED
  - Rename 'unique_id' in same yaml files when mentioned in format: "unique_id":\s+"UNIQUE_ID"
   - Updates or adds a new attribute for the NEW_FRIENDLY_NAME (if given) and assigns to relevant states)

See also `rename_device+entities`

Rename entity_id name from OLD_ENTITY_ID to NEW_ENTITY_ID and optionally:
- Renames the corresponding ‘unique_id’ to NEW_UNIQUE_ID (in core.entity_register)
- Renames ‘friendly_name’ to NEW_FRIENDLY_NAME (in core.restore_state and if present in HA db)
- Renames designated entities mentioned in configurations, automations, scripts (See variable: YAML_FILES_BASIC and YAML_FILES_ADDED) [USE carefully]
- Updates or adds a new attribute for the NEW_FRIENDLY_NAME (if given) and assigns to relevant states

#### Usage
```
rename_entity [-u NEW_UNIQUE_ID] [-O NEW_OBJECT_ID_BASE] [-o NEW_ORIGINAL_NAME] [-f NEW_FRIENDLY_NAME] [-y] [-b BACKUP_DIR] [-x HA_BASE] OLD_ENTITY_ID [NEW_ENTITY_ID]
  where:
    '-y' also changes in yaml files
    '-x' is the location of the base homeassistant directory (default: /homeassistant)
```

To rename multiple entities with similar substitution, you can use a loop, e.g.,
```
    (OLD="<old_substring>"; NEW="<new_subtring>"; for entity in $(sudo sqlite3 $DB "SELECT entity_id FROM states_meta" | grep "$OLD"); do sudo rename_entity  $entity  ${entity/$OLD/$NEW}; done)
```

#### Files affected
```
  core.entity_registry (if changing: 'entity_id', 'unique_id', 'original_name'
  core.restore_state (if changing: 'entity_id', 'friendly_name')
  homeassistant.exposed_entities (if changing: 'entity_id')
  lovelace.dashboard_* (if changing: 'entity_id')
  *.yaml (if changing 'entity_id')
```
Note: `trace.saved_traces` contains entity_id's and friendly_names but no need to update it since historical
automation debugging data only

#### HA DB tables affected
```
  states_meta [entity_id column]: (if changing 'entity_id`)
  statistics_meta [statistic_id column] (if changing: 'entity_id')
  state_attributes (if changing: 'friendly_name')
  states (if changing: 'friendly_name')
```

Note: 'events_data' is not changed as it represents historical database changes

## rename_device+entities

#### Description
Rename device and associated entities from OLD_DEVICE_NAME to NEW_DEVICE_NAME and optionally:
  - Rename: name_by_user (for device -- see script 'rename_device' for details)
  - Rename: friendly_name (for entity -- see script 'rename_entity' for details)
  - Republish: MQTT discovery topic (and delete old one)

Useful if you want to:
3. Flexibly rename all the different naming components of a device and its associated entities (plus attributes)
when the default name is non-helpful or needs to be changed

4. Republish MQTT discovery message based on new naming (and unpublish old one). \
Useful for auto-discovered rtl_433 devices that have a channel number in their device & entity names that change 
every time the battery is changed, resulting in the endless creation of new devices and entities. Next time you
need to change the battery, all you need to do is to change a single field in the MQTT discovery message
(state_topic or stat_t) to redirect the messages to your now stably and friendly-named device and entities

Note: Calls the scripts `rename_device` and `rename_entity` that are assumed to be
in the same directory as this script (unless otherwise configured)

Note: also changes the names in the yaml files identified in the script `rename_entity` unless the variable
'ADDYAML' is unset

Note: There are a fair number of heuristics used here to make the MQTT publishing work and to seemlessly update various derived device and entity entries based on the old and new device names. I wrote the heuristics based on
the common MQTT sensors I have but YMMV and the heuristics may not work for you. If so, either modify the code or make any missing changes manually.

#### Usage
```
rename_device+entity [-n NEW_NAME_BY_USER] [-f NEW_FRIENDLY_NAME_STEM] [-m NEW_MQTT_TOPIC_STEM] [-b BACKUP_DIR] [-x HA_BASE] OLD_DEVICE_NAME NEW_DEVICE_NAME
  where:
    '-x' is the location of the base homeassistant directory (default: /homeassistant)
    '-y' also changes the entity name in yaml files

  NOTE: NEW_DEVICE_NAME` is used also to automatically update the unique_id by substituting it in place of the OLD_DEVICE_NAME where it occurs in the string
```

#### Files affected
See `rename_device` and `rename_entity`

#### HA DB tables affected
See `rename_entity`

## move_states

#### Description
Move state metadata_ids from: (X1, X2,..., Xn) --> (Y1, Y2,..., Yn)

Useful if you want to:
1. Renumber the metadata_id for the states tables for a given set of sensors if you want to group
similar entities together (in particular all the entities for a given device)
2. Also helpful if you are anal and want to eliminate or rearrange gaps in metadata_id's

#### Usage
```
move_states [-b BACKUP_DIR] [-d HA_DATABASE] -i X1 X2... Xn -o Y1 Y2... Yn
```

#### HA DB tables affected
```
states_meta [metadata_id column]
states [metadata_id column]
```

## move_statistics

#### Description
Move statistic metadata_id's from: (X1, X2,..., Xn) --> (Y1, Y2,..., Yn)

Useful if you want to:
1. Renumber the metadata_id for the statistics tables for a given set of sensors if you want to group
similar entities together (in particular all the entities for a given device)
2. Also helpful if you are anal and want to eliminate or rearrange gaps in metadata_id's

#### Usage
```
Usage: move_statistics [-b BACKUP_DIR] [-d HA_DATABASE] -i X1 X2... Xn -o Y1 Y2... Yn
```

#### HA DB tables affected
```
statistics_meta [id column]
statistics [metadata_id column]
statistics_short_term [metadata_id column]
```

# Use Cases

1. Various integrations (e.g., MQTT, Alexa Media Player) can lead to the creation of unwanted sensors – and often 
even the recreation of such sensors multiple times – that needlessly clutter the database\
\
For example:\
    a. rtl_433_ESP by default listens to and discovers all my neighbors sensors that often have the same general
    name (other than the final channel designation) as my own sensors. This creates clutter and confusion

    b. Alexa Media Player sometimes likes to recreate and duplicate sensor names

    c. One of the calendar functions had a temporary bug that duplicated all calendar sensors which were persisted
    long after the bug was fixed, so I wanted to remove the persistent duplication and resolve the confusion in
    sensor naming

    d. Many rtl_433 humidity and temperature sensors change their name every time you change the battery, resulting 
    in the creation of new sensors with new random names that mess up any automation, script, or card than
    references them

2. Sensors are often created automatically with meaningless or hard to remember names. Given that many if not most
manual configurations require you to reference the actual entity_id sensor name, this becomes challenging to know
what name to use in the first place and then to remember what it all refers to if you ever come back to change or
review the YAML configurations. For examples see the above plus:

    a. Specifically, many rtl_433 sensors have names like sensor.lacrosse_tx141thbv2_0_233_temperature or sensor
    ambientweather_f007th_1_13_temperature that give no indication where they are or even if they are mine

    b. Alexa Media Player creates multiple sensors with names like sensor.this_device or sensor.this_device2, etc.

    c. Nest uses different naming conventions for thermostats vs. remote thermostats

3. Sometimes after the fact I want to change the names of various sensors because either they move, change, or
just want to change my naming scheme.

    a. For example, I have created some custom sensors using custom hardware. My initial MQTT configuration for
    those sensors was lacking and I wanted to change various configuration options without having to delete and reinstall the sensor (thereby losing data)

4. Conversely, if I swap out the sensor measuring a specific quantity (e.g., bedroom temperature) or if I change
the battery and a new sensor is created for the same function, I want to use the same sensor name name and
metadata_id so that I don’t have to change any yaml or configurations and so that if I export the data from
home-assistant_v2.db it will all have the same metadata_id across time for easy and consistent import into other
software

5. Sometimes one makes “mistakes” in configurations, in naming, in manipulations etc. and you don’t want to restore
from a backup so as not to lose data. In that case, these routines can be a lifesaver in undoing changes cleanly
and (relatively) safely.

6. Sometimes you want to go “back in time” and change historical data like fixing the spelling for the
friendly_name in the attribute data so that it is right historically for all time and not just going forward

7. The built-in GUI tools only let one change the display names (corresponding roughly to ‘name_by_user’
for devices and ‘friendly_name’ for entities) but not the sensor names themselves. But most scripts and automation
yaml code requires you to reference the sensor entity_id which is hard to remember and later understand if the name
is sensor.ambientweather_f007th_1_13_temperature

8. Similarly, the built-in tools for deleting devices and entities doesn’t fully delete devices and entities and
all their data.

9. Similarly, some integrations make it challenging if not impossible to delete or even modify existing devices and 
entities. The Forum is filled with requests from people asking how to do so. Sometimes there is an easy answer, but
often I have seen the suggested solution being to completely uninstall and reinstall the entire integration along
with all its associated devices and entities which presumably would lead to data loss and the need to reassociate
the newly installed sensors with their referenced automations and scripts This to me is a non-starter solution, so
short of fixing or re-writing the integration, I can use my code to solve the problem directly.

10. Most importantly, I am anal and OCD when it comes to the organization of my data and just HATE having clutter
and crud in the database and registers from mislabeled, orphaned, or deleted entities and devices – even if hidden from the GUI.

## Postscript
I could go on with many more reasons as I find these routines really useful in keeping my system streamlined – 
especially as I climb the HA learning curve and realize that some of my initial choices and naming conventions were
suboptimal. These routines let me get things right at the foundational level!

Personally, I much prefer manual configurations with YAML or JSON than GUI abstractions that hide the full power
(and complexity) and limit what you can do.
I understand this may not be true for the average non-technical user and per your suggestion, I will add a caution
at the beginning of my initial post.

I should also add that writing this code has given me unparalleled understanding of how HA and its various
integrations ingest, organize, transform, and store data at the most granular level. This is really helpful to
understand how everything works more generally as you abstract up to the GUI. Plus it’s fun :slight_smile:
