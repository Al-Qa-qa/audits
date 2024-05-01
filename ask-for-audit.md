## Auditing Process

My auditing process takes more than one phase, to improve the safeness of the protocol as much as I can.

You can message me on [twitter](https://twitter.com/Al_Qa_qa), [discord](https://discord.com/channels/al_qa_qa), or [telegram](https://t.me/al_qa_qa) to ask for it.

### Pre-Audit Phase
This process may take a few hours or a day at most, in this phase I go for the protocol and check its structure and code.
- The degree of the decentralization of the protocol.
- Code complexity.
- Access control Logic.

Then, I search for the common bugs like simple re-entrancy, unsafe casting, weird ERC20 case not handled, etc... this can give me an impression of how secure the protocol is (will it contain many bugs, or its robust codes and bugs will be less).

Finally, I give the sponsor (protocol devs), my view of the protocol codebase, the common bugs that I found, and the price of the Audit (Yes this process is free for all Protocols).

**NOTE:**
- Some types of protocols we do not accept, can be found [here](#protocols-i-do-not-accept).
- To know how much it will cost to audit your protocol check this [link](#pricing-and-duration).

### Auditing Phase
This is the longest process, if We reach an agreement on the price and duration. The auditing proccess starts and it is as following:
- I made a security review to the codebase.
- When auditing, I can contact protocol devs to ask for something unclear, abnormal behavior, or an issue that I am not sure about.
- After finishing, I submit All issues I found in GitRep/issues labeled by the severity of the issue.
- Developers can discuss the issue's validity, severity, and anything related to it on the issue's page.
- Developers will fix issues that have impacts and can Acknowledge some issues if they see it has no great impact.

### Post-Audit Phase
After fixing the issues, I made another audit to the protocol to check for the issues mitigations.
- Check that the issues are fixed successfully without introducing further issues.
- Give my final thoughts about the Project.
- Confirm if I see that the protocol is safe to go for Mainnet or do another audit is better before launching.

This is how the auditing process occurs, Feel free to drop a message. And I will be more than happy to secure the protocol.

---

## Pricing And Duration

### Pricing:

The price to audit the protocol depends on two different things
1. The number of solidity codes (the more the code the more expensive)
2. The code complexity (is there an integration with another protocol, heavy math, YUL code, ...)

The price ranges between [3, 6] USDC per LOC based on https://github.com/Consensys/solidity-metrics

The price per LOC is changed according to the complexity of the code.

### Duration:

The duration is not a constant period for all protocols with the same SLOCs, where complexity changes, and conditions changes too. But In normal cases, here is a table of the durations of the protocol according to the SLOC, where it should not exceeds these periods.

|SLOC|Duraiation|
|:--:|:--------:|
| SLOC <= 500 | 4 days|
| 1000 >= SLOC > 500| 7 days|
| 1500 >= SLOC > 1000| 10 day|
| 2000 >= SLOC > 1500| 14 days|

This duration may seem to be long, but I take into consideration the weekend days, and if the code is complex. And the duration can be less than that for some protocols.

---

## Protocols I do not accept

### Protocols used for frauding and stealing

If the protocol will be used to steal users funds, draining wallets, deceive users, then I will not accept to audit this protocol

### Gambling and Lottery Protocols

I do not accept auditing protocols that deal with lottery and Gambling, like Casino protocols where people bid with their amount and they may be the winner.

### Lending/Borrowing with interest rate > 0

Lending/Borrowing protocols like AAVE and Compound are one of the most famous protocols in the DeFi space, and they are being used a lot. In most of the cases, the Borrower will return the value he took from the Lender and pay fees for this.
- For example, if he borrowed 100 ETH, he will pay 100.5ETH when giving it back to the Lender.

This type of Lending/borrowing protocol, I do not accept them.

If there is no fees (interest-rate) accumulated by the Lender, then I can accept the protocol.

If you are not sure about your protocol type and interest rate mechanism, and its integrations. You can message me and I will give you weither I can accept it or not

### Leverage Protocols with fees paid to the Lender

In the case of Leverage in Forex/Crypto trading, you can trade with an amount greater than the amount you have.

- for example, if you have 10ETH, and made 10x leverage, you can open a position that worth 100ETH.

If the leverage accures a percent fee relative to the worth of leveraged money, then I can not accept this protocol. Here is an illustration:

- `Protocol I will accept`
  - Bob has 10ETH.
  - Bob wants to open a position in ETH/USDC (he made an Up position meaning ETH should go Up for him to win).
  - 1 ETH worth 3,300 USD.
  - Bob made a leverage of 10x (made a position worth 100 ETH i.e. 330,000 USD).
  - Bob's position is liquidatable only if his collateral can not pay the loss.
  - ETH price goes down instead of going Up and reaches 3,000 USD.
  - The position is now worth 100 * 3000 = 300,000.
  - 330,000 - 300,000 =  -30,000, the protocol losses 30,000.
  - Bob collateral is 10ETH i.e 10 * 3,000 = 30,000.
  - Then bob position should get liquidated at this point as his collateral can not pay for that loss.
  - Ofc, there can be fees accumulated that the protocol takes when closing the position, but it should not be a percent of the position value.

- `Protocol I will not accept`
  - Bob 10ETH.
  - Bob wants to open a position in ETH/USDC (he made an Up position meaning ETH should go Up for him to win)
  - 1 ETH worth 3,000 USD
  - Bob made leverage of 10x (made a position worth 100 ETH i.e. 300,000 USD)
  - Leverage fees is 0.5%
  - This means that Bob will be liquidatable if his position faces a loss in his position that can not pay for the loss and fees
  - i.e 0.5% * 100ETH = 0.5ETH
  - This means if bob position faces a loss equal to his collateral - fees (10 ETH - 0.5ETH = 9.5ETH), it will be liquidatable.
  - This type of protocol I do not accept.

### Perpetual/Options trading

In the case of Forex perpetual/Option is done by simulating the buying and selling process. i.e. the user is not actually buying the tokens he wants, just the system records this.

If the users are not actually taking there tokens when making a trade, and do not own it themselves, then I can not accept this protocol.

**In Brief**
- If the user owns the (tokens) position, this means he can either sell it, or transfer it to another one, and then I can accept the protocol.
- If the user position is known using the Smart Contract storage, and users can not see their tokens value, nor transfer the position, and can only close the position from the Contract, then I can not accept the protocol.

Some protocols are complex, and some protocols can integrate with other protocols that do one of the things we do not accept. So if you find that you are not sure about the protocol type, you can message me, and I will tell you wether I will be able to accept the protocol or not.

## Disclaimer

Smart Contract Auditing can not guarantee the safety of the protocol 100%. I try my best to find as many issues as I can that can put the protocol is an abnormal state. But I can not be sure that I found all issues and attacks that can occur.
