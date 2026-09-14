# EVM storage, memory and calldata
```
| Location | Where it lives             | Lifetime        | Mutable         | Typical Usage                                        | Gas Cost                  |
| -------- | -------------------------- | --------------- | --------------- | ---------------------------------------------------- | ------------------------- |
| storage  | blockchain state           | permanent       | yes             | state variables, mappings, persistent arrays         | most expensive            |
| memory   | EVM linear temporary RAM   | during the call | yes             | temp variables, return values, function computations | medium                    |
| calldata | transaction input buffer   | during the call | read-only       | external function parameters (arrays, structs)       | cheapest for large inputs |
| stack    | EVM stack (max 1024 slots) | during the call | yes             | local primitives variables (uint, bool, etc)         | cheapest                  |

Only dynamic arrays in storage support push() and pop().
```
EVM layout when a function runs:
```
┌─────────────────────────────┐
│ Stack (max 1024 items)      │
│ small values, pointers      │
├─────────────────────────────┤
│ Memory                      │
│ temporary arrays/structs    │
│ uint[] memory arr           │
├─────────────────────────────┤
│ Storage                     │
│ contract state variables    │
└─────────────────────────────┘
```

# Contract call
In Ethereum, each contract transaction contains:
```
to: contract address
value: some ETH
data: some calldata
```
The `calldata` is the raw bytes sent with the transaction that specify:
1. which function to call
2. the arguments
```
calldata = [4 bytes function selector][encoded arguments]
```

Examples:
```
to: 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4
value: 1 ETH
data: 0x // empty calldata

to: 0x5B38Da6a701c568545dCfcB03FcB875f56beddC4
value: 1 ETH
data: 0x12345678 // any data bytes
```

# receive() and fallback()
`receive()` and `fallback()` are special functions that are automatically executed when a contract receives a call that does not match any existing function.
They are mainly used for handling ETH transfers or unknown function calls.

`receive()` is executed when 1) the transaction sends ETH and 2) the calldata is empty.

`fallback()` is triggered when 1) the called function does not exist or 2) calldata exists but doesn't match any function.

```
incoming call
     │
     ▼
function exists?
     │
 ┌───┴───┐
 │       │
yes      no
 │       │
execute  calldata empty?
function │
         ├──── yes → receive()
         │
         └──── no  → fallback()
```

# visibility modifiers
`public`, `external`, `internal`, and `private` are visibility modifiers define where a function or state variable can be accessed from.
```
| Visibility | Same Contract        | Derived Contract     | Other Contracts  | External Users |
| ---------- | -------------------  | -------------------- | ---------------  | -------------- |
| public     | ✅                   | ✅                   | ✅               | ✅             |
| external   | ❌ (must use `this`) | ❌ (must use `this`) | ✅               | ✅             |
| internal   | ✅                   | ✅                   | ❌               | ❌             |
| private    | ✅                   | ❌                   | ❌               | ❌             |

```

# State mutability modifiers/specifiers
`view`, `pure`, and `payable` are state mutability modifiers that specify how a function interacts with blockchan state and Ether.
```
| Modifier  | Read state | Modify state | Receive ETH |
| --------- | ---------- | ------------ | ----------- |
| pure      | ❌         | ❌           | ❌          |
| view      | ✅         | ❌           | ❌          |
| normal    | ✅         | ✅           | ❌          |
| payable   | ✅         | ✅           | ✅          |
```

# Custom modifiers
Besides visibility modifiers and State mutability modifiers, we can define `custom modifiers` which are reusable piece of code that wraps a function to add pre-conditions, post-conditions, or common logic.
It is mainly used to:
- enforece access control
- validate conditions
- avoid repeating code across functions
Base syntax:
```
modifier modifierName() {
    // code before function
    _; // execute the original function body here
    // code after function
}

modifier onlyOwner() {
    require(msg.sender == owner, "Not owner");
    _;
}

function withdraw() public onlyOwner {
    // only owner can execute
}
```
# ERC20
ERC20 is a standard for fungible tokens on Ethereum. It defines a set of rules (functions + events) that every token contract must follow.
- ERC = Ethereum Request for Comments
- 20 = proposal number
- fungible tokens = all tokens are identical and interchangable. E.g. USDT, USDC, DAI.
- ERC20 tokens are NOT ETH, they live inside a contract, just numbers in a mapping. So No `.call{value:...}("")`.

