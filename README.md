# poa-network-monitor

Tests for network health checks and monitoring.
<br>
<ul>
<li><code>network-test</code> folder contains tests and helper script. 
Use the command line arguments to detect network name and url. 
If no command line arguments received, parameters from the toml file will be used. <br>
<code>test-data</code> folder contains test blocks for checking missing-round test. <br>
<code>contracts</code> folder contains abi and address of needed contract
</li>
<li><code>common</code> folder contains file with configuration information obtained from toml file and dao 
for working with sqlite database.
</li>
<li><code>webapp</code> folder contains web server for retrieving test results
</li>
<li><code>test-result-monitor.js</code> file checks test results via web server and send alert to slack channel. 
Also uses the command line arguments to detect network name and url</li>
<li><code>config-sample.toml</code> is example of file with settings. Needs to be renamed to <code>config.toml</code> 
and filled with valid settings (as account and password)  </li>
</ul>
<h2>Setup</h2>

<h3>Run Parity nodes</h3>
1. Install Parity and obtain spec.json and bootnodes.txt files using these instructions: <a href="https://github.com/poanetwork/wiki/wiki/POA-Installation">POA Installation</a>.<br>
2. Clone Github repository:

```sh
git clone https://github.com/Natalya11444/poa-network-monitor.git
```
Install dependencies <br>

```sh
cd poa-network-monitor 
npm init
```
3.Run parity nodes <br>
For running parity node enable JSONRPC when connecting to POA Network on Parity <code>--jsonrpc-apis all</code><br>
For running two nodes for the each network it's needed to specify different ports for them. <br><br>
Example of running Sokol node on ubuntu:<br>

```sh
nohup parity --chain /path/to/sokol/spec.json --reserved-peers /path/to/sokol/bootnodes.txt --jsonrpc-apis all --port 30300 --jsonrpc-port 8540 --ws-port 8450 --ui-port 8180 --no-ipc > /path/to/logs/parity-sokol.log 2>&1 &
```

<br>url will be http://localhost:8540<br><br>
For the Core node:<br>

```sh
nohup parity --chain /path/to/core/spec.json --reserved-peers /path/to/core/bootnodes.txt --jsonrpc-apis all --port 30301 --jsonrpc-port 8541 --ws-port 8451 --ui-port 8181 --no-ipc > /path/to/logs/parity-core.log 2>&1 &
```

<br>url will be http://localhost:8541


<h3>Create scripts for running monitor and tests. </h3>
Network name and url can be added as parameters, otherwise it will be taken from the toml file. <br>
Example for the one test: <br>

```sh
#!/bin/sh <br>
cd /project/path/poa_monitor;  node  /project/path/poa_monitor/network-test/missing-rounds.js sokol http://localhost:8540 >> /path/to/logs/missing-rounds-sokol-log 2>&1;
node  /project/path/poa_monitor/network-test/missing-rounds.js core http://localhost:8541 >> /path/to/logs/missing-rounds-core-log 2>&1;
```
For preventing duplicate cron job executions PID files can be used, in this case script can be created this way

```sh
#!/bin/bash
PIDFILE=/path/to/pids/missing-rounds-sokol.pid
if [ -f $PIDFILE ]
then
  PID=$(cat $PIDFILE)
  ps -p $PID > /dev/null 2>&1
  if [ $? -eq 0 ]
  then
    echo "Process is running"
    exit 1
  else
    ## Process is not running
    echo $$ > $PIDFILE
    if [ $? -ne 0 ]
    then
      echo "Could not create PID file"
      exit 1
    fi
  fi
else
  echo $$ > $PIDFILE
  if [ $? -ne 0 ]
  then
    echo "Could not create PID file"
    exit 1
  fi
fi

node  /project/path/poa_monitor/network-test/missing-rounds.js sokol http://localhost:8540 >> /path/to/logs/missing-rounds-sokol-log 2>&1;

rm $PIDFILE
```

The same way scripts for other tests can be created <br><br>
When running monitor the time in seconds can be specified for checking last result. <br>

```sh
#!/bin/sh <br>
cd /project/path/poa_monitor;  node  /project/path/poa_monitor/test-result-monitor.js sokol http://localhost:8540 2400 >>/path/to/logs/monitor-sokol-log 2>&1;
node  /project/path/poa_monitor/test-result-monitor.js core http://localhost:8541 2400 >>/path/to/logs/monitor-core-log 2>&1
```

<h3>Add scripts to the crontab </h3>
Run <code>sudo crontab -e -u user</code> <br>
Crontab example with timeout: 

```sh
*/10 * * * *  timeout -s 2 8m /path/to/scripts/missing-rounds.sh
*/10 * * * *  timeout -s 2 8m /path/to/scripts/mining-reward.sh
0,30  * * * *   timeout -s 2 25m /path/to/scripts/mining-block.sh
*/15 * * * *  timeout -s 2 12m /path/to/scripts/txs-public-rpc.sh
0,30 * * * *   timeout -s 2 15m /path/to/scripts/monitor.sh
```
<h3>Run web server </h3>
<code>nohup node /project/path/poa_monitor/webapp/index.js >>/path/to/logs/web_server.log 2>&1 & </code>