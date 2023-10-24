---
sip: 14
title: Redeemable Contract Offerer
description: A Seaport Contract Offerer that uses ERC-7496 Dynamic Traits to enable onchain redemptions
author: Ryan Ghods (@ryanio), 0age (@0age), Stephan Min (@stephankmin)
discussions-to: https://github.com/ProjectOpenSea/SIPs/discussions/18
status: Draft
type: Standards Track
category: ERC
created: 2023-07-28
requires: 5
---

## Abstract

This specification proposes a standard for a Seaport Contract Offerer that uses ERC-7496 Dynamic Traits to enable onchain and offchain redemptions. This also allows a Seaport zone to use the Dynamic Traits standard to ensure redemptions cannot be frontrun when NFTs are listed to be sold with a guarantee of certain traits.

## Motivation

Traits and metadata for non-fungible and semi-fungible tokens are often stored offchain. This makes it difficult to query and mutate these values in contract code. ERC-7496 Dynamic Traits brings certain traits onchain, to enable contracts like this Redeemable Contract Offerer to use them to create onchain token and offchain product redemptions.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in RFC 2119 and RFC 8174.

The contract offerer MUST have the following interface and MUST return `true` for EIP-165 supportsInterface for `0x12345678(placeholder, to be set here when finalized)`, the 4 byte interfaceId of the below.

```solidity
interface IRedeemableContractOfferer {
  /* Events */
  event CampaignUpdated(uint256 indexed campaignId, CampaignParams params, string URI);
  event Redemption(uint256 indexed campaignId, uint256 requirementsIndex, bytes32 redemptionHash);

  /* Structs */
  struct CampaignParams {
      uint32 startTime;
      uint32 endTime;
      uint32 maxCampaignRedemptions;
      address manager; // the address that can modify the campaign
      address signer; // null address means no EIP-712 signature required
      CampaignRequirements[] requirements;
  }
  struct CampaignRequirements {
      OfferItem[] offer; // items to be minted, can be empty for offchain redeemable
      ConsiderationItem[] consideration; // items transferring to recipient
      TraitRedemption[] traitRedemptions; // the trait redemptions
  }
  struct TraitRedemption {
    uint8 substandard;
    address token;
    uint256 identifier;
    bytes32 traitKey;
    bytes32 traitValue;
    bytes32 substandardValue;
  }

  /* Getters */
  function getCampaign(uint256 campaignId) external view returns (CampaignParams memory params, string memory uri, uint256 totalRedemptions);

  /* Setters */
  function createCampaign(CampaignParams calldata params, string calldata uri) external returns (uint256 campaignId);
  function updateCampaign(uint256 campaignId, CampaignParams calldata params, string calldata uri) external;
}
```

### Creating campaigns

When creating a new campaign, `createCampaign` MUST be used and MUST return the newly created `campaignId` along with the `CampaignUpdated` event. The `campaignId` MUST be an incrementing counter starting at `1`.

### Updating campaigns

Updates to campaigns MUST use `updateCampaign` and MUST emit the `CampaignUpdated` event. If an address other than the `manager` tries to update the campaign, it MUST revert with `NotManager()`. If the manager wishes to make the campaign immutable, the `manager` MAY be set to the null address.

### Offer

If tokens are set in the params `offer`, the tokens MUST implement the `IRedemptionMintable` interface in order to support minting new items. The implementation SHOULD be however the token mechanics are desired. The implementing token MUST return true for ERC-165 `supportsInterface` for the interfaceId of `IRedemptionMintable`, `0x12345678(placeholder, to be set here when finalized)`.

```solidify
interface IRedemptionMintable {
  function mintRedemption(uint256 campaignId, address recipient, SpentItem[] calldata spent) external;
}
```

### Consideration

Any token may be used in the RedeemableParams `consideration`. This will ensure the token is transferred to the `recipient`. If the token is meant to be burned the recipient SHOULD be `0x000000000000000000000000000000000000dEaD`, since it is against ERC-721 and ERC-1155 specifications to transfer items to the null address.

### Signer

A signer MAY be specified to provide a signature to process the redemption. If the signer is NOT the null address, the signature MUST recover to the signer address via EIP-712 or EIP-1271.

The EIP-712 struct for signing MUST be as follows: `SignedRedeem(address owner,address redeemedToken, uint256[] tokenIds,uint256 campaignId,uint256 requirementsIndex, bytes32 redemptionHash, uint256 salt)"`

### AdvancedOrder extraData

When interacting with the contract offerer via Seaport, the extraData/context layout for the AdvancedOrder MUST follow:

| bytes    | value                             | description / notes                                                                  |
| -------- | --------------------------------- | ------------------------------------------------------------------------------------ |
| 0-32     | campaignId                        |                                                                                      |
| 32-64    | requirementsIndex                 | index of the campaign requirements met                                                |
| 64-96    | redemptionHash                    | hash of offchain order ids                                                           |
| 96-\*    | uint256[] traitRedemptionTokenIds | token ids for trait redemptions, MUST be in same order of campaign TraitRedemption[] |
| \*-(+32) | salt                              | if signer != address(0)                                                              |
| \*-(+\*) | signature                         | if signer != address(0). can be for EIP-712 or EIP-1271                              |

