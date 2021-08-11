DINGIR-EXCHANGE SETUP (on Ubuntu)
====

## Installation

Download source code:  
```git clone git@github.com:Fluidex/dingir-exchange.git ; cd dingir-exchange```

### Prerequisite

* cmake
* librdkafka

```
# apt install cmake librdkafka-dev
```

### Starting microservice infrastructure

run 
```bash
cd docker
docker-compose up
```

Starts the following containers:  
* Kafka: `exchange_kafka`
* ZooKeeper: `exchange_zookeeper`
* PostgreSQL Timescale: `exchange_pq`
* Envoy: `exchange_envoy`

Envoy is a service proxy and it's deployment area is displayed in the following, simplified architecture design:
![simplified architecture](https://raw.githubusercontent.com/gcomte/dingir-exchange-docs/main/img/simplifiedArchitecture.svg)

(checkout overall Fluidex architecture [here](https://www.fluidex.io/en/blog/fluidex-architecture/))

### Starting the matching engine

Fire it up:  
`make startall`

See all the running processes:  
`make pgrep`

4 processes should be running:
* matchengine
* restapi
* persistor
* tokio-runtime-w

Watch logs:  
For once: `make taillogs`, continuously: `make viewlogs`

Shutdown the matching engine:
`make stopall`

## Database

As defined in `docker/docker-compose.yaml`, Postgres stores its data in the folder `./data/volumes/postgres_data`.

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

If this doesn't work and you want to start with a clean slate, consider shutting everything down (`make stopall` & `cd docker ; docker-compose down`) and remove the data directory:  
`rm -rf docker/data`  
Then start everything again (`cd docker ; docker-compose up` and `make startall`)

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
There is a test suite written in TypeScript. It connects directly to the matching engine, over the matching engine's GRPC interface (it does **not** pass through *Envoy* or *Kafka*).

### Running the test suite
Install it:  
`cd examples/js ; npm i`

and run it:  
`npx ts-node trade.ts`

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


