---

layout:  post

title:  "My conversations with LLM - How does TLS work?"

categories:  Cloud and Networking

author:  "Aalok Singh"

---

Some days you just wake up curious. Today when  [this](https://www.youtube.com/watch?v=WxxHH_4vKh4)  video showed up in my feed, I had some questions about TLS protocol.

This took my down a rabbit hole of how Symmetric Key Derivation algorithms work. I'm sharing a succinct and paraphrased version of that conversation with gpt-4o:

* * * * *

### Q: You are an expert in TLS protocol. Which keys are used to encrypt my HTTPS traffic?

-   Neither the public key nor the private key (asymmetric keys used to sign and validate certs) directly encrypts the actual traffic.
-   They are used during the  handshake  to  establish a symmetric key.
-   The  symmetric session key  is then used to encrypt the actual data.

* * * * *

### Q: Can you dive deeper into how symmetric session keys are used to encrypt traffic?

... Yes,  the actual traffic  *is*  encrypted, but  not with public/private keys. It's encrypted using a  symmetric encryption algorithm  (like AES) after the TLS handshake is complete.

Let's dive deeper into how this works:

### 1\. Purpose of Public/Private Keys (Asymmetric Encryption)

-   Used  only during the handshake  phase of a TLS connection.
-   Enables:

> Why not use public/private keys for actual data encryption?

-   Performance: Asymmetric encryption is computationally expensive and slower.
-   Security Best Practices: Symmetric encryption is faster and more suitable for large data volumes like website content.

### 2\. Actual Traffic Encryption (Symmetric Encryption)

Once the handshake is complete:

-   A  shared symmetric key  (or a set of keys) is agreed upon.
-   From then on,  all HTTP traffic is encrypted with this symmetric key, using algorithms like:

### 🔑 How Is the Symmetric Key Established?

It depends on the  key exchange algorithm:

### Modern TLS (using ECDHE):

-   Both sides generate ephemeral key pairs (private + public).
-   They exchange  public  parts.
-   Each side uses their  private key  + the other's  public key  to derive the  same shared secret  (via elliptic curve math).
-   This shared secret is passed through a  Key Derivation Function (KDF)  to generate:

* * * * *

### Q: What are the characteristics of symmetric keys?

### 🛡️Summary

-   🔁 Same key is used for both encryption and decryption
-   ⚡ Very fast and efficient
-   🔢 Typically 128--256 bits (longer = more secure)
-   🔒 Must be securely exchanged or derived
-   🌐 Used in TLS to encrypt actual traffic after handshake
-   🔧 Common algorithms include AES and ChaCha20

* * * * *

### Q: How difficult is it to get hold of session keys and why is it difficult?

### 🛡️Summary

-   ☠️ Session key compromise is fatal --- all traffic can be decrypted
-   🛡️ Extremely difficult to obtain the session key
-   🧠 Session key is never sent, is ephemeral, exists only in RAM, and is derived via secure ECDHE math
-   🚫 Attacker's best chances are malware, memory scraping, or exploiting bugs (which are rare and detectable)

* * * * *

### Q: You mentioned earlier that both parties use the same session key. However you also say that keys are never actually transmitted. How is that possible?

### ✅ Short Answer:

-   Yes,  both client and server use the  *same session key(s)*.
-   But  this session key is  never transmitted over the network.
-   Instead, both parties  independently derive the same session key, using secure mathematical operations during the  key exchange phase.

### 🔑 Summary: Same Key, But Not Shared Explicitly

-   🔑 Symmetric session key is the same on both client and server
-   📵 It is never transmitted over the network
-   🧮 Each party derives it using ECDHE math on exchanged public keys and their own private key
-   🔐 Security benefit: even if someone intercepts all traffic, they can't compute the key without a private key

* * * * *

### Q: How is it possible that same exact secret is generated when a public key and a different private key (which could be anything) goes into a mathematical function?

That's an excellent and deep question --- and it's the heart of  Elliptic Curve Diffie-Hellman (ECDH)  key exchange.

It  does  seem strange at first:

> How can  two parties, each using a  different private key, and the  other's public key, end up computing  the exact same secret?

Let's walk through it  mathematically but intuitively.

* * * * *

### 🔐 TL;DR:

-   The  shared secret  is the result of a  scalar multiplication  of elliptic curve points.
-   Due to the  commutative  nature of the math (like a × b = b × a), both parties arrive at the  same result  independently.
-   The result is the same because of how  elliptic curve point multiplication  works.

### 🔐 Final Insight

> ECDH works because both parties use their own private key + the other's public key to derive the same point --- and the math ensures that the result is identical.

They don't need to know each other's private keys --- they just rely on  shared public information  and the hard math problem underneath.

*--- Conversation Over ---*

That last bit about  ECDH  algorithm was fascinating! I hope you got to learn something from this as well.