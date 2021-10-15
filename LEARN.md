# Overflow Attack
In this quest, today we are going to learn about overflow/underflow attacks. This is one of the most common forms of vulnerability present in smart contracts. Because smart contracts are immutable, they cannot be modified once deployed, any vulnerability that is present in the contract may go unnoticed till someone points it out, or worst, it is exploited.

In this tutorial, we will start with what may look like a simple working smart contract and then explore the vulnerability present in it. Next, we are going to learn how we can make our smart contract secure from such vulnerabilities. For this quest, we are going to use Remix because it is easier to use. However, the concepts learned in this quest will remain valid irrespective of the development environment used. 

Since vulnerabilities and hacks form a crucial part of smart contract development I would request you to have a code editor ready and try to code along with this tutorial.

# Overflow/Underflow Vulnerability

Most Integer datatypes in Solidity (like uint256) have a fixed size and a specific upper and lower value. For example in the case of uint256, the lowest possible value is 0 (all 256 bits are 0) and the maximum possible value is 2256 - 1 (all 256 bits are 1).

An overflow or underflow will occur when we try to store a value greater than the maximum value or lower than the minimum value of the variable respectively. In such a case, the value stored in the variable will circle back (move from highest value to lowest value in case of overflow and lowest value to highest value in case of underflow). 

In this quest, we are going to see how we can use this to our benefit to pay less than the required amount for performing an action and also how we can prevent such vulnerabilities in our smart contract.

# The smart contract

Take a look at the following smart contract:
```
pragma solidity 0.6.0;

contract Vote {

    

    mapping(address => uint256) public received_votes;

    mapping(address => uint256) public stake;

    uint256 public PER_VOTE_ETH = 1 ether;

    

    function vote(uint256 _vote, address _candidate) public payable {

        require(msg.value == _vote \* PER_VOTE_ETH);

        

        received_votes\[_candidate\] \+= _vote;

        stake\[msg.sender\] \+= msg.value;

    }

}
```
This is a small smart contract in which users are expected to vote for a candidate of choice. Votes can cast as many votes as he/she wants but an equivalent amount of Eth has to be paid for doing so. The PER_VOTE_ETH variable is used to define how much Eth must be paid for each vote. This is defined as 1 ether which means if someone wants to cast only 1 vote he/she pays 1 Eth while if someone wants to cast 3 votes to a candidate, 3Eth has to be paid. 

A require statement is used that ensures that the user is passing the correct amount of eth which making the function call as needed by the number of votes he/she is casting.

Based on this, our initial goal will be to find how we can cast the maximum number of votes for a candidate (let’s assume their address is 0x56B53E1e4561A9fa6d309325c5537c3346A69d37) by staking the least possible amount.

Before moving to the next sub-quest, I would request you to try and find out how you can achieve this by yourself.

# A close look at the smart contract

Lets take a close look at the smart contract and try to see what approach was made by the developer. 

- The contract has defined 3 public global variables:
	- received_votes: This is a mapping from an address to an uint256 number. This mapping represents the number of votes received by each account address. 
	- stake: This is also a mapping from an address to an uint256 number. This mapping is used to store the amount of stake made by an account. This variable may be helpful to withdraw the amount staked after the voting period is over. For simplicity purposes, the withdraw function is not written.
	- PER_VOTE_ETH: This variable helps us define how much amount is to be staked for each vote cast. It is set as 1ether. ether is a prefix in solidity which signifies 1018. So writing 1ether is equivalent to writing 1 \* 10\*\*18. Here we have defined 1ether which means for every 1vote cast, a total of 1ether should be staked, for 2vote, 2ether should be staked, and so on. 
- The contract has only one function called vote that takes in two parameters, the amount of vote the user wants to cast(_vote) and the address of the candidate he/she wants to vote(_candidate).
	- The function first verifies that the correct amount of eth is sent with the function call. This is done by using the msg.value keyword that returns the amount of wei (smallest uint of ether. 1wei = 10-18ether) send with the transaction. The value returned by msg.value is also of type uint256. The value returned by msg.value is checked with the product of PER_VOTE_ETH and _vote.
	- Then the value stored in received_votes and stake mapping is updated to add the number of votes given and the staked amount respectively.

This contract is based on the idea that if someone wants to caste a higher vote that individual should pay an equally large amount. If it becomes possible to cast a very large amount of vote by paying a very low price then it would defeat the purpose of using this contract.

# The vulnerability

