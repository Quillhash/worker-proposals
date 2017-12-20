# ElasticSearch Plugin v1

The document explains motivations, technical challenges and sample use of a new plug-in created for bitshares to store account history data into an elasticsearch database.

## Motivation

There are 2 main problems this plug-in tries to solve:

- The amount of RAM needed to run a full node. Current options for this are basically store account history: yes/no, store some of the account history, track history for specific accounts, etc. Plugin allows to have all the account history data without the amount of RAM required in current implementation.
- The number of github issues in regards to new api calls to lookup account history in different ways. `elasticsearch-plugin` will allow users to search account history querying directly into the database without the bitshares-core team creating and maintaining API calls for every case.

Additionally we are after a secure way to store error free full account data. The huge amount of operations data involved in bitshares blockchain and the way it is serialized makes this a big challenge.

Even more, saving data to Elastic Search database(from now on: ES) will bring additional value to clients connected to the new full node with real time statistics and fast access to data.

## The database selection

Elastic search was selected for the main following reasons:

- open source.
- fast.
- index oriented.
- easy to install and start using.
- send data from c++ using curl.
- scalability and decentralized nodes of data.

## Technical

The `elasticsearch-plugin` is uses the basics of `account_history_plugin`.

Here is how the current account_history plugin works basically:
- with every signed block arriving to the plugin the ops are extracted from it.
- each op is added to ohi(operation history index) and to ath(account transaction history index).
- both indexes keep growing as new block gets in.
- with the memory reduction techniques currently available the 2 indexes can remove early ops of accounts reducing ram size.

And now what `elasticsearch-plugin` attempts to do:

- with every signed block arriving to the plugin the ops are extracted from it, just as in account_history.
- create indexes ohi and ath and store current op there just as account history do. this a temp indexation of 1 op only that is done to remain compatible with the previous numbers used as id(1.11.X and 2.9.X).
- send ath and ohi plus additional block data and visitor data into ES(actually we send them in bulks not one by one as we will explain below).
- remove everything in the compatibility temporal indexes except for current operation. This way the indexes always have just 1 item and don't use any ram.

### Replay and _bulk

As mentioned we don't send to ES the operations one by one, this is because in a replay the number of ops will be so big that performance will decrease drastically and time to be in sync will be too much.
For this reason we use the ES bulk api and send by default 10000 lines when we are in replay and downgrade this to 100 lines after we are in sync.

ES bulk format is one line of meta-data and one line of data itself, so 10000 is actually 2500 operations we send on every bulk and 100 is actually 50 ops in real time mode.

This values are available as plugin options to the user to change.

The optimal number of docs to bulk is hardware dependent.

The name of the created index in ES will be `graphene`

### Accessing data inside operations

There are cases where data coming from account transaction history index, operation history index and block data is not enough. We may want to index fields inside operations themselves like the fees, the asset id or the amount of transfer to make fast queries into them. 

Data inside operations is saved as a text fields into ES, this means that we can't fast search into them as the data is not indexed, we can, still search by other data and filter converting the op text to json in client side. This is possible and relatively easy so please consider.

Still, a workaround the limitation is available. A `visitor` that can be turned on/off by the command line. As an example something in common all ops have is a fee field with `asset_id` and `amount`. In `elasticserch-plugin` v1 when visitor is `true` this 2 values will be saved meaning clients can know total chain fees collected in real time, total fees in asset, fees by op among other things.

As a poc we also added amount and asset_id of transfer operations to illustrate how easy is to index more data for any competent graphene developer.

## Installation

Basically there are 2 things you need: elasticsearch and curl for c++. elasticsearch need java so those are the 3 things you will need to have. The following are instructions tested in debian(ubuntu - mint) based linux versions.

### Install java:

download the jre, add sudo to the start of the commands if installing from a non root user:

`apt-get install default-jre`

we are going to need the jdk too:

`apt-get install default-jdk`

add repository to install oracle java8:

in ubuntu:

`add-apt-repository ppa:webupd8team/java` 

in debian:

`add-apt-repository "deb http://ppa.launchpad.net/webupd8team/java/ubuntu xenial main"`

then:
`apt-get update`