Why ERC20 exists?
```
| Before ERC20                                    | After ERC20                                       |
| ------------------------------------------------| ------------------------------------------------- |
| Every token had different interfaces 😵         | ✅ Wallets (MetaMask) understand all tokens       |
| Wallets & exchanges couldn't interact easily    | ✅ DEXs (Uniswap) can trade any token             | 
|                                                 | ✅ Contracts can interact with tokens generically | 
```

1. Important State variables
- `balanceOf`, tracks how many tokens each address owns
```
mapping(address => uint256) public balanceOf;
```

- `allowance`, tracks permission to spend tokens
```
// owner -> spender -> amount
// owner = token holder
// spender = approved account (like a DEX)
// e.g. Alice allows Uniswap to spend 100 tokens
mapping(address => mapping(address => uint256)) public allowance;
```

- `totalSupply`, the total number of tokens in existence
```
uint256 public totalSupply;
```

- Matadata (optional but standard)
```
string public name;
string public symbol;
uint8 public decimals;
```

2. Core Functions (Must Have)
- `transfer`, move tokens from YOUR account to someone.
```
function transfer(address to, uint256 amount) public returns (bool);
```

- `approve`, allow someone else to spend your tokens.
```
function approve(address spender, uint256 amount) public returns (bool);
```

- `transferFrom`, spend tokens on behalf of someone else, used by DEXs, lending protocols, etc.
```
function transferFrom(address from, address to, uint256 amount) public returns (bool);
```

- How `approve` and `tranferFrom` work together
```
1. Alice calls: approve(Uniswap, 100)
2. Uniswap calls: transferFrom(Alice, Pool, 100)
This is how DeFi works.
```

3. Events (Very Important)
- `Transfer`, emitted when tokens move. Wallets rely on this to show balances.
```
event Transfer(address indexed from, address indexed to, uint256 value);
```

- `Approval`, emitted when approval is set.
```
event Approval(address indexed owner, address indexed spender, uint256 value);
```

# Contract Inheritance
- Constructors are executed from most base → most derived (GrandParent -> Parent1 -> Parent2 -> child)
- Solidity resolves inheritance from right to left 
```
Assuming foo exists, the precedence order is:
Child(super.foo()) -> Parent2(foo()) -> Parent1(foo()) -> GrandParent(foo())
```

```
contract GrandParent {}

contract Parent1 is GrandParent {}

contract Parent2 is GrandParent {}

contract Child is Parent1, Parent2 {}
```

# Contract, Abstract Contract and interface
In Solidity, `contract`, `abstract contract`, and `interface` all define blueprints for other contracts, but they differ in how complete they are and what they're allowed to contain.

- Contract (Concrete Contract), a regular contract is a fully implemented contract that can be deployed.
- Abstract Contract, an abstract contract is a partially implemented contract that cannot be deployed.
- Interface, an interface is a strict blueprint with no implementation at all.
```
| Feature             | contract  |  abstract contract  | interface     |
| ------------------- | --------- | ------------------- | ------------- |
| Deployable          | ✅        | ❌                  | ❌            |
| Function bodies     | ✅        | ✅ / ❌             | ❌            |
| State variables     | ✅        | ✅                  | ❌            |
| Constructors        | ✅        | ✅                  | ❌            |
| Inheritance         | ✅        | ✅                  | ✅            |
| Function visibility | any       | any                 | external only |
```

🧠 Intuition
- contract => “Fully built house” 🏠 (ready to live in).
- abstract contract => “Half-built house” 🚧 (needs finishing).
- interface => “Blueprint only” 📐 (no implementation at all).

