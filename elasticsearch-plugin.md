# ElasticSearch Plugin v1

The document explains motivations, rationale and technical aspects of a new plugin created for bitshares to store account history data into an elasticsearch database.

## Motivation

There are 2 main problems this plugin tries to solve

- The amount of RAM needed to run a full node. Current options for this are basically store account history: yes/no, store some of the account history, track history for specific accounts, etc. Plugin allows to have all the account history data without the amount of RAM required in current implementation.
- The number of github issues in regards to new api calls to lookup account history in different ways. Elasticsearch plugin will allow users to search account history quering directly into the database without the need of developing API calls for every case.

Additionally we are after a secure way to store error free full account data. The huge amount of operations data involved in bitshares blockchain and the way it is serialized makes this a big challenge.

## The database selection

Elastic search was selected for the main following reasons:

- open source.
- fast.
- index oriented.
- easy to install and start using.
- send data from c++ using curl.
- scalability and decentralized nodes of data.

## Technical

The elasticsearch plugin is a copy of account_history_plugin skeleton where not needeed stuff was removed saving to objects(to RAM) was replaced by saving into elastic.

Here is how the current account_history plugin works basically:
- with every signed block arraiving to the plugin the ops are extracted from it.
- each op is added to ohi(operation history index) and to ath(account transaction history index).
- both indexes keep growing as new block gets in.
- with the memory reduction techniques currently available the 2 indexes can remove early ops of accounts reducing ram size.

And now what elasticsearch plugin attempts to do:

- with every signed block arraiving to the plugin the ops are extracted from it, just as in account_history.
- create indexes ohi and ath and store current op there just as account history do. this a temp indexation of 1 op only that is done to remain constant with the previous numbers used as id(1.11.X and 2.9.X).
- send ath and ohi plus additional block data and visitor data(lini to visitor data) into elasticsearch(actually we send them in bulks not one by one - replay and bulk links).
- remove everything in the compatibility temporal indexes expect for current operation. This way the indexes always have just 1 item and dont waste any ram.

### Replay and _bulk

As mentioned we dont send to elasticsearch the operations one by one, this is because in a replay the number of ops will be so big that performance will decrease drastically and time to be in sync will be too much.
For this reason we use the elasticsearch bulk api and send by default 5000 lines when we are in replay and downgrade this to 10 lines after we are in sync.

ES bulk format is one line of metadata and the line of data itself, so 5000 is actually 2500 operations we send on every bulk and 10 is actually 5. We could be doing it 1 by 1 after sync but keep going with the bulk will allow to change the values to increase performance sacrificing some real time.

This values are available as plugins options to the user to change, so if a change in ....
[add data and plugin options]

The optimal number of docs to bulk is hardware dependent this is why we added it as an option for changing it at start time.

The name of the index for us will be `graphene`

### Accessing data inside operations


## Installation