if you don't have add-apt-repository command you can install it by: 

`apt-get install software-properties-common`

install java 8:

`apt-get install oracle-java8-installer`

### Install ES:

Get the last version zip file at: https://www.elastic.co/downloads/elasticsearch

Please do this as a non root user as ES will not run as root.

download as: 

`wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.6.3.zip`

unzip:

`unzip elasticsearch-5.6.3.zip`

and run:

```
cd elasticsearch-5.6.3/
./bin/elasticsearch
```

You can put this as a service, the binary haves a `--deamon` option, can run inside `screen` or any other option that suits you in order to keep the database running. 

Please note ES does not run a the root user, if you are a root user you need to first make a normal user account by:

`adduser elastic`

After created: `su elastic`. Execute the `wget` and `unzip` commands from above as normal user in created user home dir. 

### Install curl

We need curl to send requests from the c++ plugin into the ES database:

`apt-get install libcurl4-openssl-dev`

## Running

Make sure ES is running, can start it by:

`./elasticsearch --daemonize`

ES will listen on localhost port 9200 `127.0.0.1:9200`

Clone repo and install bitshares:

```
git clone https://github.com/bitshares/bitshares-core
cd bitshares-core
git checkout -t origin/develop
git submodule update --init --recursive
BOOST_ROOT=$HOME/opt/boost_1_63_0
cmake -DBOOST_ROOT="$BOOST_ROOT" -DCMAKE_BUILD_TYPE=RelWithDebInfo .
make
```

### Arguments

The ES plugin have the following parameters passed by command line:

- `elasticsearch-node-url` - The url od elasticsearch - default: `http://localhost:9200/`
- `elasticsearch-bulk-replay` - The number of lines(ops * 2) to send to database in replay state - default: `10000`
- `elasticsearch-bulk-sync` - The number of lines(ops * 2) to send to database at syncronized state - default: `100` 
- `elasticsearch-logs` - Save logs to database - default: `true`
- `elasticsearch-visitor` - Index visitor additional inside op data - default: `true`

### Starting node

ES plugin is not active by default, we need to start it with the `plugins` parameter. An example of starting a node with ES plugin may be:

`programs/witness_node/witness_node --data-dir data/my-blockprod --rpc-endpoint "127.0.0.1:8090" --plugins "witness elasticsearch market_history" --elasticsearch-bulk-replay 10000 --elasticsearch-logs true --elasticsearch-visitor true`

### Checking if it is working

A few minutes after the node start the first batch of 5000 ops will be inserted to the database. If you are in a desktop linux you may want to install https://github.com/mobz/elasticsearch-head and see the database from the web browser to make sure if it is working. This is optional.

If you only have command line available you can query the database directly throw curl as:

```
root@NC-PH-1346-07:~/bitshares/elastic/bitshares-core# curl -X GET 'http://localhost:9200/graphene-*/data/_count?pretty=true' -d '
{
    "query" : {
        "bool" : { "must" : [{"match_all": {}}] }
    }
}
'
{
  "count" : 360000,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  }
}
root@NC-PH-1346-07:~/bitshares/elastic/bitshares-core# 
```

360000 records are inserted at this point of the replay in ES, means it is working.

**Important: Replay with ES plugin will be always slower than the "save to ram" `account_history_plugin` so expect to wait more to be in sync than usual.**

A syncronized node will look like this(screen capture 20/12/2017):

```
root@NC-PH-1346-07:~# curl -X GET 'http://localhost:9200/graphene-*/data/_count?pretty=true' -d '
{
    "query" : {
        "bool" : { "must" : [{"match_all": {}}] }
    }
}
'
{
  "count" : 107156155,
  "_shards" : {
    "total" : 135,
    "successful" : 135,
    "skipped" : 0,
    "failed" : 0
  }
}
root@NC-PH-1346-07:~# 
```

## Indexes

The plugin creates monthly indexes in the ES database, index names are as `graphene-2016-05` and contain all the operations made inside the monthly period.

List your indexes as:

