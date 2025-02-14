# Production-grade Standalone Validator Guide

This guide presents a straightforward and focused guide to set up a production grade validator node.
It won't discuss all possible ways to set up a standalone node. The guide is tested and based on Ubuntu 20.04.

This is work in progress and some parts may change with the upcoming node releases.


## Firewall (using UFW)

Enable the ports.
```
sudo ufw allow 1234/tcp
sudo ufw allow 30000/tcp
```
## Update System
Update package repository and update system:
```
sudo apt update -y
sudo apt-get dist-upgrade
```
# Radix Node
We install the Radix node based on the standalone instructions form the documentation
https://docs.radixdlt.com/main/node-and-gateway/systemd-install-node.html. 

## Dependencies
Install the necessary dependencies and initiate randomness to securely generate keys.
```
sudo apt install -y rng-tools openjdk-17-jdk unzip jq curl
sudo rngd -r /dev/random
```

## Create Radix User
Create a specific user for running the Radix node. The user is created with a locked password
and can only be switched to via `sudo su - radixdlt`.
```
sudo useradd radixdlt -m -s /bin/bash
```

## Service Control
Allow radix user to control the radix node service.
```
sudo sh -c 'cat > /etc/sudoers.d/radixdlt << EOF
radixdlt ALL= NOPASSWD: /bin/systemctl enable radixdlt-node.service
radixdlt ALL= NOPASSWD: /bin/systemctl restart radixdlt-node.service
radixdlt ALL= NOPASSWD: /bin/systemctl stop radixdlt-node.service
radixdlt ALL= NOPASSWD: /bin/systemctl start radixdlt-node.service
radixdlt ALL= NOPASSWD: /bin/systemctl reload radixdlt-node.service
radixdlt ALL= NOPASSWD: /bin/systemctl status radixdlt-node.service
radixdlt ALL= NOPASSWD: /bin/systemctl enable radixdlt-node
radixdlt ALL= NOPASSWD: /bin/systemctl restart radixdlt-node
radixdlt ALL= NOPASSWD: /bin/systemctl stop radixdlt-node
radixdlt ALL= NOPASSWD: /bin/systemctl start radixdlt-node
radixdlt ALL= NOPASSWD: /bin/systemctl reload radixdlt-node
radixdlt ALL= NOPASSWD: /bin/systemctl status radixdlt-node
radixdlt ALL= NOPASSWD: /bin/systemctl status radixdlt-node
radixdlt ALL= NOPASSWD: /bin/systemctl restart grafana-agent
radixdlt ALL= NOPASSWD: /bin/sed -i s/fullnode/validator/g /etc/grafana-agent.yaml
radixdlt ALL= NOPASSWD: /bin/sed -i s/validator/fullnode/g /etc/grafana-agent.yaml
EOF'
```


## Systemd Service
Create the radixdlt-node service:
```
sudo curl -Lo /etc/systemd/system/radixdlt-node.service \
    https://raw.githubusercontent.com/fpieper/fpstaking/main/docs/config/radixdlt-node.service
```

Also we enable the service at boot:
```
sudo systemctl enable radixdlt-node
```

## Create config and data directories
We create the necessary directories and set the ownership:
```
sudo mkdir /etc/radixdlt/
sudo chown radixdlt:radixdlt -R /etc/radixdlt
sudo mkdir /data
sudo chown radixdlt:radixdlt /data
sudo mkdir -p /opt/radixdlt/releases
sudo chown -R radixdlt:radixdlt /opt/radixdlt
```

Add `/opt/radixdlt` to `PATH`:
```
sudo sh -c 'cat > /etc/profile.d/radixdlt.sh << EOF
PATH=$PATH:/opt/radixdlt
EOF'
```

## Install Node
Switch to radixdlt user first
```
sudo su - radixdlt
```

I developed a seamless install and update script which downloads the last release
from `https://github.com/radixdlt/radixdlt/releases` and waits until one proposal was made to
restart the node to minimise the downtime.
If the interval between proposals is higher than around 5 seconds then there will be zero missed proposals:
```
curl -Lo /opt/radixdlt/update-node \
    https://raw.githubusercontent.com/fpieper/fpstaking/main/docs/scripts/update-node && \
chmod +x /opt/radixdlt/update-node
```

