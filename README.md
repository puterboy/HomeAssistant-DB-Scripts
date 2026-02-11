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



Rename entity_id name from OLD_ENTITY_ID to NEW_ENTITY_ID and optionally:
- Renames the corresponding ‘unique_id’ to NEW_UNIQUE_ID (in core.entity_register)
- Renames ‘friendly_name’ to NEW_FRIENDLY_NAME (in core.restore_state and if present in HA db)
- Renames designated entities mentioned in configurations, automations, scripts (See variable: YAML_FILES_BASIC and YAML_FILES_ADDED) [USE carefully]
- Updates or adds a new attribute for the NEW_FRIENDLY_NAME (if given) and assigns to relevant states

#### Usage
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

### HA DB tables affected
```
  states_meta [entity_id column]: (if changing 'entity_id`)
  statistics_meta [statistic_id column] (if changing: 'entity_id')
  state_attributes (if changing: 'friendly_name')
  states (if changing: 'friendly_name')

Note: 'events_data' is not changed as it represents historical database changes


