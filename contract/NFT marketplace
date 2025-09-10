// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/IERC721.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";
import "@openzeppelin/contracts/access/Ownable.sol";

contract Project is ReentrancyGuard, Ownable {
    
    // Marketplace fee percentage (2.5%)
    uint256 public marketplaceFee = 250; // 250 basis points = 2.5%
    uint256 public constant PERCENTAGE_BASE = 10000;
    
    // Counter for listing IDs
    uint256 private _listingIdCounter;
    
    // Listing structure
    struct Listing {
        uint256 listingId;
        address nftContract;
        uint256 tokenId;
        address seller;
        uint256 price;
        bool active;
        uint256 createdAt;
    }
    
    // Mappings
    mapping(uint256 => Listing) public listings;
    mapping(address => mapping(uint256 => uint256)) public tokenToListing; // nftContract => tokenId => listingId
    
    // Events
    event ItemListed(
        uint256 indexed listingId,
        address indexed nftContract,
        uint256 indexed tokenId,
        address seller,
        uint256 price
    );
    
    event ItemSold(
        uint256 indexed listingId,
        address indexed nftContract,
        uint256 indexed tokenId,
        address seller,
        address buyer,
        uint256 price
    );
    
    event ListingCancelled(
        uint256 indexed listingId,
        address indexed nftContract,
        uint256 indexed tokenId,
        address seller
    );
    
    constructor() Ownable(msg.sender) {}
    
    /**
     * @dev List an NFT for sale on the marketplace
     * @param nftContract Address of the NFT contract
     * @param tokenId Token ID of the NFT
     * @param price Price in wei for the NFT
     */
    function listItem(
        address nftContract,
        uint256 tokenId,
        uint256 price
    ) external nonReentrant {
        require(price > 0, "Price must be greater than zero");
        require(nftContract != address(0), "Invalid NFT contract address");
        
        IERC721 nft = IERC721(nftContract);
        require(nft.ownerOf(tokenId) == msg.sender, "You don't own this NFT");
        require(
            nft.getApproved(tokenId) == address(this) || 
            nft.isApprovedForAll(msg.sender, address(this)),
            "Marketplace not approved to transfer NFT"
        );
        
        // Check if item is already listed
        uint256 existingListingId = tokenToListing[nftContract][tokenId];
        if (existingListingId != 0) {
            require(!listings[existingListingId].active, "Item already listed");
        }
        
        _listingIdCounter++;
        uint256 listingId = _listingIdCounter;
        
        listings[listingId] = Listing({
            listingId: listingId,
            nftContract: nftContract,
            tokenId: tokenId,
            seller: msg.sender,
            price: price,
            active: true,
            createdAt: block.timestamp
        });
        
        tokenToListing[nftContract][tokenId] = listingId;
        
        emit ItemListed(listingId, nftContract, tokenId, msg.sender, price);
    }
    
    /**
     * @dev Buy an NFT from the marketplace
     * @param listingId ID of the listing to purchase
     */
    function buyItem(uint256 listingId) external payable nonReentrant {
        Listing storage listing = listings[listingId];
        
        require(listing.active, "Listing not active");
        require(msg.value >= listing.price, "Insufficient payment");
        require(msg.sender != listing.seller, "Cannot buy your own NFT");
        
        IERC721 nft = IERC721(listing.nftContract);
        require(nft.ownerOf(listing.tokenId) == listing.seller, "Seller no longer owns NFT");
        
        // Mark listing as inactive
        listing.active = false;
        
        // Calculate fees
        uint256 totalPrice = listing.price;
        uint256 marketplaceFeeAmount = (totalPrice * marketplaceFee) / PERCENTAGE_BASE;
        uint256 sellerAmount = totalPrice - marketplaceFeeAmount;
        
        // Transfer NFT to buyer
        nft.safeTransferFrom(listing.seller, msg.sender, listing.tokenId);
        
        // Transfer payments
        payable(listing.seller).transfer(sellerAmount);
        payable(owner()).transfer(marketplaceFeeAmount);
        
        // Refund excess payment
        if (msg.value > totalPrice) {
            payable(msg.sender).transfer(msg.value - totalPrice);
        }
        
        emit ItemSold(
            listingId,
            listing.nftContract,
            listing.tokenId,
            listing.seller,
            msg.sender,
            totalPrice
        );
    }
    
    /**
     * @dev Cancel a listing
     * @param listingId ID of the listing to cancel
     */
    function cancelListing(uint256 listingId) external nonReentrant {
        Listing storage listing = listings[listingId];
        
        require(listing.active, "Listing not active");
        require(
            msg.sender == listing.seller || msg.sender == owner(),
            "Only seller or owner can cancel listing"
        );
        
        listing.active = false;
        
        emit ListingCancelled(
            listingId,
            listing.nftContract,
            listing.tokenId,
            listing.seller
        );
    }
    
    // View functions
    function getListing(uint256 listingId) external view returns (Listing memory) {
        return listings[listingId];
    }
    
    function getActiveListingsCount() external view returns (uint256) {
        uint256 activeCount = 0;
        for (uint256 i = 1; i <= _listingIdCounter; i++) {
            if (listings[i].active) {
                activeCount++;
            }
        }
        return activeCount;
    }
    
    function getTotalListings() external view returns (uint256) {
        return _listingIdCounter;
    }
    
    // Owner functions
    function setMarketplaceFee(uint256 _fee) external onlyOwner {
        require(_fee <= 1000, "Fee cannot exceed 10%"); // Max 10%
        marketplaceFee = _fee;
    }
    
    function withdrawFees() external onlyOwner {
        payable(owner()).transfer(address(this).balance);
    }
}
