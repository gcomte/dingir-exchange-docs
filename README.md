DINGIR-EXCHANGE SETUP (on Ubuntu)
====

## Installation

Download source code:  
```git clone --recursive git@github.com:Fluidex/dingir-exchange.git ; cd dingir-exchange```

### Prerequisite

* cmake
* librdkafka

```bash
apt install cmake librdkafka-dev
```

You might as well try to run the following script to automatically install all dependencies: `./scripts/install_all_deps.sh`.

### Starting microservice infrastructure

run 
```bash
cd orchestra/docker
docker-compose up
```

Starts the following containers:  
* Kafka: `exchange_kafka`
* ZooKeeper: `exchange_zookeeper`
* PostgreSQL Timescale: `exchange_db`
* Envoy: `exchange_envoy`

Envoy is a service proxy and it's deployment area is displayed in the following, simplified architecture design:
![simplified architecture](https://raw.githubusercontent.com/gcomte/dingir-exchange-docs/main/img/simplifiedArchitecture.svg)

(checkout overall Fluidex architecture [here](https://www.fluidex.io/en/blog/fluidex-architecture/))

### Starting the matching engine

Fire it up:  
`make startall`

See all the running processes:  
`make list`

4 processes should be running:
* matchengine
* restapi
* persistor
* tokio-runtime-w

Watch logs:  
For once: `make taillogs`, continuously: `make viewlogs`

Shutdown the matching engine:
`make stopall`

## Dependencies

### orchestra
Dingir-Exchange outsources the docker environment (PostgreSQL, Kafka, ZooKeeper, Envoy) and the Protocol Buffers (gRPC interface definition) into the library [orchestra](https://github.com/fluidex/orchestra).

## Configuration
There is an `.env` file in the root directory, which lets you pass some Rust parameters (for example for Logging) but also lets you specify which environment you want to run on. Those environments are then further defined in the yaml files under in the `config` directory (e.g. `development.yaml`)

## Database

As defined in `orchestra/docker/docker-compose.yaml`, Postgres stores its data in the folder `./orchestra/docker/volumes/exchange_db/data`.

### Manage DB

Get into postgres container:  
`make conn`

Show tables:  
`\d`

You can run queries from there:  
`SELECT * FROM account;`

### Reset DB

From the project root, run:  
`make cleardb`

If this doesn't work and you want to start with a clean slate, consider shutting everything down (`make stopall` & `make stop-compose`) and remove the data directory:  
`make clean-compose`  
Then start everything again (`cd orchestra/docker ; docker-compose up` and `make startall`)

### Migration scripts
In the file `src/matchengine/persist/state_save_load.rs` the macro `sqlx::migrate!()` is called. By default, it will check out all the scripts that are in the `migrations` (only the root directory, subdirectories are being ignored), and run them against the db, in the order of the timestamps their filename starts with.

These migration scripts will mainly establish the database structure (tables, indexes, unique contraints, ...), but with the file `migrations/20210223072038_markets_preset.sql` it will also insert some data about the available assets and the available trading pairs.

### DB abstractions in Rust
There is an abstraction of the database in Rust in the file `dingir-exchange/src/storage/models.rs`.  

Caution:  
If you change add or remove attributes/columns to the table abstractions (adding or removing lines that contain the syntax `arg.add()`), make sure to change the respective number of arguments constant accordingly (`const ARGN: i32`). This number should always be equal to the number of lines with the syntax `arg.add()`.  

Dingir-Exchange uses these db models to create *prepared statements*.

## Matchengine gRPC interface
The gRPC interface of the matchengine is defined in the file `proto/exchange/matchengine.proto`.  
Dingir-Exchange uses [tonic](https://crates.io/crates/tonic) in the file `./build.rs`, to compile the proto file into Rust code.
The created file has been checked in onto GitHub, namely into the src folder: `src/matchengine/rpc/matchengine.rs`.
This implementation of the interface forwards requests to the methods in the file `src/matchengine/server.rs`. You can find a method for each *service* that is defined in `matchengine.proto`, but the names are transformed from CamelCase to snake_case to fit the rust naming conventions.  

For example there is a service called `OrderPut` registered in the file `matchengine.proto`:
```GRPC
rpc OrderPut(OrderPutRequest) returns (OrderInfo)
    option (google.api.http) = {
      post : "/api/order"
      body : "*"
    };
  }
```
  
Requests received on this endpoint would then be forwarded to the method `async fn order_put(&self, request: Request<OrderPutRequest>) -> Result<Response<OrderInfo>, Status>` in the file `src/matchengine/server.rs`.

## Test suite
There is a test suite written in TypeScript. It connects directly to the matching engine, over the matching engine's GRPC interface (it does **not** pass through *Envoy* or *Kafka*). However, if you want to test the *consumption* of Kafka subscriptions, you may turn these tests on by adding the line `TEST_MQ=1` to the file `examples/js/.env`.

Changing the `.env`-variables might only go into effect after manually recompiling the entire TypeScript application:  
`cd examples/js && node_modules/typescript/bin/tsc --build --force ./tsconfig.json`

### Running the test suite
Install it:  
`cd examples/js ; npm i`

and run it:  
`npx ts-node tests/trade.ts`

### Environment
If you don't want to use the local standard setup but instead want to change IPs, ports and/or container names, you can copy the `.env`-template-file `examples/js/.env.advise` to `examples/js/.env` and reconfigure the variables to your needs.

### Data

The test suite will work with a couple of data sets that are stored in different places.

#### User account dummies

They're stored in the file `examples/js/accounts.jsonl`.

#### Funds deposits
The function `setupAsset()` in the file `examples/js/trade.ts` deposits funds onto user accounts.

#### Order put
The function `orderTest()` in the file `examples/js/trade.ts` creates an order and cancels it.

#### Trade
The function `tradeTest()` in the file `examples/js/trade.ts` creates orders that match and therefore result in a successful trade.

#### Config
There is some dummy data in the file `examples/js/config.ts` too.


