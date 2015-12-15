# Introducing the Tendermint Socket Protocol (TMSP)

# Motivation

After many months of fussing around with blockchain design, 
it became increasingly apparent that a restructuring was in order which would 
provide a greater separation of concerns.

In particular, there are quite clearly two elements of a blockchain: 
the consensus engine (a network protocol facilitating an eventual strict ordering on transactions)
and the application state (account balances, storage, unspent outputs, contracts, etc.).
In fact, these two elements are present in many (all?!) popular internet services today.

What is unique about blockchains, as pioneered by Bitcoin,
in comparison to any other internet application sitting above a consensus engine, 
is the way in which the application state directly incentivizes the consensus, 
through inflation and fees, and the alleged possibilities for economic decentralization therein.

Bitcoin's success, however, be it as it may dependent on economic decentralization,
has marshalled a growing appreciation in finance and industry for 
its other characteristic features: transparency, accountability, and identity via strong cryptography. 

No doubt, those features are further supported, even nurtured, by economic decentralization, 
but they are present too without it, 
motivated and nurtured by the inherent decentralized aspects of the culture of open source itself, 
which is, some may sometimes forget, orders of magnitude bigger than Bitcoin, and growing rapidly.

Hence we have been inspired to take the defining feature of a blockchain -
direct incentivation of the consensus by the application state - and remove it completely,
in order to achieve a separation of concerns between the consensus and the application that 
will give us tremendous flexibility.

Of course, we intend over time to bring back the incentivization layer, 
but to do it in a manner which is motivated more directly by the needs and use cases of the technology's users.

In the meantime, we have an open source platform which can support up to 10,000 transactions per second, 
which is Byzantine Fault Tolerant, which uses state-of-the-art digital signatures,
which has a robust and secure networking layer, 
and which can run applications written in arbitrary programming languages.

Ladies and gentlement (and everyone inbetween), we are pleased to introduce, 
the new Tendermint, and her accomplice, the TMSP.

# TMSP Overview

The tendermint socket protocol (TMSP) is an asyncronous message passing protocol
enabing a consensus engine, running in one process,
to manage an application state, running in another.

If you are of sound mind, the consensus engine will be Tendermint, 
and regardless of your soundness of mind, your application can be written in any programming language.

The only requirements we have are that applications be deterministic and implement the TMSP API.

The API is quite simple, as it boils down to the most basic interface between a consensus engine and its application state: `append_tx`, `get_hash`, `commit`, `rollback`.

That is, the consensus engine will receive new txs in the mempool, 
and run them through the application using `append_tx`.

After a few transactions, the consensus engine can ask for the state hash with `get_hash`.
This way, the only thing consensus has to know about the application is the state root hash,
which goes in the block header, so applications can support light client proofs easily.

When a block is committed, the consensus sends a `commit` message, indicating the application should save its latest state.

Converesely, a `rollback` message tells the application to go back to the latest committed state.

# First Example

Ok, let's do an example.

Make sure you have go installed and put `$GOPATH/bin` on your `$PATH`.

Install the tmsp tool and example applications:

```
go get github.com/tendermint/tmsp/cmd/...
```

Now run `tmsp --help` to see the list of commands.

The `tmsp` tool lets us send tmsp messages to our application, to help build and debug them.

As you can see, the TMSP API has more than the four messages outlined above, 
for convenience, configuration, and information purposes, but it remains quite general.

Let's start a dummy application:

```
dummy
```

In another terminal, run 

```
tmsp echo hello
tmsp info
```

A TMSP application must provide two things:
	- a socket server
	- a handler for TMSP messages

The server may be generic for a particular language, and we provide one for golang in `tmsp/server`.
There is one for python in `example/python/tmsp/server.py`, but it needs work.

The handler is specific to the application, and may be arbitrary, 
so long as it is deterministic and conforms to the TMSP interface specification.

For example, starting the `dummy` application in golang looks like:

```
server.StartListener("tcp://0.0.0.0:46658", example.NewDummyApplication())
```

Where `example.NewDummyApplication()` has methods for each of the TMSP messages and `server` handles everything else.

