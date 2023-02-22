
# Zabbix Active Agent Autoregistration

Zabbix can auto add hosts to the inventory. This mechanism is useful when hosts can come from any network (public internet) and we are unable to specify a subnet for autodiscovering hosts via SNMP.

It is highly recommended to secure the zabbix server to agent communication using a PSK and TLS. This will prevent rogue agents from registering with the server. See [[zabbix-agent2]].

## Configuring Zabbix Autoregistration Action

A zabbix autoregistration action can be used to add the new agent/host to the respective host group, link templates, and notify admins.

Reference: https://www.zabbix.com/documentation/current/en/manual/discovery/auto_registration

1. In Zabbix server, open _Configuration_, _Actions_, _Autoregistration actions_ menu and click _Create Action_ button on the top right.
2. In Action _Conditions_, add filters for _Host metadata_ to any of the strings from the _HostMetadata_ value from the zabbix agent config. We can use this to segregate host based on OS (Linux, Windows, etc), environment (prod, test, etc) Linux, or location.
3. In the Actions _Operations_ tab, select the list of actions you need to perform on the matched hosts.

## References

1. Zabbix Agent setup [[zabbix-agent2]]
2. Zabbix server setup in docker [[zabbix-docker]]