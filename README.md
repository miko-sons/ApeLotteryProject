# Lottery Project Overview

This project contains lottery games in which players buy as many tickets as they want--each for an entrance fee--to have a shot at winning the whole pool. The owner/contract deployer will choose when the lottery is opened and closed, while the other stages are automatic. The game has three different implementations, differing in the way in which they randomly[^1] select the winner. The first two are more on the side of pseudo-random while the third is closer to true random. I thought up the details of the 1st and 2nd, but they have almost definetely been thought up before me and likely have flaws I'm not thinking about :)

1. The first implementation, LotteryCaR, utilizes a self-designed commit and reveal mechanism with defined time limits for those phases for the sake of a strong and secure (for a pseudo-random at least) pseudo-random winner selection. This mechanism implements staking to make collusion unprofitable, and other features to be specified. It has the limitation of requiring that the stake to entrance fee ratio is greater than the allowed number of players in the lottery. The details and security risks of this system are provided further down in this README.

2. The second implementation, LotteryRANDAO, is not a great option as of now. It uses [block.prevrandao](https://soliditydeveloper.com/prevrandao). It follows three steps for random number generation. First, close the lottery once all players have entered and record the current block number to be n. Second, wait a bit then access block.prevrandao when the block = n + k. Third, use this value to select a random winner. This implementation would be half decent, however solidity/ethereum currently offers no way to access a certain (past) block's randao, only offering the previous block's randao through block.prevrandao. So, this contract just allows anyone to call a function (that can only be called once and which opens when the block number is >= n + k) which will get block.prevrandao. The details and security risks of this implementation are provided further down in this README. 

3. The third implementation, LotteryVRF, is the common/best lottery implementation. I came across the idea while watching Patrick Collins' "Solidity, Blockchain, and Smart Contract Course – Beginner to Expert Python Tutorial" freecodecamp course. My scripts differ from his in a handful of ways, mainly due to my choice of working with apeworx as opposed to his use of brownie as well as my choice of using chainlink's direct funding methods (and thus the VRF wrapper) as opposed to his use of the subscription method (and thus the VRF coordinator). This implementation has off-chain random number generation by calling to Chainlink's Verifiable Random Function (VRF) v2, which (in exchange for a fee paid in link), provides a "provably fair and verifiable random number generator (RNG) that enables smart contracts to access random values without compromising security or usability." - [Chainlink](https://docs.chain.link/vrf)

***

# Details of Implementation 1

## Commit and reveal scheme background info

For those that don't already know, a commit and reveal scheme is where each player commits a hash of a secret number they come up with, and then later they reveal their secret number. When the players reveal their secret numbers, the contract can verify that the originally committed hash was in fact the unique hash output that corresponds to the revealed secret number. It is infeasible for any player to submit a value that was not their original secret number they generated their hash with, as the chance of randomly finding another number that would yield the same hash output is 1 / (2<sup>256</sup>), in the case of the keccak256 hash at least. This system of taking player inputs to determine the random number would not work otherwise since all information sent to the blockchain (including the smart contract logic) is publicly accessible by design. Without a commit phase first to lock in each player's submission in a hashed (and thus unrevealed) form prior to the posting of their submissions, the last submitter would be able to choose a submission that wins them the lottery by reading the smart contract logic and other player's submissions then doing the relevant calculations.

After the reveal phase, the contract can combine the secret numbers in some way--such as by applying a keccak256 hash to the (packed) abi encoding of the private number data then turning this hash into an int x then finally returning (x modulo number_of_players)--to select a winner. 

## Finer details of implementation 1

I have the order of the (feed into the keccak256 hash to determine the winner) defined by the order of the commits in the commit phase. This would get around the issue of [front-running](https://hacken.io/discover/front-running/) (a vulnerability allowing miners/validators to influence an outcome of the smart contract by manipulating block transaction order) by making the order of the reveals in the reveal phase irrelevant. The validator could potentially manipulate the commit order, but it would not advantage them since their order choice would not give them any information on the random number since the random number's generation fully depends on the yet-to-be posted private numbers.

Typically, in a bare bones commit and reveal mechanism, the so called last-mover (the last one to reveal their reveal/secret number/submission) can determine--using everybody else's reveal that has already been posted to the blockchain--whether they will win or not (since they know everyone's secret number as well as the contract algorithm to select the winner given those numbers). If they have done this calculation and have determined they will not win, they technically have no reason to reveal their secret number. In fact, if they made a deal with the (now calculated) winner to split the prize, this last revealer may have financial incentive to not reveal. I talk about this vulnerability of collusion in the next paragraph. Overall, however, I disincentivize this last revealer's (and all players' in general) non-compliance by initially charging each player more than the entry cost (via a stake) and only returning this stake to players who reveal. The contract could simply proceed with the players who did not reveal disqualified, and their stakes either just stay in the contract forever or are given to the owner. There is thus a penalty for this last revealer choosing to not comply by not revealing their secret number.

