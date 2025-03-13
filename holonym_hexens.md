# Unauthorized vetoer setting allows Denial of Access Requests

Holonym

The first 2PC wallet powered by ZK tech

## Description

`src/AccessV2.sol::setVetoer()#L132-L136`

The `setVetoer()` function in the `AccessV2.sol` contract allows any user to set the vetoer for a specific `emailCommitment` without any authentication. This can be exploited by a malicious actor to set themselves as the vetoer for an `emailCommitment` before the legitimate user does. Once set, the malicious vetoer can deny any access requests by vetoing them, effectively blocking the legitimate user from gaining access.

### Reproduction Steps:

1. A malicious actor scans transactions in the blockchain’s mempool to find the `setVetoer()` calls

2. The attacker creates a new transaction with `setVetoer` call where he puts in the address field with their own address for a given `emailCommitment` and front-runs the legitimate user’s call.

3. The application will call the `requestAccess` method and set `electedDecryptor`, timestamp for a given email commitment, when the user wants to recover own account.

4. When the attacker tracks the `requestAccess` method in the blockchain’s mempool, he back-runs it with the `vetoAccessRequest` call where the `accessRequest.vetoed = true;` will be set.

5. The legitimate user is unable to gain access due to the veto.

Application’s backend ignores errors from the `setvetoer` call.

```rust
    pub async fn create_account(
        State(state) : State<AppState>,
        session: SilkySession,
        Json(account_data): Json<CreateAccount>,
    ) -> Result<(), Error> {
        ...
        
        // Call setVetoer on recovery contract here
        let output = Command::new("node")
            .arg(SCRIPTS_PATH.to_owned() + "/recovery/set-vetoer.js")
            .arg(account_data.email_commitment) // emailCommitment
            .arg(user_silk_address) // vetoer
            .output()
            .map_err(|_e| Error::CustomBadRequest("Encountered unknown error initializing recovery"))?;


        // If the set-vetoer script fails, it's not a big deal. It can be run later, and there
        // are edge cases where it might fail even if we still want to finish finalizing the user's
        // account. This println is helpful in case the RPC node is down or our account runs out
        // of gas. It allows us to determine when such errors occurred.
        if !output.status.success() {
            println!("set-vetoer.js failed: {}", String::from_utf8_lossy(&output.stderr));
        }

        // initialize session values 
        let mut locked = state.redis_conn.lock().expect("Mutex Poisoned");
        session.finalize_new_user(&keyshare, &mut locked)?;
        Ok(())
    }
    
}
```

## Code Snippet

```solidity
function setVetoer(bytes32 emailCommitment, address vetoer) public {
    require(commitmentToVetoer[emailCommitment] == address(0), "vetoer already set for this emailCommitment");
    commitmentToVetoer[emailCommitment] = vetoer;
    emit SetVetoer(emailCommitment, vetoer);
}
```

## Remediation

Add authentication to the `setVetoer` function to ensure only the legitimate user can set the vetoer, also terminate the account creating in a case of `"vetoer already set for this emailCommitment"` error message.


>The full report can be found [here](https://github.com/Hexens/Smart-Contract-Review-Public-Reports/blob/main/holonym-zk-audit-may-2024(Public).pdf).
