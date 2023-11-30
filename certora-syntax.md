
Part 1

1. Variable declaration for env
```js
struct env {
	address e.msg.sender;
	uint256 e.msg.value;
	uint256 e.block.number;
	uint256 e.block.timestamp;
	uint256 e.tx.origin;
}
```

2. Method declarations (tells verifier not to look for the `env` variable in `envfree`)
```js
methods {
	balanceOf(address)                      returns uint256 envfree
	transfer(address, uint256)              returns (bool)

	allowance(address, address)             returns (uint) envfree
	transferFrom(address, address, uint256) returns (bool)
	approve(address, uint256)               returns (bool)
}
```

3. Rule Declarations
```js
rule transfer(env e, address recipient, uint256 amount) {
	require e.msg.sender != recipient
	
	address myBalanceBefore = balanceOf(e.msg.sender)
	address recipientBalanceBefore = balanceOf(recipient)
	
	transfer(e, recipient, amount)
	
	address myBalanceAfter = balanceOf(e.msg.sender)
	address recipientBalanceAfter = balanceOf(recipient)
	
	assert myBalanceAfter == myBalanceBefore - amount
	assert recipientBalanceAfter == myBalanceBefore + amount
}
```

4. Types of assumptions

```js

rule check additionOfTransfer(env e, address recipient, uint256  amount){
	
	// a. require
	require recipient != e.msg.sender
	
	// b. if statement
	if (amount > 0) {
		assert balanceAfter > balanceBefore
	} else {
		assert true
	}

	// c. assert implication
	assert amount > 0 => balanceAfter > balanceBefore
	
	// d.  bi-implication
	assert amount > 0 <=> balanceAfter > balanceBefore
}
```

5. Rule Exempt: transfer@withrevert

```js
/// A user must not trasfer more tokens than they own
rule transferReverts(env e, address recipient, uint256 amount) {
	require balanceOf(e.msg.sender) < amount

	transfer@withrevert(e, recipient, amount)
	
	assert lastReverted
}
```

Non reverting paths
```js
// Solidity
function foo(uint256 bar) public pure {
	if (bar == 10) {
		revert()
	}
}
```

```js
// certora: ⛔️ will never reaches assert!!
rule checkRevert(env e) {
	foo(e, 10)
	assert false;
}

// certora: ✅ reaches assert!
rule checkRevert(env e) {
	foo@withrevert(e, 10)
	assert lastReverted
	assert false
}

// using @revert and lastReverted to prove liveness
/// want to prove a user with sufficient balance can transfer their tokens

/// if you call transfer and do have enough funds, the transaction doesn't revert

rule transferDoesntRevert(env e, address recipient, uint256 amount) {
	require balanceOf(e.msg.sender) > amount          // 
	require e.msg.value == 0                          // non-payable
	require balanceOf(recipient) + amount < max_uint; // overflow check
	require e.msg.sender != 0                         // zero-address
	require recipient != 0                            // zero-address
	
	transfer@withrevert(e, recipient, amount)
	assert !lastReverted
}
```
