---

layout:  post

title:  "Decentralized Voting Dilemma"

categories:  decentralization

author:  "Aalok Singh"

---

  

I believe that any system that can be decentralized, should be decentralized. Popular Proof of work DLTs, however, involve broadcasting all successful transactions to the network, so that all participating nodes have a copy of these transactions. Consensus is thus maintained, however privacy is compromised.

  

Privacy and decentralization often struggle to exist in the same system. Sure a public key can't be pin pointed to an individual so easily. But when it comes to scenarios where you have to ensure that one individual gets to perform an action only once, that's where it gets tricky.

  

If democracy were to be really decentralized, we would trust no authority to count votes for us. In an ideal system, citizens would interact with a smart contract to vote and the count of votes would be agreed upon by a consensus algorithm.

  

Electoral voting is a scenario where privacy and decentralization both are paramount requirements, i.e. who *can* vote should not be determined by a central authority (Voting list), AND a voter's vote should be private.

At the same time, the system should be self tallying so that any one authority is not responsible for counting votes.

  

### zk-SNARKS

![Transaction types in Zcash](/assets/images/zcash.PNG)

[Voting using Zero knowledge proofs](https://eprint.iacr.org/2017/585.pdf) are of immense help in a scenario where transactions (votes) need to be private but verifiable. The white paper linked before details how Internet voting can be done using popular zk proof based payment system called [Zcash](https://z.cash/), including anonymous voter registration. 

It is important to note that Zcash relies on zk-snark public parameters to construct and verify the zero-knowledge proofs, these parameters were created via something called as [Zcash ceremony](https://www.youtube.com/watch?v=D6dY-3x3teM). It is up to you to trust these 6 individuals who birthed Zcash. This ceremony was akin to creation of a public-private key pair, where private key was destroyed in the end.

If a voting system is truly anonymous, the voter registration system should also be anonymous, i.e. no personally identifiable information is stored in a 'whitelist' of eligible voters. Also no authority should be able to discriminate as to who makes it to the voter list.
But how do we stop an individual voting multiple times using multiple public keys? This is the dilemma of decentralized voting. 
Even the specs for [Decentralized Identifiers (DID)](https://w3c-ccg.github.io/did-spec) allow for an individual to posses more than one identifier, unless a federated/centralized identity manager is involved.
  

Here, we have to compromise a little bit, having a list of eligible voter addresses at least ensures there are no bogus votes. So such a system, using zk-SNARKS, ensures complete privacy of votes and is self tallying. However the entity that decides/oversees which voters make it to the voters list can still be biased.

[This](https://github.com/stonecoldpat/anonymousvoting) project demonstrates a self tallying voting system on Ethereum where votes are private. It does involve an election admin preparing a list of eligible voters.

### Ring Signatures

![Ring signatures](/assets/images/Ring-signature.svg)

Ring signatures are great for decentralized voting with voter anonymity. They allow for signatures to be endorsed by a group of keys without revealing which particular key is the signer. Popular cryptocurrency [Monero](https://ww.getmonero.org/) uses ring signatures. 
Two great variants of ring signatures are linkable and traceable ring signatures.

#### Linkable ring signatures

The property of linkability allows one to determine whether any two signatures have been produced by the same member (under the same private key). The identity of the signer is nevertheless preserved.

#### Traceable ring signature

In Traceable ring signatures, the public key of the signer is revealed, if they issue more than one signatures under the same private key. This can control bogus voting nicely, thus making traceable ring signatures a highly viable method to create a decentralized and anonymous e-voting system. 
[Here's](https://arxiv.org/ftp/arxiv/papers/1804/1804.06674.pdf) a good research paper on this.