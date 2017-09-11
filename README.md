# ACES Developer Guide

## Deploying your Node

These steps help you get set up a Ubuntu 16.04 linux server running ACES. You'll need at least 50GB of disk
space to sync the ethereum blockchain (might not be necessary for light syncing).

These instructions use systemd to supervise applications running in the background and nginx to serve
web traffic.

1. Install system dependencies

```
sudo apt-get install software-properties-common python-software-properties build-essential curl git maven
sudo add-apt-repository ppa:git-core/ppa
sudo add-apt-repository -y ppa:ethereum/ethereum
sudo add-apt-repository ppa:webupd8team/java -y
sudo apt-get update

sudo apt-get install ethereum solc

# install Java JVM
echo oracle-java8-installer shared/accepted-oracle-license-v1-1 select true | sudo /usr/bin/debconf-set-selections
sudo apt-get install oracle-java8-installer

# install nodejs
curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -
sudo apt-get install nodejs

# install nginx
sudo apt-get install nginx
```


2. Install ethereum service

Copy the following into `/etc/systemd/system/geth-network.service`. There are 3 different ways to run geth to
target different ethereum networks: dev, testnet, and mainnet.



```
vim /etc/systemd/system/geth-network.service
```
When in vim, press 'a' to enter insert mode. Then press ctrl+shift+V to paste the desired network chunk below. If you make a mistake, press ESC and type :1,$d to clear the page. Save the changes with ESC and type :x

Local Devnet:

```
[Unit]
Description=Ethereum devnet

[Service]
Restart=always
ExecStart=/usr/bin/geth --dev --rpc --rpcaddr=127.0.0.1 --rpcapi 'web3,eth,personal' --rpccorsdomain="*" \
--datadir=/eth-data
[Install]
WantedBy=multi-user.target
```

Ropsten Testnet:

```
[Unit]
Description=Ethereum testnet

[Service]
Restart=always
ExecStart=/usr/bin/geth --testnet --rpc --rpcaddr=127.0.0.1 --rpcapi 'web3,eth,personal' --rpccorsdomain="*" \
--datadir=/eth-data --fast --cache=1024 \
--bootnodes "enode://20c9ad97c081d63397d7b685a412227a40e23c8bdc6688c6f37e97cfbc22d2b4d1db1510d8f61e6a8866ad7f0e17c02b14182d37ea7c3c8b9c2683aeb6b733a1@52.169.14.227:30303,enode://6ce05930c72abc632c58e2e4324f7c7ea478cec0ed4fa2528982cf34483094e9cbc9216e7aa349691242576d552a2a56aaeae426c5303ded677ce455ba1acd9d@13.84.180.240:30303"

[Install]
WantedBy=multi-user.target
```

Mainnet:

```
[Unit]
Description=Ethereum network

[Service]
Restart=always
ExecStart=/usr/bin/geth --rpc --rpcaddr=127.0.0.1 --rpcapi 'web3,eth,personal' --rpccorsdomain="*" \
--datadir=/eth-data --fast --cache=1024 \
--bootnodes "enode://20c9ad97c081d63397d7b685a412227a40e23c8bdc6688c6f37e97cfbc22d2b4d1db1510d8f61e6a8866ad7f0e17c02b14182d37ea7c3c8b9c2683aeb6b733a1@52.169.14.227:30303,enode://6ce05930c72abc632c58e2e4324f7c7ea478cec0ed4fa2528982cf34483094e9cbc9216e7aa349691242576d552a2a56aaeae426c5303ded677ce455ba1acd9d@13.84.180.240:30303"

[Install]
WantedBy=multi-user.target
```


Start up ethereum service (this can take several hours to sync before transactions):

```
sudo chmod 655 /etc/systemd/system/geth-network.service
sudo systemctl daemon-reload
sudo service geth-network start
```

You can view the output of running systemd services by tailing the syslog:

```
sudo tail -f /var/log/syslog
```

It is helpful to check the status of your sync by opening a new terminal window, logging in to your server if necessary, and running the following:

```
sudo geth attach ipc:/eth-data/geth.ipc
eth.syncing
```

3. Create Service Eth Wallet

Connect to the local geth instance and create a new eth wallet for ACES service:

```
geth attach ipc:/eth-data/geth.ipc

> personal.newAccount('enter your passphrase here');
> eth.accounts
["0x77fa2ae66ff74d99da00cdd3f82a3a50750bb95d"]
```

Use the eth wallet address and password in the application.yml configuration below.

4. Install ACES application

Install the application under `/apps`:

```
sudo mkdir /apps
git clone https://github.com/ark-aces/aces-backend.git
```

Install application.yml configuration file under `/etc/aces-backend`:

```
mkdir /etc/aces-backend
```

Copy `src/main/resources/application.yml` file over into `/etc/aces-backend` and replace
wallet configurations with your own Eth and Ark wallet addresses and passphrase. Ensure that the etheruem address is quoted.

```
vim /etc/aces-backend/application.yml
```

