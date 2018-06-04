# CarryToken_Review

This smart contract audit was prepared by [Quantstamp](https://www.quantstamp.com/), the protocol for securing smart contracts.

This security audit report follows a generic template. Future Quantstamp reports will follow a similar template and they will be fully generated by automated tools.

## Specification

Our understanding of the specification was based on the following documentation:

* [Carry Token Crowdsale README](https://github.com/carryprotocol/carry-token-crowdsale/blob/25b53a973864b6ef30d70caa14bd490935a16b55/README.md)

* Comments included in Solidity files.

## Methodology

The review was conducted during 2018-May-29 through 2018-June-04 by the Quantstamp team, which included senior engineers Alex Murashkin and Leonardo Passos with reviewers Kacper Bak and Steven Stewart. 

Their procedure can be summarized as follows:

1. Code review
    1. Review of the specification
    2. Manual review of code
    3. Comparison to specification
2. Testing and automated analysis
    1. Test coverage analysis
    2. Symbolic execution (automated code path evaluation)
3. Best-practices review
4. Itemize recommendations

## Source Code

The following source code was reviewed during the audit.

|Repository	|Commit	|
|---	|---	|
|[contracts](https://github.com/carryprotocol/carry-token-crowdsale/tree/25b53a973864b6ef30d70caa14bd490935a16b55/contracts)	|[25b53a9](https://github.com/carryprotocol/carry-token-crowdsale/commit/25b53a973864b6ef30d70caa14bd490935a16b55) 	|

# Security Audit

Quantstamp's objective was to evaluate the Carry Protocol ERC20-based token and crowdsale contracts for security-related issues, code quality, and adherence to best-practices.

Possible issues include (but are not limited to):

* Transaction-ordering dependence
* Timestamp dependence
* Mishandled exceptions and call stack limits
* Unsafe external calls
* Integer overflow / underflow
* Number rounding errors
* Reentrancy and cross-function vulnerabilities
* Denial of service / logical oversights

# Test coverage

We evaluated the test coverage using truffle and solidity-coverage. The notes below outline the setup and steps that were performed.

## Setup

Testing setup: 

* Truffle v4.1.7
* TestRPC v6.0.3
* solidity-coverage v0.5.4
* Oyente v0.2.7
* Mythril v0.17.12
* truffle-flattener v1.2.5

## Steps

Steps taken to run the full test suite:

* Installed dependencies via `npm install`.
* Installed the `solidity-coverage` tool: `npm install solidity-coverage@0.5.4`.
* Ran the coverage tool: `./node_modules/.bin/solidity-coverage`.
* To workaround limitations of the `Mythril` and `Oyente` tools, we flattened the source code using `truffle-flattener`.
* Installed the `mythril` tool from Pypi: `pip3 install mythril`.
* Ran the `mythril` tool: `myth -x <Flattened-Contract>.sol`.
* Installed the `Oyente` tool from Docker Hub: `docker pull luongnguyen/oyente && docker run -i -t -v ${PWD}:/oyente/oyente/contracts luongnguyen/oyente`.
* Ran the `Oyente` tool: `cd /oyente/oyente && python oyente.py -s contracts/<Contract>.sol`.

# Evaluation

## Code Coverage

Code coverage within the repository is high: `CarryToken.sol`, `CarryTokenCrowdsale.sol`, and `CarryTokenPresale.sol` have 100% coverage; `GradualDeliveryCrowdsale.sol` has few untested parts:

* Lines 79, 89, 90, 98, and 125: the `else` paths of `require()` statements.
* Lines 92 and 93: the `if` path of the `if (to > beneficiaries.length)` statement.

```
-------------------------------|----------|----------|----------|----------|----------------|
File                           |  % Stmts | % Branch |  % Funcs |  % Lines |Uncovered Lines |
-------------------------------|----------|----------|----------|----------|----------------|
 contracts/                         97.73 |    76.92 |      100 |    97.96 |                
  CarryToken.sol                      100 |      100 |      100 |      100 |                
  CarryTokenCrowdsale.sol             100 |      100 |      100 |      100 |                
  CarryTokenPresale.sol               100 |      100 |      100 |      100 |                
  GradualDeliveryCrowdsale.sol |    97.14 |    72.73 |      100 |     97.3 |             93 |
-------------------------------|----------|----------|----------|----------|----------------|
All files                           97.73 |    76.92 |      100 |    97.96 |                
-------------------------------|----------|----------|----------|----------|----------------|
```

## Usage of External Libraries 

The contract makes extensive usage of external libraries (created by OpenZeppelin). This practice enables relatively minimal modifications to be made in order to execute a customized crowdsale. With continued widespread use throughout the community and regular audits, these libraries will ensure that low-hanging fruit often associated with typographical errors and simplistic oversights are removed. 

### Allowance Double-Spend Exploit

As it presently is constructed, the contract is vulnerable to the [allowance double-spend exploit](https://github.com/OpenZeppelin/openzeppelin-solidity/blob/a0c03ee61cdb77327f2435ba2ab1c539b1d34774/contracts/token/ERC20/StandardToken.sol#L44-L58), similarly to other ERC20 tokens. 

The exploit (as described below) can be mitigated through implementing functions that increase/decrease the allowance relative to its current value, such as the [increaseApproval](https://github.com/MonolithDAO/token/blob/94b935828408d6816849797fab8f89dca9a21db3/src/Token.sol#L98) and [decreaseApproval](https://github.com/MonolithDAO/token/blob/94b935828408d6816849797fab8f89dca9a21db3/src/Token.sol#L106) functions in the MonolithDAO Token. As the Carry Protocol token contract inherits these functions from OpenZeppelin contracts, no further action is necessary. ERC20 tokens may be exploited as follows:

1. Alice allows Bob to transfer `N` amount of Alice's tokens (`N>0`) by calling the `approve` method on `Token` smart contract (passing Bob's address and `N` as method arguments)
2. After some time, Alice decides to change from `N` to `M` (`M>0`) the number of Alice's tokens Bob is allowed to transfer, so she calls the `approve` method again, this time passing Bob's address and `M` as method arguments
3. Bob notices Alice's second transaction before it was mined and quickly sends another transaction that calls the  `transferFrom` method to transfer `N` Alice's tokens somewhere
4. If Bob's transaction will be executed before Alice's transaction, then Bob will successfully transfer `N` Alice's tokens and will gain an ability to transfer another `M` tokens
5. Before Alice notices any irregularities, Bob calls `transferFrom` method again, this time to transfer `M` Alice's tokens.


## Adherence to Specification 

While minimal specification is proposed within the [README](https://github.com/carryprotocol/carry-token-crowdsale/blob/25b53a973864b6ef30d70caa14bd490935a16b55/README.md), multiple checks and constants are presented through the Truffle project to ensure intended actions are successfully performed. 

## Toolset Warnings

Oyente tool has not detected any vulnerabilities of kinds Parity Multisig Bug 2, Callstack Depth Attack, Timestamp Dependency, and Re-Entrancy.

Mythril tool has not detected any vulnerabilities of kinds Integer underflow, Unprotected functions, Missing check on `call` return value, Re-entrancy, Multiple sends in a single transaction, External call to untrusted contract, `delegatecall` or `callcode` to untrusted contract, Timestamp dependence, Use of `tx.origin`, Predictable RNG, Transaction order dependence, Use `require()` instead of `assert()`, Use of deprecated functions, Detect tautologies.

Both Mythril and Oyente have shown an `Integer Overflow` warning at the method `SafeMath.add`; however, we believe this is a false-positive. The method is designed to handle integer overflows, and the statement `c = a + b` is followed by the assertion `assert(c >= a);` which causes the method to throw in case of an overflow. The assertion was also flagged by Mythril as `reachable exception`, which is expected and does not raise any concerns.

Oyente has shown a warning of a possible transaction-ordering dependency at `_wallet.transfer(depositedWeiAmount)` of the `_transferRefund` method of the contract `GradualDeliveryCrowdsale`. This proved to be a false-positive, as both flows reported by the tool are the same. This warning
can be safely ignored.

Oyente has also shown a warning of a possible integer underflow at the line `string public symbol = "CRE"`; however, we believe this is a bug of Oyente: the line does not contain any integer operations.

# Recommendations

## Contracts Code

`GradualDeliveryCrowdsale.sol`

* ***Major issue in logic***. Currently, `_processPurchase` is defined as:
  ```
      function _processPurchase(
          address _beneficiary,
          uint256 _tokenAmount
      ) internal {
          beneficiaries.push(_beneficiary);
          balances[_beneficiary] = balances[_beneficiary].add(_tokenAmount);
      }
  ```
  Executing `beneficiaries.push(_beneficiary);` may lead to cases where
  `beneficiaries` are added multiple times, compromising the gradual delivery of tokens, as given by functions `deliverTokensInRatio` and `deliverTokensInRatioFromTo`. 

  To illustrate how this could happen, consider the following execution flow:

  1. `Bob` buys `100` CRE tokens. Thus, `Bob` is added as a beneficiary in the `beneficiaries` array at position `k' = beneficiaries.length`. His balance is now `100` CRE.
  2. `Bob` buys an extra `200` CRE tokens. Again, `Bob` is added as a beneficiary in the
  `beneficiaries` array at a position `k'' = beneficiaries.length` and `k'' > k'`. His balance is now `300` CRE.
  3. The contract `Owner` now wishes to deliver `1/2` of the tokens. As such, `Onwer`
  calls `deliverTokensInRatio(1, 2)`. The expectation here is that `Bob` should receive `150` CRE.
  4. Function  `deliverTokensInRatio` executes, leading to a call to `_deliverTokensInRatio(1, 2, 0, beneficiaries.length`). 
  5. The execution of `_deliverTokensInRatio` iterates over the `beneficiaries` array. Delivering tokens to `Bob` leads to:
      1. When `i = k'`, `Bob` receives `1/2` of his `300` balance, i.e., `150` CRE. His remaining balance is now `300 - 150 = 150`. 
      2. When `i = k''`, the contract will again deliver tokens to `Bob`. Specifically, `Bob` receives 1/2 of his 150 balance, i.e., 75 CRE. His remaining balance is `300 - 225 = 75`.
As shown from the flow, `Bob` receives `225` CRE, instead of the expected `150`.

One possible way to address this flaw is to assure that beneficiaries are only
pushed once:

  ```
      function _processPurchase(
          address _beneficiary,
          uint256 _tokenAmount
      ) internal {
          if (_tokenAmount > 0 && balances[_beneficiary] == 0) {
            beneficiaries.push(_beneficiary);
          }
          
          balances[_beneficiary] = balances[_beneficiary].add(_tokenAmount);
      }
  ```

Adding the if statement as shown guards against the cases when a `_beneficiary` has already contributed; thus, it enforces uniqueness of items in the `_beneficiaries` array.

## General Remarks

* Document function parameters in all contracts.

* Ensure function names in comments are consistent with function names in the code. Examples where this needs to be fixed: 
  * `receiverRefund` -> `receiveRefund` (line 148, `GradualDeliveryCrowdsale.sol`)
  * `deliverTokenRatio` -> `deliverTokensInRatio` (line 27, `GradualDeliveryCrowdsale.sol`)

* Variable names `_from` and `_to` have been overloaded. Typically they mean a source and a destination when transferring ether. Such names, however, have been used to express ranges over an array (e.g., `GradualDeliveryCrowdsale.sol`, lines 76 and 77). Our recommendation is to rename them to `_startIndex` and `_endIndex`, respectively. Moreover, document that `_endIndex` is exclusive.

* Fix grammar issues. Some examples: `they has` -> `they have`, `ethers` -> `ether`

* The `constructor` keyword should have been used instead of naming it after the
contract. In fact, after doing so, running

```
npm run lint
```

does not report any issues (this is in contrast to the documentation
in the code).

# Conclusion

We found one major issue in the logic of gradually distributing tokens.
Also, the contract is vulnerable to the
ERC20 allowance double spend exploit. The latter, however, is inherent to any ERC20 token contract.

Quantstamp had no additional findings of potential vulnerabilities at the time of analysis. 

# Appendix

## File Signatures

Below are SHA256 file signatures of the relevant files reviewed in the audit.

```
$ shasum -a 256 ./contracts/*
6ce3d1e88167ca24c24f636bd930fb01fa05ac3fb3b0ced0c8f6a897f8e78a37  ./contracts/CarryToken.sol
264b80605fe9a6ca9f9d4ae0df5cb8ecbb5bdee405e6ad2ba7a93fb403f9b14c  ./contracts/CarryTokenCrowdsale.sol
3956b8e95658f6cd6b611328df30e9bfa764f2bc95f1155fdd0b7cc105a0267e  ./contracts/CarryTokenPresale.sol
1239e1fd969899678f82ce80fafebbeed9c64cf2db9d5275ff9a2572a4d16eab  ./contracts/GradualDeliveryCrowdsale.sol
db2ef484d9c59a7b1aa9d99be90f51c92f574b61ff4552996d2ec0be4578b344  ./contracts/Migrations.sol

$ shasum -a 256 ./test/*
486a144f6e28dbdbef8e50bd591bca061c8cc2f6e1940d48de063dbde38ac898  ./test/SampleGradualDeliveryCrowdsale.sol
93e06c72bb0dfab7687f8863887f3ee416cc5524cb6a34818ec59c2b4a50b4f2  ./test/carryToken.js
86185a9828b4b5506822a65a7fe510819077e9a710b43c85b9b2074aa817c148  ./test/carryTokenCrowdsale.js
4cdbbad7a562059255a51f06d3071f9546baca0f98ba307bac92ed30f6d49160  ./test/gradualDeliveryCrowdsale.js
ca88c4b385e302933c495b4280fd401f13382f6b07c266c60cd7aa2d8bcf075b  ./test/utils.js
```

## Truffle Test Results

```
  Contract: CarryToken
    ✓ cannot be minted more than TOTAL_CAP (49ms)
    ✓ should transfer token correctly (112ms)
    ✓ should fail to transfer if sender has not enough balance (68ms)

  Contract: CarryTokenCrowdsale
    ✓ should not receive ETH from address not whitelisted (378ms)
    ✓ should not receive less than individualMinPurchaseWei (399ms)
    ✓ should not receive more than individualMaxCapWei per contributor (454ms)
    ✓ should not receive more than total individualMaxCapWei per contributor (499ms)
    ✓ should receive if all conditions are satisfied (110ms)

  Contract: CarryTokenPresale
    ✓ should not receive ETH from address not whitelisted (395ms)
    ✓ should not receive less than individualMinPurchaseWei (464ms)
    ✓ should not receive more than individualMaxCapWei per contributor (435ms)
    ✓ should not receive more than total individualMaxCapWei per contributor (458ms)
    ✓ should receive if all conditions are satisfied (62ms)

  Contract: SampleGradualDeliveryCrowdsale
    ✓ should not withdraw tokens immediately after purchase (50ms)
    ✓ disallows withdrawal by other than the owner
    ✓ delivers tokens in the specified ratio (304ms)
    ✓ can deliver tokens from & to a particular offset (450ms)
    ✓ disallows to be requested to refund by other than the fund owner or the fund wallet (177ms)
    ✓ allows the fund owner to request to refund a purchase (195ms)
    ✓ disallows to refund more than purchased (143ms)
    ✓ allows the fund wallet to request to refund a purchase (195ms)
    ✓ disallows to refund more than purchased (147ms)
    ✓ disallows to receive the refund if there is no refunded deposit (120ms)
    ✓ disallows to receive the refund by other than the fund owner or the beneficiary (156ms)
    ✓ allows the fund owner to receive the refunded deposit (411ms)
    ✓ allows the beneficiary to receive the refunded deposit (374ms)
    ✓ allows the beneficiary to receive the refunded deposit to his another account (address) (363ms)
    ✓ accumulate the deposit if refunded multiple times (226ms)
    ✓ disallows any other than the beneficiary to receive the refunded deposit to a specified address (274ms)

  Contract: CarryTokenPresale
    ✓ should not withdraw tokens immediately after purchase (130ms)
    ✓ disallows withdrawal by other than the owner
    ✓ delivers tokens in the specified ratio (270ms)
    ✓ can deliver tokens from & to a particular offset (441ms)
    ✓ disallows to be requested to refund by other than the fund owner or the fund wallet (214ms)
    ✓ allows the fund owner to request to refund a purchase (191ms)
    ✓ disallows to refund more than purchased (169ms)
    ✓ allows the fund wallet to request to refund a purchase (211ms)
    ✓ disallows to refund more than purchased (178ms)
    ✓ disallows to receive the refund if there is no refunded deposit (149ms)
    ✓ disallows to receive the refund by other than the fund owner or the beneficiary (176ms)
    ✓ allows the fund owner to receive the refunded deposit (469ms)
    ✓ allows the beneficiary to receive the refunded deposit (404ms)
    ✓ allows the beneficiary to receive the refunded deposit to his another account (address) (384ms)
    ✓ accumulate the deposit if refunded multiple times (212ms)
    ✓ disallows any other than the beneficiary to receive the refunded deposit to a specified address (262ms)

  45 passing (13s)
```

# Disclosure

## Purpose of report

The scope of our review is limited to a review of Solidity code and only the source code we note as being within the scope of our review within this report. Cryptographic tokens are emergent technologies and carry with them high levels of technical risk and uncertainty. The Solidity language itself remains under development and is subject to unknown risks and flaws. The review does not extend to the compiler layer, or any other areas beyond Solidity that could present security risks.

The report is not an endorsement or indictment of any particular project or team, and the report does not guarantee the security of any particular project. This report does not consider, and should not be interpreted as considering or having any bearing on, the potential economics of a token, token sale or any other product, service or other asset.

No third party should rely on the reports in any way, including for the purpose of making any decisions to buy or sell any token, product, service or other asset. Specifically, for the avoidance of doubt, this report does not constitute investment advice, is not intended to be relied upon as investment advice, is not an endorsement of this project or team, and it is not a guarantee as to the absolute security of the project.

## Links to other websites

You may, through hypertext or other computer links, gain access to web sites operated by persons other than Quantstamp Technologies Inc. (QTI). Such hyperlinks are provided for your reference and convenience only, and are the exclusive responsibility of such web sites' owners. You agree that QTI are not responsible for the content or operation of such web sites, and that QTI shall have no liability to you or any other person or entity for the use of third-party web sites. Except as described below, a hyperlink from this web site to another web site does not imply or mean that QTI endorses the content on that web site or the operator or operations of that site. You are solely responsible for determining the extent to which you may use any content at any other web sites to which you link from the report. QTI assumes no responsibility for the use of third-party software on the website and shall have no liability whatsoever to any person or entity for the accuracy or completeness of any outcome generated by such software.

## Timeliness of content

The content contained in the report is current as of the date appearing on the report and is subject to change without notice, unless indicated otherwise by QTI; however, QTI does not guarantee or warrant the accuracy, timeliness, or completeness of any report you access using the internet or other means, and assumes no obligation to update any information following publication.

