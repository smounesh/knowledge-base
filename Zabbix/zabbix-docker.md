# Full Stack Zabbix 6.2 Using docker-compose

Full stack Zabbix  6.2 using docker-compose. Sets up the following 4 dependent containers:

| Serial | Container Name         | Description/Purpose               |
| ------ | ---------------------- | --------------------------------- |
| 1      | zabbix-web-nginx-pgsql | Zabbix front end                  |
| 2      | zabbix-server-pgsql    | Zabbix application server         |
| 3      | zabbix-snmp-traps      | Zabbix SNMP trap server           | 
| 4      | timescaledb            | Postgres with Timescale extension |

#TODO: zabbix-snmp-traps is untested.

## docker-compose

```yaml
# Filename: /srv/zabbix.localhost/docker-compose.yml
# Purpose: To run zabbix server and zabbix web along with postgres database

version: "2.4"
services:
  zabbix-web:
    image: zabbix/zabbix-web-nginx-pgsql:6.2-ubuntu-latest
    container_name: zabbix-web.zabbix
    restart: always

    mem_limit: 4G

    environment:
      DB_SERVER_HOST: timescale
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      ZBX_SERVER_HOST: zabbix-server
      PHP_TZ: Asia/Kolkata

    ports:
      - 80:8080
      - 443:8443

    volumes:
      - /etc/localtime:/etc/localtime:ro

    healthcheck:
      test: curl -f http://localhost:8080/ping || exit 1
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 30s

    depends_on:
      timescale:
        condition: service_healthy

    labels:
      - com.centurylinklabs.watchtower.enable=true


  zabbix-server:
    image: zabbix/zabbix-server-pgsql:6.2-ubuntu-latest
    container_name: zabbix-server.zabbix
    restart: always

    mem_limit: 4G

    ports:
      - "10051:10051"

    environment:
      DB_SERVER_HOST: timescale
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

    volumes:
      - /etc/localtime:/etc/localtime:ro
      - ./data-server/alertscripts:/usr/lib/zabbix/alertscripts:ro
      - ./data-server/externalscripts:/usr/lib/zabbix/externalscripts:ro
      - ./data-server/dbscripts:/var/lib/zabbix/dbscripts:ro
      - ./data-server/export:/var/lib/zabbix/export
      - ./data-server/modules:/var/lib/zabbix/modules:ro
      - ./data-server/enc:/var/lib/zabbix/enc:ro
      - ./data-server/ssh_keys:/var/lib/zabbix/ssh_keys:ro
      - ./data-server/mibs:/var/lib/zabbix/mibs:ro
      - ./data-snmptraps:/var/lib/zabbix/snmptraps

    depends_on:
      timescale:
        condition: service_healthy

    labels:
      - com.centurylinklabs.watchtower.enable=true

  zabbix-snmptraps:
    image: zabbix/zabbix-snmptraps:6.2-ubuntu-latest
    container_name: zabbix-snmptraps.zabbix
    restart: always

    mem_limit: 4G

    volumes:
      - ./data-snmptraps:/var/lib/zabbix/snmptraps

    depends_on:
      timescale:
        condition: service_healthy

    labels:
      - com.centurylinklabs.watchtower.enable=true


  timescale:
    image: timescale/timescaledb:latest-pg14
    container_name: zabbix-timescale.zabbix
    restart: always
    user: postgres

    mem_limit: 6G

    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: zabbix
  
    volumes:
      - ./data-timescale/data:/var/lib/postgresql/data

    healthcheck:
      test: pg_isready
      interval: 5s
      timeout: 5s
      retries: 3
    labels:
      - com.centurylinklabs.watchtower.enable=true
```

## .env file

```sh
# Filename: .env

POSTGRES_USER=postgres
POSTGRES_PASSWORD=b7bbd341bc51ef1c394a215e2482b05e

# zabbix/zabbix-server-pgsql will automatically create timescale hypertables on init
ENABLE_TIMESCALEDB=true
```

## References

1. Zabbix Agent setup [[zabbix-agent2]]
2. Autoregistering hosts with the server [[zabbix-active-agent-autoregistration]]
3. Monitoring docker with zabbix https://www.zabbix.com/integrations/docker