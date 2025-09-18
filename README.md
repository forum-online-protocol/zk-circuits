# Zero-Knowledge Circuits Documentation for Votta E-Voting System

This document provides comprehensive guidance on implementing, deploying, and using zero-knowledge circuits in the Votta e-voting system.

## Table of Contents

- [Overview](#overview)
- [System Architecture](#system-architecture)
- [Circuit Components](#circuit-components)
- [Implementation Guide](#implementation-guide)
- [Security Considerations](#security-considerations)
- [Testing & Verification](#testing--verification)
- [Deployment Process](#deployment-process)
- [Troubleshooting](#troubleshooting)
- [Advanced Topics](#advanced-topics)

## Overview

The Votta e-voting system leverages Zero-Knowledge Proofs (ZKPs) using the PLONK proving system to ensure:

- **Privacy**: Individual votes remain confidential
- **Integrity**: Vote counts are verifiable without revealing individual choices
- **Efficiency**: Batched verification reduces on-chain computation costs
- **Trust**: Cryptographic guarantees eliminate the need for trusted intermediaries

### Key Benefits

1. **Voter Privacy**: Zero-knowledge proofs verify vote validity without revealing the actual vote
2. **Batch Processing**: Multiple votes can be aggregated and verified together
3. **Fraud Detection**: Invalid batches can be challenged and proven fraudulent
4. **Gas Efficiency**: On-chain verification is more efficient than processing individual votes

## System Architecture

The ZK circuit system consists of several interconnected components:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Vote Client   │────│   Aggregator    │────│  Smart Contract │
│                 │    │    Service      │    │   (PlonkVerifier)│
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │                        │
                                │                        │
                         ┌─────────────────┐    ┌─────────────────┐
                         │   ZK Circuit    │    │   VotingBatch   │
                         │ (voting_circuit)│    │   Contract      │
                         └─────────────────┘    └─────────────────┘
```

### Components

1. **Circom Circuit**: Defines the voting verification logic
2. **PLONK Verifier**: Smart contract that verifies ZK proofs on-chain
3. **Aggregator Service**: Collects votes and generates proofs
4. **VotingBatch Contract**: Manages batch submissions and challenges

## Circuit Components

### Current Circuit Structure

The voting verification circuit (`voting_circuit.circom`) includes:

```circom
template VotingVerification() {
    // Public inputs (visible on-chain)
    signal input merkle_root;     // Root of Merkle tree containing all votes
    signal input counts_hash;     // Hash of vote count results
    
    // Private inputs (kept secret)
    signal input vote_data;       // Individual vote data
    signal input secret_key;      // Voter's private key
    
    // Output
    signal output valid_batch;    // Proof that batch is valid
    
    // Constraints (verification logic)
    valid_batch <== merkle_root * counts_hash - vote_data * secret_key;
}
```

### Production Circuit Requirements

For a production-ready voting system, the circuit should include:

#### 1. Vote Structure Validation
```circom
// Ensures votes are properly formatted
component voteValidator = VoteStructureValidator();
voteValidator.vote <== individual_vote;
voteValidator.poll_id <== poll_id;
```

#### 2. Merkle Tree Proof Verification
```circom
// Verifies vote inclusion in batch
component merkleProof = MerkleTreeInclusionProof(tree_depth);
merkleProof.leaf <== vote_hash;
merkleProof.path_elements <== merkle_path;
merkleProof.path_indices <== merkle_indices;
merkleProof.root === merkle_root;
```

#### 3. Vote Counting Logic
```circom
// Aggregates votes and verifies counts
component counter = VoteCounter(max_votes);
counter.votes <== vote_array;
counter.expected_counts <== claimed_counts;
```

#### 4. Credential Verification
```circom
// Ensures voter eligibility and uniqueness
component credentialCheck = CredentialVerifier();
credentialCheck.credential <== voter_credential;
credentialCheck.nullifier <== vote_nullifier;
```

#### 5. Signature Verification
```circom
// Verifies vote authenticity
component sigVerify = EdDSAMiMCSpongeVerifier();
sigVerify.signature <== vote_signature;
sigVerify.public_key <== voter_public_key;
sigVerify.message <== vote_message;
```

## Implementation Guide

### Step 1: Environment Setup

Install required dependencies:

```bash
# Install Node.js packages
npm install

# Install global tools
npm install -g circom snarkjs

# Install Rust (for advanced circuits)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### Step 2: Circuit Development

1. **Design the Circuit**
   - Define public and private inputs
   - Implement verification constraints
   - Optimize for gas efficiency

2. **Write the Circuit**
   ```bash
   # Create new circuit file
   touch circuits/production_voting_circuit.circom
   ```

3. **Test Locally**
   ```bash
   # Compile circuit
   circom circuits/voting_circuit.circom --r1cs --wasm --sym -o circuits/build
   
   # Generate witness
   node circuits/build/voting_circuit_js/generate_witness.js \
        circuits/build/voting_circuit_js/voting_circuit.wasm \
        circuits/build/input.json \
        circuits/build/witness.wtns
   ```

### Step 3: Trusted Setup

For production deployment, you need a secure trusted setup:

```bash
# Download or generate Powers of Tau
curl -L https://hermez.s3-eu-west-1.amazonaws.com/powersOfTau28_hez_final_12.ptau \
     -o circuits/build/pot12_final.ptau

# Generate circuit-specific setup
cd circuits/build
snarkjs plonk setup voting_circuit.r1cs pot12_final.ptau voting_circuit.zkey
```

### Step 4: Verifier Generation

Generate the Solidity verifier contract:

```bash
# Export verification key
snarkjs zkey export verificationkey voting_circuit.zkey verification_key.json

# Generate Solidity verifier
snarkjs zkey export solidityverifier voting_circuit.zkey PlonkVerifier.sol

# Copy to contracts directory
cp PlonkVerifier.sol ../../contracts/
```

### Step 5: Integration

Update your `VotingBatch` contract to use the new verifier:

```solidity
contract VotingBatch {
    IPlonkVerifier public immutable plonkVerifier;
    
    function submitBatch(
        bytes32 root,
        bytes32 countsHash,
        bytes calldata proof
    ) external {
        // Prepare public inputs
        bytes32[] memory pubInputs = new bytes32[](2);
        pubInputs[0] = root;
        pubInputs[1] = countsHash;
        
        // Verify proof
        require(
            plonkVerifier.verify(proof, pubInputs),
            "Invalid ZK proof"
        );
        
        // Process batch...
    }
}
```

## Security Considerations

### 1. Circuit Security

- **Constraint Completeness**: Ensure all necessary constraints are implemented
- **Underconstraint Detection**: Use tools like `circomspect` to detect issues
- **Trusted Setup**: Use a proper Powers of Tau ceremony for production
- **Circuit Auditing**: Have cryptography experts review the circuit

### 2. Implementation Security

```circom
// Good: Proper range checks
component rangeCheck = LessThan(8);
rangeCheck.in[0] <== vote_option;
rangeCheck.in[1] <== max_options;
rangeCheck.out === 1;

// Bad: Missing constraints
signal input vote_option;
// Missing: vote_option should be validated!
```

### 3. Key Management

- Store private keys securely
- Use hardware security modules (HSMs) for production
- Implement key rotation mechanisms
- Separate setup keys from operational keys

### 4. Proof Generation Security

```javascript
// Secure proof generation in Aggregator Service
const witness = await snarkjs.wtns.calculate(
    input,
    wasmPath,
    zkeyPath
);

const { proof, publicSignals } = await snarkjs.plonk.prove(
    zkeyPath,
    witness
);

// Validate proof before submission
const verified = await snarkjs.plonk.verify(
    verificationKey,
    publicSignals,
    proof
);
```

## Testing & Verification

### Unit Testing

Test individual circuit components:

```javascript
// Test vote structure validation
describe("Vote Structure Validation", () => {
    it("should accept valid votes", async () => {
        const circuit = await wasm_tester("circuits/vote_validator.circom");
        const input = {
            vote: validVote,
            poll_id: 1
        };
        const witness = await circuit.calculateWitness(input);
        await circuit.assertOut(witness, { valid: 1 });
    });
});
```

### Integration Testing

Test the complete proof generation and verification flow:

```javascript
describe("Full ZK Proof Flow", () => {
    it("should generate and verify valid proofs", async () => {
        // Generate proof
        const proof = await generateProof(validInputs);
        
        // Verify on-chain
        const result = await plonkVerifier.verify(
            proof.proof,
            proof.publicSignals
        );
        
        expect(result).to.be.true;
    });
});
```

### Security Testing

```javascript
// Test invalid inputs
describe("Security Tests", () => {
    it("should reject invalid vote counts", async () => {
        const invalidInput = { ...validInput, counts: [999, 999] };
        await expect(generateProof(invalidInput)).to.be.rejected;
    });
    
    it("should reject replayed proofs", async () => {
        const proof1 = await generateProof(validInput);
        const proof2 = await generateProof(validInput); // Same input
        
        // First submission should succeed
        await votingBatch.submitBatch(proof1);
        
        // Second should fail (replay protection)
        await expect(votingBatch.submitBatch(proof2)).to.be.reverted;
    });
});
```

## Deployment Process

### 1. Pre-deployment Checklist

- [ ] Circuit audited by cryptography experts
- [ ] Trusted setup completed securely
- [ ] All tests passing (unit, integration, security)
- [ ] Gas costs optimized and acceptable
- [ ] Monitoring and alerting configured

### 2. Deployment Steps

```bash
# 1. Deploy contracts
npx hardhat run scripts/deploy.js --network mainnet

# 2. Verify contracts on Etherscan
npx hardhat verify --network mainnet DEPLOYED_ADDRESS

# 3. Start aggregator service
cd services/aggregator
cargo run --release -- --config mainnet.toml

# 4. Start watchtower service
cd services/watchtower
go run main.go --config mainnet.yaml
```

### 3. Post-deployment Verification

```javascript
// Verify verifier contract works correctly
const testProof = await generateTestProof();
const result = await deployedVerifier.verify(
    testProof.proof,
    testProof.publicSignals
);
console.log("Verifier working:", result);
```

## Troubleshooting

### Common Issues

#### 1. Circuit Compilation Errors

```bash
# Error: "Not enough constraints"
# Solution: Ensure all signals are properly constrained
component dummy = IsEqual();
dummy.in[0] <== unconstrained_signal;
dummy.in[1] <== 0;
```

#### 2. Proof Generation Failures

```bash
# Error: "Witness calculation failed"
# Check input format and constraints
node circuits/build/voting_circuit_js/generate_witness.js \
     circuits/build/voting_circuit_js/voting_circuit.wasm \
     debug_input.json \
     debug_witness.wtns
```

#### 3. Gas Limit Issues

```solidity
// Optimize verifier for gas efficiency
contract OptimizedVerifier {
    // Use assembly for gas-critical operations
    function verify(bytes calldata proof, bytes32[] calldata inputs) 
        external view returns (bool) {
        assembly {
            // Low-level verification logic
        }
    }
}
```

#### 4. Trusted Setup Problems

```bash
# Re-generate if setup is corrupted
rm circuits/build/voting_circuit.zkey
snarkjs plonk setup \
    circuits/build/voting_circuit.r1cs \
    circuits/build/pot12_final.ptau \
    circuits/build/voting_circuit.zkey
```

## Advanced Topics

### 1. Circuit Optimization

```circom
// Use efficient hash functions
component hasher = MiMCSponge(2, 220, 1);
hasher.ins[0] <== data1;
hasher.ins[1] <== data2;
hash_output <== hasher.outs[0];

// Minimize constraint count
component efficientCheck = IsEqual();
efficientCheck.in[0] <== computed_value;
efficientCheck.in[1] <== expected_value;
efficientCheck.out === 1;
```

### 2. Batched Verification

For processing multiple proofs efficiently:

```solidity
contract BatchVerifier {
    function verifyBatch(
        bytes[] calldata proofs,
        bytes32[][] calldata publicInputs
    ) external view returns (bool[] memory results) {
        results = new bool[](proofs.length);
        for (uint i = 0; i < proofs.length; i++) {
            results[i] = plonkVerifier.verify(proofs[i], publicInputs[i]);
        }
    }
}
```

### 3. Upgradeability

Use proxy patterns for verifier upgrades:

```solidity
contract VerifierProxy {
    address public implementation;
    
    function upgrade(address newImplementation) external onlyOwner {
        implementation = newImplementation;
    }
    
    fallback() external {
        address impl = implementation;
        assembly {
            calldatacopy(0, 0, calldatasize())
            let result := delegatecall(gas(), impl, 0, calldatasize(), 0, 0)
            returndatacopy(0, 0, returndatasize())
            switch result
            case 0 { revert(0, returndatasize()) }
            default { return(0, returndatasize()) }
        }
    }
}
```

### 4. Layer 2 Integration

Deploy on L2 networks for reduced costs:

```javascript
// Optimism deployment
const optimismConfig = {
    url: "https://mainnet.optimism.io",
    chainId: 10,
    gasPrice: 1000000000, // 1 gwei
};

// Arbitrum deployment
const arbitrumConfig = {
    url: "https://arb1.arbitrum.io/rpc",
    chainId: 42161,
    gasPrice: 100000000, // 0.1 gwei
};
```

### 5. Performance Monitoring

```javascript
// Monitor proof generation time
const startTime = Date.now();
const proof = await generateProof(input);
const generationTime = Date.now() - startTime;

console.log(`Proof generation took ${generationTime}ms`);

// Monitor gas usage
const tx = await contract.submitBatch(proof, { gasLimit: 1000000 });
const receipt = await tx.wait();
console.log(`Gas used: ${receipt.gasUsed}`);
```

## Resources and References

### Documentation
- [Circom Documentation](https://docs.circom.io/)
- [snarkjs Documentation](https://github.com/iden3/snarkjs)
- [PLONK Paper](https://eprint.iacr.org/2019/953)

### Tools
- [circomspect](https://github.com/trailofbits/circomspect) - Circuit security analyzer
- [circom-mutator](https://github.com/aviggiano/circom-mutator) - Property testing
- [ZoKrates](https://zokrates.github.io/) - Alternative ZK toolchain

### Security Resources
- [ZK Security Guidelines](https://github.com/0xPARC/zk-security)
- [Common Circom Pitfalls](https://www.rareskills.io/post/circom-tutorial)

---

This documentation provides a complete guide for implementing and using zero-knowledge circuits in the Votta e-voting system. For additional support, refer to the existing `VERIFIER_GUIDE.md` and the generated scripts in the `scripts/` directory.