There is only one line in the function vote that checks whether an individual is paying the correct amount of Eth. That line being:
``
require(msg.value == _vote \* PER_VOTE_ETH);
```
So as a hacker, our target will be to find a loophole in this check statement. Let’s get back to the concept of overflow. If we try to store a certain value in a variable that is larger than the maximum possible value for that variable, it will cycle back to the lowest possible value. We know that both the variables _vote and PER_VOTE_ETH are of type uint256 which means their product will also be of type uint256. Similarly, the type of msg.value is also uint256. If we can somehow overflow the value of _vote\*PER_VOTE_ETH then we can pass a very high value of _vote by paying a low value.

Max. value of uint256  =                    2256 - 1

PER_VOTE_ETH        =  1 ether  =  1018

Min. value of _vote for overflow  =      (Max. value for uint256)  /  (PER_VOTE_ETH)                                                    =      (2256 - 1) / 1018                                                    

=      115792089237316195423570985008687907853269984665640564039457.584007…

Round up we get

115792089237316195423570985008687907853269984665640564039458

This large number will push us to __Overflow Region__

Now we must calculate how much value should be passed while making the transaction.

Lets assume the large number obtained as N.

Then the value of msg.sender should be  = (P \* 1018) - (2256-1) = 415992086870360065

Therefore the Hack will be:

Cast a vote of 115792089237316195423570985008687907853269984665640564039458 for the desired candidate and pass 415992086870360065 wei as the paid value. 

This will overflow the multiplication and as a result, the require statement will validate even though the amount sent is significantly low. By this attack, it is possible to cast a vote with a very high value yet by staking a very small amount.

# Seeing it in action

We are going to use Remix IDE to see a live demo for this. Copy past the code into Remix.

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/7f12f167-b727-44cd-a3bd-08e19ce782a8.jpg)

Now let’s deploy it and try to interact with it. 

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/4475b454-92e4-432f-8f71-a936ba7f5a22.jpg)

The address we are casting the vote to is not important. After entering this value, when clicked on the transact button the transaction will go through. Now we can check the amount of votes received by the candidate whose address we passed in and the balance of the account making the transaction. And it will be something like:

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/edb06616-0fbb-4e39-a568-18800feccb48.jpg)

Congratulations 🎉🎉🎉 ! You have successfully hacked the system 😎😎😎.

# Prevention

Now that we understand what is the hack, it is important to secure our smart contract from such attacks. The main requirement for securing our smart contract is to ensure that in case of an overflow/underflow our smart contract should generate an error instead of wrapping the value.

This security feature can be achieved in either of three ways:

1. Using Solidity version 0.8.0 or higher: The simplest and easiest solution will be to use solidity version 0.8.0 or higher. From version 0.8.0, no variable will overflow/underflow, and an error is generated.
2. Using SafeMath: If for some reason or other, version 0.8.0 or higher is not used then the best option is to use the SafeMath library. This library makes sure that no overflow/underflow can take place. We will see how to use SafeMath in a later subquest.
3. Building it from scratch: We always have the option to check from scratch for every mathematical operation whether there is an overflow or not. However this is not easy nor recommendable as if not done properly, it may give rise to other forms of vulnerabilities.

# Using SafeMath

The following code is the same code as before, rewritten by using SafeMath:

```
pragma solidity 0.6.0;

import "https://github.com/OpenZeppelin/openzeppelin-contracts/blob/solc-0.6/contracts/math/SafeMath.sol";

contract Vote {
    using SafeMath for uint256;

    mapping(address => uint256) public received_votes;

    mapping(address => uint256) public stake;

    uint256 public PER_VOTE_ETH = 1 ether;

    

    function vote(uint256 _vote, address _candidate) public payable {

        require(msg.value == _vote.mul(PER_VOTE_ETH));

        

        received_votes\[_candidate\].add(_vote);

        stake\[msg.sender\].add(msg.value);

    }

}
```


Let’s understand the code:

- We can import the SafeMath library in our smart contract by using the import statement with the link to the code. If we used some other framework like Truffle, HardHat, or EthBrownie we would first install the OpenZeppelin package and then import the SafeMath.sol smart contract from that package.
- We want to use SafeMath for the uint256 datatype. This is defined with the using keyword.

	using SafeMath for uint256

This tells the smart contract to use SafeMath for performing various operations on this datatype.

- We use the .add() function for adding and .mul() function for multiplying values.

# Testing updated contract

We again deploy our contract and try to make a similar transaction. This time we get the following error while trying to make the transaction:

![](https://qb-content-staging.s3.ap-south-1.amazonaws.com/public/fb231f7d-06af-4aff-bca3-fd51cb633f77/77e2ba66-4884-4359-9f89-925636dbc7f1.jpg)

If we look closely to the error message you can see, “SafeMath: multiplication overflow”. This error is generated when we try to overflow the uint256 variable by multiplying large numbers. 

This shows that using SafeMath we can prevent overflow and also underflow of variables. You can also try the same without SafeMath and using solidity version 0.8.0 or higher. Let it be an exercise for you.

# Conclusion

Through this quest, we learned about the Overflow/Underflow attack, what it is and how it can be prevented in our smart contract. As an assignment I would like to assign the readers two tasks:

1. Try using solidity version 0.8.0 or higher and check if the overflow attack is still valid.
2. Try to use SafeMath in creating an ICO smart contract where a token is sold based on the amount of Eth send to the contract. 

# References

I considered the following references while learning and believe it will be helpful for everyone:

1. [https://ledgerops.com/blog/capture-the-ether-part-2-of-3-diving-into-ethereum-math-vulnerabilities/\#:~:text=A%20uint256%20variable%20has%20a,of%202\*\*256%20%E2%80%93%201](https://ledgerops.com/blog/capture-the-ether-part-2-of-3-diving-into-ethereum-math-vulnerabilities/#:~:text=A%20uint256%20variable%20has%20a,of%202**256%20%E2%80%93%201).
2. https://solidity-by-example.org/hacks/overflow/