One significant exploit is if players collude with one another. For example if there are 10 players and four of them decide to cheat by working together, they could wait til the other 6 reveal and then the four on the cheating team could choose who of them should reveal, doing intermediate calculations to see which if any combination of their reveals/non-reveals lets one of their teammates win the prize. For n teammates colluding, they would have 2^n options for how to reveal (reveal or not reveal for each of the n teammates). Depending on the stake to entry_fee ratio, the number of teammates forgoing their stakes, and the ratio of teammates to non-teammates, it may end up being profitable for the team to forego at least one team member's stake to be able to ensure the winner is one of the team members:

The pay out (entry_fee * total_players) has to be greater than the cost/loss to the team (entry_fee * total_teammates + stake * foregoing_teammates) for it to be worth it to forego a (or multiple) yield(s) and not reveal at least one of the teammates secret numbers.

In other words, if stake = x * entry_fee, then (by dividing both sides by entry_fee) we get that total_players must be >= (total_teammates + x * forgoing_teammates) for cheating to be profitable.

For a conservative estimate of how large x needs to be to make cheating never profitable, we can ignore the total_teammates term and just require that (x * foregoing_teammates) > total_players. At minimum, foregoing_teammates must be at least 1, otherwise the result of the lottery is the correct/valid conclusion of the lottery where all players reveal their secret numbers.

To ensure it would never be worth cheating, you would therefore just have to make sure the staking-to-entry-fee factor (x) is > total_players. For example, a lottery with a maximum of 9 players, a staking deposit of 0.5 ETH, and a entry-fee of 0.05 ETH would be completely protected from all of the mentioned vulnerabilities in this section.

# Details of Implementation 2

## Discussion of this implementation


## Discussion of potential future feature

There is a chance that block.randao(x), where x specifies the past block number of interest, is eventually implemented in ethereum/solidity. This would get around the current issues of needing someone to call the random getter function at block n + k (in current block = n + k implementations), code for cases in which nobody calls the random getter function at block n + k (in current block = n + k implementations), and potential biasing from validators/builders manipulating transaction block assignment (in current block >= n + k implementations).

Even if ethereum did add the feature to access a specified (past) block's randao with something like block.randao(x), there would still be risks. 
//TODO: One is the risk of the block's validator choosing not to sign the block they were assigned to. This would cost the validator that block's reward (which tends to average around 0.04-0.05 ETH), and result in the script's chosen slot to reference. such as the risk of tail-end (within an epoch) validators influencing a block's randao value by choosing not to sign the slot they were assigned to. This risk is further explained in the link at the top of the paragraph. Another risk, but one that is much less signficant, is that of a future block's randao being determinable should a (large) number of validators be in collusion with one another. This is because future randao values are derived from the randomness of each block's validator's BLS signature (with the current epoch as the message of this signature). Should a large number of validators who are going to be chosen in a consecutive string of blocks collude with one another by sharing their signatures, they would be able to calculate the randao of future blocks.


# Details of Implementation 3



[^1]: I am not claiming any of these methods have *true* randomness--in fact I know 1 and 2 are not. I definetely don't know enough to make any judgement on how *truly* random 3 is. In practice, though, it definetely seems to be the best current solution for getting random numbers on to the blockchain.
