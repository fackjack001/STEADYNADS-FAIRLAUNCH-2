https://monadvision.com/address/0xA16984b04D2041af93165fC297475641FA71D9f6?tab=Contract

合约的区块链地址

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

/**
 * STEADY v3 - Final (Updated)
 * - Token name: STEADY
 * - Symbol: STEADY
 * - Decimals: 18
 * - Total supply (fixed): 1,000,000,000 STEADY (1B)
 * - Pre-minted treasury reserve: 100,000,000 STEADY (100M) minted to `treasury` at deployment
 * - Max mintable supply: 900,000,000 STEADY (900M)
 * - Reward per mint: 2,500 STEADY
 * - Fee per mint: 1 MON (native coin) forwarded to `treasury`
 *
 * Key guarantees per your request:
 * 1) After the 900M mintable cap is reached, mint() will automatically revert and no more minting is possible.
 * 2) The contract maximum total supply is fixed at 1B (100M pre-minted + 900M mintable). No functions allow increasing that cap.
 * 3) Owner does NOT have any special mint privilege — owner cannot mint tokens beyond the 900M mintable pool.
 *
 * Other features (same as before): minimal ERC20 functions, anti-bot logic, per-address nonces, cooldown, temporary blacklist with owner remediation, pause/unpause, reentrancy guard, rescue ETH.
 */

