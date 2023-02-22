# Oxidized - Docker container

* Oxidized is used to backup network devices configurations and maintain in a version control system like git
* Run with custom models with custom commands to run on network devices  with BIT specific environment


# Note:

* It creates a bare repository during the first run. To convert it into a unbare repo remove the .git directory in the data folder and git clone from the remote repository
* It dont provides authentication out of the box for web interface. Authentication is handles via traefik reverse proxy

# Oxidize-setup:

* Docker-compose file to start oxidize docker container with traefik as a reverse proxy 

```yaml
# Filename: /srv/netwatcher.bit.lan/docker-compose.yaml
# Purpose: Used to backup configuraions from network devices and store it in version control system

version: "2.4"
services:
  oxidized:
    image: oxidized/oxidized:latest
    container_name: netwatcher.bit.lan
    restart: always

    mem_limit: 512M

    environment:
      CONFIG_RELOAD_INTERVAL: 600

    volumes:
      - ./ssh:/root/.ssh
      - ./config/config:/root/.config/oxidized/config:ro
      - ./config/router.db:/root/.config/oxidized/router.db:ro
      - ./model:/root/.config/oxidized/model:ro
      - ./webhooks:/root/.config/webhooks:ro
      - ./data:/root/.config/data

    labels:
      - com.centurylinklabs.watchtower.enable=true
      - traefik.enable=true
      - traefik.http.routers.oxidize.rule=Host(`netwatcher.bit.lan`) && ClientIP(`127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `100.64.0.0/10`)
      - traefik.http.routers.oxidize.tls=true
      - traefik.http.routers.oxidize.middlewares=oxidize-auth
      - traefik.http.middlewares.oxidize-auth.basicauth.users=oxidize:$$2y$$05$$F41tqCPhw1cX6iZ69aS1MeDqNXaViAEn7hfz.wfDm4Xjko3gpuyA2
      - traefik.http.routers.oxidize.service=oxidize-svc
      - traefik.http.services.oxidize-svc.loadbalancer.server.port=8888
```

## Configuration

```yaml
# Filename: /root/.config/oxidized/config/config
# Purpose: Config for oxidized service

---
interval: 86400
log: /dev/stdout
debug: true
threads: 4
use_max_threads: false
timeout: 30
retries: 3
pid: "/root/.oxidized_pid"

stats:
  history_size: 10

groups: {}

crash:
  directory: /root/.config/crashes
  hostnames: false

rest: 0.0.0.0:8888

input:
  default: ssh
  debug: false
  ssh:
    secure: false

source:
  default: csv
  csv:
    file: /root/.config/oxidized/router.db
    delimiter: !ruby/regexp /:/
    map:
      ip: 0
      name: 1
      group: 2
      model: 3

output:
  default: git
  git:
      single_repo: true
      user: Oxidized
      email: shankarmounesh.cs20@bitsathy.ac.in
      repo: "/root/.config/data/oxidized-backups/.git"
      # After first run change to
      # repo: "/root/.config/data/oxidized-backups"

hooks:
  push_to_remote:
    type: githubrepo
    events: [post_store]
    remote_repo: ssh://git@git.bitsathy.ac.in:2222/DevOps/oxidized-backups.git
    privatekey: /root/.ssh/id_ed25519

 # TODO: remove the post-commit hook in the git-repos/config-backups.git/hooks/poost-commit after checking if push_to remote hook works for ssh protocol
  post_commit_hook:
    type: exec
    events: [post_store]
    cmd: 'git -C /root/.config/data/oxidized-backups/.git push origin master'
    # After first run change to
    # cmd: 'git -C /root/.config/data/oxidized-backups push origin master'

  notify_when_backup_fails:
    type: exec
    events: [node_fail]
    async: true
    cmd: 'bash /root/.config/webhooks/backup_failure.sh'

model_map:
  juniper: junosbit
  cisco: ciscobit

models:
  junosbit:
    username: nil
    password: nil

  ciscobit:
    username: nil
    password: nil
```
## Devices-list:
```txt
# Filename: /root/.config/oxidized/config/router.db
# Purpose: To configure network devices-list with ip. hostname and map with model similar to rancid's router.db

# ip:name:group:model

10.0.0.0:RACK1-NO5:OFFICE:junosbit
```

## Models

* Custom models used  on BIT devices
```rb
# Filename: root/.config/oxidized/model/ciscobit.rb
# Purpose: Custom model for cisco sg 300 series

