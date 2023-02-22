	## Ctop #install
1. Ctop is a gui tool for docker cli similar to UI when we install ubuntu from the iso.
2. It gives a insight about wat docker containers are running and how much ram, cpu, io and rx, tx each container is using and logs of the container 
3. Containers can also be restarted, stopped and removed using ctop. However, we can not create containers using ctop
4. Below script will download ctop binary from official github repo and place it in the /usr/local/bin folder

### Script: #keep-up-to-date
#### <div dir="rtl"> amd64 </div>

```bash
wget https://github.com/bcicen/ctop/releases/download/v0.7.7/ctop-0.7.7-linux-amd64 -O /tmp/ctop
mv /tmp/ctop /usr/local/bin/
chmod +x /usr/local/bin/ctop
```
#### <div dir="rtl"> arm64 </div>

```bash
wget https://github.com/bcicen/ctop/releases/download/v0.7.7/ctop-0.7.7-linux-arm64 -O /tmp/ctop
mv /tmp/ctop /usr/local/bin/
chmod +x /usr/local/bin/ctop
```
### Reference: 
1. [Ctop github repo(github.com)](https://github.com/bcicen/ctop)