Installs or updates the radix node with the latest available version.
```
update-node
```

The argument `force` bypasses the check of the current installed version (mostly useful for testing). 
```
update-node force
```

Change directory for following steps.
```
cd /etc/radixdlt/node
```

## Secrets
Create secrets directories (one for validator and one for full node mode)
```
mkdir /etc/radixdlt/node/secrets-validator
mkdir /etc/radixdlt/node/secrets-fullnode
```

### Key Copy or Generation
The idea is to have two folders with configurations for a validator and a fullnode setting with different keys.
`/etc/radixdlt/node/secrets-validator` contains the configuration for a validator.
`/etc/radixdlt/node/secrets-fullnode` contains the configuration for a fullnode.
We will later to be able to switch between being a validator or fullnode.
This is useful for failover scenarios.

Either copy your already existing keyfiles `node-keystore.ks` to `/etc/radixdlt/node/secrets-validator` or `/etc/radixdlt/node/secrets-fullnode` or create a new keys.
Use a password generator of your choice to generate a secure password, don't use your regular one because
it will be written in plain text on disk and loaded as environment variable.
```
./bin/keygen --keystore=secrets-validator/node-keystore.ks --password=YOUR_VALIDATOR_PASSWORD
./bin/keygen --keystore=secrets-fullnode/node-keystore.ks --password=YOUR_FULLNODE_PASSWORD
```

Don't forget to set the ownership and permissions (and switch user again):
```
sudo chown -R radixdlt:radixdlt /etc/radixdlt/node/secrets-validator/
sudo chown -R radixdlt:radixdlt /etc/radixdlt/node/secrets-fullnode/
sudo su - radixdlt
cd /etc/radixdlt/node
```

To achieve high uptime, it is important to also have a backup node for maintenance or failover.
Your main and backup node will have the same validator key (node-keystore.ks), but they both have different fullnode keys
(which leads to 3 different keys in total: 1 key used as validator, 2 keys used for the full nodes).
Please also checkout this article for further details: https://docs.radixdlt.com/main/node-and-gateway/maintaining-uptime.html.

### Environment file
Set java options and the previously used keystore password.
```
cat > /etc/radixdlt/node/secrets-validator/environment << EOF
JAVA_OPTS="-server -Xms8g -Xmx8g -XX:+HeapDumpOnOutOfMemoryError -XX:+UseCompressedOops -Djavax.net.ssl.trustStore=/etc/ssl/certs/java/cacerts -Djavax.net.ssl.trustStoreType=jks -Djava.security.egd=file:/dev/urandom -DLog4jContextSelector=org.apache.logging.log4j.core.async.AsyncLoggerContextSelector --enable-preview"
RADIX_NODE_KEYSTORE_PASSWORD=YOUR_VALIDATOR_PASSWORD
EOF

cat > /etc/radixdlt/node/secrets-fullnode/environment << EOF
JAVA_OPTS="-server -Xms8g -Xmx8g -XX:+HeapDumpOnOutOfMemoryError -XX:+UseCompressedOops -Djavax.net.ssl.trustStore=/etc/ssl/certs/java/cacerts -Djavax.net.ssl.trustStoreType=jks -Djava.security.egd=file:/dev/urandom -DLog4jContextSelector=org.apache.logging.log4j.core.async.AsyncLoggerContextSelector --enable-preview"
RADIX_NODE_KEYSTORE_PASSWORD=YOUR_FULLNODE_PASSWORD
EOF
```

### Restrict Access To Secrets
```
chown -R radixdlt:radixdlt /etc/radixdlt/node/secrets-validator
chown -R radixdlt:radixdlt /etc/radixdlt/node/secrets-fullnode
chmod 500 /etc/radixdlt/node/secrets-validator && chmod 400 /etc/radixdlt/node/secrets-validator/*
chmod 500 /etc/radixdlt/node/secrets-fullnode && chmod 400  /etc/radixdlt/node/secrets-fullnode/*
```