# Solidity Library
A Solidity library is a specialized, reusable smart contract deployed once at a specific address (except libraries only have internal functions) to provide helper functions, reducing deployment gas costs and promoting modular code.
In Solidity, libraries can be used in two fundamentally different ways:
- Internal libraries (inlined into your contract)
- External libraries (deployed and called separately)

1. Internal library
A library whose functions are marked `internal` (or `private`)
- functions are referenced directly
- functions are inlined(copied) into the contract at compile time

2. External library
A library with `public` or `external` functions.
- Deployed separately
- Your contract stores its address
- Calls it via DELEGATECALL
  - Code runs in your contract's storage context
  - Library can read/write your storage

```
| Feature       | Internal Library | External Library    |
| ------------- | ---------------- | ------------------- |
| Deployment    | ❌ Not needed    | ✅ Required         |
| Call type     | Direct (inlined) | DELEGATECALL        |
| Gas (runtime) | ✅ Cheaper       | ❌ More expensive   |
| Bytecode size | ❌ Larger        | ✅ Smaller          |
| Reusability   | Limited          | High                |
| Safety        | ✅ Safer         | ⚠️ Depends on trust |
```

3. A Solidity library is stateless:
- ❌ cannot have its own persistent storage
- ❌ cannot declare state variables like a contract
- ❌ cannot hold balances or ownership
- ❌ cannot be inherited

# using for directive
The Solidity `using for` directive is used to attach library functions as member functions to a specific data type.
```
using LibraryName for Type;
```

# What is a Solidity event?
A Solidity `event` is a way for a contract to emit structured logs during execution.
These logs are:
- stored in the transaction receipt
- not part of contract storage
- mainly used by off-chain apps (UIs, indexers, analytics)
```
Blockchain
├─ Blocks
│   ├─ Transactions
│   │   ├─ Receipt
│   │   │   └─ Logs (events) ⭐ Events在这里
│
└─ World State (全局状态)
    ├─ Account A
    │   ├─ balance
    │   ├─ nonce
    │   ├─ codeHash (empty)
    │   └─ storage root (empty)
    │
    ├─ Account B
    └─ Contract Account
        ├─ balance
        ├─ nonce
        ├─ codeHash
        └─ storage root ⭐ Contract States在这里
```

# What's inside a events?
when you emit an event, it becomes a log entry with this structure:
```
Log
├─ address        (contract that emitted it)
├─ topics[]       (indexed fields)
└─ data           (non-indexed fields)
```
`topic` stored as fixed-size 32-byte values and used for filtering/searching
 - Topic[0] - event signature code (e.g. Transfer(address,address,uint256), hashed using keccak256)
 - Topic[1..n] - indexed parameters

# What's Anonymous Events in Solidity?
An anonymous event is a special type of event declare with the `anomymous` keyword:
```
event MyEvent(address indexed user, uint256 amount) anomymous;
```
It does NOT include the event signature hash in `topic[0]`.

Why use anonymous events?

✅ 1. Save gas. No need to store signature hash, slightly cheaper per emit.

✅ 2. More indexed fields, can have up to four indexed fields.

Downsides:
- Hard to filter, must filter manually by contract + topics.
- Hard to decode.

So rarely used in practice.

# Set up Dapp frontend
- system level
```
sudo apt update
sudo apt install nodejs npm

node -v
npm -v
```

- project level
```
mkdir my-dapp
cd my-dapp
npm init -y
npm install web3
```

# call, delegatecall and staticcall
1. `call`, a low-level external call that executes code in the target contract's context.
```
(bool success, bytes memory data) = target.call{options}(data);

e.g.
(bool success, bytes memory data) = payable(msg.sender).call{value: msg.value, gas: 5000}(data);
```
 - Executes target contract's code
 - Uses target contract's storage
 - `msg.sender` is caller contract
 - Can modify state
 - Can send ETH