```
NC-PH-1346-07:~# curl -X GET 'http://localhost:9200/_cat/indices' 
yellow open logs             PvSvmtcFSCCF_-3HBjJaXg 5 1   358808 0   5.8gb   5.8gb
yellow open graphene-2017-09 ZUUi5xeuTeSb7QPP1Pnk0A 5 1 11075939 0   8.4gb   8.4gb
yellow open graphene-2017-04 HtCKnM_UTLyDWBR7-PopGQ 5 1  3316120 0   2.4gb   2.4gb
yellow open graphene-2017-10 SAUFtI0CRx6MnISukeqA-Q 5 1  9326346 0   7.1gb   7.1gb
yellow open graphene-2017-07 CTWIim51SZy0Ir55sjlT7w 5 1 17890903 0  12.8gb  12.8gb
yellow open graphene-2017-01 sna676yfR1OT5ww9HRiVZw 5 1  1197124 0 917.7mb 917.7mb
yellow open graphene-2016-06 FZ_FyTLwQCKObjKgLQcF3w 5 1   656358 0 473.9mb 473.9mb
yellow open graphene-2017-03 Ogsw0PlbRAWeHRS86o4vTA 5 1  2167788 0   1.6gb   1.6gb
yellow open graphene-2017-11 -9-oz-DRRTOMLdZVOUV5xw 5 1 14107174 0  10.8gb  10.8gb
yellow open graphene-2017-02 ZE3LxFWkTeO7CBsxMMmIFQ 5 1  1104282 0 822.4mb 822.4mb
yellow open graphene-2017-12 bZRFc5aLSlqbBMYEnKmF7Q 5 1  8758628 0     8gb     8gb
yellow open graphene-2015-11 iXyIZkj_T0O-NjPxEf0L_A 5 1   301079 0 193.1mb 193.1mb
yellow open graphene-2017-05 ipVA6yEKTH68nEucBIZP7w 5 1 10278394 0   7.5gb   7.5gb
yellow open graphene-2017-08 Z2C_pciwQSevJzcl9kRhhQ 5 1  9916248 0   7.5gb   7.5gb
yellow open graphene-2016-04 vtQSi-kdQ0OblRq2naPu-A 5 1   413798 0 297.4mb 297.4mb
yellow open graphene-2016-12 hsohZ0cfTGSPPVWUM_Fgeg 5 1   917034 0 711.3mb 711.3mb
yellow open graphene-2016-01 PzjbXCXFShCzZFyTBgv2IQ 5 1   372772 0 247.1mb 247.1mb
yellow open graphene-2016-03 8j1yyLIeTUWXb6TZntqQhg 5 1   597461 0   431mb   431mb
yellow open graphene-2016-08 6qBByeOPSkCDqWYrOR_qPg 5 1   551835 0 389.3mb 389.3mb
yellow open graphene-2015-10 DTHVtNpIT2KAoMiO4-c61w 5 1   161004 0 122.2mb 122.2mb
yellow open graphene-2017-06 Jei8GvRYSkif4zZ-V9mFRg 5 1 10795239 0   8.1gb   8.1gb
yellow open .kibana          8q5behLlQRi_hxKSWurYDw 1 1        7 0  42.7kb  42.7kb
yellow open graphene-2016-05 s7Ie2crTS_uismCP5U2dIQ 5 1   498493 0 353.4mb 353.4mb
yellow open graphene-2016-07 7yx03R2hQX6XyOR8Sdu7ZA 5 1   609087 0 435.8mb 435.8mb
yellow open graphene-2016-02 0SVSrhn3QK2AxWNv9NRASQ 5 1   468174 0 321.8mb 321.8mb
yellow open graphene-2016-10 C28juWi8Rf-1A0h2YnShBw 5 1   416570 0 299.9mb 299.9mb
yellow open graphene-2016-09 3rHI_5HFR3SFl9bhroL5EA 5 1   409164 0 296.1mb 296.1mb
yellow open graphene-2015-12 aLlUxO_tSG-CMC3SPcFgkw 5 1   349985 0 222.4mb 222.4mb
yellow open graphene-2016-11 i00upS94Ruii1zJYXI0S9Q 5 1   495556 0 367.3mb 367.3mb
root@NC-PH-1346-07:~# 
```

