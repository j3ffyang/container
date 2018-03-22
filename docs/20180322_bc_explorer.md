# Blockchain Explorer

Reference: [https://github.com/hyperledger/blockchain-explorer](https://github.com/hyperledger/blockchain-explorer)

## Steps to Build Explorer

- Pull Docker Image

  ```shell
  docker pull mysql:tag
  ```

- Start Mysql container

  ```shell
  docker run --name some-mysql -p 3306:3306 -v /path/to/dir:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=my-secret-pw -d mysql:tag
  ```

- Clone Repository

  ```shell
  git clone https://github.com/hyperledger/blockchain-explorer.git
  cp fabricexplorer.sql /path/to/mysql-volume
  ```

- Initialize Database

  ```shell
  docker exec -it some-mysql bash
  cd /var/lib/mysql
  mysql -u root -p < ./fabricexplorer.sql
  ```

- Startup Fabric Network

- Modify configuration

  Here is my own ```config.json```, please ensure all the key files are suitable.

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
  	"keyValueStore": "/tmp/fabric-client-kvs",
  	"eventWaitTime": "30000",
  	"mysql": {
  		"host": "127.0.0.1",
  		"port": "3306",
  		"database": "fabricexplorer",
  		"username": "root",
  		"passwd": "secret"
  	}
  }

  ```

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
