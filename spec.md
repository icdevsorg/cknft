# Schema and Functional Spec for the ckNFT Canister

**Overview:**

The ckNFT canister acts as a manager or controller over a set of child NFT canisters. Each child corresponds to a specific NFT and network pair. The canister must maintain robustness, scalability, and network compatibility, particularly as it seeks to integrate with additional blockchains such as Ethereum, Solana, and Bitcoin in the future.

The ckNFT canister is designed to manage and oversee the creation and maintenance of individual NFT canisters. When a request is made to interact with a specific NFT, the ckNFT canister first checks its registry to determine if the corresponding NFT canister already exists for the given NFT and network pair. If it does not find a matching entry, the ckNFT canister will initiate the process of spawning a new NFT canister. Before this new canister is created, the ckNFT canister requires the user to pay a fee to cover the computational and storage costs associated with spawning the canister. This fee acts as a deterrent against unnecessary or spam canister creation and ensures that resources are allocated efficiently. Once the fee is confirmed, the ckNFT canister deploys the new NFT canister, registers it in its internal registry, and makes it available for future interactions.

**Schema Requirements:**

**CanisterRegistry**: A registry containing mappings from NFT and network pairs to their corresponding canister IDs.

  ```plaintext
  type Map = vec {record {text, Value}};
  type Network = variant { Ethereum: EthChainType; Solana; Bitcoin: Text; ICP:Text; Other: Map; }
  type RemoteContractPointer = record{
    contractId: Text; 
    network: Network;
  };
  type EthChainType = {
    chanId: nat
    networkId: nat
  };
  type NFTCanisterRegistry = vec { record{RemoteContractPointer, canisterID : Principal}};
  type NFTRemoteRegistry = vec { record{canisterID: Principal, remoteContract : RemoteContractPointer}};
  
  ```

**WASMVersionManagement**: Manage different versions of WebAssembly (WASM) binaries used by child canisters.
 
  ```plaintext
  type WASMVersionMap = record(NFTType = text; version = [nat];binary = Blob);  // Keyed by version identifier
  ```

How will we approve/deprecate new wasm binaries?  Maybe need to look at how ckXXX is doing it with the ckERC20 canisters?

**Interaction with Ethereum RPC Canister**: Define methods for communication between the ckNFT canister and the Ethereum RPC canister.

  ```candid

  type RemoteNFTPointer = record {
    network: NetworkDescriptor;
    contract: Text;
    tokenId: Nat;
  };
  type OwnerResponse = variant {
    Ok = record {
      consensus: bool;
      owner: text;
    };
    Err = tbd
  };
  type FoundMetadata = variant {
    Ok = record {
      metadata = vec Map;
      consensus: bool;
    };
    Err = tbd
  };
  type CreateCanisterResponse = variant{
    Ok: Principal;
    Err: tbd
  }
  type CreateCanisterRequest = record {
    version: ?[Nat];
    icrc7: {
      name: ?Text;
      description: ?Text;
      symbol: ?Text;
      logo: ?Text;
    };
    descriptor: RemoteContractPointer;
    payment: variant { //where does the payment come from. Will use icrc2 to transfer and then convert cycles.
      icp: ?Account;
      cycles: ?Account;
    };
  };
  type MintResult = variant {
    Ok: MintRequestID;
    Err: tbd
  };
  type MintStatus = varian {
    Transferring;
    CheckingOwner: {
      retries: Nat;
      nextQuery: Nat;
    };
    RetrievingMetadata;
    Minting;
    Complete: {
      remoteTrx: ?Text;
      mintTrx: Nat;
      approvalTrx: ?Nat
    };
    Err : variant {
      InvalidTransfer : Text; //Error from RPC
      OwnershipNotVerified : variant {
        TooManyRetries: nat;
        NoConsensus;
      };
      MetadataError: Text;
      MintError: Text;
      ApprovalError: Text;
      GenericError: Text;
    };
    
  };
  type MintResumeOption: variant {
    StartOwnershipVerification;
    StartMetadataTransfer;
    StartMint;
  };
  type MintRequest: record {
    nft:RemoteNFTPointer;
    mintToAccount: Account;
    spender: ?Account;
    resume: ?(?Nat, ResumeOption)
  }
  type CastResult : variant {
    Ok: Nat; //cast status id
    Err: CastError; //
  };
  type CastStatus : variant {
    VerifyingOwnership;
    RetrievingCollectionMetadata;
    RetrievingNFTMetadata;
    WritingContract: {
      trxId: ?Text;
      nextQuery: Nat;
      retries: Nat
    };
    MintingNFT: {
      trxId: ?Text;
      nextQuery: Nat;
      retries: Nat
    };
    TransferringNFT: {
      trxId: ?Text;
      nextQuery: Nat;
      retries: Nat
    };
    Complete: {
      contractTrx: ?Text;
      mintTrx: ?Text;
      transferTrx: ?Nat
    };
    Err : variant {
      Unauthorized;
      CollectionError: Text;
      NFTError: Text;
      InvalidTransaction : Text; //Error from RPC. includes not enough gas
      ContractNotVerified : variant {
        TooManyRetries: nat;
        NoConsensus;
      };
      MintNotVerified : variant {
        TooManyRetries: nat;
        NoConsensus;
      };
      TransferNotVerified : variant {
        TooManyRetries: nat;
        NoConsensus;
      };
      MintError: Text;
      ApprovalError: Text;
      GenericError: Text;
    };
  };
  type UpgradeResult : variant{
    Ok: bool;
    Err: variant{
      GenericError: Text;
    };
  }
  type CastRequest : record{
    ckNFTCanister: canister;
    tokenId: nat;
    targetNetwork: Network;
    targetAccount: Text;
  };
  type service = actor {
    get_remote_contract: query(vec (Principal, Network)) -> vec opt RemoteContractPointer
    get_ck_nft_canister: query(vec RemoteContractPointer) -> vec opt Principal
    get_mint_cost: query() -> async (Nat,Nat); //balance in ICP and cycles needed
    get_cast_cost: query() -> async RemoteTokenCost; //balance in remote token that is needed at deposit address
    create_canister: (CreateCanisterRequest) -> async CreateCanisterResponse;
    get_owner : (RemoteNFTPointer) -> async Text;
    get_remote_metadata: (RemoteContractPointer) -> async FoundMetadata;
    get_remote_nft_metadata: (RemoteNFTPointer) -> async FoundMetadata;
    get_approval_address: query(Account, RemoteNFTPointer) -> Text;
    mint: (MintRequest) -> MintResult; //Resume with the mint status ID, you will be issued a new mint status Id. If using transfer method, pass null for the status and StartOwnershipVerification for the item.
    get_mint_status: query(Nat) -> MintStatus;
    add_approved_wasm_chunk: (record {
      version: ?[nat];
      chunkId: nat
      chunk: ?blob}) -> async (nat, nat); //current chunk, total chunks; null deletes a chunk
    activate_current_wasm_version: ([nat]) -> bool;
    get_wasm_version: query(Text, ?[nat]) -> ?([nat],nat,blob); //version, chunks, hash; input can be null for current version
    get_wasm_versions: query(Text, prev: ?[nat], take: nat) -> [([nat],nat,blob)]; //paginated list of items
    get_wasm_chunk: query(Text, [nat], chunk) -> blob;
    upgrade_ck_nft_canister: async(principal, Type, [nat], blob) -> UpgradeResult; //target, version, candid arguments for upgrade
    cast_nft: (CastRequest) -> async CastResult;
    get_cast_status: query(vec Nat) -> async vec ?CastStatus;
    get_remote_funds(Network, Text, Text) -> async (nat, nat, symbol); //Network, contract, account -> (balance, decimals, symbol)
  };
  
  ```

