# Blockchain Explorer

Reference: [https://github.com/hyperledger/blockchain-explorer](https://github.com/hyperledger/blockchain-explorer)

## Steps to Build Explorer

- Clone Repository

  ```shell
  git clone https://github.com/hyperledger/blockchain-explorer.git
  cp ~/blockchain-explorer/db/fabricexplorer.sql /path/to/mysql_vol
  ```

> You need to create ```/path/to/mysql_vol``` manually first

- Start MySQL container

  ```shell
  docker run --name mysql-bcexplorer -p 3306:3306 -v /path/to/mysql_vol:/var/lib/mysql \
    -e MYSQL_ROOT_PASSWORD=mysecret -d mysql
  ```

- Initialize Database

  ```shell
  docker exec -it mysql-bcexplorer bash
  cd /var/lib/mysql
  mysql -u root -p < ./fabricexplorer.sql
  ```

- Startup Fabric network. Then modify configuration of ```config.json``` accordingly

  Here is a sample of ```config.json```

```json
{
  "network-config": {
    "org1": {
      "name": "peerOrg1",
      "mspid": "Org1MSP",
      "peer1": {
        "requests": "grpcs://host2:7051",
        "events": "grpcs://host2:7053",
        "server-hostname": "peer0.org1.test.io",
        "tls_cacerts": "/path/to/peerOrganizations/org1.test.io/peers/peer0.org1.test.io/tls/ca.crt"
      },
      "admin": {
        "key": "/path/to/peerOrganizations/org1.test.io/users/Admin@org1.test.io/msp/keystore",
        "cert": "/path/to/peerOrganizations/org1.test.io/users/Admin@org1.test.io/msp/signcerts"
      }
    }
  },
  "host": "localhost",
  "port": "8080",
  "channel": "mychannel",
  "keyValueStore": "/path/to/fabric-client-kvs",
  "eventWaitTime": "30000",
    "mysql": {
    "host": "127.0.0.1",
    "port": "3306",
    "database": "fabricexplorer",
    "username": "root",
    "passwd": "mysecret"
    }
}
```

> If ```tls``` is disabled, (see Tips section for the details of how to disable TLS for demo)
> - use ```grpc://host2:7051``` instead of ```grpcs://host2:7051```
> - execute to create ```touch ~/blockchain-explorer/fabric-path/first-network/crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt``` to avoid ```ca.crt``` not found error

- Start Service

  ```shell
  cd blockchain_explorer
  npm install
  ./start.sh
  ```
  Or

  ```
  cd blockchain_explorer
  npm install
  node main.js
  ```

#### Tips

- List container_id of images from ```hyperledger``` and ```dev-peer```

  ```
  docker ps -a | grep 'hyperledger\ | dev-peer' | cut -f1 -d" "
  ```

    Or

  ```
  docker ps -a | awk '($2 ~ /hyperledger/) || ($2 ~ /dev-peer/) {print $1}'
  ```

- Disable __peer_tls__, edit ```~/fabric-samples/basic-network/docker-compose.yml```

```
orderer.example.com:
  container_name: orderer.example.com
  image: hyperledger/fabric-orderer
  environment:
    - ORDERER_GENERAL_LOGLEVEL=debug
...
    - ORDERER_GENERAL_TLS_ENABLED=false
  working_dir: /opt/gopath/src/github.com/hyperledger/fabric/orderer
  command: orderer
  ports:
    - 7050:7050
  volumes:
      - ./config/:/etc/hyperledger/configtx
      - ./crypto-config/ordererOrganizations/example.com/orderers/orderer.example.com/:/etc/hyperledger/msp/orderer
      - ./crypto-config/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/:/etc/hyperledger/msp/peerOrg1
  networks:
    - basic

peer0.org1.example.com:
  container_name: peer0.org1.example.com
  image: hyperledger/fabric-peer
  environment:
...
    - CORE_PEER_TLS_ENABLED=false
```
