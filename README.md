# Hyperledger Composer

Hyperledger Composer is an extensive, open development toolset and framework to make developing blockchain applications easier. Primary goal of Hyperledger Composer is to accelerate time to value, and make it easier to integrate your blockchain applications with the existing business systems. Hyperledger Composer supports the existing Hyperledger Fabric blockchain infrastructure and runtime. The key concept for Hyperledger Composer is the business network definition (BND).

Steps to integrate blockchain application with Hyperledger fabric are as follows:
* Use Hyperledger Composer to create a business network definition consisting of Model(.cto), Script(.js), Access Control List(.acl), Query Definitions(.qry) files.
* Package up business network definition into an archive(.bna) which can be deployed to an existing business network (Might be a fabric network).
* Start fabric network.
* Prepare fabric peers by installing .nba file on each peer.
* Start business network on fabric.
* Start REST API server. 
* Generate Angular Application.

## Installation
  Make sure you have the required pre-requisites by following [Installing pre-requisites](https://hyperledger.github.io/composer/latest/installing/installing-prereqs.html).
#### Install CLI tools
 ```sh
 $ npm install -g composer-cli
 $ npm install -g composer-rest-server
 $ npm install -g generator-hyperledger-composer
 $ npm install -g yo
 ```
 #### Install Playground
 You can try composer [online](https://composer-playground.mybluemix.net/) . To try locally on the development machine, run the following command.
 ```sh
 npm install -g composer-playground
 ```
 #### Set up IDE
 - Install VSCode from this URL: https://code.visualstudio.com/download
 - Open VSCode, go to Extensions, then search for and install the Hyperledger Composer extension from the Marketplace.
 #### Install Hyperledger Fabric
```sh
$ mkdir ~/fabric-dev-servers && cd ~/fabric-dev-servers
$ curl -O https://raw.githubusercontent.com/hyperledger/composer-tools/master/packages/fabric-dev-servers/fabric-dev-servers.tar.gz
$ tar -xvf fabric-dev-servers.tar.gz
$ cd ~/fabric-dev-servers
$ ./downloadFabric.sh
```
Start fabric network by running startFabric script.
```sh
$ cd ~/fabric-dev-servers
$ ./startFabric.sh
$ ./createPeerAdminCard.sh
```
You can start the playground by using following command
```sh
$ composer-playground
```
You can see the playground in http://localhost:8080/login. If you are seeing PeerAdmin@hlfv1 Card in the browser playground then everything has installed correctly.

As fabric network is already running. Now the task is to start a business network and mount in on existing fabric network. To do that a business network archive(.bna) should be made and integrate with existing fabric network. Following development section will clarify all the integration and create an application.

## Development
Following steps will walk you through building a Hyperledger Composer blockchain solution from scratch.

#### Creation of Business Network Structure
Create a skeleton Business Network using Yeoman generator. 
```sh
$ yo hyperledger-composer:businessnetwork
```
1. Enter `tutorial-network` as Business network name (or you can give a name as per your choice). Give the description, author fields etc.
2. Select `Apache-2.0` as the license.
3. Select `org.example.biznet` as the namespace.
4. Select `No` when asked whether to generate an empty network or not.

#### Defining a Business Network
Update the model file(.cto) to define assets, participants and transactions. To do that open `org.example.biznet.cto` in models directory and replace the contents with the following and save the changes.
```cto
/**
 * My commodity trading network
 */
namespace org.example.biznet
asset Commodity identified by tradingSymbol {
    o String tradingSymbol
    o String description
    o String mainExchange
    o Double quantity
    --> Trader owner
}
participant Trader identified by tradeId {
    o String tradeId
    o String firstName
    o String lastName
}
transaction Trade {
    --> Commodity commodity
    --> Trader newOwner
}
```
Now in the model file transaction `Trade` involves participant `Trader` and asset `Commodity`. To execute the transaction it should be defined as a function in javascript `logic.js` in `lib` directory. To do that replace the content of `logic.js` file with the following and save the changes.
```js
/**
 * Track the trade of a commodity from one trader to another
 * @param {org.example.biznet.Trade} trade - the trade to be processed
 * @transaction
 */
async function tradeCommodity(trade) {
    trade.commodity.owner = trade.newOwner;
    let assetRegistry = await getAssetRegistry('org.example.biznet.Commodity');
    await assetRegistry.update(trade.commodity);
}
```
You can add the access control in your business network by replacing the content of `permissions.acl` file with the following.
```acl
/**
 * Access control rules for tutorial-network
 */
rule Default {
    description: "Allow all participants access to all resources"
    participant: "ANY"
    operation: ALL
    resource: "org.example.biznet.*"
    action: ALLOW
}

rule SystemACL {
  description:  "System ACL to permit all access"
  participant: "ANY"
  operation: ALL
  resource: "org.hyperledger.composer.system.**"
  action: ALLOW
}
```
#### Generation of Business Network Archive (.bna)
From the `tutorial-network` directory run the following command
```sh
$ composer archive create -t dir -n .
```
Now, you can see `tutorial-network@0.0.1.bna` archive file in the `tutorial-network` directory which is used to integrate with the existing business network (can be Hyperledger Fabric).
#### Deployment of Business Network
Here we will use `PeerAdmin` business network card which is already created during the installation step of Hyperledger fabric, to deploy the business network. `PeerAdmin` card is used to install and instantiate chaincodes on peers of fabric network.

Deploying a business network to the Hyperledger Fabric requires the Hyperledger Composer business network to be installed on the peer, then the business network can be started, and a new participant, identity, and associated card must be created to be the network administrator. Finally, the network administrator business network card must be imported for use, and the network can then be pinged to check it is responding.
1. To install the business network, from the `tutorial-network` directory, run the following command:
```sh
$ composer network install --card PeerAdmin@hlfv1 --archiveFile tutorial-network@0.0.1.bna
```
2. To start the business network, run the following command:
```sh
$ composer network start --networkName tutorial-network --networkVersion 0.0.1 --networkAdmin admin --networkAdminEnrollSecret adminpw --card PeerAdmin@hlfv1 --file networkadmin.card
```
3. To import the network administrator identity as a usable business network card, run the following command:
```sh
$ composer card import --file networkadmin.card
```
4. To check that the business network has been deployed successfully, run the following command to ping the network:
```sh
$ composer network ping --card admin@tutorial-network
```
After these steps, business network has been deployed and integrated with Hyperldeger fabric. Now we can define the REST API for our business network as well as create an application for the same.

#### Generation of REST Server
To generate REST API, run the following command from tutorial-network
```sh
$ composer-rest-server
```
Select the options as following:
* Enter the name of the business network card to use: admin@tutorial-network
* Specify if you want namespaces in the generated REST API: never use namespaces
* Specify if you want to use an API key to secure the REST API: No
* Specify if you want to enable authentication for the REST API using Passport: No
* Specify if you want to enable event publication over WebSockets: Yes
* Specify if you want to enable TLS security for the REST API: No

Now The generated API is connected to the deployed blockchain and business network.

#### Application Generation
Hyperledger Composer can also generate an Angular 4 application running against the REST API. To create Angular 4 application run the following command from `tutorial-network` directort in new terminal window.
```sh
$ yo hyperledger-composer:angular
```
Select the options as following, if not mentioned below fill it as per your choice
* Do you want to connect to a running Business Network? Yes
* Project name: `Fill it as per your choice`
* License: Apache-2.0
* Name of the Business Network card: admin@tutorial-network
* Do you want to generate a new REST API or connect to an existing REST API?  Connect to an existing REST API
* REST server address: http://localhost
* REST server port: 3000
* Should namespaces be used in the generated REST API? Namespaces are not used

After creating angular application, a directory inside `tutorial-network` will be created with the name as specified project name in the above step. Navingate to that project directory and run the following command to start the application
```sh
$ npm start
```
You will see in the terminal as
```
** NG Live Development Server is running on http://0.0.0.0:4200 **
```
That means Angular 4 application is running against your REST API at http://localhost:4200 .
From the application you can add assets, participants and invoke the transactions. You can do the same from http://localhost:3000/explorer/#/ and use the GET and POST methods.

Congratulations !! You have successfully created a blockchain application.