2. `delegatecall`, executes another contract's code, but in the caller's context.
```
(bool succcess, bytes memory data) = target.delegatecall{options}(data);
Note that options of delegatecall can only be gas, it doesn't support value.
```
 - Execute target contract's code
 - Uses caller's storage ❗
 - `msg.sender` is original external caller ❗ It's the very first sender in the call chain, not changed across calls
 - Can modify caller's state
 - Cannot tranfer ETH separately (uses caller's balance implicitly)

3. `staticcall` a read-only call that guarantees no state modification.
```
(bool success, bytes memory data) = target.staticcall{options}(data);
Note that staticcall supports only {gas: ...}.
```
 - Executes target's code
 - Uses target's storage
 - `msg.sender` is caller contract
 - Cannot modify state ❗
 - Cannot send ETH ❗

 4. 🧠 Final intuition, think of them like this:
- call → "Go run code over there"
- delegatecall → "Run their code here using my data"
- staticcall → "Ask them a question, but don't let them change anything"

# Solidity Safety
Solidity safety is all about writing smart contracts that can't be manipulated or broken, especially since code on blockchains like Ethereum is immutable (they can't be easily changed after deployment).

1. Reentrancy attach
 - A contract calls another contract, and that contract calls back into the original before it finishes.
 - Example problem
```
payable(msg.sender).call{value: amount}("");
balanceOf[msg.sender] -= amount;
```
An attacker can repeatedly withdraw funds before the balance updates.

 - Fix
   - Use Checks-Effects-Interactions(CEI) patthern
   - or a guard like `nonReentrant` from OpenZeppelin

2. Integer Overflow / Underflow
 - Before Solidity 0.8, numbers could wrap around
 ```
 uint8 x = 255;
 x += 1; // becomes 0

uint8 y = 0;
y -= 1; // becomes 255
 ```
  - Fix
    - Solidity >= 0.8 automatically checks this 
    - Older code uses `SafeMath`

3. Access Control Issues
 - Functions that should be restricted are accidentally public.
 - Derives from `Ownable` and uses `onlyOwner` modifier from OpenZeppelin
 ```
 import "@openzeppelin/contracts/access/Ownable.sol";

 contract OwnableToken is Ownable {
    constructor() Ownable() {

    }

    function mint(address to, uint256 amount) external onlyOwner {
        // update to's balance
    }
 }
 ```
 - Fine-grained role-based `AccessControl` from OpenZeppelin
 ```
 contract TokenWithRoles is AccessControl {
    bytes32 public constant MINTER_ROLE = keccak256("MINTER_ROLE");
    bytes32 public constant BURNER_ROLE = keccak256("BURNER_ROLE");

    constructor() {
        _setupRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _setupRole(MINTER_ROLE, msg.sender);
        _setupRole(BURNER_ROLE, msg.sender);
    }

    function mint(address to, uint256 amount) external onlyRole(MINTER_ROLE) {
        // update to's balance
    }

    function burn(address from, uint256 amount) external onlyRole(BURNER_ROLE) {
        // update from's balance
    }
 }
 ```

4. Unchecked External Calls
 - Calling another contract is risky, that contract could
   - Fail sliently
   - Reenter
   - Consume all gas
 - Fix
   - Check return values
   - Use `try/catch`

5. Gas Limit and Loops
 - Unbounded loops can break your contract
 ```
 for (uint i = 0; i < users.length; i++) {...}
 ```
 if `users` grows too big, function becomes unusable
 - Fix
   - Limit the loops iteration
   - Pagination

6. `tx.origin` Vulnerability
 - Use `tx.origin` for authentication is dangerous. Imagine this phishing attack:
 ```
 Victim Contract:
 function withdraw() public {
    require(tx.origin == owner);
    payable(msg.sender).transfer(address(this).balance);
 }
 
 Malicious Contract:
 function trick() public {
    victimContract.withdraw();
 }

 user -> maliciou contract -> victim contract (require check passes, the attacker drains victim contract's funds)
 ```
 - Fix
 ```
 require(msg.sender == owner);
 ```

7. Front-Running
 - Since transactions are public before confirmation, attackers can see and beat your transaction.
 - Example
 ```
 You submit a trade, which is sitting in the mempool before it's confirmed and added to a block.
 Bot copies it with higher gas to make it executes first.
 ```
 - Fix
   - Commit-reveal schemes
   - Use private mempools