## Node Configuration
Create and adapt the node configuration to your needs.
Especially, set the `network.host_ip` to your own IP (`curl ifconfig.me`) and
bind both apis to localhost `127.0.0.1`.

```
curl -Lo /etc/radixdlt/node/default.config \
    https://raw.githubusercontent.com/fpieper/fpstaking/main/docs/config/default.config
nano /etc/radixdlt/node/default.config
```

Setting `api.archive.enable=true` enables archive mode otherwise the node is running as full node.
Then you may also want to enable the construction endpoint with `api.construction.enable=true`.

You also may want set a `seed_node` from another region instead of the one from the `EU` above.

If you want to run on stokenet (testnet) instead of mainnet, you can set `network.id=2` and use this seed node: 
```
radix://tn1qt9kqzzqyj27zv4n67f2jrzgd24hsxfwe8d4kw9j4msze7rpdg3guvk07jy@54.76.86.46:30000
```

For further detail and explanation check out the official documentation
https://docs.radixdlt.com/main/node-and-gateway/systemd-install-node.html#_configuration


## Failover

Until now the service the Radix node does not find the secrets (environment and key).
Depending on whether we want to run the node in validator or full node mode we create a symbolic link 
to the corresponding directory. For example to run in validator mode:
```
/etc/radixdlt/node/secrets -> /etc/radixdlt/node/secrets-validator 
```

To streamline this process of promoting in case of a failover from our primary node, I wrote a small script:
```
curl -Lo /opt/radixdlt/switch-mode \
    https://raw.githubusercontent.com/fpieper/fpstaking/main/docs/scripts/switch-mode && \
chmod +x /opt/radixdlt/switch-mode
```

To switch the mode simply pass the mode as first argument. Possible modes are: `validator` and `fullnode`
```
switch-mode <mode>
```

For example:
```
switch-mode validator
switch-mode fullnode
```

It also supports `force` in case you need to switch, but your validator isn't fully working or making proposals:
```
switch-mode fullnode force
```

For bootstrapping a new validator it is a good idea to start as a `fullnode` and then after full sync
switch to `validator` mode because this also directly tests failover or promoting to validator works fine.

Switching to fullnode waits for the next made proposal still in validator mode
and stops immediately afterwards to minimise downtime (or specifically the missed proposals)  

For maintenance failover just open SSH connections to both of your servers side-by-side.
1. Switch to fullnode mode on your validator: `/opt/radixdlt/switch-mode.sh fullnode` 
2. Wait until switching mode was successful
3. Immediately switch to validator mode on your backup node: `/opt/radixdlt/switch-mode.sh validator`

## Node-Runner CLI

The node-runner cli was already installed by the `update-node` script.
We only just need to fit the following environment variables to our setup: 
```
echo '
export NGINX_SUPERADMIN_PASSWORD=""
export NGINX_ADMIN_PASSWORD=""
export NGINX_METRICS_PASSWORD=""
export NODE_END_POINT="http://localhost:3333"' >> ~/.bashrc
```

Now you need to logout and login back into the shell to enable the environment variables.

The radix node-runner cli can afterwards be called with for example:
```
radixnode api system health
```

For further details checkout the official documentation https://github.com/radixdlt/node-runner.
Though only use the `api` feature of the cli to interact with the node endpoints in 
an easier way and not the `setup`/`update`/`nginx`/`monitoring` commands, since these
conflict with the minimal setup approach in this guide.


## Registering as a validator
First of all we make sure that our node is running in `validator mode` to register the correct node key.
```
switch-mode validator
```

To register as validator please refer to the official documentation https://docs.radixdlt.com/main/node-and-gateway/systemd-register-as-validator.html.
Since our setup is a bit different and simpler (without Nginx, because it is not needed for a validator) we need to use a different curl command.

Instead of e.g.:
```
curl -k -u admin:{nginx-admin-password} -X POST 'https://localhost/entity' --header 'Content-Type: application/json' -d '{...}'
```

