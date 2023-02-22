## Webhook

1. To listen for http post requests to trigger a preconfigured event

 2. Used as a simple cd tool

### Install 
```bash
apt install webhook
```

### Config
```yaml
# Filename: /etc/webhook.conf
# Purpose: to create hook rules for the incoming webhook

- id: autodeploy
  execute-command: /srv/app/autodeploy.sh
  command-working-directory: /srv/app 
  
  include-command-output-in-response: true
  include-command-output-in-response-on-error: true
  
  trigger-rule:
    and:
    -match:
	    type: payload-hmac-sha1
	    secret: <secret>
	    parameter:
		    source: header
		    name: X-Hub-Signature
		    
    - match:
        type: value
        value: refs/heads/master
        parameter:
          source: payload
          name: ref
          
    -match:
       type: ip-whitelist
       ip-range: "192.168.0.1/24" 
       
``` 
#### autodeploy.sh
```bash
#!/bin/bash

# cmd triggered by webhook to run on /srv/app dir
set -e
git pull
docker-compose down
docker-compose up -d
```

### Autostart on reboot
```service
[Unit]
Description=Small server for creating HTTP endpoints (hooks)
Documentation=https://github.com/adnanh/webhook/
ConditionPathExists=/etc/webhook.conf

[Service]
ExecStart=/usr/bin/webhook -nopanic -hooks /etc/webhook.conf -verbose

[Install]
WantedBy=multi-user.target
```


### Gitea webhook 
```url
http://<ipaddress>:9000/hooks/autoredeploy 
```


### Reference:  https://github.com/adnanh/webhook

### Note:
 1. secrets not work apt webhook package
