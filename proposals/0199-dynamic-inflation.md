---
simd: '0200'
title: Dynamic Inflation
authors:
  - A2KDefi
  - Ansel (Tokamai)
category: Standard
type: Core
status: Review
created: 2024-11-21
---

## Summary

The purpose of this feature is to enable Solana reward emission to be dynamic 
on the established inflation schedule parameters.

## Motivation

The inflation rate on Solana should be as low as required to enable more than 
50% of the circulating supply of tokens staked. 

## Detailed Design

The inflation schedule will be implemented on a dynamic curve: 

- the minimum inflation on the curve would be 0%
- the maximum inflation on the curve would be the current Solana curve inflation rate (4.8%)

The inflation shall adjust upwards or downwards at the start of each epoch. 

Upon implementation, inflation would start at 4.8% (this is the current inflation) and:

- inflation would decrease by .05 if the total Solana staked was more than 50%. 
- inflation would increase by .05 if the total Solana staked was less than 50%.

The curve will be bound by :

- <current Solana curve inflation rate> at the maximum 
- and 0% at the minimum.

## Alternatives Considered

N/A

## Impact

With the evolution of the ecosystem, validators now rely on MEV and priority 
fees over token inflation. This positive direction means that we can reduce 
reliance on inflation for rewarding stakers to maintain network security. 

dApp developers, token holders, and core contributors will have a more sound 
long-term sustainability tokenomics.

## Security Considerations

We want to ensure as many validators as possible are economically sufficient 
to maximize the decentralization of the network.

To ensure this long term, we might need a controller dependent on revenue/opportuniy 
cost rates.

## New Terminology

N/A