Use:
```
curl -s -X POST 'http://localhost:3333/entity' -H 'Content-Type: application/json' -d '{...}'
```
`--data-raw` / `-d` and `--header` / `-H` are synonyms. `-s` is optional and means the request progress is silent / hidden.


# Monitoring with Grafana Cloud
I can recommend watching this comprehensive introduction to Grafana Cloud
https://grafana.com/go/webinar/intro-to-prometheus-and-grafana/.
First, sign up for a Grafana Cloud free account and follow their quickstart introductions to install
Grafana Agent on your node (via the automatic setup script). This basic setup is out of the scope of this guide.
You can find the quickstart introductions to install the Grafana Agent under
`Onboarding (lightning icon) / Walkthrough / Linux Server` and click on `Next: Configure Service`.
The Grafana Agent is basically a stripped down Promotheus which is directly writing to Grafana Cloud instead of storing metrics locally
(Grafana Agent behaves like having a built-in Promotheus). 
You should now have a working monitoring of your system load pushed to Grafana Cloud.

## Extending Grafana Agent Config
Add the `scrape_configs` configuration to `etc/grafana-agent.yaml`: 
```
sudo nano /etc/grafana-agent.yaml
```
```
prometheus:
    configs:
    - name: integrations
      scrape_configs:
        - job_name: radix-mainnet-fullnode
          static_configs:
            - targets: ['localhost:3333']
          metrics_path: /prometheus/metrics
      remote_write:
        - basic_auth:
          password: secret
          username: 123456
          url: https://prometheus-blocks-prod-us-central1.grafana.net/api/prom/push
```

The prefixes like `radix-mainnet` before `fullnode` or `validator` are arbitrary and can be used
to have two dashboards (one for mainnet and one for stokenet) in the same Grafana Cloud account.

Just set the template variable `job` to `radix-mainnet-validator` in your mainnet dashboard
and `radix-stokenet-validator` in your stokenet dashboard.

The switch-mode script replaces `fullnode` with `validator` and vice versa.
Set `job_name` in the config above to e.g. `radix-mainnet-fullnode` if you are running in fullnode mode and
`radix-mainnet-validator` if you are running as validator.

And restart to activate the new settings:
```
sudo systemctl restart grafana-agent
```

## Radix Dashboard

I adapted the official `Radix Node Dashboard`
https://github.com/radixdlt/node-runner/blob/main/monitoring/grafana/provisioning/dashboards/sample-node-dashboard.json
and modified it a bit for usage in Grafana Cloud (including specific job names for `radix-validator` and `radix-fullnode` for failover).
You can get the `dashboard.json` from https://github.com/fpieper/fpstaking/blob/main/docs/config/dashboard.json.
You only need to replace `<your grafana cloud name>` with your own cloud name
(three times, since it seems the alerts have problems to process a datasource template variable).
It is a good idea to replace the values and variables in your JSON and then import the JSON as dashboard into Grafana Cloud.

## Alerts

### Spike.sh for phone calls
To get phone proper notifications via phone calls in case of Grafana Alerts I am using Spike.sh.
It only costs 7$/month and is working great.
How you can configure Spike.sh as `Notification Channel` is described here:
https://docs.spike.sh/integrations-guideline/integrate-spike-with-grafana.
Afterwards you can select `Spike.sh` in your alert configurations.

### Grafana Alerts
You can find the alerts by clicking on the panel title / Edit / Alert.

I set an alert on the proposals made panel, which fires an alert if no proposal was made in the last 2 minutes.
However, this needs a bit tuning for real world condition (worked fine in betanet conditions).

You also need to set `Notifications` to `Spike.sh` (if you configured the `Notification Channel` above).
Or any other notification channel if you prefer `PagerDuty` or `Telegram`.

# More Hardening
## SSH
- https://serverfault.com/questions/275669/ssh-sshd-how-do-i-set-max-login-attempts  
- Restrict access to the port:
    - use a VPN
    - only allow connections from a fix IP address
      ```
      sudo ufw allow from 1.2.3.4 to any port ssh
      ```