If you dont see any index here then something is wrong with the bitshares-core node setup with elasticsearch plugin. Make sure you are running elasticsearch version 5.X.

Todo: Make plugin work with version 5 of elasticsearch too.

## Usage

After your node is in sync you are in posesion of a full node without the ram issues. A syncronized witness_node with ES will be using less than 10 gigs of ram:

```
 total          9817628K
root@NC-PH-1346-07:~# pmap 2183
```

Compare against a tradtional full node:

```
 total         60522044K
[bitshares@lantea ~]$
```

Please note start command was with `markets_history` plugin activated, only account data should use even less than 10 gigs. 

What client side apps can do with this new data is kind of unlimited to client developer imagination but lets check some real world examples to see the benefits of this new feature.

### Get operations by account, time and operation type

References:
https://github.com/bitshares/bitshares-core/issues/358
https://github.com/bitshares/bitshares-core/issues/413
https://github.com/bitshares/bitshares-core/pull/405
https://github.com/bitshares/bitshares-core/pull/379
https://github.com/bitshares/bitshares-core/pull/430
https://github.com/bitshares/bitshares-ui/issues/68

This is one of the issues that has been requested constantly. It can be easily queried with ES plugin by calling the _search endpoint doing:

```
root@NC-PH-1346-07:~/bitshares/elastic/bitshares-core# curl -X GET 'http://localhost:9200/graphene/data/_search?pretty=true' -d '
{
    "query" : {
        "bool" : { "must" : [{"term": { "account_history.account.keyword": "1.2.282"}}, {"range": {"block_data.block_time": {"gte": "2015-10-26T00:00:00", "lte": "2
015-10-29T23:59:59"}}}] }
    }
}
'
{
  "took" : 99,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 3,
    "max_score" : 9.865524,
    "hits" : [
      {
        "_index" : "graphene",
        "_type" : "data",
        "_id" : "2.9.121234",
        "_score" : 9.865524,
        "_source" : {
          "account_history" : {
            "id" : "2.9.121234",
            "account" : "1.2.282",
            "operation_id" : "1.11.114383",
            "sequence" : 11,
            "next" : "2.9.113951"
          },
          "operation_history" : {
            "trx_in_block" : 0,
            "op_in_trx" : 0,
            "operation_results" : "[0,{}]",
            "virtual_op" : 14811,
            "op" : "[0,{'fee':{'amount':4210936,'asset_id':'1.3.0'},'from':'1.2.89940','to':'1.2.282','amount':{'amount':2000000,'asset_id':'1.3.535'},'memo':{'from
':'BTS8LWkZLmsnWjgtT1PNHT5XGAu1z1ueQkBHBQTVfECFVQfD3s7CF','to':'BTS5TPTziKkLexhVKsQKtSpo4bAv5RnB8oXcG4sMHEwCcTf3r7dqE','nonce':'370147387190899','message':'559c6966
ccf3db2583f9636489c7be0339a88c1703f3b60dde9f165825720486'},'extensions':[]}]"
          },
          "operation_type" : 0,
          "block_data" : {
            "block_num" : 375534,
            "block_time" : "2015-10-26T19:37:18",
            "trx_id" : ""
          },
          "fee_data" : {
            "fee_asset" : "1.3.0",
            "fee_amount" : "4210936"
          },
          "transfer_data" : {
            "transfer_asset_id" : "1.3.535",
            "transfer_amount" : "2000000"
          }
        }
      },
      {
        "_index" : "graphene",
        "_type" : "data",
        "_id" : "2.9.143682",
        "_score" : 9.720996,
        "_source" : {
          "account_history" : {
            "id" : "2.9.143682",
            "account" : "1.2.282",
            "operation_id" : "1.11.136390",
            "sequence" : 13,
            "next" : "2.9.126760"
          },
          "operation_history" : {
            "trx_in_block" : 0,
            "op_in_trx" : 0,
            "operation_results" : "[0,{}]",
            "virtual_op" : 36818,
            "op" : "[0,{'fee':{'amount':4335936,'asset_id':'1.3.0'},'from':'1.2.376','to':'1.2.282','amount':{'amount':1010000,'asset_id':'1.3.535'},'memo':{'from':
'BTS6zT2XD7YXJpAPyRKq8Najz4R5ut3tVMEfK8hqUdLBbBRjTjjKy','to':'BTS5TPTziKkLexhVKsQKtSpo4bAv5RnB8oXcG4sMHEwCcTf3r7dqE','nonce':'370212206009967','message':'02b9072a43
b3e1ed960e7710b227984600b17fa3d6837bed56fba07e69c6e2793eac9ea7d3851eb48b04e9ab48933900996b00016d2f145808d30dc81e61232f1b29afb38caf1361e46927d8749826a26435bd9a227533
d5f1b181bfca5e26ac'},'extensions':[]}]"
          },
          "operation_type" : 0,
          "block_data" : {
            "block_num" : 459202,
            "block_time" : "2015-10-29T17:57:18",
            "trx_id" : ""
          },
          "fee_data" : {
            "fee_asset" : "1.3.0",
            "fee_amount" : "4335936"
          },
          "transfer_data" : {
            "transfer_asset_id" : "1.3.535",
            "transfer_amount" : "1010000"
          }
        }
      },
      {
        "_index" : "graphene",
        "_type" : "data",
        "_id" : "2.9.126760",
        "_score" : 9.720996,
        "_source" : {
          "account_history" : {
            "id" : "2.9.126760",
            "account" : "1.2.282",
            "operation_id" : "1.11.119823",
            "sequence" : 12,
            "next" : "2.9.121234"
          },
          "operation_history" : {
            "trx_in_block" : 0,
            "op_in_trx" : 0,
            "operation_results" : "[0,{}]",
            "virtual_op" : 20251,
            "op" : "[0,{'fee':{'amount':4000000,'asset_id':'1.3.0'},'from':'1.2.96086','to':'1.2.282','amount':{'amount':78970118,'asset_id':'1.3.0'},'extensions':[
]}]"
          },
          "operation_type" : 0,
          "block_data" : {
            "block_num" : 394952,
            "block_time" : "2015-10-27T11:51:30",
            "trx_id" : ""
          },
          "fee_data" : {
            "fee_asset" : "1.3.0",
            "fee_amount" : "4000000"
          },
          "transfer_data" : {
            "transfer_asset_id" : "1.3.0",
            "transfer_amount" : "78970118"
          }
        }
      }
    ]
  }
}
root@NC-PH-1346-07:~/bitshares/elastic/bitshares-core# 
```

