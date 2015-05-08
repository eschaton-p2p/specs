# Extensible Scalable Coopetitive High Availability Trade Optimization Network: ESCHATON


> I am the Eschaton; I am not your God.

> I am descended from you, and exist in your future.

> Thou shalt not violate causality within my historic light cone. Or else.

**Singularity Sky** by Charles Stross

TODO:
* References
* More technical details
* Diagrams, tables, code and pseudocode

This is a very rough draft in the spirit of [Readme Driven Development][RDD]. Much more work remains to be done in terms of design and specification, as well as actual implementation.
[RDD]: http://tom.preston-werner.com/2010/08/23/readme-driven-development.html
## Motivation

Bitcoin's blockchain is a great timestamping and consensus mechanism. However, it has the following disadvantages:

* Instant payments suitable for point of sale require either 0-confirmation transactions, which are vulnerable to double spend, or centralization-increasing green addresses
* Insufficient blockchain scalability for trillions of automated micropayments per day
* Traceability of payments on the blockchain can reduce privacy for users
* Blockchain use for advanced contracts is also subject to scalability limitations
* Trustless exchanges - including between Bitcoin, altcoins, colored coins, and in the future, side chains - require multiple on-chain transactions, reducing scalability and speed of trading

### Project Goals

This project aims to build a network primarily on top of Bitcoin and eventually side chains, though it's simple to add support for other similar cryptocurrency networks. The intent is to move as much trade as possible off the blockchain for privacy and scalability while maintaining a no- or minimal-trust environment. Support for advanced contracts that can be maintained off the blockchain is described. The network will have the following features:

* Friend-to-friend, instant, off-blockchain, trustless payments of arbitrary size with optional escrow, composable into distributed inter-blockchain instant payments and exchanges between arbitrary parties
* Friend-to-friend, instant, off-blockchain, trustless bets composable into distributed derivatives and prediction markets, crowdfunding with performance bonds, insurance, and more
* Metered network services bought using micropayments, composable into meshnets, mixnets, microgrids, social networks, grid computing networks, and more
* A modular design permitting alternative implementations of various services as new innovations occur, such as potentially improving the DHT with a proof of storage based side chain

### Overall Design

The network will have a friend-to-friend financial structure, while the connectivity structure may be more freeform. 

