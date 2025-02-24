# tiered-erc20


````
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract FanToken is ERC20, Ownable {
    struct Tier {
        uint256 minTokens;
        uint256 maxTokens;
    }

    mapping(uint256 => Tier) public tierThresholds; // index => tier range
    uint256[] private tierIndexes;

    mapping(uint256 => string) private _tokenURIs; // tokenId -> tokenURI

    event TierUpdated(uint256 indexed tier, uint256 minTokens, uint256 maxTokens);
    event TiersCleared();

    constructor(
        string memory name, 
        string memory symbol, 
        uint256 initialSupply,
        uint256[] memory tiers,
        uint256[] memory minTokens,
        uint256[] memory maxTokens,
        string[] memory tierURIs // new parameter for token URIs
    ) ERC20(name, symbol) {
        require(minTokens.length == maxTokens.length && minTokens.length == tiers.length, "Arrays must match");
        require(tiers.length == tierURIs.length, "URIs must match tiers");
        _mint(msg.sender, initialSupply * 10 ** decimals());
        
        for (uint256 i = 0; i < tiers.length; i++) {
            tierThresholds[tiers[i]] = Tier(minTokens[i], maxTokens[i]);
            tierIndexes.push(tiers[i]);
            _tokenURIs[tiers[i]] = tierURIs[i]; // setting the tokenURI during deployment
            emit TierUpdated(tiers[i], minTokens[i], maxTokens[i]);
        }
    }

    function setTier(uint256 tier, uint256 minTokens, uint256 maxTokens) external onlyOwner {
        if (tierThresholds[tier].minTokens == 0 && tierThresholds[tier].maxTokens == 0) {
            tierIndexes.push(tier);
        }
        tierThresholds[tier] = Tier(minTokens, maxTokens);
        emit TierUpdated(tier, minTokens, maxTokens);
    }

    function clearTiers() external onlyOwner {
        for (uint256 i = 0; i < tierIndexes.length; i++) {
            delete tierThresholds[tierIndexes[i]];
        }
        delete tierIndexes;
        emit TiersCleared();
    }

    function getTier(address account) public view returns (uint256[] memory) {
        uint256 balance = balanceOf(account);
        uint256[] memory tempTiers = new uint256[](tierIndexes.length);
        uint256 count = 0;

        for (uint256 i = 0; i < tierIndexes.length; i++) {
            uint256 tier = tierIndexes[i];
            if (tierThresholds[tier].minTokens > 0 && balance >= tierThresholds[tier].minTokens && balance <= tierThresholds[tier].maxTokens) {
                tempTiers[count] = tier;
                count++;
            }
        }
        
        uint256[] memory finalTiers = new uint256[](count);
        for (uint256 j = 0; j < count; j++) {
            finalTiers[j] = tempTiers[j];
        }
        return finalTiers;
    }
    
    function hasAccess(address account, uint256 requiredTier) public view returns (bool) {
        uint256[] memory userTiers = getTier(account);
        for (uint256 i = 0; i < userTiers.length; i++) {
            if (userTiers[i] == requiredTier) {
                return true;
            }
        }
        return false;
    }

    // Function to get the tokenURI for a specific token ID (tier-based URI)
    function tokenURI(uint256 tierId) external view returns (string memory) {
        return _tokenURIs[tierId];
    }
}
````
