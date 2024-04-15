---
title: Read-Only Reentrancy Attacks 
layout: post
date: 2023-11-06 09:33:02 -0200
category: blog
author: Vijay
description: A blog post explaining about read-only reentracny attacks in smart contracts.
---
In this blog post I explained about the popular Read-only Reentrancy attacks.
<!--more-->

<br/>


In January, Jarvis Network Protocol was exploited by using a read-only reentrancy issue in the Curve Protocol. The attacker added a huge amount of liquidity to the Curve pool. And while removing it, he was able to trigger a callback to his malicious contract because one of the pool's tokens is native MATIC, which triggers the fallback function in the attacker's contract. As the `remove_liquidity()` function decreases the totalSupply and makes a callback to the attacker's contract before updating the pool balances, the `get_virtual_price()` function of Curve returns an inflated price when invoked within the callback. Thus, when the attacker called the `borrow()` function of Jarvis Protocol within the callback, which relies on the `get_virtual_price()` function of the Curve protocol to price the LP tokens, he was able to borrow more assets than he should have due to the inflated LP token price.


<br/>

In July, Conic Finance ended up being exploited too, despite their efforts to prevent this exact attack: They had implemented a reentrancy check that was supposed to be automatically turned on when interacting with Curve pools that handled native ether. Their `_isETH()` method checked whether the pool was configured to manage a token of address `0xEee...eE`, which most Curve pools use to represent ETH. Unfortunately, this was not the case for the rETH/ETH pool that instead used the address of the WETH token to represent the ETH side. With the reentrancy protection bypassed, the same issue as explained above could be used to manipulate the price while removing liquidity from Curve and using this to steal funds from Conic.