more samples here ...

### Get operations by account and block

References:
https://github.com/bitshares/bitshares-core/issues/61

```
root@NC-PH-1346-07:~# curl -X GET 'http://localhost:9200/graphene/data/_search?pretty=true' -d '
{
    "query" : {
        "bool" : { "must" : [{"term": { "account_history.account.keyword": "1.2.356589"}}, {"range": {"block_data.block_num": {"gte": "17824289", "lte": "17824290"}                                                                                                                  
}}] }                                                          
    }
}
'
{
  "took" : 284,
  "timed_out" : false,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : 1,
    "max_score" : 15.487353,
    "hits" : [
      {
        "_index" : "graphene",
        "_type" : "data",
        "_id" : "2.9.35267898",
        "_score" : 15.487353,
        "_source" : {
          "account_history" : {
            "id" : "2.9.35267898",
            "account" : "1.2.356589",
            "operation_id" : "1.11.34664169",
            "sequence" : 2,
            "next" : "2.9.34665677"
          },
          "operation_history" : {
            "trx_in_block" : 1,
            "op_in_trx" : 0,
            "operation_results" : "[0,{}]",
            "virtual_op" : 27469,
            "op" : "[0,{'fee':{'amount':22941,'asset_id':'1.3.0'},'from':'1.2.152768','to':'1.2.356589','amount':{'amount':100000000,'asset_id':'1.3.0'},'memo':{'fr
om':'BTS6o6byTjfThztXDD4nhzV6tR4MpjRaLVmkfiDCiTeeJfmQNKCHb','to':'BTS6kY7wQtCAbagTj8YmyRXBDTrdLra6Qh8F4oucCiqZRMAWtMPif','nonce':'383662087143284','message':'57bd2a
f0f786557ed343b76c37ac3b59'},'extensions':[]}]"
          },
          "operation_type" : 0,
          "block_data" : {
            "block_num" : 17824289,
            "block_time" : "2017-06-28T19:59:51",
            "trx_id" : ""
          },
          "fee_data" : {
            "fee_asset" : "1.3.0",
            "fee_amount" : "22941"
          },
          "transfer_data" : {
            "transfer_asset_id" : "1.3.0",
            "transfer_amount" : "100000000"
          }
        }
      }
    ]
  }
}
root@NC-PH-1346-07:~#
```