The financial connections are minimal trust contracts settled on a blockchain (Bitcoin, a side chain, or an alt coin's blockchain). However, they're maintained for as long as years off chain, minimizing blockchain resource usage.

The communication connectivity can be varied: through the Internet, local Wi-Fi, Bluetooth, cellular networks, etc. The aim is to create a survivable, sustainable, scalable and highly available system to prevent interruptions in communication and trade while guarding privacy and reducing censorship.

## Trade Channel Management

### Traditional Off Chain Trade Contract

The traditional off chain trade contract functions as [previously described by Mike Hearn][MICRO]. A 2-of-2 multisignature output is created, and a refund transaction with nLockTime in the future, a nominal fee, and a version number of 1, is signed and countersigned.
[MICRO]: https://en.bitcoin.it/wiki/Contracts#Example_7:_Rapidly-adjusted_.28micro.29payments_to_a_pre-determined_party
The parameters of the contract can be modified to include risk deposits, maximum payment size, and other rules to enforce incentives to play by the rules. These rules are enforced by each party to the contract in refusing to countersign any version of the refund transaction which doesn't follow the rules.

As an example, consider a situation where Alice and Bob would like to open a 100 mBTC trade channel between themselves, with each contributing half to start. The first thing they would do is create a contract transaction:

![Contract Transaction](http://aakselrod.github.io/contract.svg "Contract Transaction")

The output of the transaction is set up so that it's only possible to redeem if both Alice and Bob agree to do so together. Note that nLockTime and input versions are set per Peter Todd's [suggestion](http://sourceforge.net/p/bitcoin/mailman/bitcoin-development/thread/CAAS2fgSHi3Ms17JSkxirE5RkH4BEg6GP5SKLuCj6c39k0OQnhg@mail.gmail.com/) to prevent intentionally orphaned blocks as the block subsidy gets reduced. When any input version number is lower than 0xFFFFFFFF and nLockTime specifies a block or time in the future, the transaction cannot be added to a block until nLockTime expires (or rather, real time catches up with nLockTime).

In order to prevent the funds from being permanently locked should one party disappear, Alice and Bob assign a maximum lifetime to the trade channel (in this example, until block 400,000), and create a refund transaction as follows:

![Initial Refund Transaction](http://aakselrod.github.io/refund-initial.svg "Initial Refund Transaction")

Both Alice and Bob sign off on the refund transaction, which specifies that both parties get their money back at block 400,000 minus fees. If a finalized version of the refund transaction that uses the contract output isn't confirmed before block 400,000, this refund transaction becomes valid to include in a block. In case either of the parties disappears right after the contract transaction is broadcast, the refund transaction can permit the other party to get their money back at nLockTime expiration.

From this point, trade can take place between Alice and Bob off the blockchain by creating new versions of the refund transaction. In some situations described below, the new version of the refund transaction will need to have an earlier nLockTime to ensure that it can be mined before a previously-created refund transaction becomes valid. Off-chain trade between Alice and Bob can continue based on the contract until the version of the refund transaction with the earliest nLockTime becomes valid for inclusion in a block.

There are caveats regarding transaction malleability between the time the refund transaction is generated and the contract transaction is confirmed. There are also caveats regarding the broadcast of the contract transaction before both peers have a fully co-signed copy of the refund transactions. Both of these have mitigations which were excluded from the above for simplicity's sake but will be implemented in the software.

This structure has the following advantages:

* Payments and other trades over the off-chain contract can be executed instantly without waiting for confirmation once the contract transaction itself has been confirmed
* Temporary blockchain issues such as forks, rollbacks, drops in hashing power, etc. do not reduce the effectiveness of trade over already-confirmed contracts
* No trust is required to prevent losses from one peer running away with the other's money or getting hacked or shut down

### Traditional Off Chain Payment

To begin making off chain payments in one direction, a new version of the refund transaction is signed with an earlier nLockTime, a slightly higher fee, and an incremented version number. The portion of the output signed over to the payee is increased, and that signed over to the payer is decreased, by the amount of the first payment.

Continuing the example from the previous section, Alice wants to pay Bob 1 mBTC over their new trade channel. She creates and signs a new version of the refund transaction, and sends it directly to Bob: 

![Refund Transaction After Payment](http://aakselrod.github.io/refund-payment.svg "Refund Transaction After Payment")

The new version has four differences from the original refund transaction:

* An earlier nLockTime permits the new version to be mined before the old version, thus overriding the old version
* A higher fee makes miners more likely to mine the new version than the old in case the new version doesn't get mined before the old version becomes valid for inclusion in a block (in reality, the fee increment can be as low as a single satoshi)
* A higher version number for the input permits transaction replacement in full nodes' mempools in case the original design for transaction replacement is re-enabled
* Alice's share of the refund is decreased by 1 mBTC, while Bob's is increased by the same amount

To make a subsequent payment in the same direction, it is only necessary to increase Bob's share of the refund and decrease Alice's. Alice can assume that Bob will only broadcast the version of the transaction with the highest possible share of the refund going to him. Thus, increasing the fee, incrementing the version, and decrementing nLockTime is only required when switching payment directions.

### Off Chain Payment with Escrow

An [escrowed payment][TXCHAIN] is similar to a traditional payment. The difference is in the new version of the refund transaction: each output is encumbered not just with a required signature, but the same required hash preimage for both outputs. In this way, the party that knows the preimage can't redeem its output without enabling the other party to do the same.
[TXCHAIN]: https://github.com/aakselrod/libtxchain-java
The payment is committed when the payer tells the payee the preimage to the required hash. To roll back the the payment, a traditional off chain payment is sent back in the reverse direction, complete with incremented version number, decremented nLockTime, and increased fee.

To continue the example, Bob wants to pay Alice 1 mBTC but would like to guarantee delivery by way of escrow. Bob generates a random number *P* and hashes it into *hash(P)*. He then signs the transaction below and sends it to Alice:

![Refund Transaction After Escrow](http://aakselrod.github.io/refund-escrow.svg "Refund Transaction After Escrow")

Since the payment flow is being reversed (instead of going from Alice to Bob, it's now going from Bob to Alice), Bob must decrement nLockTime, increment the version number, and increase the fee. The outputs for both Alice and Bob are encumbered not just with their respective public keys, necessitating a signature from each to redeem his/her output, but also with *hash(P)*, necessitating the knowledge (and public revelation) of *P*.

When Alice delivers the goods, Bob tells her *P*, which means that she can redeem her output if she were to broadcast the new version of the refund transaction. If the refund transaction gets broadcast without Bob telling Alice *P*, she can learn it when he reveals it by redeeming his own output from the contract transaction. If Alice doesn't deliver the goods, Bob does not reveal *P* to Alice, and refuses to do any additional trade over the channel until Alice rolls back the escrow by sending a new (non-escrowed) payment for the 1 mBTC back to him.

### Off Chain Bet

With a [bet][BET], an oracle publishes a signed list of possible outcomes and an associated hash for each. Upon the occurrence of the event being wagered on, the oracle publishes the preimage for the hash corresponding to the outcome. One refund transaction similar to an escrow refund transaction is created for each possible outcome (except an optional default case). The new refund transactions must have an earlier nLockTime, higher version number, and higher fee than the most recent redeemable refund transaction. Cheating by the oracle can be detected if there exists any signed statement from which two or more conflicting preimages are known.
[BET]: https://en.bitcoin.it/wiki/Off-Chain_Betting
The parties betting then accept the version of the refund transaction containing the hash for the published preimage as current, committing the outcome of the bet. When the default case occurs, the most recent redeemable refund transaction becomes current. With this scheme, counterparty risk is virtually eliminated by way of the risk deposit used to establish the channel - all bets are fully funded when agreed.

For example, assume Carole, an oracle of good repute, generates two random numbers: *P1* and *P2*. She hashes the numbers into *hash(P1)* and *hash(P2)*, and signs/publishes a statement saying "Upon the occurrence of event *E*, I will publish the preimage of *hash(P1)* if outcome 1 happens, and the preimage of *hash(P2)* if outcome 2 happens."

Alice and Bob decide to make a bet of 10 mBTC at 50/50 odds. Alice bets on outcome 1 and Bob bets on outcome 2. They create and sign the following two new versions of a refund transaction together:

![Refund Transaction - Bet Outcome 1](http://aakselrod.github.io/refund-bet1.svg "Refund Transaction - Bet Outcome 1")
![Refund Transaction - Bet Outcome 2](http://aakselrod.github.io/refund-bet2.svg "Refund Transaction - Bet Outcome 2")

When event *E* occurs, Carole publishes *P1* or *P2*, depending on the outcome of *E*. This makes either the first or the second version of the refund transaction redeemable; if the other transaction were published, all of the money would be gone forever. If somehow both P1 and P2 end up being published, that would serve as proof that Carole cheated and would ruin her reputation as an oracle. Alice and Bob cannot cheat except by colluding with Carole.

It's also possible to adjust the amount being wagered or cancel the wager altogether by creating a new bet (or other refund transaction) with an earlier nLockTime, higher fee and higher version number for the input. There are additional caveats regarding ensuring that both versions of the bet transaction get signed and cosigned; the mitigation is omitted above for simplicity but will be implemented in the software.

## Network Service Primitives

### Distributed Database/Hash Table with Map/Reduce

In order for nodes to be able to find information, a distributed database similar to a DHT is required. Nodes must be able to answer the following queries according to best effort:

* Preimage for a hash, for hash types including SHA256, double SHA256, SHA256+RIPEMD160, and other types related to altcoins
* Retrieval and proof of inclusion/exclusion of elements in a Merkle or Patricia tree (or multiple trees, for searching through a blockchain)
* Retrieval of defined sets of elements from Merkle and Patricia trees
* Search for signed statements by signing key or sub-element of statement (with statements having a defined format such as protocol buffers or bencode and structured as a potentially nested dictionary-like object)
* Queries that function like map/reduce, with the reduced output being the root of an authenicated, Merkle tree-like (but containing additional summary data and not necessarily balanced) data structure

Queries should be able to test candidate results against Bloom filters, explicit values, wildcards, and other arguments to multiple operators, and to be able to combine all of these methods using boolean logic. Use cases for these queries will be described in the sections below. For any query that returns no results, a node can query its neighbors. All queries are paid for using micropayments over a trade channel and price can be negotiated based on any criteria (number of results returned, size of returned data, maximum price per result or KB, etc.) and forwarding of queries to neighbors should take this cost into consideration. One of the most important applications of the DHT will be to look up transactions in the blockchain.

### Distributed Publish/Subscribe

Similarly to a DHT, a pub/sub facility is useful to watch for new data. Neighbor relationships for pub/sub traffic are published in the DHT as signed statements, and subscribers can use the data about those relationships to send subscription messages to their own neighbors. Cost can again be negotiated between nodes, and each node must take into account its own cost of acquiring the requested data during negotiation. This can be used for new block/transaction alerts, online broadcasting/multicasting, etc.

### Distributed Timestamping Service

There are two ways to timestamp: the first is to prove that something existed after a certain time, and the second to prove that something existed before a certain time. A combination of the two can prove that something existed during a certain time.

To prove that something was created only after a certain time, the hashes of the latest several blocks (in case of fork) should be included in the data. For example, a signed statement that wants to specify its validity as starting just after block 300,000 could include the hashes to blocks 299,995-300,000 to prove the earliest time it could have been created.

To prove that something was created before a block, it must be included in the block. For this network, nodes provide a timestamp facility which accepts a hash of the data to be timestamped. The node then adds it to a Merkle tree containing all of the other hashes it has been asked to timestamp, and eventually broadcasts a provably unspendable OP_RETURN transaction with the root hash as the data. The node then returns the Merkle branch from the block's Merkle root to the transaction, and the Merkle branch from the transaction's Merkle root to the data hash, back to the original requester. Note that this also permits additional consolidation as nodes can, instead of broadcasting/committing their Merkle root directly into the blockchain, send their Merkle root as data to an upstream node. Then, the upstream Merkle tree becomes technically unbalanced when fully unrolled, but has the advantage of consolidating timestamped data from many sources, lowering transaction fees and blockchain resource utilization.

### Service Directory

Each node should publish a directory of the services it provides. The specification for services should be self-describing to the maximum degree possible.

## Advanced Services

This project aims to develop Raspberry Pi versions of these advanced services for cheap, low-powered, mass-producible meshnet nodes. This project also aims to develop corresponding client capability in an Android app, as well as an Android app with limited, configurable service functionality for users who want to run some services on their own portable devices. Commercial potential exists in more scalable implementations for "supernodes" of various services. This project will gladly accept open source patches for greater functionality, scalability, and platform compatibility; for example, running on a VPS, multicore dedicated server, or local datacenter-sized cluster. However, the focus is on making the technology accessible in remote locations and in developing nations, and using local, renewable, or cheaply mass-producible resources.

### Distributed Instant Payments and Exchanges

Distributed instant payments take place over chains of trade channels using escrowed payments. Each node pays the next node in the chain using an escrowed payment, with each payment using the same hash. Once the entire path has allocated funds to the payment, the hash preimage is revealed to all nodes in the chain, committing the payment across the entire chain. Escrow is built in, as the revelation of the hash preimage can be delayed until performance by the ultimate payee is verified. Rollbacks are performed one by one, from the last node to get an escrowed payment back to the first.

Routing information for trade channels that can be used for this purpose can be stored in the DHT. Fees can be charged based on a number of factors such as how long a trade channel will need to be reserved, how big a payment is, number of payments to make, etc. A tradeoff exists between low cost (fewer well-connected nodes acting as competitive transaction processors, shorter payment channel chains, and lower fees) vs. decentralization (more locally-connected nodes, longer chains, and higher fees). Both models could thrive: well-connected nodes would be more vulnerable to regulation, while locally-connected nodes would be better for transferring funds in a more private manner. The choice will be made by individual nodes who choose their interconnection partners.

Since trade contracts can be created on any blockchain with support for scripting similar to that of Bitcoin, the distributed payment network can include altcoins and side chains. Trade contracts can also be established using colored coin outputs, permitting off blockchain trade of colored coins. It then becomes possible for the distributed payment network to act as a distributed cryptocurrency exchange as well. For example, a customer could pay with Litecoin and the merchant could automatically get paid with Bitcoin.

### Distributed Derivatives/Prediction Markets

Each node that participates in a distributed derivatives/prediction market must maintain an internal order book and bet channels with multiple trading partners, acting as a bookmaker. Market makers would seek to create connections to as many people as possible and maintain their positions to be as neutral as possible, locking in arbitrage profits and hedging against losses. Then, bet price/probability movements propagate from market actors who are hedging and speculating through market makers and to the rest of the market. Much like the instant payment scheme, there's a tradeoff between cost/efficiency and decentralization.

This project aims to implement a basic distributed derivatives-based hedging client for Android and a specification for connecting to market makers in order to permit merchants to keep accounts in their preferred currency regardless of the currency being used for payments. Patches will be accepted for additional functionality.

### Distributed Identity/Reputation

If identities are public keys, then their reputation involves signed statements by other keys listing some or all of the following information:

* Subject, or the entity (public key, file, etc.) about which the statement is being signed
* Metadata about the subject (ratings for trustworthiness, taste, timeliness, etc.)
* Timestamp of rating and number of blocks the rating is valid

In addition, the signed statements could be combined to include multiple subjects, multiple attributes, multiple instances (for example, a visit to the same business on multiple occasions). The signed statements would be accessible via DHT, and should also be cached for best performance and lowest cost. Each node should cache at least signed statements about itself that it can present as credentials to others and signed statements about others that its own key and neighboring keys have signed. The nodes should also cache the same data for several hops away in each direction, and for its most frequent trading partners. This will let nodes discover relationships to each other in the web of trust more easily by searching for intersections in the sets of signed statements.

Reputation can also be indirectly applied to other things, such as files. For example, if a client would like to download a song but wants to know whether it's a high quality, complete copy, it can search for signed statements from public keys it trusts, directly or not, about the file. The attribute/rating pair, like a key/value pair, can describe any other metadata in addition to trust/quality ratings.

### File Sharing and Streaming

With the DHT structure, each node can act as a file sharing and streaming service in concert with the rest. Instead of tit-for-tat traffic allocation that must keep track of availability of each chunk in order to avoid bottlenecks, the profit motive will cause local nodes to cache locally favored content intelligently, permit local clients to subscribe to streams cheaply, and stream existing content from the cheapest acceptable source. In essence, a peer-to-peer YouTube with broadcast support where each broadcast is likely to be indefinitely available.

Since streaming media across the network would cause the media to be stored in the DHT, a method of funding content creation using the below scheme for off-chain crowdfunded performance bonds may be more effective than current business models in allowing for inevitable piracy.

### Meshnets, Mixnets, and Virtual ISPs

To pay an access point for metered bandwidth or access time, it's possible to create a trade channel with the AP and directly pay for each unit of bandwidth used over traditional micropayments. It's also possible to create a trade channel with just one node, and pay through that node to any AP which has a direct or indirect connection, thus turning that node into a virtual ISP.

In a meshnet-style environment, each permanent or semi-permanent installation should have trade channels with all of its permanent or semi-permanent direct neighbors. Ephemeral clients can pay any AP in the meshnet through any node to which they have connectivity for bandwidth utilization and other network services (locally-relevant entries from the DHT which are cheaper to cache locally than request from a remote peer all the time, for example). These can interconnect over the Internet or other long-distance media, bridging both connectivity and trade gaps.

Onion routing can also be funded with trade channels. If a node wants to anonymously connect to another node, it can map out a path using routing information in the DHT (where the DHT replaces the Tor directory authorities and can include reputation info for each node along the way in regards to reliability, bandwidth, etc.). Then, it sends the first hop enough money to cover the entire packet transmission, instructions to forward most of that money onto the next hop in the path, and an encrypted packet with instructions to the next hop. The next hop would decrypt the instructions, which would tell it to forward the inner encrypted packet to the subsequent hop along with most of the money it received. This would continue until the innermost encrypted packet reaches its destination, which can decrypt it and send a response back the same way. It is also possible to route payments, not just traffic, this way.

## Future Speculation

### Outsourced Storage

Much like Dropbox and Wuala permit outsourced storage, this service can be commoditized with an open API. Client-side encryption, regular requests for proof-of-stored-data via a Merkle branch, and client-defined redundancy can make private data in the public cloud a reality. This can even apply to publicly-accessible files and other data as it permits clients to ensure that their public data doesn't get deleted from the DHT for non-use. As blockchain technology continues to evolve, proof of storage blockchains may become possible and could be used for this feature.

### Crowdfunding and Performance Bonds

One narrow application of a prediction market is a crowdfunded performance bond using a construct similar to a dominant assurance contract. To make this work, an oracle would commit to reveal the preimage to a hash when a project is completed, or reveal the preimage to another hash if the project isn't completed to its satisfaction by a specified time. Those working on the project would bet on its completion, while those wanting to fund the project bet against its completion. If the project isn't completed, the funders win their bet and make money; if the project is completed, those working on the project win their bet and get paid for their work.

### Decentralized Supermarkets and Online Markets

The ability to publish signed statements into the DHT permits producers and merchants to let the world know about a product they make available, creating an Alibaba-like functionality for the DHT. Wholesalers can buy large lots directly from the manufacturers and use outsourced shippers to deliver those large lots to outsourced warehouses, and then resell to individual buyers or, in smaller lots, to local stores.

It is desirable to separate the DHT records for products vs. the records for offers; this way, it's possible to use the reputation system to evaluate the product and the merchant selling the product separately as well as provide metadata for offers such as terms of purchase, fulfillment details, etc.

### Microgrids and Other Distributed Utilities

In a local environment where each house has well, septic, power generation and storage, rainwater collection and storage, etc., it's possible to create distributed utility grids. Each house would be connected through automated shutoffs and flow meters to its neighbors, as well as through direct trade channels. Then, each house could buy electricity or water from its neighbors or sell to its neighbors using trade channels to pay each other. Electricity, water, sewage and money could be routed around entire neighborhoods as necessary. Most of this can be built with existing off-the-shelf components and could make locally built, sustainable, decentralized and survivable infrastructure in remote parts of the developing world.

### Semantic Web

Signed statements can also be about hashes of documents in RDF or other formats of knowledge representation. This permits the semantic web to be stored in an instantly-queryable, scalable, globally available distributed database with built in reputation that can be applied to specific content.

### Machine Learning

The semantic web and the reputation network can be mined, and the results themselves can be stored in the DHT. This can create the basis for distributed machine learning to make reputation more effective, permit users to automatically mine the DHT for recommendations similar to Netflix but applicable to any information, and enable crowdsourced scientific research.

### Social Networking

BitMessage style messaging can be propagated through the network along the most desirable (private and/or cost-efficient) paths. The messages can contain microblog-style updates, including hypertext-like content that can point to other content stored in the DHT. Because the infrastructure is publicly available, distributed, and open-source, users can innovate on top of it without having to worry about data lock-in similar to that of proprietary companies. For example, unlike Facebook and Twitter, users will be able to finely control what they see and what they don't: instead of Facebook's choices for "top stories," users will be able to set their own criteria.

### Distributed Source Control

Source code commits to a repository are a very good fit for data which can be stored in a DHT. Git already uses hashes to identify commits; tracking the relationships of commits to each other can also be added to signed statements. Adding distributed identity and reputation of developers, testers, and users of software can enable a decentralized replacement of websites like GitHub, etc. It would be possible for systems to update themselves from source a la Gentoo's portage system, but to fetch that source from the DHT rather than from a centralized repository, and use the reputation system along with the system configuration to determine what code to download, compile and run. In this way, it would be possible to automate adoption of new, innovative software to make service provision over the network more effective, efficient, and profitable. The same can apply to bitstreams for FPGAs or kernels for GPUs, permitting automatic adoption of profitable innovations.

### Education

The network's above listed features will make it possible to crowdfund, develop, maintain, and host open source, multimedia educational materials. The reputation facility will allow students to pick the materials that best fit with their learning style, language, and other needs. The global availability of content on the network will permit anyone to access the materials, and education can be commoditized in the same way Khan Academy, Udemy, and various open source textbooks and other materials have begun to do.

### Grid Computing for Scientific Research

The instant, distributed micropayment and communication structure designed into this network will be a great fit for replacing or augmenting BOINC-style grid computing software with economic incentive as well as permit independent researchers to rent computation time. This could help empower crowdsourced research in multiple scientific and engineering fields.
