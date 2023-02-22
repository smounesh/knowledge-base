# Zabbix Agent 2 Setup

1. In order to monitor all host level parameters, zabbix agent 2 is installed natively on the host and not inside docker.
2. zabbix agent 2 uses PSK to securely authenticate and encrypt zabbix server/agent communication.

## Downloading and Installing the Agent

1. Download the appropriate Zabbix Agent 2 version from https://www.zabbix.com/download. Eg. `Zabbix 6.2`, `Ubuntu Jammy`, `Agent 2`.
2. After installing the `zabbix-release` deb, install `apt install zabbix-agent2`. The zabbix-agent2 plugins are only for Postgres and Mongo.
3. If you are monitoring docker containers,  add the zabbix user to the docker group using`adduser zabbix docker`.

## Creating and Securing the PSK

Reference: https://www.zabbix.com/documentation/current/en/manual/encryption/using_pre_shared_keys

```bash
PSKFILE=/etc/zabbix/zabbix-agent2.psk
openssl rand -hex 32 > ${PSKFILE}
chown zabbix. ${PSKFILE}
chmod 400 ${PSKFILE}
```

## zabbix-agent2 config fragment

```txt
# Filename: /etc/zabbix/zabbix_agentd.conf.d/00-agt.conf
# zabbix-agent config fragment

# Unset Hostname as it's set by default in the main config
Hostname=

# Permit any Zabbix server to connect to this agent, but correct PSK and TLS is mandatory
Server=0.0.0.0/0
ServerActive=100.75.158.19
HostMetadata=Linux Server Production Docker 7712cef1e30cf474e0f46f93727c4eba

# See Zabbix Server menu _Administration_ > _General_ > _Autoregistration_
TLSConnect=psk
TLSAccept=psk
TLSPSKFile=/etc/zabbix/zabbix_agent2.psk
TLSPSKIdentity=AGT
```

## Securing Zabbix Server Autoregistration

We want to ensure all zabbix agents that attempt to auto register with the server are secured with the PSK and TLS encryption. This prevents unknown agents/clients from registering with the server.

1. In Zabbix server, open _Administration_ > _General_ > _Autoregistration_ menu.
2. Encryption Level: ensure _No Encryption_ is disabled and _PSK_ is enabled. We want all clients to authenticate and encrypt in transit data using the PSK.
3. Set the _PSK Identity_ field to _TLSPSKIdentity_ from the zabbix config fragment above.
4. Set the  _PSK_ field value to the contents of the _zabbix-agent2.psk_ file.

## References

1. Zabbix server setup in docker [[zabbix-docker]]
2. Autoregistering hosts with the server [[zabbix-active-agent-autoregistration]]