8. Denial of Service (DoS)
 - Attackers can block contract functionality
 - Example
 ```
 for (uint i = 0; i < users.length; i++) {
    payable(users[i]).transfer(amounts[i]);
 }

 An attacker adds a malicious contract to users[]:
 receive() external payable {
    revert(); // always fails
 }
 
 Then one failing transfer in a loop stops everything.
 ```
 - Fix
   - Use pull over push pattern, instead of sending funds in a loop, let user withdraw themselves
   ```
   public withdraw() public {
       uint amount = balanceOf[msg.sender];
       balanceOf[msg.sender] = 0;
       payable(msg.sender).call{value: msg.value}("");
   }
   ```
   - Avoid dependency on single external calls
     - If this external call fails… does my contract still work? If NO, dangerous design, resilient design if YES.

9. Delegatecall Risks
 - `delegatecall` execute code in your contract's context
 - If misused, attacker can overwrite storage.

10. Private data isn't Private
 - Even if marked `private`, data is still visible on-chain.
 - Never store 
   - Passwords
   - Secrets and 
   - Private keys

# Solidity Design Patterns
1. Security Patterns
 - Checks-Effects-Interactions (CEI), prevents reentrancy attacks. Always update state `before` external calls.
 - Reentrancy Guard, use a lock to prevent reentrant calls.
 - Ownable (Access Control), restricts functions to an admin. Use for admin functions like upgrades or withdrawals.
 - Role-Based Access Control (RBAC), more flexible than Ownable. Used when multiple roles exist (admin, minter, etc.)
 - Pull over Push Payments, instead of sending ETH automatically, let users withdraw it. Avoid failed transfers blocking execution.
2. Architectural Patterns
 - Factory Patterns, deploy contracts from another contract. Useful for DAOs, wallets, NFT collections.
 ```
 function createContract() public {
    new MyContract();
 }
 ```
 - Proxy Pattern, separate logic and storage. Enables contract upgrades without losing state.
   - Proxy holds storage
   - implementation holds logic
   - use `delegatecall` to execute logic in the implementation contract
3. Gas Optimization
 - Struct Packing, optimize storage slots.
 - Mapping over Arrays. Mappings are cheaper and O(1). Avoid loops when possible.
 - Event Sourcing, Use events instead of storage for history. Much cheaper than arrays in storage.
4. Behavioral Patterns
 - State Machine, control contract flow with states. Used in auctions, escrow, workflows.
 - Commit-Reveal, prevent front-running attack. Common in games, voting sealed bids.
 - Circuit Breaker (Pausable), emergency stop mechanism.

 # ERC Standards
 1. ERC-165, interface detection standard for smart contracts, it specifies an `supportInterface(bytes4 interfaceId)` function that returns a boolean indicating whether a contract implements an interface identified by a unique 4-byte ID.
 2. ERC-20, it's the standard interface for `fungible` tokens on Ethereum. It defines a set of rules(functions + events) that a token contract must implement.
 3. ERC-721, its' the standard for `non-fungible` tokens on Ethereum.
 4. ERC-2981, it defines NFT `royalty info` - who gets paid and how much when an NFT is sold.

 # ERC-721
- Core functions and events
```
function ownerOf(uint256 tokenId) external view returns (address);
function balanceOf(address owner) external view returns (uint256);

function transferFrom(address from, address to, uint256 tokenId) external;
function safeTransferFrom(address from, address to, uint256 tokenId) external;

function approve(address to, uint256 tokenId) external;
function setApprovalForAll(address operator, bool approved) external;

function tokenURI(uint256 tokenId) external view returns (string memory);

event Transfer(address indexed from, address indexed to, uint256 indexed tokenId);
event Approval(address indexed owner, address indexed approved, uint256 indexed tokenId);
event ApprovalForAll(address indexed owner, address indexed operator, bool approved);
```
- Example flow
  - Mint NFT
  ```
  mint -> Token #1 -> Alice
  ``` 
  - Transfer
  ```
  Alice -> Bob (tokenId = 1)
  ```
  - Check
  ```
  ownerOf(1) -> Bob
  ```