**Functional Specifications:**

- Implement `get_remote_contract` to find if a ckNFT canister has a remote contract on another chain.
- Implement `get_ck_nft_canister` to find if a canister exists for a contract on a remote network.
- Implement a `get_creation_cost` query that returns the current cost in ICP for cycles to spawn a new ckNFT canister
- Implement a `create_canister` method that deploys a new ckNFT canister if one doesn't already exist for the given NFT/network.
  - As part of this process the admin canister will call icrc2_transfer_from to pull the ICP into a known account
  - The process will use the retrieved ICP to create cycles by sending cycles to the CycleMinter using ICP.transfer teo the CycleMinting Canister
  - The process will deposit the requested cycles to the Cycles Ledger using the deposit function
  - The process will mint the canister using the Cycles Ledger 
  - The process will then deploy the canister from the indicated wasm with any fields supplied.
  - The process will query the NFT contract for any other data items
  - The process will configure ICRC7 metadata.
- Implement a `get_owner` method that queries who the current owner of an NFT is on another network.
- Implement a `get_remote_metadata` method that gets the metadata for an NFT, potentially even pulling the item if it points to an https or ipfs file.
- Implement a `get_approval_address` method that calculates a tecdsa address for the owner to approve or transfer to so that the Admin canister can transfer it and/or detect control over it it.
  - This should be a hash of RemoteNFTPointer and the ICRC7 based Account the user wants to mint to.
- Implement a `mint` method that starts the process of moving the NFT to the ckNFT canister
  - The process will kick off a workflow and return a mint id.
- Implement the mint workflow
  - The Process will attempt to transfer the asset to the approval address via the approval address unless this has been skipped(the approval pathway requires funding the approval address with gas fees).
  - The Process will check the owner with `get_owner`.
  - If it doesn't already exist
    - If the proper user owns the nft the admin canister will retrieve the NFT metadata using `get_remote_nft_metadata`.
    - The process will Mint the NFT on the target ckNFT Canister and assign it to the target Account.
  - If it does exist
     The process will transfer the NFT on the target ckNFT Canister to the target Account.
  - The process will approve an icrc37 approval to a spender account if provided.
- Integrate with Ethereum RPC canister at administrative canister to abstract away EVM-specific logic from NFT canisters.
- Provide methods for governing and updating WASM versions for new deployments.
  - `add_approved_wasm_chunk` - adds or deletes a chunk for a wasm.
  - `activate_current_wasm_version` - makes a version a canonical version for deployment
  - `upgrade_ck_nft_canister` - upgrades a canister to a wasm version
  - `get_wasm_versions` - gets a paginated list of wasm versions, number of chunks, and hash
  - `get_wasm_version` - gets a version of a  wasm, number of chunks and hash; a null param can be used to get the canonical version
  - `get_wasm_chunk` - gets a chunk of the version
- `upgrade_ck_nft_canister` - upgrade a specific canister to a new version of the wasm.
- Plan for cross-chain NFT casting, ensuring NFTs aren't cast back to their original chain.
- Implement ICRC-3 event logging for significant state changes (deposit, mint, burn, withdraw) for auditability and tracking.

------------------------

|ICRC|Title|Author|Discussions|Status|Type|Category|Created|
|:----:|:----:|:----:|:----:|:----:|:----:|:----:|:----:|
|99|Minimal Non-Fungible Token (NFT) Standard|Austin Fatheree (@skilesare)|https://github.com/dfinity/ICRC/issues/99|Pre-Draft|Standards Track||2024-08-14|



# ICRC-99: ChainFusion Extension for ICRC-7 (NFT) Standard

