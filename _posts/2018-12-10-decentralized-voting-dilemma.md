---
layout: post
title: "Decentralized Voting Dilemma"
categories: decentralization
author: "Aalok Singh"
---

I believe that any system that can be decentralized, should be decentralized. Popular Proof of work DLTs, however, involve broadcasting all successful transactions to the network, so that all participating nodes have a copy of these transactions. Consensus is thus maintained, however privacy is compromised.

Privacy and decentralization often struggle to exist in the same system. Sure a public key can't be pin pointed to an individual so easily. But when it comes to scenarios where you have to ensure that one individual gets to perform an action only once, that's where it gets tricky.

If democracy were to be really decentralized, we would trust no authority to count votes for us. In an ideal system, citizens would interact with a smart contract to vote and the count of votes would be agreed upon by a consensus algorithm.

Electoral voting is a scenario where privacy and decentralization both are paramomunt requirements, i.e. who can vote should not be determined by a central authority (Voting list), AND a voter's vote should be private. 
At the same time, the system should be self tallying so that any one authority is not responsible for counting votes.

## zk-SNARKS
[Voting using Zero knowledge proofs](https://eprint.iacr.org/2017/585.pdf) are of immense help in a scenario where transactions (votes) need to be private but verifiable.

Zero knowledge proofs solve one problem, the votes are private, but anonymity is still a problem.
If a voting system is truly anonymous, i.e. no personally identifiable information is stored in a 'whitelist' of eligible voters, how do we stop an individual voting multiple times using multiple public keys? This is the dilemma of decentralized voting.

Here, we have to compromise a little bit, having a list of eligible voter addresses atleast ensures there are no bogus votes. So such a systemm, using zk-SNARKS, ensures complete privacy of votes and is self tallying. However the entity that decides which voters make it to the voters list can still be biased.

[This](https://github.com/stonecoldpat/anonymousvoting) project demonstrates a self tallying voting system on Ethereum where votes are private. It does involve an election admin preparing a list of eligible voters.

