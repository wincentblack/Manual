## Forta setup guide
Official guide: 
https://docs.forta.network/en/latest/api-example-use-cases/ 

https://forta.notion.site/Node-Operator-Onboarding-Process-6dfdfb0c717a429ebeb37e23e5532e3f 
### Form for request to be a node validatro (it`s closed on this moment):
```https://docs.google.com/forms/d/e/1FAIpQLSfO1D7NLUcv-FsYKMcUlVwTUXAdmFraXqnXsmUA_qkSR2d79A/viewform?pli=1```

### Installation tutorial: I decided start it ont lilght client for Ethereum network (not full node)


1. Install the light client for Ethereum network, I used this one:
https://geth.ethereum.org/docs/install-and-build/installing-geth#run-inside-docker-container
```
# docker pull ethereum/client-go
# mkdir /root/.ethereum
# screen -S Etherium_noda
# docker run -it -p 30303:30303 -p 8545:8545 -p 8546:8546 -p 30304:30304 -v /root/.ethereum/:/root/.ethereum/ ethereum/client-go --syncmode light --http --http.addr 0.0.0.0 --http.port 8545 --http.api personal,eth,net,web3 --ws.api eth,net,web3 --http.corsdomain --http.vhosts 
```
https://www.freecodecamp.org/news/how-to-run-geth-from-a-docker-container-b6d30620ca74/ 
https://ethereum.org/ru/developers/tutorials/run-light-node-geth/

##Note!!! The metod sync in the light mode is not supported the trace and as result cannot be use here, as alternative we can try use it in the SNAP mode but, I was not able to check it, unfortunately,
For this current test net I used Alchemi paid service to get access to ETH RPC node (see config below)

### 2. Deploy Forta node
#### 2.1 Get repositories and instal package
```
# curl https://dist.forta.network/pgp.public -o /usr/share/keyrings/forta-keyring.asc -s
# echo 'deb [signed-by=/usr/share/keyrings/forta-keyring.asc] https://dist.forta.network/repositories/apt stable main' | sudo tee -a /etc/apt/sources.list.d/forta.list
# apt-get update
# apt-get install forta
```
#### 2.2 Initializate stage
```
# forta init --passphrase "your password"
Scanner address: 0x4751ca9aa2EaA551d26d1993A14BbE05a5404049

Successfully initialized at /root/.forta

- Please make sure that all of the values in config.yml are set correctly.
- Please fund your scanner address with some MATIC.
- Please enable it for the chain ID in your config by doing 'forta register --owner-address <your_owner_wallet_address>'.
root@vmi621203:~#
```

#### 2.3 We should edit this file and point ot our Etherium service:
For Alchemy
```
chainId: 1

# The scan settings are used to retrieve the transactions that are analyzed
scan:
  jsonRpc:
    url: https://eth-mainnet.alchemyapi.io/v2/z5Jwxxxxxxxxxxxxxxxxxxxx7Zp

# The trace endpoint must support trace_block (such as alchemy)
trace:
  jsonRpc:
    url: https://eth-mainnet.alchemyapi.io/v2/z5Jwxxxxxxxxxxxxxxxxxxxx7Zp

```
Also, to avoid high trafics from bots we need use a RPC Proxy, I use this one:
https://moralis.io/ - we need register account there and then navigate to the Speedy Nodes button, press on our network (this is Ethereum in my case) copy mainet link, somthing like this:
![image](https://user-images.githubusercontent.com/7540778/165710588-5f90f2c0-171f-49f0-bab1-f3bdce09e470.png)
open our forta config and put it there:
```
# The jsonRpcProxy settings are used make query requests (defaults to scan url)
jsonRpcProxy:
  jsonRpc:
    url: https://speedy-nodes-nyc.moralis.io/xxxxxxxxxxxxxx/eth/mainnet

```
save changes and restart service:
```
# systemctl stop forta && sleep 20 && systemctl start forta
```
Be sure that it goes well, check ```forta status``` and logs

For local node (geth) - I did not use it here
```
root@vmi621203:~# cat /root/.forta/config.yml |head -n20
# Auto generated by 'forta init' - safe to modify
# The chainId is the chainId of the network that is analyzed (1=mainnet)
chainId: 1

# The scan settings are used to retrieve the transactions that are analyzed
scan:
  jsonRpc:
    url: http://127.0.0.1:8545

# The trace endpoint must support trace_block (such as alchemy)
trace:
  jsonRpc:
    url: http://127.0.0.1:8545

# The registry settings are used to discover and load agents
```
#### 2.4 Registering Forta node
We shoud fund our address in the Polygon network by 1 MATIC and check the balance:
https://polygonscan.com/address/0x4751ca9aa2EaA551d26d1993A14BbE05a5404049  
#### !!! Note thet we should fund our Node scan address, but place in the command bellow our MM address (owner)
```
# forta register --owner-address 0x1418559Ea8b2d5A29Ce34A8c512c764F80A68D4A --passphrase "your password"
Sending a transaction to register your scan node to chain 1...
Successfully sent the transaction!

Please ensure that https://polygonscan.com/tx/0xbca62d6b2e4c2d9068b6ec1bc251c8b23d6cd31fe42b33df829beea62d5f6adf succeeds before you do 'forta run'. This can take a while depending on the network load.
root@vmi621203:~#
```
#### 2.5 Node start
Before start I've edited and created these environment for my service:
To override systemd service environment, you can set the variables in /etc/systemd/system/forta.service.d/env.conf like:
```
[Service]
Environment="FORTA_DIR=<your_forta_config_dir>"
Environment="FORTA_PASSPHRASE=<your_forta_passphrase>"

Run the systemd service to start Forta
# systemctl daemon-reload
# systemctl enable forta
# systemctl start forta
```
#### 2.6 Check status:
```
 # forta status
```
![image](https://user-images.githubusercontent.com/7540778/165150636-cd6fd826-addd-4f19-90c3-d2eb2ab2e09b.png)


### 3. Checking logs
```
# journalctl -u forta -n20 --output cat
```
#### We should see somthing like this:
![image](https://user-images.githubusercontent.com/7540778/165150850-d19471e6-ac3c-48f3-8562-77f1dcb4655b.png)

check that all docker containres were started:
```
# docker ps
```
![image](https://user-images.githubusercontent.com/7540778/164965835-60aaafb8-76b9-424c-b650-84c385ca4fe0.png)

 How to check count of connected agents:
```# docker ps |grep docker-entrypoint |wc -l```

 How to chekck node`s avg:
```
# curl https://api.forta.network/stats/sla/scanner/your node address?startTime=2022-04-24T00:00:00Z |jq |grep avg
```
![image](https://user-images.githubusercontent.com/7540778/166000947-875ab9eb-6cc1-42a1-bf6b-752f9726e4bd.png)

 Gradation:  
100% rewards, if SLA more then >0.9  
80% rewards, if SLA more then >0.75-0,89  
No rewards, if SLA less then 0.75  

#### check your node in the explorer lists here:
https://explorer.forta.network/network 