ICRC-99 is an extension to the [ICRC-7 minimal NFT standard](https://github.com/dfinity/ICRC/blob/main/ICRCs/ICRC-7/ICRC-7.md). The primary purpose of ICRC-99 is to facilitate the interoperability of Non-Fungible Tokens (NFTs) across different blockchain networks. This allows seamless transfer and management of NFTs on the Internet Computer (IC) and other supported blockchains such as Ethereum, Solana, and Bitcoin.

ICRC-99 introduces a set of methods and schema enhancements to manage cross-chain ownership, transformation, and cost assessment functionalities for NFTs. These include methods for querying the original and remote status of NFTs, requesting updates on remote ownership, and handling cross-chain casting (transferring) of NFTs. The standard also integrates functional specifications to ensure secure transactions, state management, and metadata handling, thereby enhancing the versatility and auditability of NFT operations across multiple blockchain environments.

## Data Representation

This section elaborates on the data representations utilized in ICRC-99, extending and enhancing the foundational structures provided by ICRC-7 to support cross-chain functionalities and management of remote ownership.
### Network

The `Network` type extends the data representations from ICRC-7 to support various blockchain networks and ensure robust cross-chain interoperability. It includes different variants to distinguish between supported networks like Ethereum, Solana, Bitcoin, ICP, and other arbitrary networks. Each variant carries specific details required to identify and interact with these networks accurately.

```plaintext
type Network = variant {
  Ethereum: EthereumNetwork;
  Solana: Text;
  Bitcoin: Text;
  ICP: Text;
  Other: RemoteNetwork;
};

type EthereumNetwork = record {
  chainId: Nat;
  networkId: Nat;
};

type RemoteNetwork = vec { record { key: Text; value: Value }};
```

- **Ethereum**: Specifies an Ethereum-based network using `EthereumNetwork`, which includes `chainId` and `networkId` to identify the specific Ethereum network (Mainnet, Ropsten, etc.).
- **Solana**: Uses a simple text field to describe the Solana network (Mainnet, Devnet, etc.).
- **Bitcoin**: Uses a text field to specify the Bitcoin network type.
- **ICP**: Uses a text field to describe the Internet Computer network.
- **Other**: A generic variant for other arbitrary networks, represented by a map of key-value pairs in `RemoteNetwork`.

### Example Usage

#### Ethereum Network

The `EthereumNetwork` variant provides specific fields for identifying Ethereum mainnet and testnets such as Ropsten, Kovan, etc.

```plaintext
type EthereumNetwork = record {
  chainId: Nat; // The ID of the Ethereum chain
  networkId: Nat; // The specific network ID within Ethereum-based networks
};
```

For example:
```plaintext
let ethereumNetworkExample: Network = Ethereum({ chainId = 1, networkId = 1 });
```

#### Solana Network

Solana uses a simple text identifier to denote different instances of Solana networks:

```plaintext
type Network = variant {
  Solana: Text; // Solana network name or other unique identifier
};
```

For example:
```plaintext
let solanaNetworkExample: Network = Solana("Mainnet");
```

#### Bitcoin Network

Similarly, Bitcoin uses a textual representation to specify the network type:

```plaintext
type Network = variant {
  Bitcoin: Text; // Bitcoin network name or other unique identifier
};
```

For example:
```plaintext
let bitcoinNetworkExample: Network = Bitcoin("Testnet");
```

#### Other Networks

For other undefined or custom networks, `RemoteNetwork` provides the flexibility to accommodate additional key-value pair information.

```plaintext
type RemoteNetwork = vec { record { key: Text; value: Value }};
```

For example:
```plaintext
let customNetworkExample: Network = Other(vec { ("customKey", Text("customValue")) });
```

### RemoteContractPointer

The `RemoteContractPointer` type points to a specific contract on a remote blockchain network. It includes the contract's unique ID and the network descriptor.

```plaintext
type RemoteContractPointer = record {
  contractId: Text;
  network: Network;
};
```

### RequestRemoteOwnerRequest

The `RequestRemoteOwnerRequest` type is utilized to request an update of the remote ownership status for a specific NFT token. This record includes necessary details about the token and the remote contract pointer where ownership should be queried.

```plaintext
type RequestRemoteOwnerRequest = record {
  tokenId: nat;                             // Identifier of the token whose remote ownership status is being requested.
  currentContract: RemoteContractPointer;   // Pointer to the current remote contract associated with the token.
  memo: opt Blob;                           // Optional memo field for additional information.
  createdAtTime: opt Nat;                   // Optional timestamp indicating when the request was created.
};
```

- `tokenId`:
  - **nat**: The unique identifier of the token whose remote ownership status is being queried.

- `currentContract`:
  - **RemoteContractPointer**: Details of the contract associated with the token on the remote network.

- `memo`:
  - **opt Blob**: An optional blob for adding additional information or instructions related to the request.

- `createdAtTime`:
  - **opt Nat**: An optional timestamp indicating the creation time of the request.

### CastCostRequest

The `CastCostRequest` type is designed to request the cost associated with casting a specific NFT token to a target contract on a remote network. This record captures the token ID and target contract details required for the cost calculation.

```plaintext
type CastCostRequest = record {
  tokenId: nat;                             // Identifier of the token for which the casting cost is being calculated.
  targetContract: RemoteContractPointer;    // Target contract on the remote network where the token is to be cast.
};
```

- `tokenId`:
  - **nat**: The unique identifier of the token for which the casting cost is being calculated.

- `targetContract`:
  - **RemoteContractPointer**: Details of the target contract on the remote network where the token will be cast.

### RemoteTokenCost

The `RemoteTokenCost` type encapsulates the cost associated with casting an NFT to a remote blockchain network. This includes details about the remote token's symbol, amount required, decimal precision, network identifier, and contract ID.

```plaintext
type RemoteTokenCost = {
  symbol: Text;
  amount: Nat;
  decimals: Nat;
  network: Network;
  contractId: Text;
}
```

### RemoteOwner

The `RemoteOwner` type specifies the ownership status of an NFT, both locally (i.e., on the current chain) and remotely (i.e., on a different blockchain network). The local ownership is represented by the `Account` type, whereas the remote ownership includes additional metadata.

```plaintext
type RemoteOwner = variant {
  local: Account;
  remote: {
    contract: RemoteContractPointer;
    owner: Text;
    timestamp: Nat;
  };
}
```

### CastRequest

The `CastRequest` record is used to initiate the casting process of an NFT to a remote network. This process includes transferring the ownership of the NFT from its current location within the Internet Computer (IC) to a specified contract on a different blockchain network. The record encapsulates the necessary data to facilitate this operation, ensuring that all required parameters for the casting process are available.

```plaintext
type CastRequest = record {
  tokenId: Nat;
  remoteContract: RemoteContractPointer;
  memo: opt Blob;
  created_at_time : opt Nat;
};
```
### RemoteOwnershipUpdateRequest

The `RemoteOwnershipUpdateRequest` record is used to request an update on the ownership status of an NFT that has been cast to a remote network. This ensures that the local registry is synchronized with the current ownership details on the remote blockchain.

```plaintext
type RemoteOwnershipUpdateRequest = record {
  tokenId: Nat;
  remoteContract: RemoteContractPointer;
  memo: opt Blob;
};
```


### RemoteOwnershipUpdateResult

The `RemoteOwnershipUpdateResult` variant represents the result of a remote ownership update request. This variant can either indicate a successful update with the new ownership information or provide an error describing why the update failed.

```plaintext
type RemoteOwnershipUpdateResult = variant {
  Ok: RemoteOwnership;
  Err: RemoteOwnershipUpdateError;
};
```

### RemoteOwnershipUpdateError

The `RemoteOwnershipUpdateError` variant encapsulates different types of errors that may occur while attempting to update the remote ownership status of an NFT. This helps in identifying and handling specific error conditions appropriately.

```plaintext
type RemoteOwnershipUpdateError = variant {
  QueryError: Text;
  GenericError: Text;
  NotFound;
};
```

These schemas and data representations provide a structured and efficient way to manage cross-chain casting, ownership updates, and error handling within the ICRC-99 framework, ensuring seamless interoperability and reliable operations across different blockchain networks.

### CastRequest

The `CastRequest` record is utilized to initiate the casting process for NFTs. It includes all the necessary details to facilitate the cross-chain casting operation, ensuring all parameters required are available for successful execution.

```plaintext
type CastRequest = record {
  tokenId: nat; // The unique identifier of the token to be cast.
  targetNetwork: Network; // The network to which the NFT is to be cast.
  targetAccount: Text; // The account identifier on the target network.
};
```

- **tokenId**:
  - **nat**: The unique identifier of the NFT being cast.

- **targetNetwork**:
  - **Network**: The destination network to which the NFT is to be cast. This is defined with the expanded `Network` type for supporting various blockchain networks.

- **targetAccount**:
  - **Text**: The account identifier on the target network where the NFT will reside after casting.

### CastResult

The `CastResult` variant is used to encapsulate the outcome of casting an NFT to a remote network. It provides a structured way of handling both successful and unsuccessful casting operations.

```plaintext
type CastResult = variant {
  Ok: Nat; // Indicates success and returns the casting status ID.
  Err: CastError; // Indicates an error and provides details about the error.
};
```

- **Ok**:
  - **Nat**: On successful casting, a unique status ID is provided to track the casting process.

- **Err**:
  - **CastError**: On failure, a detailed error message is provided to describe the specific reason for the failure.

### Cast Error

The `CastError` variant in the ICRC-99 standard is used to describe the errors that can arise during the casting process of NFTs to remote blockchain networks. Each error type in this variant provides specific details about the nature of the problem encountered, facilitating better handling and user feedback.

```plaintext
type CastError = variant {
  Unauthorized;
  InvalidContract;
  ExistingCast: Nat;
  NetworkError(Text);
  ContractNotVerified {
       TooManyRetries: nat;
       NoConsensus;
  };
  MintNotVerified {
       TooManyRetries: nat;
       NoConsensus;
  };
  TransferNotVerified {
       TooManyRetries: nat;
       NoConsensus;
  };
  InvalidTransaction (Text);
  GenericError (Text);
};
```

#### Unauthorized

- **Description**: Indicates that the casting operation was not authorized. This typically occurs if the caller does not have the necessary permissions to perform the cast. 
- **Handling**: Verify that the caller is properly authenticated and has the required permissions to initiate the casting process.

#### InvalidContract

- **Description**: The specified remote contract is invalid, either because it doesn't exist, is incorrectly formatted, or is not compatible with the NFT.
- **Handling**: Check the contract ID and ensure that it corresponds to a valid, active contract capable of receiving NFTs on the target network.

#### NetworkError(Text)

- **Description**: A network-related error has occurred during the casting operation. The accompanying text provides additional context or details about the network issue.
- **Handling**: Check network connectivity and retry the casting operation. Review the error message for specific details that might indicate the nature of the network problem.

#### Existing(Cast)

- **Description**: A cast is already underway. Check its status for an update.

#### ContractNotVerified 

- **Description**: The remote contract could not be verified on the target network. This can be due to several retries without reaching consensus.
- **Fields**:
  - `TooManyRetries: nat`: Indicates that the maximum retry limit for verifying the contract has been reached.
  - `NoConsensus`: Indicates that consensus could not be reached regarding the contract's existence or state.
- **Handling**: Review the contract details and network status. Consider increasing the retry limit or investigating potential issues with the network or contract.

#### MintNotVerified 

- **Description**: The minting operation on the remote chain could not be verified.
- **Fields**:
  - `TooManyRetries: nat`: Indicates that the maximum retry limit for minting verification has been reached.
  - `NoConsensus`: Indicates that consensus could not be reached regarding the minting operation.
- **Handling**: Investigate the minting process on the remote chain. Review any logs or messages to understand the reasons for reaching the retry limit or failing to achieve consensus.

#### TransferNotVerified 

- **Description**: The transfer operation on the remote chain could not be verified.
- **Fields**:
  - `TooManyRetries: nat`: Indicates that the maximum retry limit for transfer verification has been reached.
  - `NoConsensus`: Indicates that consensus could not be reached regarding the transfer operation.
- **Handling**: Review the transfer logs and network conditions. Consider extending the retry limit or analyzing why consensus could not be reached.

#### InvalidTransaction(Text)

- **Description**: An error specific to the transaction itself, such as insufficient gas, incorrect transaction format, or other transaction-related issues.
- **Fields**:
  - `Text`: A message providing details about the transaction error.
- **Handling**: Review the transaction parameters and ensure they are correct. The error message will provide specific details to aid in troubleshooting the transaction issue.

#### GenericError(Text)

- **Description**: A catch-all for other errors that do not fall into the more specific categories defined above.
- **Fields**:
  - `Text`: A message providing details about the generic error.
- **Handling**: Use the error message to diagnose the issue. This may involve general troubleshooting steps and deeper investigation depending on the provided details.


### Token Metadata and State Management

The metadata and state management for NFTs within the ICRC-99 standard includes several specialized metadata keys and values to track the cross-chain status and provenance of NFTs.

```plaintext
  icrc99:adminCanister: Principal;
  icrc99:castCycles: Nat;
  icrc99:remoteOwnerRequestCycles: Nat;
  icrc99:castCostCycles: Nat;
  icrc99:originChain: Network; 
  icrc99:originContract: Text; 
  icrc99:remoteChain: Network; 
  icrc99:remoteContract: Text; 
  icrc99:metadataURL: Text;
  icrc99:status: variant { Casting; Remote; Local }; // The current status of the NFT
```

- `icrc99:adminCanister`: Specifies the canister on the Internet Computer that manages deposits and casts for the icrc99 implementation.
- `icrc99:castCycles`: Optional: Specifies the minimum cycles that must be available to call the cast function;
- `icrc99:remoteOwnerRequestCycles`: Optional: Specifies the minimum cycles that must be available to call the remote owner request function;
- `icrc99:castCostCycles`: Optional: Specifies the minimum cycles that must be available to call the cast cost function;
- `icrc99:originChain`: Specifies the originating blockchain network of the NFT.
- `icrc99:originContract`: The contract ID where the NFT was initially minted.
- `icrc99:remoteChain`: Indicates the remote blockchain network where the NFT is currently residing.
- `icrc99:remoteContract`: The contract ID on the remote chain.
- `icrc99:metadataURL`: A URL pointing directly to the NFT metadata.
- `icrc99:status`: Represents the current state of the NFT which could be one of:
  - `Casting`: Indicates the NFT is in the process of being cast to a remote blockchain and can't be moved.
  - `Remote`: Signifies the NFT is currently located on a different blockchain.
  - `Local`: Denotes that the NFT resides within the local blockchain network.
  
These metadata elements and types facilitate the robust management and cross-chain operability of NFTs. They ensure comprehensive tracking and accurate state representation of NFTs, enabling seamless integration and interaction between different blockchain networks.

## Methods

### Generally-Applicable Specification

We next outline general aspects of the specification and behavior of query and update calls defined in this standard. Those general aspects are not repeated with the specification of every method, but specified once for all query and update calls in this section.

#### Batch Update Methods

Please refer to the Batch Update Methods section of ICRC-7

#### Batch Query Methods

Please refer to the Batch Query Methods section of ICRC-7

#### Error Handling

Please refer to the Error Handling section of ICRC-7

#### Other Aspects

Please refer to the Other Aspects section of ICRC-7

#### icrc99_native_chain

```plaintext
icrc99_native_chain: query() -> RemoteContractPointer;
```

##### Description
- **Function**: This query method returns the origin details of the NFT collection, including the network and contract information where the NFT was originally minted.
- **Returns**: A `RemoteContractPointer` object which includes fields for the originating network and contract.

#### icrc99_remote_owner_of

```plaintext
icrc99_remote_owner_of: query( vec nat) -> vec opt RemoteOwner;
```

##### Description
- **Function**: This query method returns the current ownership status of each NFT specified by token IDs within the input list. The ownership can be local or remote depending on the location of the NFT. Due to the asynchronous nature of cast NFTs, this may not be the canonical owner of the NFT if it has changed hands on the remote network.
- **Parameters**: 
  - A vector of NFT token IDs (`[nat]`).
- **Returns**: 
  - A vector of `opt RemoteOwner` objects where each object corresponds to the ownership status of the queried token ID. If the item does not exist, the value will be null

#### icrc99_request_remote_owner_status

```plaintext
icrc99_request_remote_owner_status: (vec nat) -> async vec ?RemoteOwner;
```

##### Description
- **Function**: This update method requests the most recent remote ownership status for each NFT specified by token IDs within the input list, updating the local registry to reflect these updates. This must be an update method due to the asynchronous nature of cross chain communication.
- **Parameters**: 
  - A vector of NFT token IDs (`[nat]`).
- **Returns**: 
  - A vector of updated `opt RemoteOwner` objects. If the item does not exist it will return null.

#### icrc99_cast

```plaintext
icrc99_cast: (vec CastRequest) -> async CastResponse;
```

##### Description
- **Function**: This method initiates the casting process of an NFT specified by `tokenId` to a remote blockchain network designated by the `RemoteContractPointer`.
- **Parameters**: 
  - `vec CastRequest`: Information for the cast operation.
- **Returns**: 
  - A `vec CastResponse` object indicating the success or failure of the casting process.

### Schema

```plaintext
type CastResponse = variant {
  Ok(Nat); // Casting completed, returns casting status ID.
  Err(CastError);
};
```

#### icrc99_cast_cost

```plaintext
icrc99_cast_cost: (nat, RemoteContractPointer) -> async RemoteTokenCost;
```

##### Description
- **Function**: This update method calculates the cost associated with casting an NFT (specified by `tokenId`) to a remote blockchain network (specified by `RemoteContractPointer`). This must be an update method due to the async method of checking gas fees for remote chains. Due to the time and potential cost of calls, this does not provide a batch interface
- **Parameters**: 
  - `nat`: The ID of the NFT for which the casting cost is being calculated.
  - `RemoteContractPointer`: A pointer to the remote contract where the NFT will be cast.
- **Returns**: 
  - A `RemoteTokenCost` object containing the cost details including symbol, amount, decimals, network, and contract ID.

### icrc99_cast_status

```candid
icrc99_cast_status: query(vec nat) -> async vec opt CastStatus;
```

##### Description
- **Function**: This query method retrieves the status of ongoing or completed casting operations for a list of specified NFT token IDs. It provides detailed information about where each NFT is in the casting process, whether it has completed successfully, or if any errors have occurred.
- **Parameters**: 
  - `vec nat`: A vector of CastStatusIds returned from icrc99_cast for which the casting statuses are being requested.
- **Returns**: 
  - `vec opt CastStatus`: A vector of optional `CastStatus` objects corresponding to each token ID provided in the input. Each `CastStatus` object can contain information such as:

    ```plaintext
    type CastStatus = variant {
      VerifyingOwnership;
      RetrievingCollectionMetadata;
      RetrievingNFTMetadata;
      WritingContract: {
        trxId: opt Text;
        nextQuery: Nat;
        retries: Nat;
      };
      MintingNFT: {
        trxId: opt Text;
        nextQuery: Nat;
        retries: Nat;
      };
      TransferringNFT: {
        trxId: opt Text;
        nextQuery: Nat;
        retries: Nat;
      };
      Complete: {
        contractTrx: opt Text;
        mintTrx: opt Text;
        transferTrx: opt Nat;
      };
      Err: variant {
        Unauthorized;
        CollectionError: Text;
        NFTError: Text;
        InvalidTransaction: Text;
        ContractNotVerified: variant {
          TooManyRetries: Nat;
          NoConsensus;
        };
        MintNotVerified: variant {
          TooManyRetries: Nat;
          NoConsensus;
        };
        TransferNotVerified: variant {
          TooManyRetries: Nat;
          NoConsensus;
        };
        MintError: Text;
        ApprovalError: Text;
        GenericError: Text;
      };
    };
    ```
  
#### Usage

This method is crucial for tracking the entire lifecycle of the casting process, enabling users to understand exactly where their NFTs are in the cross-chain transfer journey. Given the complex nature of these operations, this granular status reporting helps ensure transparency and immediate troubleshooting if issues arise. By returning detailed status information for each NFT, this method provides a comprehensive view of the state of casting operations, allowing users and processes to react accordingly based on a multitude of potential scenarios and outcomes.

### icrc10_supported_standards

An implementation of ICRC-99 MUST implement the method `icrc10_supported_standards` as put forth in ICRC-10.

The result of the call MUST always have at least the following entries:

```candid
record { name = "ICRC-7"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-7"; }
record { name = "ICRC-10"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-10"; }
record { name = "ICRC-99"; url = "https://github.com/dfinity/ICRC/ICRCs/ICRC-99"; }
```

### Cycle Costs

Cycle costs MAY be implemented by ICRC-99 implementations or a service provider MAY fund cycles.

When cycle costs are implemented, when interacting with cross-chain functionalities through the ICRC-99 standard, specific methods such as `icrc99_request_remote_owner_status`, `icrc99_cast`, and `icrc99_cast_cost` require cycles to be included with the request or for the admin canister to be approved by the requestor on the Cycles Ledger for an appropriate number of cycles. This is necessary due to the computational and storage resources required for these operations, especially since some methods involve querying or updating remote chains.

In all cases, the provided cycles are managed to ensure efficient utilization:

- Upon invocation, the necessary cycles are used to perform the cross-chain operation or computation.
- Any excess cycles that are not consumed during the process are refunded to the original account or canister.
- If the user has authorized the admin canister to withdraw cycles, it will only withdraw the exact amount required for the operation, thus minimizing the likelihood of overcharging or resource wastage.

# ICRC-99 Block Schema

### cast Block Schema

This block type records the request to cast an NFT to a remote network, capturing essential details required to track and audit the casting process.

1. The `btype` field of the block MUST be set to `99cast`.
2. The `tx` field:
    1. MUST contain a field `tid: Nat` representing the token ID being cast.
    2. MUST contain a field `from: Account` indicating the account initiating the cast.
    3. MUST contain a field `rcp: RemoteContractPointer` specifying the remote contract details.
    4. MAY contain a field `memo: Blob` if a memo was provided with the cast request.
    5. MUST contain a field `ts: Nat` indicating the timestamp of the cast request.
    6. MAY contain a field `memo: Blob` if a memo was provided with the cast request.

This schema ensures all necessary information to reconstruct the state and audit the casting process efficiently is recorded.

### remoteOwnerUpdated Block Schema

This block type records the update of a remote owner for an NFT, capturing all critical details required to maintain accurate and up-to-date ownership records.

1. The `btype` field of the block MUST be set to `99remoteOwnerUpdated`.
2. The `tx` field:
    1. MUST contain a field `tid: Nat` representing the token ID of the NFT.
    2. MUST contain a field `newOwner: RemoteOwner` specifying the updated remote ownership details.
    3. MAY contain a field `from: Account` indicating the account initiating the update.
    4. MUST contain a field `ts: Nat` indicating the timestamp when the ownership was updated.
    5. MAY contain a field `memo: Blob` if a memo was provided with the cast request.

### Outputting RemoteContractPointer
When outputting a `RemoteContractPointer` in the ICRC-3 value schema, use the `Map` type to ensure each field is represented clearly.

In the ICRC-3 `Value` schema, this would translate to:

```plaintext
variant {
  Map: vec {
    record {
      "contractId";
      Text: contractId;
    };
    record {
      "network";
      Map: [
        // Include fields for Network enum
        record {
          "type";
          Text: "Ethereum"; // or "Solana", "Bitcoin", etc.
        };
        // Based on the type, include appropriate nested fields
      ];
    }
  }
}
```
This setup captures each field distinctly, converting the network description into a nested map where necessary.

### Outputting Account
When outputting an `Account` in the ICRC-3 value schema, represent both the `owner` and optional `subaccount` fields.


This corresponds to:

```plaintext
variant {
  Map: vec {
    record {
      "owner";
      Text: owner;
    };
    opt record {
      "subaccount";
      Blob: subaccount; // if subaccount is present
    };
  }
}
```
This schema ensures both the `owner` and `subaccount` are accurately captured.

### Outputting RemoteOwner
The `RemoteOwner` variant can be either a `local` `Account` or a `remote` pointer with additional metadata.

```plaintext
type RemoteOwner = variant {
  local: Account;
  remote: {
    contract: RemoteContractPointer;
    owner: Text;
    timestamp: Nat;
  };
};
```

In the ICRC-3 `Value` schema, this would be represented as:

```plaintext
variant {
  // For local variant
  Map: [
    record {
      "type";
      Text: "local";
    };
    record {
      "account";
      Account;
    };
  ],
  
  // For remote variant
  Map: [
    record {
      "type";
      Text: "remote";
    };
    record {
      "contract";
      // Nested representation
      Map: [
        record {
          "contractId";
          Text: contractId;
        };
        record {
          "network";
          Text: "Ethereum"; // or "Solana", "Bitcoin", etc.
        }
      ];
    };
    record {
      "owner";
      Text: owner;
    };
    record {
      "timestamp";
      Nat: timestamp;
    }
  ]
}
```
## Transaction Deduplication

Please see the Transaction Deduplication section in ICRC-7

----------------------------

ckNFT canister Functional Specification

type ICRCXService = record {
    icrc99_native_chain: query() -> RemoteNFTPointer; //where did this NFT Collection come from?
    icrc99_remote_owner_of: query([nat]) ->  [RemoteOwner]; //where is this nft now?
    icrc99_request_remote_owner_status: (vec RequestRemoteOwnerRequest) -> [RemoteOwner]; //updates the remote ownership status of an nft
    icrc99_cast: (vec CastRequest) -> async vec CastResult;   // NFT ID to be cast to the remote network
    icrc99_cast_cost: (CastCostRequest) -> async RemoteTokenCost;   // NFT ID to be cast to the remote network
    icrc99_cast_status: vec nat -> async vec opt CastStatus;
  };

**Functional Specifications:**

- Store the native chain in metadata
- Keep the remote owner in metadata
- Child canisters implements icrc99_cast to cast tokens to other chains
  - Tokens are also cast back to their home chain
  - Canister will Call the cast() method of the admin canister to retrieve a castStatusID
- Child canister implements icrc99_cast_status to get the status of a cast
- Child canister implements icrc99_cast_cost and calls get_cast_cost of admin canister.
- Implement icrc99_cast_stats which should call the get_cast_status method of the admin canister
- Child canister must handle cycle management for cycle management requirements of icrc99
- Must be able to check the approval on the Cycle Ledger for the Admin canister account if necessary.
- Child canister creates icrc99_request_remote_owner_status which call get_owner of the admin canister
- Ensure these transactions are authenticated and validated, considering potential security issues such as re-entrance or double spends.
- Implement ICRC-3 event logging for significant state changes (cast, status_update) for auditability and tracking.
- Allow the manual updating of remote ownership status by calling `icrcX_request_remote_owner_status`.

** Diagrams **

```
sequenceDiagram
    participant EthOperator as Eth Operator
    participant RemoteNFT as Remote NFT
    participant AdminCanister as Admin Canister
    participant ckNFTCanister as ckNFT Canister
    participant User as NFT Owner / Eth Operator
    participant EVMRPC as EVMRPC Canister
    participant ICP as ICP Canister
    participant CM as Cycle Minter

    Note over EthOperator, CM: Deposit EVM NFT into an IC ckNFT canister    
    EthOperator->>AdminCanister: Check if ERC-721 has an assigned ckNFT canister
    AdminCanister->>EthOperator: ckNFT canister exists?
    alt ckNFT canister does not exist
        EthOperator->>AdminCanister: Request creation cost
        EthOperator->>ICP: icrc2_approve creation cost to Admin
        EthOperator->>AdminCanister: Request creation of ckNFT canister Admin.create_canister
        AdminCanister->>ICP: ICP icrc2_transfer_from creation cost
        AdminCanister->>CM: Convert to Cycles
        AdminCanister->>ckNFTCanister: Spawn Canister w/cycles
        AdminCanister-->>ckNFTCanister: ckNFT WASM Installed
        AdminCanister-->>EVMRPC: Retrieve EVM contract details admin.get_remote_metadata
        AdminCanister-->>ckNFTCanister: configure metadata
        
    end

    EthOperator->>ckNFTCanister: Request evm spender address
    ckNFTCanister-->>EthOperator: Spender address provided

    alt transfer method
        User->>RemoteNFT: ERC71.transfer to approval address
    else approval method
        User->>RemoteNFT: ERC71.approve to approval address
        User->>AdminCanister: Fund approval address with native evm gas
    end

    User->>AdminCanister: Start mint with call to ckNFT.mint
    AdminCanister->>User: MintStatusID

    AdminCanister->>EVMRPC: Broadcast transfer from Function if necessary
    EVMRPC->>RemotNFT: transfer from function if necessary
    AdminCanister->>EVMRPC: Poll for ownership record at deposit address
    loop Check Ownership
        AdminCanister-->>EVMRPC: Check until ownership confirmed
    end
    AdminCanister->>EVMRPC: Retrieve NFT Metadata
    AdminCanister->>ckNFTCanister: Mint ICRC7 NFT to Target Account
    AdminCanister->>ckNFTCanister: Assign approval to marketplace spender account if necessary


    Note over EthOperator, CM: Cast back to EVM network
    User->>ckNFTCanister: Request cast to external contract with icrcX_cast
    ckNFTCanister-->ckNFTCanister: Transfer NFT to Admin Account
    ckNFTCanister-->>AdminCanister: Start Cast Process - Get Cast StatusID
    AdminCanister-->>ckNFTCanister: Confirm ownership
    AdminCanister-->>AdminCanister: Check remote contract existance
    alt Contract does not exist
        AdminCanister-->>ckNFTCanister: get collection metadata
        AdminCanister-->>ckNFTCanister: get nft metadata
        AdminCanister-->>EVMRPC: Broadcast contract creation 
        EVMRPC-->>RemoteNFT: Contract Created
        AdminCanister-->>EVMRPC: Check Creation Status
        AdminCanister-->>AdminCanister: Record Creation   
    else Contract does exist
        AdminCanister-->>EVMRPC: Check for NFT Existance 
        EVMRPC-->>RemoteNFT: Query Existance
    end
    alt NFT Doesn't Exist
        AdminCanister-->>EVMRPC: Mint on Remote Chain
        AdminCanister-->>EVMRCP: Confirm Mint
    else NFT Does Exist
        AdminCanister-->>EVMRPC: Transfer on Remote Chain
        AdminCanister-->>EVMRPC: Confirm Transfer
    end
    AdminCanister-->>ckNFT: Notify of Transfer
    Note over EthOperator, CM: Update Ownership info form remote chain
    User->>ckNFTCanister: Request remote ownership info
    ckNFTCanister-->ckNFTCanister: Check that item is remote
    ckNFTCanister-->>AdminCanister: Request Ownership Check
    AdminCanister-->>EVMRPC: Request owner from Contract
    AdminCanister-->>ckNFTCanister: Report updated response
```



[link to diagram](https://mermaid.live/edit#pako:eNqdV99T4zYQ_lc0fulLyOFQIPjhOtRwHR5yxwHXdjrMZFR5TTTYkispx6UM_3tXkn_EdpyY5iWxsvtptfvtt_JrwGQCQRRo-GcNgsEVp0-K5o-C4KegynDGCyoMuTarLwUoaqQiVNtHUj33je8glwY-f3qwpv6B4FPf8DLJuYip4NqAw3ULpFrpO7BnxNl2cAt7HL5pb2etvrwIfPhwIPjr3xd3t7E7pP81DH4T31o7-zVsFC-sTbxhGZAFF87GW322eZHfMaat7E7QISJXUEjNXTAudPSThArcqjwxq3KAH4-2hXH08WMrsxGJV8CeCU_J9V18dD4LyQpjQjyqNX8SkHRQPWILAzG3doi6YcAP_Na_eEeame7_iQRNhDTe0JsdjPrOslIjlgJquBSEyWFfrEJEOFNstqRFoTCvbT-CGXT4_3dzmXYP5TymzgCW7dztyp-L0JLFR2kUFToFtUyVzIfO2MWw5IilQNK4Azla6WHzVr9E5L6gL017kZcPbK9_H8An4I_L-wW5EdrQLINkj7fvIJtLozhgRSyfmRR4dGZIAobyDHno0vgEZqmcVixz_COhhr4jLilS_rRWQPq-_heIpOq7dtk7SFXZ4XtOdIFeVj-SRIEu09SyP-q2xX3bhVgi8gTqvW1rVIW3sa7kVv6sWCFgLZ-RbdfzcFo7WBFw3KZZOyjINDR_jcWt-mQQdsu50x2f1iLpOZEXjtIqkMcIahP4RHUv-7vx7g1KJsEl40EYUsvG5bI9teu7NcmiRU5XEcGs9c1VtU9PvUou_qokTRjVW3VwDYgnYq4BUSUFMDwOVZuSMM61yqBLYNs3HfYdCuNW4gFTnKbSTiW94gVRwKTCrNrO8PLfKkYmZVEK-ZfKZ0TveY-1MDzb2ss1jMqr7nX12Rdu3cG2_xetHjugObY2qHpIOOeLNX2gCpudXDIm10N17YBcujnVEA5RcqqewRQZZdA0qofslGHEvI0tH_6mmChEtiIlwLxI9dyi7IBSeC5JnGy4LDC4WuAck63Y_7m0Rrv1o4P6UBGrTJa_EbVy1VOgnb3kjnSrpE0DOSK_QbnU9MkYYY09TxrmDLjtvG94OW_y4WY_xWtmo4VxPQwGLgiHArRMYthL4Bvw_aPDAojUjPHsS0h9tHp8b90uvGq0hbc-b-xuDcnoBo6rDXz99vj17jFOVWr_eiDakdFO_6HUtyOy4mVJel1V9dDZv65BbRrzjvJYNli4K4xE_GS83YhgnL7gucr3jHhFt694O93sPawi9qIeLS4hVQSjt6_79X0huDSWIVQQe6W4Im5kZYynG3sdbTvukbdvBTIbmrGB7xOptAXM6x5tYt6vdqW9bEGNUjbPGrPC-cYN5ITrEmycqlUBNIdwgAO5auaW93Lx-mFdcX6cAN7hKEY1XbsMJhixLqTQ4KdKMAlyUDnlCb5Dv1rAx8CsIIfHIMKfCaR0nZnH4FG8oSldG3m_ESyIUopcmwQetHznrlch4Vi4hX8xd-_nkwDfJIPoNfgRRLPj0-nZeXgxn1-chBfnJ5NgE0RHs1k4Dc_Ck_n8ZHb8czgP3ybBv1IiaDidzU-Pzy7C8OR8dnpxOnNgf7n_3IZv_wG3l11S)


---------------------------

Existing markets contact log:

- Open Sea
  - 08/15/2024 - Reach out via website
- Element
  - 08/15/2024 - Reach out via discord
- Rareable
  - 08/15/2024 - Reach out via discord and airtable submission
- Blur
  - 08/15/2024 - Reach out via discord ticket
- x2/y2
  - 08/15/2024 - Reach out via discord ticket
- Looks Rare
  - 08/15/2024 - Reach out via discord
- Sand Box
  - 08/15/2024 - Reach out via discord

---------------------------

## High Level Workplan Approach:

1. **Phase 1: Design and Specification**:
   - Finalize the detailed canister schemas and functional specs.
   - Determine the infrastructure and tools needed for development and testing.

2. **Phase 2: Development and Testing**:
   - Develop ckNFT master canister and child canisters.
   - Implement interfaces and testing with Ethereum RPC canisters in a controlled environment.
   - Unit testing, integration testing, and simulation testing for cross-chain functionality.

3. **Phase 3: Deployment and Maintenance**:
   - Deploy canisters to the Internet Computer and conduct real-world testing with limited external users.
   - Gradually open up the platform, monitoring usage and performance, and iterating based on user feedback and system performance.

4. **Future Phases**:
   - Integrate additional blockchains such as Solana and Bitcoin.
   - Extend the platform's functionality based on emerging needs and blockchain developments.

This detailed approach should provide a robust foundation for developing the ckNFT system on the Internet Computer, facilitating future expansion and upgrades.

