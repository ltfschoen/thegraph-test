# Integrate into a dApp queries to subgraph indexes of a smart contract that is deployed to a local Graph Node and connected to a local Ethereum testnet using Ganache

## Terminal 1 - Run Ganache CLI

```
yarn global add truffle ganache-cli
npm uninstall -g truffle ganache-cli
ganache-cli -h 0.0.0.0
```

## Terminal 2 - Run local Graph Node to connect to Ganache

* Close and update to latest version, then run it
```
git clone https://github.com/graphprotocol/graph-node/
cd graph-node/docker
docker pull graphprotocol/graph-node:latest
docker-compose up
```

## Terminal 3 - Initialize new subgraph

```
yarn global add @graphprotocol/graph-cli
graph init \
  --from-example <GITHUB_USERNAME>/my-subgraph-project
```

* Note: Manually add further datasources in subgraph.yaml after running `graph init`

## Define a smart contract

* Build "autogenerated" ids for an entity when handling events by combining the transaction hash + log index to be unique, and optionally obfuscate by converting to Bytes and then piping through crypto.keccak256

## Define a subgraph

* Reference: https://thegraph.com/docs/define-a-subgraph

* Subgraph indexes include event handlers that are triggered by smart contract events if they are defined and are used to query data the fastest, otherwise if no events are defined the subgraph may trigger indexing using call and block handlers instead but with slower performance.
* Subgraph index syncing performance may be improved by specifying the optional start block to start indexing only from the block when the smart contract was deployed 
* Subgraph indexing configuration is defined in `subgraph.yml` and may include specifying the block (i.e. where the smart contract was created) for the Data Source to start indexing from (i.e. `dataSources.source.startBlock`) 
* Subgraphs need separate names for multiple networks (see https://thegraph.com/docs/deploy-a-subgraph#redeploying-a-subgraph)
* Subgraph processes events in the order they appear in blocks regardless of whether they are across different smart contracts
* Subgraph mappings are written in AssemblyScript at the moment https://thegraph.com/docs/assemblyscript-api
* Subgraphs deployed to the Graph Node support be queried directly using JSON-RPC API to determine the latest block number it has indexed, and use `indexingStatusForPendingVersion` insted if the subgraph is still pending
```
curl -X POST -d '{ "query": "{indexingStatusForCurrentVersion(subgraphName: \"organization/<SUBGRAPH_NAME>\") { chains { latestBlock { hash number }}}}"}' https://api.thegraph.com/index-node/graphql
``` 

### Microservices

* [Apollo Federation](https://www.apollographql.com/docs/federation/) is not yet supported. Use schema stitching on client or via proxy service in the interim

### Templates

* Define in a Template how you want to index a smart contract
* Subgraph creates a dynamic Data Source (by supplying the smart contract address) whilst your subgraph is indexing smart contracts (i.e. a smart contract may be spawning new contracts whilst users interact with it) that have been defined upfront in the structure of the smart contract (i.e. ABI, events, etc)  

## Terminal 3 - Deploy an example smart contract (i.e. GravatarRegistry) that the subgraph indexes to Ganache, and execute a couple of tx against the contract so there's data to index, then record to the <CONTRACT_ADDRESS> where the smart contract is deployed to.

```
truffle compile
truffle migrate
```

* Add the <CONTRACT_ADDRESS> from of the smart contract to the subgraph:
```
sed -i -e \
    's/0x2E645469f354BB4F5c8a05B3b30A929361cf77eC/<CONTRACT_ADDRESS>/g' \
    subgraph.yaml
```

* Install dependencies of the subgraph
```
yarn
```

* Run the Graph CLI code generation
```
yarn codegen
```

* Allocate the subgraph name in the Graph Node, and then deploy the subgraph to the local Graph Node

```
yarn create-local
yarn deploy-local
```

  * Note: Subgraph's must be redeployed when duplicated to another account or endpoint, but if no change to subgraph ID (IPFS hash) then do not have to sync from the start 

* Review terminal output logs of the Graph Node to troubleshoot

## Terminal 4 - Integrate the subgraph into a dApp by querying The Graph

* Clone dApp. Configure dApp to use the GraphQL endpoint of the example subgraph
Afterwards, enter the app directory and configure it to use the GraphQL endpoint of your example subgraph:

```
git clone https://github.com/graphprotocol/ethdenver-dapp/
cd ethdenver-dapp
echo 'REACT_APP_GRAPHQL_ENDPOINT=http://localhost:8000/subgraphs/name/<GITHUB_USERNAME>/<SUBGRAPH_NAME>' > .env
yarn && yarn start
```

* Differentiate between networks (i.e. Ethereum mainnet, Kovan, Rinkeby, Ropsten, Goerli, PoA-Core, xDAI, Sokol, local) from within event handlers by importing `dataSource ` from `@graphprotocol/graph-ts`
```
import { dataSource } from '@graphprotocol/graph-ts'

dataSource.network()
dataSource.address()
```

* Query more than the default query response limit of 100 items per collection (i.e. up to 1000) by using pagination, e.g.
```
someCollection(first: 1000, skip: <number>) { ... }
``` 

* On Rinkeby block handlers are supported but without `filter: call`, and call handlers are not supported

* Call a smart contract function or access a public state variable from the subgraph mappings
  * https://thegraph.com/docs/assemblyscript-api