<img src="assets/profile.png" alt="Profile Picture">

## About Rappie
Rappie is the Founder, CTO, and Lead Fuzzing Specialist at [Perimeter](https://cantina.xyz/guilds/perimeter), an Associate Security Researcher at [Spearbit](https://spearbit.com/), and active in Bug Bounty on [Immunefi](https://immunefi.com/).

As a security researcher, he specializes in fuzzing EVM-based smart contracts. 

Beyond his professional roles, Rappie is an active member of the fuzzing community, contributing to its growth through various initiatives, including maintaining a [List of Public Fuzzing Campaigns](https://github.com/perimetersec/public-fuzzing-campaigns-list).

## Testimonials
> Rappie found some extremely subtle behaviors in our code that many others missed. He not only uses the cutting edge of multiple fuzzing engines, but also helps shape how these fuzzers are built. We've been delighted to use his mastery to make our contracts more secure.
> 
>   - [DanielVF](https://x.com/danielvf), [Origin Protocol](https://www.originprotocol.com/)

> Rappie went above and beyond to deeply understand our protocol and cover all the edge cases. His experience and knowledge about the art of fuzzing is unparalleled. Overall he is an incredible security expert, we certainly will be returning to him with our future smart contracts.
>
>   - [Igor Zuk](https://x.com/code_sandwich), [Drips Network](https://www.drips.network/)

## Fuzzing & Security Research

| Protocol                                           | Engagement Type                                    | Completed | Report                                                                                                                                   | Code                                                                                            |
| -------------------------------------------------- | -------------------------------------------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| Private                                            | Fuzzing Specialist during Spearbit Security Review | 2024-05   |                                                                                                                                          |                                                                                                 |
| [Origin Protocol](https://www.originprotocol.com/) | Fuzzing OETHVault                                  | 2024-05   | [Report](https://github.com/perimetersec/origin-oeth-fuzzing/blob/main/reports/Origin%20Protocol%20OETHVault%20-%20Fuzzing%20Report.pdf) | [Code](https://github.com/perimetersec/origin-oeth-fuzzing/tree/main/src/fuzz/oethvault)        |
| Private                                            | Fuzzing Engagement with Perimeter                  | 2024-04   |                                                                                                                                          |                                                                                                 |
| Private                                            | Fuzzing Specialist during Spearbit Security Review | 2024-03   |                                                                                                                                          |                                                                                                 |
| [Drips Network](https://www.drips.network/)        | Fuzzing Suite & Spearbit Security Review           | 2024-01   | [Report](https://docs.drips.network/assets/files/Spearbit_Drips_Network_Security_Review-d5cda225c36d4c2f1185e154431812b5.pdf)            | [Code](https://github.com/perimetersec/drips-fuzzing/tree/main/src/echidna)<br>                 |
| Private                                            | Fuzzing Engagement with Perimeter                  | 2023-11   |                                                                                                                                          |                                                                                                 |
| [Origin Protocol](https://www.originprotocol.com/) | Fuzzing OUSD                                       | 2023-09   |                                                                                                                                          | [Code](https://github.com/OriginProtocol/origin-dollar/tree/master/contracts/contracts/echidna) |
| [Origin Protocol](https://www.originprotocol.com/) | Fuzzing based Audit                                | 2023-03   | [Report]( reports/Origin%20Protocol%20-%20Security%20assessment%20of%20PR%20%231239.md)                                                  |                                                                                                 |

## Bug Bounty & Competitions
| Description                                                                               | Severity<br> | Report                                                                                                              | Platform  | Protocol                                           |
| ----------------------------------------------------------------------------------------- | ------------ | ------------------------------------------------------------------------------------------------------------------- | --------- | -------------------------------------------------- |
| Incorrect argument passed to `Utils.characterToUnicodeBytes` in `Namespace.fuse`          | High         | [Report](https://github.com/code-423n4/2023-03-canto-identity-findings/issues/101)                                  | Code4rena | [Canto Identity](https://www.cantoidentity.build/) |
| Calling `OUSD.burn()` on an address with zero balance causes the `totalSupply` to go down | Low          | [Report](reports/Origin%20Protocol%20-%20Token%20burn%20bug.md)                                                     | Immunefi  | [Origin Protocol](https://www.originprotocol.com/) |
| `Vault.redeem()` fails with only non-rebasing credits in the protocol                     | Low          | [Report](reports/Origin%20Protocol%20-%20Redeem%20with%20no%20rebasing%20credits.md)                                | Immunefi  | [Origin Protocol](https://www.originprotocol.com/) |
| Total supply can become larger than max supply                                            | Low          | [Report](reports/Origin%20Protocol%20-%20Total%20supply%20can%20become%20larger%20than%20max%20supply.md)           | Immunefi  | [Origin Protocol](https://www.originprotocol.com/) |
| `LiquidityTree.push()` does not always update state correctly                             | Low          | [Report](reports/Azuro%20-%20Function%20push%20does%20not%20always%20update%20correctly.md)                         | Immunefi  | [Azuro](https://azuro.org/)                        |
| `OUSD.burn()` allows for destroying supply while balance remains                          | Low          | [Report](reports/Origin%20Protocol%20-%20OUSD%20burn%20allows%20destroying%20supply%20while%20balance%20remains.md) | Immunefi  | [Origin Protocol](https://www.originprotocol.com/) |

## Other Work & Initiatives

| Project                                                                           | Link                                                                                                                                                                                             |
| --------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Maintaining a List of Public Fuzzing Campaigns                                    | [Link](https://github.com/perimetersec/public-fuzzing-campaigns-list)                                                                                                                            |
| Contributing to [Antonio Viggiano](https://x.com/agfviggiano)'s Eureka initiative | [Link](https://x.com/agfviggiano/status/1767899333620363432)                                                                                                                                     |
| Reproduction of the Rari Finance hack using on-chain fuzzing with Echidna         | [Link](https://github.com/rappie/echidna-rari-hack)                                                                                                                                              |
| Author of Echidna Exercise: Solve Damn Vulnerable DeFi - Side Entrance            | [Exercise](https://github.com/crytic/building-secure-contracts/blob/master/program-analysis/echidna/exercises/Exercise-7.md), [PR](https://github.com/crytic/building-secure-contracts/pull/143) |

## Contact
Don't hesitate to contact me with questions, discussions, or business requests.
- X: [rappie_eth](https://x.com/rappie_eth)
- Discord: `rappie`
- Telegram: `@rappenstein`
- Cantina: [Rappie](https://cantina.xyz/u/Rappie)