contract STEADY {
    // ERC20 basic info
    string public constant name = "STEADY";
    string public constant symbol = "STEADY";
    uint8 public constant decimals = 18;

    // Supply constants
    uint256 public constant TOTAL_SUPPLY = 1_000_000_000 * 10**18; // 1B
    uint256 public constant TREASURY_RESERVE = 100_000_000 * 10**18; // 100M pre-minted
    uint256 public constant MAX_MINTABLE_SUPPLY = 900_000_000 * 10**18; // 900M
    uint256 public constant REWARD_PER_MINT = 2500 * 10**18; // 2,500

    // Minting limits
    uint256 public constant MINT_COST = 1 ether; // 1 MON
    uint256 public constant COOLDOWN_PERIOD = 10; // seconds default
    uint256 public constant MAX_MINTS_PER_WALLET = 1000;

    // Owner / admin
    address public owner;

    // Treasury (receives MINT_COST)
    address public treasury = 0xa71407ED688B2E97bc217C0d05E366DEcD236731;

    // Token state
    uint256 private _totalMinted; // tracks tokens minted (includes treasury reserve set at constructor)
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    // Minting state
    mapping(address => uint256) public lastMintTime;
    mapping(address => uint256) public mintCount;
    mapping(address => uint256) public lastMintBlock;
    mapping(address => uint8) public offenseCount;
    mapping(address => uint256) public blacklistedUntil; // timestamp

    // Nonces per-address for mintWithNonce
    mapping(address => mapping(bytes32 => bool)) public usedNonces;

    // Controls
    bool public paused;
    bool public blockContracts; // optional: owner can enable/disable blocking contract callers

    // Reentrancy guard
    uint256 private _status;
    uint256 private constant _NOT_ENTERED = 1;
    uint256 private constant _ENTERED = 2;

    // Events
    event Transfer(address indexed from, address indexed to, uint256 value);
    event Approval(address indexed owner_, address indexed spender, uint256 value);
    event Minted(address indexed minter, uint256 amount);
    event Blacklisted(address indexed account, uint256 until);
    event Unblacklisted(address indexed account);
    event Paused(address indexed who);
    event Unpaused(address indexed who);
    event TreasuryUpdated(address indexed oldTreasury, address indexed newTreasury);
    event OwnershipTransferred(address indexed oldOwner, address indexed newOwner);
    event RescueETH(address indexed to, uint256 amount);

    // Modifiers
    modifier onlyOwner() {
        require(msg.sender == owner, "STEADY: only owner");
        _;
    }

    modifier nonReentrant() {
        require(_status != _ENTERED, "STEADY: reentrant");
        _status = _ENTERED;
        _;
        _status = _NOT_ENTERED;
    }

    modifier notPaused() {
        require(!paused, "STEADY: paused");
        _;
    }

    modifier notBlacklisted(address account) {
        require(block.timestamp >= blacklistedUntil[account], "STEADY: temporarily blacklisted");
        _;
    }

    constructor() {
        owner = msg.sender;
        _status = _NOT_ENTERED;

        // Mint treasury reserve immediately at deployment and count it toward total minted
        balanceOf[treasury] = TREASURY_RESERVE;
        _totalMinted = TREASURY_RESERVE;
        emit Transfer(address(0), treasury, TREASURY_RESERVE);
    }

    // ======== Ownership ========
    function transferOwnership(address newOwner) external onlyOwner {
        require(newOwner != address(0), "STEADY: zero owner");
        emit OwnershipTransferred(owner, newOwner);
        owner = newOwner;
    }

    function renounceOwnership() external onlyOwner {
        emit OwnershipTransferred(owner, address(0));
        owner = address(0);
    }

    // ======== Treasury ========
    function setTreasury(address newTreasury) external onlyOwner {
        require(newTreasury != address(0), "STEADY: zero treasury");
        emit TreasuryUpdated(treasury, newTreasury);
        treasury = newTreasury;
    }

    // ======== Controls ========
    function pause() external onlyOwner {
        paused = true;
        emit Paused(msg.sender);
    }

    function unpause() external onlyOwner {
        paused = false;
        emit Unpaused(msg.sender);
    }

    function setBlockContracts(bool enable) external onlyOwner {
        blockContracts = enable;
    }

    // Admin: manually set temporary blacklist (until timestamp). Use 0 to clear.
    function setTemporaryBlacklist(address account, uint256 untilTimestamp) external onlyOwner {
        blacklistedUntil[account] = untilTimestamp;
        emit Blacklisted(account, untilTimestamp);
    }

    // Admin: clear blacklist and offense count
    function unblacklist(address account) external onlyOwner {
        blacklistedUntil[account] = 0;
        offenseCount[account] = 0;
        emit Unblacklisted(account);
    }

    // ======== Views ========
    function totalSupply() external pure returns (uint256) {
        return TOTAL_SUPPLY;
    }

    function totalMinted() external view returns (uint256) {
        return _totalMinted;
    }

    function remainingMintableSupply() external view returns (uint256) {
        if (_totalMinted >= MAX_MINTABLE_SUPPLY + TREASURY_RESERVE) return 0;
        // _totalMinted already includes the TREASURY_RESERVE from constructor
        uint256 mintedBeyondReserve = _totalMinted - TREASURY_RESERVE;
        if (mintedBeyondReserve >= MAX_MINTABLE_SUPPLY) return 0;
        return MAX_MINTABLE_SUPPLY - mintedBeyondReserve;
    }

    function remainingMintsForWallet(address account) external view returns (uint256) {
        if (mintCount[account] >= MAX_MINTS_PER_WALLET) return 0;
        return MAX_MINTS_PER_WALLET - mintCount[account];
    }

    function timeUntilNextMint(address account) external view returns (uint256) {
        if (block.timestamp >= lastMintTime[account] + COOLDOWN_PERIOD) return 0;
        return (lastMintTime[account] + COOLDOWN_PERIOD) - block.timestamp;
    }

    function canMint(address account) external view returns (bool) {
        if (paused) return false;
        if (block.timestamp < blacklistedUntil[account]) return false;
        // Check remaining mintable pool (excluding the treasury reserve)
        uint256 mintedBeyondReserve = _totalMinted - TREASURY_RESERVE;
        if (mintedBeyondReserve + REWARD_PER_MINT > MAX_MINTABLE_SUPPLY) return false;
        if (mintCount[account] >= MAX_MINTS_PER_WALLET) return false;
        if (block.timestamp < lastMintTime[account] + COOLDOWN_PERIOD) return false;
        if (blockContracts && _isContract(account)) return false;
        return true;
    }

    // ======== Internal helpers ========
    function _isContract(address account) internal view returns (bool) {
        uint256 size;
        assembly { size := extcodesize(account) }
        return size > 0;
    }

    // Anti-bot policy: if same-block mint attempted, increment offense counter and revert.
    // After 3 offenses, apply temporary ban (duration 1 hour here).
    function _antiBotOnMint(address account) internal {
        if (lastMintBlock[account] == block.number) {
            offenseCount[account] = offenseCount[account] + 1;
            if (offenseCount[account] >= 3) {
                uint256 until = block.timestamp + 1 hours;
                blacklistedUntil[account] = until;
                emit Blacklisted(account, until);
                revert("STEADY: repeated same-block mint attempts - temporarily banned");
            }
            revert("STEADY: multiple mints in same block detected");
        }
    }

    // Minting internal (updates only mintable portion)
    function _mintTo(address to) internal {
        // Ensure we don't exceed MAX_MINTABLE_SUPPLY (excluding treasury reserve)
        uint256 mintedBeyondReserve = _totalMinted - TREASURY_RESERVE;
        require(mintedBeyondReserve + REWARD_PER_MINT <= MAX_MINTABLE_SUPPLY, "STEADY: max mintable reached");

        _totalMinted += REWARD_PER_MINT;
        balanceOf[to] += REWARD_PER_MINT;
        emit Minted(to, REWARD_PER_MINT);
        emit Transfer(address(0), to, REWARD_PER_MINT);
    }

    // ======== Mint functions ========
    function mint() external payable nonReentrant notPaused notBlacklisted(msg.sender) {
        require(msg.value == MINT_COST, "STEADY: fee is 1 MON");
        if (blockContracts) require(!_isContract(msg.sender), "STEADY: contracts blocked");
        // Check mintable pool
        uint256 mintedBeyondReserve = _totalMinted - TREASURY_RESERVE;
        require(mintedBeyondReserve + REWARD_PER_MINT <= MAX_MINTABLE_SUPPLY, "STEADY: max mintable reached");
        require(mintCount[msg.sender] < MAX_MINTS_PER_WALLET, "STEADY: max mints per wallet reached");
        require(block.timestamp >= lastMintTime[msg.sender] + COOLDOWN_PERIOD, "STEADY: cooldown not passed");

        // Anti-bot check
        _antiBotOnMint(msg.sender);

        // Update state BEFORE external call
        lastMintTime[msg.sender] = block.timestamp;
        lastMintBlock[msg.sender] = block.number;
        mintCount[msg.sender] += 1;

        // Forward fee to treasury - send and require success
        (bool sent, ) = payable(treasury).call{value: msg.value}("");
        require(sent, "STEADY: treasury transfer failed");

        // Mint tokens to sender
        _mintTo(msg.sender);
    }

    function mintWithNonce(bytes32 nonce) external payable nonReentrant notPaused notBlacklisted(msg.sender) {
        require(msg.value == MINT_COST, "STEADY: fee is 1 MON");
        if (blockContracts) require(!_isContract(msg.sender), "STEADY: contracts blocked");
        require(!usedNonces[msg.sender][nonce], "STEADY: nonce already used");

        // Check mintable pool
        uint256 mintedBeyondReserve = _totalMinted - TREASURY_RESERVE;
        require(mintedBeyondReserve + REWARD_PER_MINT <= MAX_MINTABLE_SUPPLY, "STEADY: max mintable reached");
        require(mintCount[msg.sender] < MAX_MINTS_PER_WALLET, "STEADY: max mints per wallet reached");
        require(block.timestamp >= lastMintTime[msg.sender] + COOLDOWN_PERIOD, "STEADY: cooldown not passed");

        // Anti-bot check
        _antiBotOnMint(msg.sender);

        // Mark nonce used for this address
        usedNonces[msg.sender][nonce] = true;

        // Update state BEFORE external call
        lastMintTime[msg.sender] = block.timestamp;
        lastMintBlock[msg.sender] = block.number;
        mintCount[msg.sender] += 1;

        // Forward fee
        (bool sent, ) = payable(treasury).call{value: msg.value}("");
        require(sent, "STEADY: treasury transfer failed");

        // Mint tokens
        _mintTo(msg.sender);
    }

    // ======== ERC20 standard functions (minimal) ========
    function transfer(address to, uint256 amount) external returns (bool) {
        require(to != address(0), "STEADY: zero address");
        require(balanceOf[msg.sender] >= amount, "STEADY: insufficient balance");
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        require(spender != address(0), "STEADY: zero spender");
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        require(to != address(0), "STEADY: zero address");
        require(balanceOf[from] >= amount, "STEADY: insufficient balance");
        require(allowance[from][msg.sender] >= amount, "STEADY: insufficient allowance");

        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        allowance[from][msg.sender] -= amount;

        emit Transfer(from, to, amount);
        return true;
    }

    // convenience: increase/decrease allowance
    function increaseAllowance(address spender, uint256 addedValue) external returns (bool) {
        allowance[msg.sender][spender] += addedValue;
        emit Approval(msg.sender, spender, allowance[msg.sender][spender]);
        return true;
    }

    function decreaseAllowance(address spender, uint256 subtractedValue) external returns (bool) {
        uint256 old = allowance[msg.sender][spender];
        if (subtractedValue >= old) {
            allowance[msg.sender][spender] = 0;
        } else {
            allowance[msg.sender][spender] = old - subtractedValue;
        }
        emit Approval(msg.sender, spender, allowance[msg.sender][spender]);
        return true;
    }

    // ======== Emergency / Owner actions ========
    // Rescue any ETH accidentally sent to the contract
    function rescueETH(address payable to) external onlyOwner nonReentrant {
        require(to != address(0), "STEADY: zero address");
        uint256 bal = address(this).balance;
        require(bal > 0, "STEADY: no eth");
        (bool sent, ) = to.call{value: bal}("");
        require(sent, "STEADY: rescue failed");
        emit RescueETH(to, bal);
    }

    // Fallback/receive to accept ETH (but normal operation forwards fees to treasury)
    receive() external payable {}
}
