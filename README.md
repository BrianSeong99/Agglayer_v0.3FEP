# Agglayer v0.3.0 Documentation

## Background

### Problem Statement
In the early versions of AggLayer, the system only focused on securing cross-chain transactions through a single verification method. While this approach worked for basic cross-chain operations, it didn't verify whether the individual chains themselves were operating correctly. Think of it like having a security guard only checking IDs at the entrance of a building, but not verifying if the building's internal systems are functioning properly.

The v0.3 release introduces a more comprehensive security approach, similar to having both a building security system and a security guard. First, it verifies that each chain is operating correctly by checking its internal state transitions (like verifying that all transactions are valid and balances are correct). Then, it verifies the cross-chain operations to ensure assets are being transferred safely between chains. This dual-layer security system allows different types of chains to participate in the network using their preferred verification methods - whether they're using advanced execution proofs or simpler consensus mechanisms. By implementing this two-step verification process, AggLayer v0.3 provides an extra layer of security, making cross-chain transfers significantly more secure while maintaining flexibility for different types of blockchain networks.

### What is Agglayer?
Agglayer is a revolutionary protocol that addresses the fundamental challenge of blockchain fragmentation by creating a unified framework for cross-chain interoperability. At its core, it serves as an aggregation layer that sits between blockchain ecosystems, offering three key phases of operation: Pre-Confirmation (verifying dependencies), Confirmation (validating proofs), and Finalization (aggregating proofs into a single proof posted to Ethereum). The protocol enables unified liquidity sharing, low-latency interoperability, and asset fungibility across connected chains, without compromising chain sovereignty or decentralization. It achieves this through sophisticated components including the Unified Bridge for cross-chain operations, Pessimistic Proof System for state verification, and various supporting services like AggProver, AggSender, and AggOracle.

What sets Agglayer apart is its ability to provide atomic and asynchronous interoperability while maintaining safety and modularity. The protocol ensures cryptographic guarantees against malicious actions like double-spending, enables cross-chain transactions to execute as atomic bundles, and allows chains to interact safely without waiting for Ethereum finality. This design creates a Web3 experience that feels unified, similar to how TCP/IP unified the internet. For example, users can transfer ETH from Polygon to mint NFTs on Zora in a single transaction without using third-party bridges or wrapped tokens. The protocol's intentionally minimal design focuses on safety and interoperability while allowing chains to retain their sovereignty and choose their level of integration, making it a potential backbone for a truly unified decentralized future.

### What is a Unified Bridge?
The Unified Bridge is a fundamental component of the Agglayer ecosystem that facilitates seamless cross-chain communication and asset transfers between different networks. It consists of three main components: Bridge Contracts (smart contracts deployed on each network to handle bridging operations), Bridge Service (manages inter-network communication and synchronizes global exit roots), and Tools (utilities for proof generation and verification). The bridge supports two primary operations: Asset Bridging for transferring tokens and other assets between networks, and Message Bridging for cross-chain communication. To maintain security and consistency, it employs several key data structures including Local Exit Root (LER), Rollup Exit Root, Mainnet Exit Root, Global Exit Root, L1 Info Tree, and Global Index. This standardized interface ensures secure and efficient cross-chain operations while providing a unified approach to bridging different networks within the Agglayer ecosystem.

For a more detailed understanding of the Unified Bridge architecture, implementation, and usage, please refer to the [Agglayer Unified Bridge Repository](https://github.com/BrianSeong99/Agglayer_UnifiedBridge).

![Unified Bridge Data Structure](./pics/UnifiedBridgeTree.png)

### What is a Pessimistic Proof?
The Pessimistic Proof is a critical security mechanism in Agglayer that verifies state transitions across different networks. It works through a four-step process: First, local chains prepare their state data and transition information in a Certificate format, which includes previous and new local exit roots, bridge exits, and imported bridge exits. Second, the Agglayer client populates a MultiBatchHeader using this certificate data. Third, the system runs the Pessimistic Proof program in native Rust to verify the state transition's validity, comparing the computed new state with the expected state in the batch header. Finally, the same program is executed in a zkVM (currently using SP1 Provers Network from Succinct Labs) to generate a zero-knowledge proof of the computation. The system uses several key data structures including PessimisticProofOutput (which contains roots and hashes of the state transition), Certificate (representing a chain's state transition), and various tree structures for tracking balances and nullifiers. This proof system is designed to be flexible and secure, supporting both ECDSA and Generic consensus types, while maintaining compatibility with existing systems.