```
spring:
  datasource:
    url: "jdbc:h2:/tmp/testdb;DB_CLOSE_ON_EXIT=FALSE;AUTO_RECONNECT=TRUE"
    driver-class-name: "org.h2.Driver"
  jpa:
    hibernate:
      ddl-auto: "update"

arkNetwork:
  name: "mainnet"

ethBridge:
  ethServerUrl: "http://localhost:8545"
  scriptPath: "./bin"
  nodeCommand: "/usr/bin/node"

  serviceArkWallet:
    address: "change me"
    passphrase: "change me"
    passphrase2:

  serviceEthAccount:
    address: "change me"
    password: "change me"

arkPerEthAdjustment: "100.00"

ethContractDeployService:
  arkFlatFee: "2.00"
  arkPercentFee: "2.25"
  requiredArkMultiplier: "1.2"

ethTransferService:
  arkFlatFee: "1.00"
  arkPercentFee: "1.25"
  requiredArkMultiplier: "1"
```

OPTIONAL: For production instances, you should configure the application to use a real database like `posgresql`:

```
spring:
  datasource:
    url: jdbc:postgresql://{host}:{port}/{database_name}
    username: {username}
    password: {password}
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQLDialect
    generate-ddl: true
    hibernate:
      ddl-auto: update
```
END OPTIONAL

Copy the following into `/etc/systemd/system/aces-backend.service`:

```
vim /etc/systemd/system/aces-backend.service
```

```
[Unit]
Description=Aces Backend

[Service]
Restart=always
WorkingDirectory=/apps/aces-backend
ExecStart=/usr/bin/mvn spring-boot:run -Dspring.config.location=file:/etc/aces-backend/application.yml

[Install]
WantedBy=multi-user.target
```
Install npm dependencies:

```
cd /apps/aces-backend/bin
npm install
```

Start the ACES backend service:

```
sudo chmod 655 /etc/systemd/system/aces-backend.service
sudo systemctl daemon-reload
sudo service aces-backend start
```

5. Installing ACES frontend

Install the application under `/apps`:

```
cd /apps
git clone https://github.com/ark-aces/aces-frontend.git
```

Copy prod configuration template into custom configuration file:
```
cd /apps/aces-frontend
cp src/environments/environment.prod.ts src/environments/environment.custom.ts
```
If using an Ethereum Testnet, copy the following into `/apps/aces-frontend/src/environments/environment.custom.ts`:
```
vim /apps/aces-frontend/src/environments/environment.custom.ts
```
```
export const environment = {
  production: true,
  title: 'Custom Ark Contract Execution Services (ACES)',
  isEthTestnet: true,
  ethNetworkName: 'ropsten',
  etherscanBaseUrl: 'https://ropsten.etherscan.io',
  ethArkRateFraction: '1/100',
  arkExplorerBaseUrl: 'https://explorer.ark.io',
  acesApiBaseUrl: 'http://localhost/aces-api'
};
```
If using the Ethereum Mainnet, copy the following into /apps/aces-frontend/aces-app/src/environments/environment.custom.ts:
```
vim /apps/aces-frontend/src/environments/environment.custom.ts
```
```
export const environment = {
  production: true,
  title: 'Custom Ark Contract Execution Services (ACES)',
  isEthTestnet: false,
  etherscanBaseUrl: 'https://etherscan.io',
  arkExplorerBaseUrl: 'https://explorer.ark.io',
  acesApiBaseUrl: 'http://localhost/aces-api'
};
```

Install angular for the front-end UI

```
cd /apps/aces-frontend
npm install -g @angular/cli
npm install
```
Update the baseURL in the front-end config, replacing the aces-ark.io url with your server ip.
```
vim /apps/aces-frontend/src/app/aces-server-config.ts
```

It should appear something like this:
```

export abstract class AcesServerConfig {
  abstract getBaseUrl(): string;
}

export class ProdAcesServerConfig extends AcesServerConfig {
  getBaseUrl() {
    return 'http://23.92.18.143/aces-api';
  }
}

export class LocalAcesServerConfig extends AcesServerConfig {
  getBaseUrl() {
    return 'http://localhost:8080';
  }
}
```

Build the application
```
ng build --target=production --base-href /aces-app/
```

6. Set up nginx web server

Allow backend and frontend to be served by nginx by adding the following to `/etc/nginx/sites-available/default`
under root `server` directive.

First vim into the file

```
vim /etc/nginx/sites-available/default
```

Then add the snippet below under the `server` directive.

```
location /aces-app/ {
    alias /apps/aces-frontend/dist/;
    try_files $uri $uri/ /aces-app/;
}

location /aces-api/ {
    proxy_pass http://127.0.0.1:8080/;
}
```
Now restart the nginx service

```
sudo service nginx restart
```


7. Confirm setup

Check the backend API by making a GET request to `http://localhost/aces-api/test-service-info`:

```
curl http://localhost/aces-api/test-service-info
{
  "capacity" : "∞",
  "flatFeeArk" : "0",
  "percentFee" : "0",
  "status" : "Up"
}
```

Open up the aces-app frontend in your browser: `https://localhost/aces-app/`