See the dummy app in `example/golang/dummy.go`. It simple adds transaction bytes to a merkle tree, hashing when we call `get_hash` and saving when we call `commit`.

So when we run `tmsp info`, we open a new connection to the tmsp server, which calls the `Info()` method on the application, which tells us the number of transactions in our merlke tree.

Now, since every command opens a new connection, we provide the `tmsp console` and `tmsp batch` commands.

Running `tmsp console` should drop you in an interactive console for speaking TMSP messages to your application.

Try running these commands:

```
> echo hello
> info
> get_hash
> append_tx abc
> info
> get_hash
```

Similarly, you could put the commands in a file and run `tmsp batch < myfile`.

# Another Example

Now that we've got the hang of it, let's try another application, the "counter" app.

This application has two modes: `serial=off` and `serial=on`.

When `serial=on`, transactions must be a big-endian encoded incrementing integer, starting at 0.

Let's kill the console and the dummy application, and start the counter app:

```
counter
```

Again, the code is just 

```
server.StartListener("tcp://0.0.0.0:46658", example.NewCounterApplication())
```

where the CounterApplication is defined in `example/golang/counter.go`.

In another window, start the `tmsp console`:

```
> echo hello
> info
> get_hash
> info
> append_tx abc
> get_hash
> set_option serial on
> append_tx def
> append_tx 0x01
> append_tx 0x02
> append_tx 0x05
> info
> commit
> info
```

Now, this is a very simple application, but between the counter and the dummy, its easy to see how you can build out arbitrary application states on top of the TMSP. Indeed, `erisdb` of Eris Industries can run atop TMSP now, bringing with it ethereum like accounts, the ethereum virtual machine, and eris's permissioning scheme and native contracts extensions.

But the ultimate flexibility comes from being able to write the application easily in any language. 

We have implemented the counter app in python:

```
cd example/python
python app.py
```

(you'll have to kill the other counter application process). 
In another window, run the console and those previous commands. 
You should get the same results.

Want to write the counter app in your favorite language?! We'd be happy to accept the pull request.

Before continuing, please kill the `python app.py` process.

# Tendermint

Now that we've seen how TMSP works, and even played with a few applications using the `tmsp` tool,
let's run an actual tendermint node.

First, some config files. Save the following in `~/.tendermint`:

genesis.json

```
{"chain_id": "mychain","validators": [{"pub_key": [1,"3D528E80F0635F1ABB992CDDAC653107E216283D4FBD20C8B1E8C2B9ED2A1877"],"amount": 1000}]}
```

priv_validator.json

```
{"address":"773B908D26E254EECD3BB2FE8346396BFABD51AE","pub_key":[1,"3D528E80F0635F1ABB992CDDAC653107E216283D4FBD20C8B1E8C2B9ED2A1877"],"priv_key":[1,"24701186281CD9D97E2324189BCF7714454F6C0C2EB19DE5CF934996C54C4DA83D528E80F0635F1ABB992CDDAC653107E216283D4FBD20C8B1E8C2B9ED2A1877"],"last_height":0,"last_round":0,"last_step":0}
```

config.toml

```
proxy_app = "tcp://127.0.0.1:46658"
moniker = "anonymous"
node_laddr = "0.0.0.0:46656"
seeds = ""
fast_sync = false
db_backend = "leveldb"
log_level = "debug"
rpc_laddr = "0.0.0.0:46657"
```

Now, install tendermint:

```
go get github.com/tendermint/tendermint/cmd/tendermint
```

And run 

```
tendermint node
```

You should see `Failed to connect to proxy for mempool: dial tcp 127.0.0.1:46658: getsockopt: connection refused`

That's because we don't have an application process running, and tendermint will only run if there's an application to talk TMSP to.

So lets start the dummy app,

```
dummy
```

and in anotheer window, start tendermint:

```
tendermint node
```

You should start seeing blocks stream in!

Now you can send transactions with curl requests, or from your browser!


TODO: expose application hash through /status.
TODO: expose TMSP info through rpc