For a more detailed understanding of the Pessimistic Proof architecture, implementation, and usage, please refer to the [Agglayer Pessimistic Proof Repository](https://github.com/BrianSeong99/Agglayer_PessimisticProof_Benchmark).

![How does Pessimistic Proof works](./pics/PessimisticProofFlow.png)

# Proof of Settlement

Proof of Settlement is a fundamental concept in AggLayer that ensures the security and validity of cross-chain operations. Think of it as a comprehensive verification system that works in two layers:

1. **Internal Settlement Verification**: This layer verifies that each chain's internal state transitions are valid. It's like checking that all transactions within a chain are properly executed and the chain's state is consistent. This is done through either:
   - Full Execution Proof (FEP): A detailed verification of every operation in the chain
   - Proof of Consensus: A verification that the chain's consensus mechanism has approved the state change

2. **Cross-Chain Settlement Verification**: This layer verifies that cross-chain operations (like asset transfers between chains) are valid. It ensures that when assets move between chains, the operations are atomic and secure.

The combination of these two layers provides a robust security model similar to two-factor authentication - both the internal chain operations and the cross-chain transfers must be verified for a transaction to be considered valid. This dual-layer approach ensures that AggLayer can maintain security while supporting different types of chains with varying consensus mechanisms.

## Aggchain Proof

Aggchain Proof is a flexible verification system in AggLayer that supports different types of consensus mechanisms for proving chain state transitions. Think of it as a universal adapter that can work with various types of chains, whether they use simple signature-based verification or more complex proof systems.

The system supports two main types of proofs:

1. **ECDSA Proof**: A simpler verification method where a trusted sequencer signs off on state changes. This is like having a security guard verify and approve changes to a building's access system.

2. **Generic Proof**: A more flexible system that can work with any type of chain-specific proof system. This is like having a universal translator that can understand and verify different types of security protocols.

The key innovation of Aggchain Proof is its ability to combine these different proof types with bridge verification, ensuring that both the chain's internal operations and cross-chain transfers are secure. This flexibility allows AggLayer to support a wide range of chains while maintaining strong security guarantees.

### Aggchain Proof Data Structure
```rust
pub enum AggchainData {
    /// ECDSA signature.
    ECDSA {
        /// Signer committing to the state transition.
        signer: Address,
        /// Signature committing to the state transition.
        signature: Signature,
    },
    /// Generic proof and its metadata.
    Generic {
        /// Chain-specific commitment forwarded by the PP.
        aggchain_params: Digest,
        /// Verifying key for the aggchain proof program.
        aggchain_vkey: Vkey,
    },
}
```

## ECDSA

### Description

The ECDSA (Elliptic Curve Digital Signature Algorithm) implementation is the original consensus mechanism used in AggLayer. In this system, a trusted sequencer (a designated address) acts as a security guard, signing off on state changes to ensure they are valid. When a chain wants to update its state or perform cross-chain operations, the trusted sequencer must verify and sign these changes using their private key. This signature serves as proof that the changes are legitimate and authorized. The system uses this signature to verify state transitions across three important data structures: the local exit tree (tracking cross-chain exits), the local balance tree (tracking token balances), and the nullifier tree (preventing double-spending).

### Aggchain Parameters

The system uses a single Ethereum address (20 bytes) as the trusted sequencer:

```solidity
/**
* aggchain_params:
* Field:           | trusted_sequencer |
* length (bits):   | 160               |
*/
uint256 aggchain_params = keccak256(abi.encodePacked(trusted_sequencer))
```

Where:
- `trusted_sequencer`: The Ethereum address that is authorized to sign state changes. This address signs a message containing the new local exit root and the commitment to imported bridge exits.

### Program Implementation

#### Public Inputs
- `public_values`: A hash combining the old local exit root, L1 info tree root, rollup ID, new local exit root, commitment to imported bridge exits, and the trusted sequencer's parameters.

#### Execution
1. Create a message to sign by combining the new local exit root and the commitment to imported bridge exits
2. Verify the signature using the trusted sequencer's public key
3. Ensure the signature matches the trusted sequencer's address

```javascript
// Public inputs
const old_ler;
const l1_info_tree_root;
const rollup_id;
const new_ler;
const commit_imported_bridge_exits;
const trusted_sequencer;

// Build message to sign
const message_to_sign = keccak(new_ler # commit_imported_bridge_exits);

// Verify signature
const from = ecrecover(message_to_sign, r, s, v);
assert(from === trusted_sequencer);
```

## FEP (Full Execution Proof)

### Description

The Full Execution Proof (FEP) is a more advanced consensus mechanism that provides comprehensive verification of chain operations. Unlike the simpler ECDSA approach, FEP is a proof system that verifies every aspect of a chain's state transition, in this case `op-geth` operations. This system is particularly useful for chains that need to prove their entire state transition is valid, not just that it was authorized by a trusted party. Then Aggchain Proof will combine FEP state transition proofs with bridge checks to ensure both internal chain operations and cross-chain transfers are valid.

### Aggchain Parameters

The system uses several parameters to verify the chain's state:

```solidity
/**
  * aggchain_params:
  * Field:           | l2PreRoot         | claimRoot          | claimBlockNum      | rollupConfigHash     | optimisticMode  | trustedSequencer |
  * length (bits):   | 256               | 256                | 256                | 256                  | 8               | 160              |
  */
uint256 aggchain_params = keccak256(abi.encodePacked(
    l2PreRoot,
    claimRoot,
    claimBlockNum,
    rollupConfigHash,
    optimisticMode,
    trustedSequencer
))
```

These parameters include:
- `l2PreRoot`: The previous state of the chain
- `claimRoot`: The new state being claimed
- `claimBlockNum`: The block number being verified
- `rollupConfigHash`: Configuration settings for the chain
- `optimisticMode`: Whether to use optimistic verification
- `trustedSequencer`: The address authorized to make changes

### Program Implementation

#### Public Inputs
- `public_values`: A hash combining various state elements including the old and new local exit roots, L1 info tree root, rollup ID, and bridge exit commitments.

#### Execution

The verification process happens in two main steps:

1. State Transition Verification:
   - Verify that the chain's state transition is valid
   - Check that all operations in the transition are correct
   - Ensure the new state is properly derived from the old state

2. Bridge Constraint Verification:
   - Verify that cross-chain transfers are valid
   - Check that global exit roots exist on the main chain
   - Validate that all claims in the state transition are legitimate

```javascript
// Constants
const rangeVkeyCommitment;
const aggregationVkey;
const bridge_address;
const ger_manager_address;

// Public inputs
const old_ler;
const l1_info_tree_root;
const rollup_id;
const new_ler;
const commit_imported_bridge_exits;
const l2_pre_root;
const claim_root;
const claim_block_num;
const rollup_config_hash;

// Verify state transition
const publics_op_fep = sha256({
    l1_blockHash,
    l2_pre_root,
    claim_root,
    claim_block_num,
    rollup_config_hash,
    range_vkey_commitment
});

// Handle different verification modes
if (optimisticMode === 0) {
    // Full verification mode
    proveL1BlockHashExist(l1_info_tree_root, l1_blockHash);
    sp1.verify(aggregation_vkey, publics_op_fep);
} else {
    // Optimistic mode - faster but requires trusted sequencer
    const message_to_sign = keccak(publics_op_fep # new_ler # commit_imported_bridge_exits);
    const from = ecrecover(message_to_sign, r, s, v);
    assert(from === trusted_sequencer);
}

// Verify bridge constraints
// ... (bridge constraint verification logic)
```

## Components

### AggProver
The AggProver is responsible for generating proofs that are then aggregated. It handles:
- State transition verification
- Proof generation
- Bridge constraint verification

### AggSender
The AggSender manages the communication of proofs to the Agglayer for verification. It ensures:
- Proper proof transmission
- Network communication
- State synchronization

### AggOracle
The AggOracle provides necessary on-chain data for proof verification. It supplies:
- L1 block data
- Bridge state information
- Network state updates

### Other Key Components
- **Trusted Sequencer Address**: Manages consensus verification
- **SP1 Verifier**: Handles proof verification
- **PLONK Verifier**: Manages generic proof verification
- **zkEVM State**: Maintains the state of the system

## How It Works

### Step 1: Initial State Verification
1. Verify previous local exit root
2. Check pessimistic root validity
3. Validate L1 info root
4. Confirm origin network

### Step 2: Proof Generation and Verification
1. Generate aggchain proof
   - For ECDSA: Create and verify signature
   - For Generic: Generate SP1 proof
2. Verify proof validity
3. Check version consistency
4. Validate state transitions

### Step 3: Bridge Constraint Verification
1. Verify imported bridge exits
2. Check global exit roots
3. Validate bridge operations
4. Confirm state consistency

### Step 4: State Update
1. Update local exit root
2. Modify pessimistic root
3. Update bridge state
4. Commit changes

### Error Handling
The system handles various error scenarios:
- Invalid signatures
- Inconsistent payload versions
- Invalid signers
- Height overflow
- Balance overflows/underflows
- Invalid bridge exits

## Implementation Details

### Aggchain Hash Computation
```rust
impl AggchainData {
    pub fn aggchain_hash(&self) -> Digest {
        match &self {
            AggchainData::ECDSA { signer, .. } => keccak256_combine([
                &(ConsensusType::ECDSA as u32).to_be_bytes(),
                signer.as_slice(),
            ]),
            AggchainData::Generic {
                aggchain_params,
                aggchain_vkey,
            } => {
                let mut aggchain_vkey_hash = [0u8; 32];
                BigEndian::write_u32_into(aggchain_vkey, &mut aggchain_vkey_hash);

                keccak256_combine([
                    &(ConsensusType::Generic as u32).to_be_bytes(),
                    aggchain_vkey_hash.as_slice(),
                    aggchain_params.as_slice(),
                ])
            }
        }
    }
}
```

## v0.2 vs v0.3 Comparison

### Major Changes

1. **Consensus Type Evolution**
   - **v0.2**: Single consensus type (`CONSENSUS_TYPE = 0`) using ECDSA signatures
   - **v0.3**: Introduces generic consensus mechanism (`CONSENSUS_TYPE = 1`) while maintaining backward compatibility

2. **Proof Generation**
   - **v0.2**: 
     - Fixed proof structure
     - Limited to ECDSA-based verification
     - Single verification path
   - **v0.3**:
     - Flexible proof structure through generic aggchains
     - Multiple verification paths (ECDSA, FEP)
     - Enhanced proof generation pipeline

3. **State Management**
   - **v0.2**:
     - Simple state transitions
     - Basic bridge constraints
     - Limited state verification
   - **v0.3**:
     - Enhanced state transition handling
     - Comprehensive bridge constraints
     - Advanced state verification mechanisms

### Technical Improvements

1. **Architecture Changes**
   ```solidity
   // v0.2: Fixed consensus structure
   struct ConsensusData {
       bytes32 consensus_hash;
       bytes signature;
   }

   // v0.3: Generic consensus structure
   struct GenericConsensusData {
       uint32 consensus_type;
       bytes32 aggchain_vkey;
       bytes32 aggchain_params;
       bytes proof;
   }
   ```

2. **Verification Process**
   - **v0.2**:
     ```solidity
     function verifyConsensus(bytes32 messageHash, bytes memory signature) {
         address signer = ecdsa.recover(messageHash, signature);
         require(signer == trustedSequencer, "Invalid signature");
     }
     ```
   - **v0.3**:
     ```solidity
     function verifyGenericConsensus(
         uint32 consensusType,
         bytes32 aggchainVkey,
         bytes32 aggchainParams,
         bytes memory proof
     ) {
         if (consensusType == 0) {
             // Legacy ECDSA verification
             verifyLegacyConsensus(proof);
         } else {
             // Generic consensus verification
             verifyGenericProof(aggchainVkey, aggchainParams, proof);
         }
     }
     ```

3. **Bridge Integration**
   - **v0.2**:
     - Basic bridge exit verification
     - Simple global exit root management
     - Limited cross-chain message support
   - **v0.3**:
     - Enhanced bridge exit verification
     - Advanced global exit root management
     - Comprehensive cross-chain message support
     - Improved bridge constraint validation

### Performance Enhancements

1. **Proof Generation**
   - **v0.2**: Single-threaded proof generation
   - **v0.3**: 
     - Parallel proof generation
     - Optimized proof structure
     - Enhanced verification efficiency

2. **State Updates**
   - **v0.2**: Sequential state updates
   - **v0.3**:
     - Batched state updates
     - Optimized storage access
     - Improved atomicity guarantees

3. **Bridge Operations**
   - **v0.2**: Basic bridge operation handling
   - **v0.3**:
     - Efficient GER management
     - Optimized index tracking
     - Enhanced constraint verification

### Security Improvements

1. **Verification Mechanisms**
   - **v0.2**:
     - Basic ECDSA signature verification
     - Simple state transition checks
   - **v0.3**:
     - Multiple verification paths
     - Enhanced state transition validation
     - Comprehensive security checks

2. **Error Handling**
   - **v0.2**: Basic error handling
   - **v0.3**:
     - Detailed error types
     - Enhanced error reporting
     - Improved error recovery

### Migration Path

1. **From v0.2 to v0.3**
   ```solidity
   // v0.2 consensus data
   struct V2Consensus {
       bytes32 consensus_hash;
       bytes signature;
   }

   // v0.3 generic consensus data
   struct V3Consensus {
       uint32 consensus_type;
       bytes32 aggchain_vkey;
       bytes32 aggchain_params;
       bytes proof;
   }

   // Migration helper
   function migrateConsensus(V2Consensus memory v2Data) 
       internal 
       pure 
       returns (V3Consensus memory) 
   {
       return V3Consensus({
           consensus_type: 0, // Legacy type
           aggchain_vkey: bytes32(0),
           aggchain_params: keccak256(abi.encodePacked(v2Data.consensus_hash)),
           proof: v2Data.signature
       });
   }
   ```

2. **Backward Compatibility**
   - v0.3 maintains support for v0.2 consensus type
   - Gradual migration path for existing implementations
   - Deprecation timeline for legacy consensus type

### Future Considerations

1. **Planned Deprecations**
   - `CONSENSUS_TYPE = 0` will be deprecated
   - Legacy verification paths will be removed
   - Migration to generic consensus required

2. **Upcoming Features**
   - Enhanced generic consensus support
   - Additional verification mechanisms
   - Improved performance optimizations

[Diagram 7: v0.2 vs v0.3 architecture comparison]


## References
- [Agglayer Documentation](https://docs.polygon.technology/Agglayer)
- [GitHub Repository](https://github.com/0xPolygon/Agglayer)
- [Technical Specifications](https://github.com/0xPolygon/Agglayer-specs)
- [Integration Guide](https://docs.polygon.technology/Agglayer/integration)