### Get operations by transaction hash

Refs: https://github.com/bitshares/bitshares-core/pull/373

## New stats

After we solve some of the issues needed by the community and generating a framework for future issues of the same kind lets go a bit beyond and explore how rich is to have account data operations stored in ES in regards to stats. This are just a few samples.

### Get total ops

- get total all history ops by type.
```
root@NC-PH-1346-07:~# curl -X GET 'http://localhost:9200/graphene/data/_count?pretty=true' -d '
{
    "query" : {
        "bool" : { "must" : [ {"term": {"operation_type": "34"}}] }
    }
}
'
{
  "count" : 63,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  }
}
root@NC-PH-1346-07:~# curl -X GET 'http://localhost:9200/graphene/data/_count?pretty=true' -d '
{
    "query" : {
        "bool" : { "must" : [ {"term": {"operation_type": "0"}}] }
    }
}
'
{
  "count" : 1225858,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  }
}
root@NC-PH-1346-07:~# 
```
- get total ops in a period of time. 

```
root@NC-PH-1346-07:~# curl -X GET 'http://localhost:9200/graphene/data/_count?pretty=true' -d '
{
    "query" : {
        "bool" : { "must" : [ {"term": {"operation_type": "0"}}, {"range": {"block_data.block_time": {"gte": "2017-01-01T00:00:00", "lte": "2017-09-01T00:00:00"}}}]
 }   
    }
}
'
{
  "count" : 697324,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  }
}
root@NC-PH-1346-07:~# curl -X GET 'http://localhost:9200/graphene/data/_count?pretty=true' -d '
{
    "query" : {
        "bool" : { "must" : [ {"term": {"operation_type": "0"}}, {"range": {"block_data.block_time": {"gte": "2017-08-25T00:00:00", "lte": "2017-09-01T00:00:00"}}}]
 }
    }
}
'
{
  "count" : 25872,
  "_shards" : {
    "total" : 5,
    "successful" : 5,
    "skipped" : 0,
    "failed" : 0
  }
}
root@NC-PH-1346-07:~# 
```

- get total ops by type inside a range of blocks.

### Get speed data

- ops per second
- trxs per second

### Get fee data

- total collected fee
- total colected fee for op type
- total fees collected this month
- total fee collected in asset

### Transfer data

- get amount transfered from account to account.

### More visitor code = more indexed data = more filters to use

Just as an example, it will be easy to index asset of trading operations by extending the visitor code of them. point 3 of https://github.com/bitshares/bitshares-core/issues/358 request trading pair, can be solved by indexing the asset of the trading ops as mentioned.

Remember ES already have all the needed info in the `op` text field of the `operation_history` object. Client can get all the ops of an account, loop throw them and convert the `op` string into json being able to filter by the asset or any other field needed. There is no need to index everything but it is possible.

## Duplicates

By using the `op_type` = `create` on each bulk line we send to the database and as we use an unique ID(ath id(2.9.X)) the plugin will not index any operation twice. If the node is on a replay, the plugin will start adding to database when it find a new record and never before. 

## The future

elasticsearch-plugin aims to be included and be part bitshares-core project as a cheaper ram full node alternative to `account_history_plugin` while obtaining the benefits in querying the huge amounts of data present in the blockchain in a new efficient and more versatile way.

Plugin should be improved in speed and performance by the developer community and active workers, some basic maintenance will be needed  - ex: if a new operation came in we need to add it to the visitor. Interested third parties can improve it for their own needs.

Market data plugin alternative is the next structure of objects we may want to move to ES. 
