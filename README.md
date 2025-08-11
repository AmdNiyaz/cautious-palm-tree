// SPDX-License-Identifier: MIT
pragma solidity ^0.8.21;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/Pausable.sol";
import "@openzeppelin/contracts/utils/math/SafeMath.sol";

import "@uniswap/v2-periphery/contracts/interfaces/IUniswapV2Router02.sol";
import "@uniswap/v2-core/contracts/interfaces/IUniswapV2Factory.sol";

contract RFToken is IERC20, Ownable, Pausable {
    using SafeMath for uint256;

    string public constant name = "RF Token";
    string public constant symbol = "RFT";
    uint8 public constant decimals = 9;
    uint256 private _totalSupply = 1_000_000_000_000_000 * 10**decimals;

    mapping(address => uint256) private _balances;
    mapping(address => mapping(address => uint256)) private _allowances;

    mapping(address => bool) private _isExcludedFromFee;

    address public constant deadAddress = 0x000000000000000000000000000000000000dEaD;

    address public ownerWallet;
    address public ecovisionWallet;

    uint256 public taxFee = 5;          // Max 7%
    uint256 public liquidityFee = 2;    // Max 3%
    uint256 public burnFee = 1;         // Max 2%
    uint256 public ecovisionFee = 2;    // Max 3%

    uint256 public constant MAX_TAX_FEE = 7;
    uint256 public constant MAX_LIQUIDITY_FEE = 3;
    uint256 public constant MAX_BURN_FEE = 2;
    uint256 public constant MAX_ECOVISION_FEE = 3;

    IUniswapV2Router02 public uniswapV2Router;
    address public uniswapV2Pair;
    address public usdtToken;

    bool private inSwapAndLiquify;
    bool public swapAndLiquifyEnabled = true;
    uint256 public numTokensSellToAddToLiquidity = 500_000 * 10**decimals;

    modifier lockTheSwap {
        inSwapAndLiquify = true;
        _;
        inSwapAndLiquify = false;
    }

    event SwapAndLiquifyEnabledUpdated(bool enabled);
    event OwnerWalletUpdated(address indexed newOwnerWallet);
    event EcovisionWalletUpdated(address indexed newEcovisionWallet);

    constructor(
        address _router,
        address _usdtToken,
        address _ownerWallet,
        address _ecovisionWallet
    ) {
        require(_ownerWallet != address(0), "Owner wallet zero address");
        require(_ecovisionWallet != address(0), "Ecovision wallet zero address");

        ownerWallet = _ownerWallet;
        ecovisionWallet = _ecovisionWallet;
        usdtToken = _usdtToken;

        _balances[msg.sender] = _totalSupply;
        emit Transfer(address(0), msg.sender, _totalSupply);

        IUniswapV2Router02 _uniswapV2Router = IUniswapV2Router02(_router);
        uniswapV2Pair = IUniswapV2Factory(_uniswapV2Router.factory())
            .createPair(address(this), _usdtToken);
        uniswapV2Router = _uniswapV2Router;

        _isExcludedFromFee[msg.sender] = true;
        _isExcludedFromFee[address(this)] = true;
        _isExcludedFromFee[ownerWallet] = true;
        _isExcludedFromFee[ecovisionWallet] = true;
    }

    // ERC20 standard functions

    function totalSupply() external view override returns (uint256) {
        return _totalSupply;
    }

    function balanceOf(address account) external view override returns (uint256) {
        return _balances[account];
    }

    function transfer(address recipient, uint256 amount) external override whenNotPaused returns (bool) {
        _transfer(msg.sender, recipient, amount);
        return true;
    }

    function allowance(address owner_, address spender) external view override returns (uint256) {
        return _allowances[owner_][spender];
    }

    function approve(address spender, uint256 amount) external override whenNotPaused returns (bool) {
        _approve(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address sender,address recipient,uint256 amount) external override whenNotPaused returns (bool) {
        _transfer(sender, recipient, amount);
        uint256 currentAllowance = _allowances[sender][msg.sender];
        require(currentAllowance >= amount, "ERC20: transfer amount exceeds allowance");
        unchecked {
            _approve(sender, msg.sender, currentAllowance - amount);
        }
        return true;
    }

    function increaseAllowance(address spender, uint256 addedValue) external whenNotPaused returns (bool) {
        _approve(msg.sender, spender, _allowances[msg.sender][spender].add(addedValue));
        return true;
    }

    function decreaseAllowance(address spender, uint256 subtractedValue) external whenNotPaused returns (bool) {
        uint256 currentAllowance = _allowances[msg.sender][spender];
        require(currentAllowance >= subtractedValue, "ERC20: decreased allowance below zero");
        unchecked {
            _approve(msg.sender, spender, currentAllowance - subtractedValue);
        }
        return true;
    }

    // Owner functions

    function excludeFromFee(address account) external onlyOwner {
        _isExcludedFromFee[account] = true;
    }

    function includeInFee(address account) external onlyOwner {
        _isExcludedFromFee[account] = false;
    }

    function setOwnerWallet(address _ownerWallet) external onlyOwner {
        require(_ownerWallet != address(0), "Owner wallet zero address");
        ownerWallet = _ownerWallet;
        _isExcludedFromFee[_ownerWallet] = true;
        emit OwnerWalletUpdated(_ownerWallet);
    }

    function setEcovisionWallet(address _ecovisionWallet) external onlyOwner {
        require(_ecovisionWallet != address(0), "Ecovision wallet zero address");
        ecovisionWallet = _ecovisionWallet;
        _isExcludedFromFee[_ecovisionWallet] = true;
        emit EcovisionWalletUpdated(_ecovisionWallet);
    }

    function setSwapAndLiquifyEnabled(bool _enabled) external onlyOwner {
        swapAndLiquifyEnabled = _enabled;
        emit SwapAndLiquifyEnabledUpdated(_enabled);
    }

    function setNumTokensSellToAddToLiquidity(uint256 amount) external onlyOwner {
        numTokensSellToAddToLiquidity = amount;
    }

    function setTaxFee(uint256 fee) external onlyOwner {
        require(fee <= MAX_TAX_FEE, "Tax fee too high");
        taxFee = fee;
    }

    function setLiquidityFee(uint256 fee) external onlyOwner {
        require(fee <= MAX_LIQUIDITY_FEE, "Liquidity fee too high");
        liquidityFee = fee;
    }

    function setBurnFee(uint256 fee) external onlyOwner {
        require(fee <= MAX_BURN_FEE, "Burn fee too high");
        burnFee = fee;
    }

    function setEcovisionFee(uint256 fee) external onlyOwner {
        require(fee <= MAX_ECOVISION_FEE, "Ecovision fee too high");
        ecovisionFee = fee;
    }

    // Pause/unpause trading
    function pause() external onlyOwner {
        _pause();
    }

    function unpause() external onlyOwner {
        _unpause();
    }

    // Internal transfer with fees

    function _transfer(address sender, address recipient, uint256 amount) internal {
        require(sender != address(0), "ERC20: transfer from zero");
        require(recipient != address(0), "ERC20: transfer to zero");
        require(amount > 0, "Transfer amount zero");

        uint256 transferAmount = amount;

        if (!_isExcludedFromFee[sender] && !_isExcludedFromFee[recipient]) {
            uint256 totalFees = taxFee.add(liquidityFee).add(burnFee).add(ecovisionFee);
            uint256 fees = amount.mul(totalFees).div(100);

            uint256 taxAmount = amount.mul(taxFee).div(100);
            uint256 liquidityAmount = amount.mul(liquidityFee).div(100);
            uint256 burnAmount = amount.mul(burnFee).div(100);
            uint256 ecovisionAmount = amount.mul(ecovisionFee).div(100);

            // Collect fees in contract
            _balances[address(this)] = _balances[address(this)]
                .add(liquidityAmount)
                .add(taxAmount)
                .add(ecovisionAmount);

            // Burn tokens
            if (burnAmount > 0) {
                _balances[deadAddress] = _balances[deadAddress].add(burnAmount);
                emit Transfer(sender, deadAddress, burnAmount);
            }

            emit Transfer(sender, address(this), liquidityAmount.add(taxAmount).add(ecovisionAmount));

            transferAmount = amount.sub(fees);
        }

        _balances[sender] = _balances[sender].sub(amount, "ERC20: transfer exceeds balance");
        _balances[recipient] = _balances[recipient].add(transferAmount);
        emit Transfer(sender, recipient, transferAmount);

        // Swap and liquify trigger
        if (
            swapAndLiquifyEnabled &&
            !inSwapAndLiquify &&
            sender != uniswapV2Pair &&
            _balances[address(this)] >= numTokensSellToAddToLiquidity
        ) {
            swapAndLiquify(numTokensSellToAddToLiquidity);
        }
    }

    // Swap and liquify

    function swapAndLiquify(uint256 contractTokenBalance) private lockTheSwap {
        // Calculate liquidity tokens portion
        uint256 totalFeeExcludingBurn = taxFee.add(liquidityFee).add(ecovisionFee);
        uint256 liquidityTokens = contractTokenBalance.mul(liquidityFee).div(totalFeeExcludingBurn);
        uint256 tokensToSwapForUSDT = contractTokenBalance.sub(liquidityTokens);

        uint256 halfLiquidity = liquidityTokens.div(2);
        uint256 otherHalfLiquidity = liquidityTokens.sub(halfLiquidity);

        uint256 initialUSDTBalance = IERC20(usdtToken).balanceOf(address(this));

        // Swap tokens (tax + ecovision + half liquidity) for USDT
        swapTokensForUSDT(tokensToSwapForUSDT);

        uint256 newUSDTBalance = IERC20(usdtToken).balanceOf(address(this)).sub(initialUSDTBalance);

        // USDT for liquidity corresponding to half liquidity tokens swapped
        uint256 usdtForLiquidity = newUSDTBalance.mul(halfLiquidity).div(tokensToSwapForUSDT);

        // Add liquidity
        addLiquidity(otherHalfLiquidity, usdtForLiquidity);

        // Remaining USDT = tax + ecovision fees sent to wallets
        uint256 usdtForEcovision = newUSDTBalance.mul(ecovisionFee).div(totalFeeExcludingBurn);
        uint256 usdtForOwner = IERC20(usdtToken).balanceOf(address(this)).sub(usdtForEcovision);

        if (usdtForEcovision > 0) {
            IERC20(usdtToken).transfer(ecovisionWallet, usdtForEcovision);
        }

        if (usdtForOwner > 0) {
            IERC20(usdtToken).transfer(ownerWallet, usdtForOwner);
        }
    }

    // Swap tokens for USDT using PancakeSwap router

    function swapTokensForUSDT(uint256 tokenAmount) private {
        address ;
        path[0] = address(this);
        path[1] = uniswapV2Router.WETH();
        path[2] = usdtToken;

        _approve(address(this), address(uniswapV2Router), tokenAmount);

        uniswapV2Router.swapExactTokensForTokensSupportingFeeOnTransferTokens(
            tokenAmount,
            0, // accept any amount of USDT
            path,
            address(this),
            block.timestamp
        );
    }

    // Add liquidity to PancakeSwap

    function addLiquidity(uint256 tokenAmount, uint256 usdtAmount) private {
        _approve(address(this), address(uniswapV2Router), tokenAmount);
        IERC20(usdtToken).approve(address(uniswapV2Router), usdtAmount);

        uniswapV2Router.addLiquidity(
            address(this),
            usdtToken,
            tokenAmount,
            usdtAmount,
            0,
            0,
            ownerWallet,
            block.timestamp
        );
    }

    // Approve helper

    function _approve(address owner_, address spender, uint256 amount) internal {
        require(owner_ != address(0), "ERC20: approve from zero");
        require(spender != address(0), "ERC20: approve to zero");

        _allowances[owner_][spender] = amount;
        emit Approval(owner_, spender, amount);
    }
}