class CiscoBIT < Oxidized::Model

  prompt /([#>]\s?)$/
  comment  '! '

  cmd :all do |cfg|
    lines = cfg.each_line.to_a[1..-2]
    # Remove \r from beginning of response
    lines[0].gsub!(/^\r.*?/, '') unless lines.empty?
    lines.join
  end

  cmd :secret do |cfg|
    cfg.gsub! /^(snmp-server community).*/, '\\1 <configuration removed>'
    cfg.gsub! /username (\S+) privilege (\d+) (\S+).*/, '<secret hidden>'
    cfg.gsub! /^(username \S+ password encrypted) \S+(.*)/, '\\1 <secret hidden> \\2'
    cfg.gsub! /^(enable password level \d+ encrypted) \S+/, '\\1 <secret hidden>'
    cfg.gsub! /^(encrypted radius-server key).*/, '\\1 <configuration removed>'
    cfg.gsub! /^(encrypted radius-server host .+ key) \S+(.*)/, '\\1 <secret hidden> \\2'
    cfg.gsub! /^(encrypted tacacs-server key).*/, '\\1 <secret hidden>'
    cfg.gsub! /^(encrypted tacacs-server host .+ key) \S+(.*)/, '\\1 <secret hidden> \\2'
    cfg.gsub! /^(encrypted sntp authentication-key \d+ md5) .*/, '\\1 <secret hidden>'
    cfg
  end

  cmd 'show bootvar' do |cfg|
    cfg.insert(0,"\n ####################################### show bootvar ############################################## \n")
    comment cfg
    cfg << "\n #################################################################################################### \n"

  end

  cmd 'show running-config' do |cfg|
    cfg.insert(0,"\n ####################################### show running-config ######################################## \n")
    cfg = cfg.each_line.to_a[0..-1].join
    cfg.gsub! /^Current configuration : [^\n]*\n/, ''
    cfg.sub! /^(ntp clock-period).*/, '! \1'
    cfg.gsub! /^ tunnel mpls traffic-eng bandwidth[^\n]*\n*(
                  (?: [^\n]*\n*)*
                  tunnel mpls traffic-eng auto-bw)/mx, '\1'
    cfg
    cfg << "\n #################################################################################################### \n"
  end

  cfg :telnet, :ssh do
    post_login 'terminal datadump' # Disable pager
    post_login 'terminal width 0'
    pre_logout 'exit'
  end
end
```
```rb
# Filename: root/.config/oxidized/model/junosbit.rb
# Purpose: Custom model for juniper Ex3400, Ex3300 series

class JunOSBIT < Oxidized::Model
  comment '# '

  def telnet
    @input.class.to_s.match(/Telnet/)
  end

  cmd :all do |cfg|
    cfg = cfg.cut_both if screenscrape
    cfg.gsub!(/  scale-subscriber (\s+)(\d+)/, '  scale-subscriber                <count>')
    cfg.lines.map { |line| line.rstrip }.join("\n") + "\n"
  end

  cmd :secret do |cfg|
    cfg.gsub!(/community (\S+) {/, 'community <hidden> {')
    cfg.gsub!(/ "\$\d\$\S+; ## SECRET-DATA/, ' <secret removed>;')
    cfg
  end

  cmd 'show version' do |cfg|
    cfg.insert(0,"\n ####################################### show version ############################################## \n")
    @model = Regexp.last_match(1) if cfg =~ /^Model: (\S+)/
    comment cfg
    cfg << "\n #################################################################################################### \n"
  end

  cmd 'show virtual-chassis' do |cfg|
    cfg.insert(0,"\n ####################################### show virtual-chassis ############################################## \n")
    comment cfg
    cfg << "\n #################################################################################################### \n"
  end

  cmd 'show chassis hardware' do |cfg|
    cfg.insert(0,"\n ####################################### show chassis hardware ############################################## \n")
    comment cfg
    cfg << "\n #################################################################################################### \n"
  end

  cmd 'show system license' do |cfg|
    cfg.insert(0,"\n ####################################### show system license ############################################## \n")
    cfg.gsub!(/  fib-scale\s+(\d+)/, '  fib-scale                       <count>')
    cfg.gsub!(/  rib-scale\s+(\d+)/, '  rib-scale                       <count>')
    comment cfg
     cfg << "\n #################################################################################################### \n"
  end

  cmd 'show system license keys' do |cfg|
    cfg.insert(0,"\n ####################################### show system license keys ############################################## \n")
    comment cfg
    cfg << "\n #################################################################################################### \n"
  end

  cmd 'show configuration | display omit' do |cfg|
    cfg.insert(0,"\n ####################################### show configuration ############################################## \n")
    comment cfg
    cfg << "\n #################################################################################################### \n"
  end

  cfg :telnet do
    username(/^login:/)
    password(/^Password:/)
  end

  cfg :ssh do
    exec true # don't run shell, run each command in exec channel
  end

  cfg :telnet, :ssh do
    post_login 'set cli screen-length 0'
    post_login 'set cli screen-width 0'
    pre_logout 'exit'
  end
end
```
## Webhooks:
```sh
# Filename: /root/.config/webhooks/backup_failure.sh
# Purpose: To send notifications in google spaces when a configuration backup is failed

#!/bin/bash

#Google-Space URL
BASE_URL="<googlespace-endpoint url>"

curl -X POST ${BASE_URL} \
        -H 'Content-Type: application/json' \
        -d "{ \"text\": \"DEVICE_NAME: ${OX_NODE_NAME}\nDEVICE_IP: ${OX_NODE_IP}\nJOB_STATUS: ${OX_JOB_STATUS}\nBLOCK: ${OX_NODE_GROUP}\nDEVICE_MODEL: ${OX_NODE_MODEL}\"}"
```
# Device setup

## Juniper configuartion (EX 3300 series)

* Create a login class with access to commands used by oxidized and deny all other commands
```sh
set system login class cfg-view permissions view-configuration
set system login class cfg-view allow-commands "(show)|(set cli screen-length)|(set cli screen-width)"
set system login class cfg-view deny-commands "(clear)|(file)|(file show)|(help)|(load)|(monitor)|(op)|(request)|(save)|(set)|(start)|(test)"
set system login class cfg-view deny-configuration all
```

* Create a user and map it to the login class
```sh
set system login user oxidized class cfg-view
set system login user oxidized authentication encrypted-password "passowrd"
```

## Cisco  configuration (SG 300 series )
* Create a user with user privileadge 15
* Disable the username prompt during ssh via "ip ssh password-auth" command

# Reference:  https://github.com/ytti/oxidized

# TODO:
### Remove post-commit hook from config if oxidized supports ssh protocol for pushing to remote git
