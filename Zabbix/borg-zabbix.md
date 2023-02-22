# Borg Backup of Zabbix

Zabbix DB contains metadata about the zabbix server itself and the telemetry data collected from the hosts. The telemetry data is extensive and accounts for over 95% of the DB backup.

The below script dumps only the schema for the large tables and schema + data for the smaller tables.

Note, this script is for Zabbix v4.x using MySQL DB only. This script needs to be adapted for Postgres + TimescaleDB.

#obsolete all new deployments use Postgres + TimescaleDB, no more MySQL
#TODO update this script for postgres + timescale.

## borg-zabbix.sh

```bash
#!/bin/bash

# Backup zabbix using borg
# Dump tables with zabbix configuration, ignore event data from large tables
# Only the schema is dumped for large data tables
# Raja Subramanian <rajasuperman@gmail.com>

# Works with Zabbix 4.x
# Table names for dumping data and/or schema are extracted from
#  https://zabbix.org/wiki/Docs/howto/mysql_backup_script

BASEDIR='/data/svr2/svr1'
STAGING="${BASEDIR}/staging"
DUMPFILE="${STAGING}/mysqldump-all.sql"

HOSTNAME='10.10.4.51'
REPO="${BASEDIR}/borg-repo"

DBNAME='zabbix'

# Large tables, dump only schema
SCHEMA_TABLES='acknowledges alerts auditlog auditlog_details event_recovery events event_suppress event_tag history history_log history_str history_text history_uint problem problem_tag task task_acknowledge task_check_now task_close_problem task_remote_command task_remote_command_result trends trends_uint '

# Smaller tables, dump data
DATA_TABLES='actions application_discovery application_prototype applications application_template autoreg_host conditions config corr_condition corr_condition_group corr_condition_tag corr_condition_tagpair corr_condition_tagvalue correlation corr_operation dashboard dashboard_user dashboard_usrgrp dbversion dchecks dhosts drules dservices escalations expressions functions globalmacro globalvars graph_discovery graphs graphs_items graph_theme group_discovery group_prototype host_discovery host_inventory hostmacro hosts hosts_groups hosts_templates housekeeper hstgrp httpstep httpstep_field httpstepitem httptest httptest_field httptestitem icon_map icon_mapping ids images interface interface_discovery item_application_prototype item_condition item_discovery item_preproc items items_applications maintenances maintenances_groups maintenances_hosts maintenances_windows maintenance_tag mappings media media_type opcommand opcommand_grp opcommand_hst opconditions operations opgroup opinventory opmessage opmessage_grp opmessage_usr optemplate profiles proxy_autoreg_host proxy_dhistory proxy_history regexps rights screens screens_items screen_user screen_usrgrp scripts service_alarms services services_links services_times sessions slides slideshows slideshow_user slideshow_usrgrp sysmap_element_trigger sysmap_element_url sysmaps sysmaps_elements sysmap_shape sysmaps_links sysmaps_link_triggers sysmap_url sysmap_user sysmap_usrgrp tag_filter timeperiods trigger_depends trigger_discovery triggers trigger_tag users users_groups usrgrp valuemaps widget widget_field'

echo ========= $DBNAME BACKUP OF $HOSTNAME started on `hostname` =========
echo

echo ====== mysqldump started `date`
# Dump schema only
mysqldump --order-by-primary -u backup_admin -h ${HOSTNAME} --routines --opt --single-transaction --skip-lock-tables --no-data $DBNAME --tables $SCHEMA_TABLES >  $DUMPFILE

# Dump schema + data and append to DUMPFILE
mysqldump --order-by-primary -u backup_admin -h ${HOSTNAME} --routines --opt --single-transaction --skip-lock-tables --extended-insert=FALSE $DBNAME --tables $DATA_TABLES >>  $DUMPFILE

echo ====== mysqldump completed `date`

borg create -s -C lzma,5 ${REPO}::${HOSTNAME}-${DBNAME}-{now} ${STAGING}
```