The contract offerer MUST check that the campaign is active (using the same boundary check as Seaport, `startTime <= block.timestamp < endTime`). If it is not active, it MUST revert with `NotActive()`.

### Redemptions

The `Redemption` event MUST be emitted when any valid redemptions occur. The `Redemption` event is simplified to deduplicate information that is present in the `OrderFulfilled` event from Seaport.

### Trait redemptions

The contract offerer MUST respect the TraitRedemption substandards as follows:

| substandard ID | description                     | substandard value                                                  |
| -------------- | ------------------------------- | ------------------------------------------------------------------ |
| 1              | set value to `traitValue`       | prior required value. if blank, cannot be the `traitValue` already |
| 2              | increment trait by `traitValue` | max value                                                          |
| 3              | decrement trait by `traitValue` | min value                                                          |

Additional substandards MAY be specified in subsequent SIPs that inherit this SIP.

### Max campaign redemptions

The contract offerer MUST check that the `maxCampaignRedemptions` is not exceeded. If the redemption does exceed `maxCampaignRedemptions`, it MUST revert with `MaxCampaignRedemptionsReached(uint256 total, uint256 max)`

### Metadata URI

The metadata URI MUST conform to the below JSON schema:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "properties": {
    "name": {
      "type": "string"
    },
    "description": {
      "type": "string",
      "description": "A one-line summary of the redeemable. Markdown is not supported."
    },
    "details": {
      "type": "string",
      "description": "A multi-line or multi-paragraph description of the details of the redeemable. Markdown is supported."
    },
    "imageUrls": {
      "type": "string",
      "description": "A list of image URLs for the redeemable. The first image will be used as the thumbnail. Will rotate in a carousel if multiple images are provided. Maximum 5 images."
    },
    "bannerUrl": {
      "type": "string",
      "description": "The banner image for the redeemable."
    },
    "faq": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "question": {
            "type": "string"
          },
          "answer": {
            "type": "string"
          },
          "required": ["question", "answer"]
        }
      }
    },
    "contentLocale": {
      "type": "string",
      "description": "The language tag for the content provided by this metadata. https://www.rfc-editor.org/rfc/rfc9110.html#name-language-tags"
    },
    "translations": {
      "type": "object",
      "properties": {
        "locale": {
          "type": "string"
        },
        "originalContent": {
          "type": "string"
        },
        "translatedContent": {
          "type": "string"
        }
      },
      "description": "Translations for content provided by this metadata."
    },
    "maxRedemptionsPerToken": {
      "type": "string",
      "description": "The maximum number of redemptions per token. When isBurn is true should be 1, else can be a number based on the trait redemptions limit."
    },
    "isBurn": {
      "type": "string",
      "description": "If the redemption burns the token."
    },
    "uuid": {
      "type": "string",
      "description": "An optional unique identifier for the campaign, for backends to identify when draft campaigns are published onchain."
    },
    "productLimitForRedemption": {
      "type": "number",
      "description": "The number of products which are able to be chosen from the products array for a single redemption."
    },
    "products": {
      "type": "object",
      "properties": "https://schema.org/Product",
      "required": ["name", "url", "description"]
    },
    "traitRedemptions": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "substandard": {
            "type": "number"
          },
          "token": {
            "type": "string",
            "description": "The token address"
          },
          "traitKey": {
            "type": "string"
          },
          "traitValue": {
            "type": "string"
          },
          "substandardValue": {
            "type": "string"
          }
        },
        "required": [
          "substandard",
          "token",
          "traitKey",
          "traitValue",
          "substandardValue"
        ]
      }
    }
  },
  "required": ["name", "description", "isBurn"]
}
```

Future SIPs MAY inherit this one and add to the above metadata to add more features and functionality.

### ERC-1155 (Semi-fungibles)

This standard MAY be applied to ERC-1155 but the redemptions would apply to all token amounts for specific token identifiers. If the ERC-1155 contract only has tokens with amount of 1, then this specification MAY be used as written.

## Rationale

The intention of this SIP is to define a consistent standard to enable redeemable entitlements for tokens and onchain traits. This contract offerer pattern allows for websites to discover, display, and interact with redeemable campaigns.

## Backwards Compatibility

As a new SIP, no backwards compatibility issues are present.

## Test Cases

Test cases can be found in [https://github.com/ProjectOpenSea/redeemables/tree/main/test](https://github.com/ProjectOpenSea/redeemables/tree/main/test)

## Reference Implementation

The reference implementation for the Redeemable Contract Offerer can be found at [https://github.com/ProjectOpenSea/redeemables/blob/main/src/RedeemableContractOfferer.sol](https://github.com/ProjectOpenSea/redeemables/blob/main/src/RedeemableContractOfferer.sol)

## Security Considerations

Tokens must properly implement EIP-7496 Dynamic Traits and allow the Redeemable Contract Offerer to use the `setTrait` method to enforce the full security of the standard.

For tokens to be minted as part of the params `offer`, the `mintRedemption` function contained as part of `IRedemptionMintable` MUST be permissioned and ONLY allowed to be called by redeemable contract offerers specified by address.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
