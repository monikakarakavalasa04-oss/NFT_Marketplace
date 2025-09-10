// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";
import "@openzeppelin/contracts/utils/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

/**
 * @title NFT Marketplace
 * @dev A decentralized marketplace for buying and selling NFTs
 */
contract NFTMarketplace is ReentrancyGuard, Ownable, IERC721Receiver {
    
    // Struct to represent a marketplace listing
    struct Listing {
        address seller;
        address nftContract;
        uint256 tokenId;
        uint256 price;
        bool active;
    }
    
    // State variables
    mapping(bytes32 => Listing) public listings;
    mapping(address => uint256) public proceeds;
    
    uint256 public marketplaceFee = 250; // 2.5% fee (250 basis points)
    uint256 private constant BASIS_POINTS = 10000;
    
    // Events
    event ItemListed(
        address indexed seller,
        address indexed nftContract,
        uint256 indexed tokenId,
        uint256 price
    );
    
    event ItemSold(
        address indexed buyer,
        address indexed nftContract,
        uint256 indexed tokenId,
        uint256 price
    );
    
    event ItemCanceled(
        address indexed seller,
        address indexed nftContract,
        uint256 tokenId
    );
    
    event ProceedsWithdrawn(
        address indexed user,
        uint256 amount
    );
    
    event MarketplaceFeeUpdated(
        uint256 oldFee,
        uint256 newFee
    );
    
    // Modifiers
    modifier notListed(address nftContract, uint256 tokenId) {
        bytes32 listingId = getListingId(nftContract, tokenId);
        require(!listings[listingId].active, "Item already listed");
        _;
    }
    
    modifier isListed(address nftContract, uint256 tokenId) {
        bytes32 listingId = getListingId(nftContract, tokenId);
        require(listings[listingId].active, "Item not listed");
        _;
    }
    
    modifier isOwner(address nftContract, uint256 tokenId, address spender) {
        IERC721 nft = IERC721(nftContract);
        address owner = nft.ownerOf(tokenId);
        require(spender == owner, "Not the owner");
        _;
    }
    
    modifier validAddress(address addr) {
        require(addr != address(0), "Invalid address");
        _;
    }
    
    constructor() Ownable(msg.sender) {}
    
    /**
     * @dev Core Function 1: List an NFT for sale
     * @param nftContract The contract address of the NFT
     * @param tokenId The token ID of the NFT
     * @param price The price in wei to list the NFT for
     */
    function listItem(
        address nftContract,
        uint256 tokenId,
        uint256 price
    )
        external
        validAddress(nftContract)
        notListed(nftContract, tokenId)
        isOwner(nftContract, tokenId, msg.sender)
    {
        require(price > 0, "Price must be above zero");
        
        IERC721 nft = IERC721(nftContract);
        require(
            nft.getApproved(tokenId) == address(this) || 
            nft.isApprovedForAll(msg.sender, address(this)),
            "Marketplace not approved to transfer NFT"
        );
        
        bytes32 listingId = getListingId(nftContract, tokenId);
        listings[listingId] = Listing({
            seller: msg.sender,
            nftContract: nftContract,
            tokenId: tokenId,
            price: price,
            active: true
        });
        
        emit ItemListed(msg.sender, nftContract, tokenId, price);
    }
    
    /**
     * @dev Core Function 2: Buy an NFT that's listed for sale
     * @param nftContract The contract address of the NFT
     * @param tokenId The token ID of the NFT to buy
     */
    function buyItem(address nftContract, uint256 tokenId)
        external
        payable
        nonReentrant
        validAddress(nftContract)
        isListed(nftContract, tokenId)
    {
        bytes32 listingId = getListingId(nftContract, tokenId);
        Listing memory listing = listings[listingId];
        
        require(msg.value >= listing.price, "Price not met");
        require(msg.sender != listing.seller, "Cannot buy your own NFT");
        
        // Calculate marketplace fee
        uint256 fee = (listing.price * marketplaceFee) / BASIS_POINTS;
        uint256 sellerProceeds = listing.price - fee;
        
        // Update seller's proceeds
        proceeds[listing.seller] += sellerProceeds;
        proceeds[owner()] += fee;
        
        // Remove the listing
        delete listings[listingId];
        
        // Transfer the NFT
        IERC721(nftContract).safeTransferFrom(
            listing.seller,
            msg.sender,
            tokenId,
            ""
        );
        
        // Refund excess payment
        if (msg.value > listing.price) {
            (bool refundSuccess, ) = payable(msg.sender).call{value: msg.value - listing.price}("");
            require(refundSuccess, "Refund failed");
        }
        
        emit ItemSold(msg.sender, nftContract, tokenId, listing.price);
    }
    
    /**
     * @dev Core Function 3: Cancel a listing
     * @param nftContract The contract address of the NFT
     * @param tokenId The token ID of the NFT to cancel
     */
    function cancelListing(address nftContract, uint256 tokenId)
        external
        validAddress(nftContract)
        isListed(nftContract, tokenId)
        isOwner(nftContract, tokenId, msg.sender)
    {
        bytes32 listingId = getListingId(nftContract, tokenId);
        delete listings[listingId];
        
        emit ItemCanceled(msg.sender, nftContract, tokenId);
    }
    
    /**
     * @dev Withdraw proceeds from sales
     */
    function withdrawProceeds() external nonReentrant {
        uint256 amount = proceeds[msg.sender];
        require(amount > 0, "No proceeds to withdraw");
        
        proceeds[msg.sender] = 0;
        
        (bool success, ) = payable(msg.sender).call{value: amount}("");
        require(success, "Transfer failed");
        
        emit ProceedsWithdrawn(msg.sender, amount);
    }
    
    /**
     * @dev Update the marketplace fee (only owner)
     * @param newFee The new fee in basis points (e.g., 250 = 2.5%)
     */
    function updateMarketplaceFee(uint256 newFee) external onlyOwner {
        require(newFee <= 1000, "Fee cannot exceed 10%"); // Max 10% fee
        
        uint256 oldFee = marketplaceFee;
        marketplaceFee = newFee;
        
        emit MarketplaceFeeUpdated(oldFee, newFee);
    }
    
    /**
     * @dev Update listing price (only by seller)
     * @param nftContract The contract address of the NFT
     * @param tokenId The token ID of the NFT
     * @param newPrice The new price in wei
     */
    function updateListingPrice(
        address nftContract,
        uint256 tokenId,
        uint256 newPrice
    )
        external
        validAddress(nftContract)
        isListed(nftContract, tokenId)
        isOwner(nftContract, tokenId, msg.sender)
    {
        require(newPrice > 0, "Price must be above zero");
        
        bytes32 listingId = getListingId(nftContract, tokenId);
        listings[listingId].price = newPrice;
        
        emit ItemListed(msg.sender, nftContract, tokenId, newPrice);
    }
    
    /**
     * @dev Get listing details
     * @param nftContract The contract address of the NFT
     * @param tokenId The token ID of the NFT
     * @return The listing details
     */
    function getListing(address nftContract, uint256 tokenId)
        external
        view
        returns (Listing memory)
    {
        bytes32 listingId = getListingId(nftContract, tokenId);
        return listings[listingId];
    }
    
    /**
     * @dev Check if an NFT is listed
     * @param nftContract The contract address of the NFT
     * @param tokenId The token ID of the NFT
     * @return bool indicating if the NFT is listed
     */
    function isItemListed(address nftContract, uint256 tokenId)
        external
        view
        returns (bool)
    {
        bytes32 listingId = getListingId(nftContract, tokenId);
        return listings[listingId].active;
    }
    
    /**
     * @dev Get user's proceeds
     * @param user The address of the user
     * @return The amount of proceeds
     */
    function getProceeds(address user) external view returns (uint256) {
        return proceeds[user];
    }
    
    /**
     * @dev Handle the receipt of an NFT
     * @return bytes4 selector to confirm token transfer
     */
    function onERC721Received(
        address,
        address,
        uint256,
        bytes calldata
    ) external pure override returns (bytes4) {
        return IERC721Receiver.onERC721Received.selector;
    }
    
    /**
     * @dev Generate a unique listing ID
     * @param nftContract The contract address of the NFT
     * @param tokenId The token ID of the NFT
     * @return The unique listing ID
     */
    function getListingId(address nftContract, uint256 tokenId)
        internal
        pure
        returns (bytes32)
    {
        return keccak256(abi.encodePacked(nftContract, tokenId));
    }
    
    /**
     * @dev Emergency function to pause contract (only owner)
     * Note: This is a basic implementation - consider using OpenZeppelin's Pausable for production
     */
    function emergencyWithdraw() external onlyOwner {
        uint256 balance = address(this).balance;
        require(balance > 0, "No balance to withdraw");
        
        (bool success, ) = payable(owner()).call{value: balance}("");
        require(success, "Emergency withdrawal failed");
    }
    
    /**
     * @dev Fallback function to reject direct ETH transfers
     */
    receive() external payable {
        revert("Direct payments not allowed");
    }
}
