---
layout: post
title:  "Tornado cash Governance takeover"
date:   2023-12-25 16:05:11 -0200
categories: blog
layout: post
---

On May 20 Tornado Cash Governance was hacked when an attacker was able to get the full control of governance. In this post I explained how the attacker tricked the community and took control of the Tornado cash goverance using a seemingly ordinary governance proposal.




<!--more-->

<!-- <img class="img img-responsive" alt="pong-almost-from-scratch" height="600" width="500" style="display: block; margin: 0 auto;" src="https://academy-public.coinmarketcap.com/srd-optimized-uploads/475169d126aa45d18e22c680910f5560.jpeg"/> -->

<br />
<br />



## Tornado Cash - Overview

Tornado Cash is a cryptocurrency smart contract mixer, facilitating users to deposit digital assets using one address and  withdraw them using another address, without any traceable link between the two addresses. 

<br />
Tornado Cash is governed by community using a Goveranace mechanism. Users can submit their proposals to the goveranace smart contracts and $TORN token holders can vote for approval or rejection of the proposal. The proposal must be submitted as a verified smart contract, and if approved by the community, the code in the proposal contract will be executed by governance contract via delegate-call. 

<br />
In this carefully crafted attack the attacker made a seemingly ordinary governance proposal, which was claimed to be the same as an earlier proposal that has been accepted by the governance. But the attacker included an [`emergencyStop`](https://twitter.com/samczsun/status/1660012960176279552/photo/2) function in the proposal contract which allowed them to selfdestruct the contract.  Upon passing the vote , the attacker took advantage of the selfdestruct to replace the existing proposal contract and deploy a new malicious one at the same address.

<br />
In this post, we'll delve into how the attacker leveraged the newly introduced selfdestruct function to replace the original proposal logic with malicious code. The attacker used a trick of combining CREATE/CREATE2 with selfdestructable proposal contract to create a contract with the same address but with different bytecode.
<br />
Before diving into the technical aspects of the attack, let's first understand how CREATE/CREATE2 works.

<br />
<br />
<br />





### CREATE

`CREATE` is used to deploy a smart contract where the address of the deployed contract will be depend on the sender and the nonce of the sender.

```
new_address = hash(sender, nonce)
```

| ----------|---------------------------------------------------- |
| `sender`    |  The sender’s own address                           |
| `nonce`     |  Every account has an associated nonce: for regular accounts it is increased on every transaction, while for       contract accounts it is increased on every contract creation |

This means it is possible to predict the address where the next created contract will be deployed, but only if no other transactions happen before then.  
  
<br />
<br />



### CREATE2

The whole idea behind `CREATE2` opcode is to make the resulting address independent of nonce. Regardless of the nonce of an account, it will always be possible to deploy the contract at the precomputed address.

`CREATE2` calculate the address of the deployed contract as below:

```
new_address = hash(0xFF, sender, salt, bytecode)
```

| ----------|---------------------------------------------------- |
| `0xFF`      |  a constant that prevents collisions with CREATE    |
| `sender`    |  The sender’s own address                           |
| `salt`      |  A salt (an arbitrary value provided by the sender) |
| `bytecode`  |  creation code of the contract to be deployed       |

`CREATE2` guarantees that if a sender ever deploys a smart contract using a particular bytecode and salt the `new_address` will always be deterministic regardless of the nonce of the account.  


<br />
<br />


## The Attack 



Now that we understand how CREATE and CREATE2 works let's see how the attacker was able to update the original proposal logic with malicious code. In other words how the attacker was able to deploy a new contract in place of the old one under the same address.


{:start="0"}
0. First the attacker deployed a contract let's call it  `Deployer Factory`.
    <br />
    
1. Then created a new contract called the `Deployer` contract from `Deployer Factory` using CREATE2.
    <br />

2. Then created the Proposal contract from Deployer using CREATE opcode. While the code purportedly used the same logic as a previous proposal, the attacker had added an extra function called `emergencyStop` which allowed them to selfDestruct the contract. The attacker used the name `emergencyStop` to hide the intent. 

<br />



![How can the attacker deploy different contracts at the same address](/assets/img/ta1.png)
{:refdef: style="text-align: center;"}
*source: [https://youtu.be/whjRc4H-rAc](https://youtu.be/whjRc4H-rAc)*
{: refdef}

 At this point the attacker waited for the proposal to be voted. Upon passing the vote , the attacker took advantage of the selfdestruct to replace the existing proposal contract and deploy a new malicious one at the same address. And then executed the proposal with malicious code.

![How can the attacker deploy different contracts at the same address](/assets/img/tornado1.png)
{:refdef: style="text-align: center;"}
*source: [https://youtu.be/whjRc4H-rAc](https://youtu.be/whjRc4H-rAc)*
{: refdef}
<br />


 If the attacker deploys a new malicious proposal contract from `Deployer` using CREATE at this point they would've get a different address for the proposal contract because the nonce of `Deployer` is now different.

<br />

![How can the attacker deploy different contracts at the same address](/assets/img/ta2.png)
{:refdef: style="text-align: center;"}
*source: [https://youtu.be/whjRc4H-rAc](https://youtu.be/whjRc4H-rAc)*
{: refdef}

<br />
<br />

 So the attacker was somehow able to reset the nonce of the `Deployer` contract to deploy the malicious code at the same address as the original proposal code. Let's see how they did it.

 <br />

{:start="3"}
3. First the attacker destructed the orginal proposal contract using `emergencyStop` funtion.
    <br />
    <br />

4. Then destroyed the `Deployer` contract which resets the nonce of the `Deployer` to zero.
    <br />

    ![How can the attacker deploy different contracts at the same address](/assets/img/ta5.png)
    {:refdef: style="text-align: center;"}
    *source: [https://youtu.be/whjRc4H-rAc](https://youtu.be/whjRc4H-rAc)*
    {: refdef}

5. Then deployed the `Deployer` again. As CREATE2 is used the Deployer get the same address again (`0xFFF`).

    <br />

6. Now the attacker deployed malicious proposal contract using CREATE again. As the sender(0xFFF) and nonce(0) are same the malicious proposal contract get the same address(`0xABC`) as original proposal contract.

    <br />
 

    ![How can the attacker deploy different contracts at the same address](/assets/img/ta7.png)
    {:refdef: style="text-align: center;"}
    *source: [https://youtu.be/whjRc4H-rAc](https://youtu.be/whjRc4H-rAc)*
    {: refdef}

## Final Thoughts
Finally what should we do to prevent these kinda goveranace attacks in future:
> “Be careful what you vote for! While we all know that proposal descriptions can lie, proposal logic can lie too! If you're depending on the verified source code to stay the same, make sure the contract doesn't have the ability to selfdestruct ”  
>    --- <cite>[Samczcun](https://twitter.com/samczsun)</cite>

<br />

Happy Hunting!