## Restrict Local Access (TTY1, etc)
We can additionally restrict local access.
However, this obviously leads results in that you won't be able to login without SSH in emergencies.
(booting into recovery mode works with most virtual servers, but causes downtime).
But since we have multiple backup servers this can be a fair trade-off.

Uncomment or add in this file
```
sudo nano /etc/pam.d/login
```
the following line:
```
account required pam_access.so
```

Then uncomment or add in this file
```
sudo nano /etc/security/access.conf
```
the following line:
```
-:ALL:ALL
```

For further details:
- https://linuxconfig.org/how-to-restrict-users-access-on-a-linux-machine

# Logs & Status

Shows radix node logs with colours:
```
sudo journalctl -f -u radixdlt-node --output=cat
```

Shows node health (`BOOTING`, `SYNCING`, `UP`, `STALLED`, `OUT_OF_SYNC`)
```
curl -s localhost:3333/system/health | jq
```

Show node keys information:
```
curl -s -X POST localhost:3333/key/list -H "Content-Type: application/json" -d '{"network_identifier": {"network": "mainnet"}}' | jq
```

Shows current validator information:
```
curl -s -X POST localhost:3333/entity -H "Content-Type: application/json"
    -d '{"network_identifier": {"network": "mainnet"}, "entity_identifier": {"address": "rv.....", "sub_entity": {"address": "system"}}}' | jq
```

Get network peers:
```
curl -s localhost:3333/system/peers | jq
```

Get node configuration:
```
curl -s localhost:3333/system/configuration | jq
```


# Upgrade from node 1.0.6 with the old api to node 1.1.0 with the new core api
In general, try these instructions on your backup node first to verify that there are no issues.

## Requirements
First we install Java 17 which the new version requires.
```
sudo apt install openjdk-17-jdk
```

Change user to `radixdlt`.
```
sudo su - radixdlt
```

## Configuration
Compare the new configuration https://raw.githubusercontent.com/fpieper/fpstaking/main/docs/config/default.config with your own and adapt your configuration.
```
nano /etc/radixdlt/node/default.config
```
You can also refer to the official documentation for more details: https://docs.radixdlt.com/main/node-and-gateway/systemd-install-node.html#_configuration

## Update the node
Now we can use the old update-node script to update our node to version 1.1.0 which also starts the node with the new version.
```
update-node
```

## Update-node and swich-mode script
Finally, we update the `update-node` and `switch-mode` script which will work together with the new node version (for future updates).
```
curl -Lo /opt/radixdlt/update-node \
    https://raw.githubusercontent.com/fpieper/fpstaking/main/docs/scripts/update-node && \
chmod +x /opt/radixdlt/update-node
```
```
curl -Lo /opt/radixdlt/switch-mode \
    https://raw.githubusercontent.com/fpieper/fpstaking/main/docs/scripts/switch-mode && \
chmod +x /opt/radixdlt/switch-mode
```

## Uninstall Java 11
Switch to your main user again
```
exit
```

Assuming you don't need Java 11 for anything else we can now remove it.
```
sudo apt --purge autoremove openjdk-11-jdk
```

## Grafana Cloud
You need to add `metrics_path: /prometheus/metrics` to your `grafana-agent.yaml` like
described in the section `Extending Grafana Agent Config` above.
```
sudo nano /etc/grafana-agent.yaml
```

Restart your grafana agent afterwards.
```
sudo systemctl restart grafana-agent
```

## Grafana Dashboard
The `info_counters_bft_proposals_made` metric was renamed to `info_counters_bft_pacemaker_proposals_sent`.
In the Radix dashboard on Grafana Cloud edit the panel "Proposals Made", rename the metric and save.
Now the data should show up again.
I will upload an updated dashboard.json after the `missed proposals` metric is supported again.


## Failover
Now switch over validator over from your primary to your backup node with (prepare two SSH sessions that you just need to press enter)

On your primary node:
```
sudo su - radixdlt
switch-mode fullnode
```

After the script has finished directly active validator mode on your (updated) backup node

On your backup node:
```
sudo su - radixdlt
switch-mode validator
```

Now repeat all the upgrade steps on your old primary node (which is still running the old version) and 
can be your new backup node or you switch modes again after you are done.
