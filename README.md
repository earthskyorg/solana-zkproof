# solana-zk-proof

## Overview

This is an example of creating a ZK-SNARK proof and verifying it on-chain. This is a technique used for "zk-rollups", among other use cases.

This is the basic order of events:

1. A prover generates a proof.
2. The proof is sent to an on-chain contract.
3. The contract checks the validity of the proof.
4. If the proof is valid, the contract performs some action (like updating account state).

What are some of the challenges associated with verifying ZK proofs on-chain? Here are a few:

1. Computational cost:
   While ZK proofs are generally more efficient to verify than to generate, the verification process still requires significant computational resources.
2. Trusted setup:
   Generally, ZK-SNARK systems require a trusted setup phase. This initial setup generates public parameters used for creating and verifying proofs. This is a security concern as the setup may become compromised.
3. Complexity:
   This stuff is hard...there is a steep learning curve. Correctly implementing ZK proof verification in smart contracts is challenging and requires cryptographic expertise. Fudging it can lead to security vulnerabilities.

Let's get into it!

### Running the example

Start your local validator
```shell
solana-test-validator
```

```shell
cd proof-verify
cargo build-bpf
solana program deploy ./target/deploy/solana_zk_example.so
```

Replace the program ID in main.rs 9PMYmoKdNk67c9Gumo8WWNFpGwmmHfZ4BvFR2rh1winq with your program ID.
```shell
cd on-chain-program-example
cargo build
cargo test
```

## Let's Begin

This tutorial will mostly use libraries from [Solana](https://github.com/anza-xyz/agave) and [Arkworks](https://github.com/arkworks-rs)

### Proof

We'll be focusing on ZK-SNARK which stands for “Zero-Knowledge Succinct Non-Interactive Argument of Knowledge”. That sounds totally awesome. Essentially, it's a mathematical way for one party to prove to another that they know a secret without revealing the secret itself.

In this case we're using a Groth16 BN254 proof (Groth16 SK-SNARKs over BN254 elliptic curve constructions). Okay...wtf does that mean?

Groth16 is a (pairing-based) proof system. We aren't gonna get into the weeds here, you can do that on your own time, but here are some things to note:

- Short proofs (only 3 group elements)
- Fast verification (only a few pairing operations)
- Requires a trusted setup (as noted in the concern above)
- Popular with various blockchains

BN254 is a pairing friendly elliptic curve. Again, not gonna get super detailed here, so it's up to you to do research. Here is what's important for this tutorial:
- 254-bit prime field
- Suitable for pairing-based cryptography
- Popular in blockchain and zk proof systems
- Also known as BN128 (128 previously referred to the bits of security) or **alt_bn_128** (foreshadowing)

Enough already, let's see some code...

Okay, in order to generate a proof we need a circuit.

```rust
let circuit = ExampleCircuit {
    some_value: Some(Fr::from(100)),
};
```

Alright, so what? What this really means is:

```rust
///@see circuit.rs
impl ConstraintSynthesizer<Fr> for ExampleCircuit {
    fn generate_constraints(self, cs: ConstraintSystemRef<Fr>) -> Result<(), SynthesisError> {
        // Allocate public inputs
        let some_value_var =
            cs.new_input_variable(|| self.some_value.ok_or(SynthesisError::AssignmentMissing))?;

        // Constraint: Ensure computed addresses_hash matches the provided addresses_hash
        cs.enforce_constraint(
            lc!() + some_value_var,
            lc!() + Variable::One,
            lc!() + some_value_var,
        )?;

        Ok(())
    }
}
```

This is where the actual constraints of the circuit are defined. Luckily, this is a simple example:

- It creates a new variable in the constraint system representing some_value.
- It adds a constraint that essentially says "some_value multiplied by 1 should equal some_value". This is a trivial constraint for tutorial purposes, but these can become complicated quickly as noted in the concerns.

Now that we have our circuit, we'll need a verifying key, proving key and random number generator.

```rust
let rng = &mut thread_rng();
let (proving_key, verifying_key) =
    Groth16::<Bn254>::circuit_specific_setup(circuit, rng).unwrap();
```
This creates a mutable reference to a random number generator (rng) via thread_rng() which provides a cryptographically secure random number generator that's local to the current thread.
As a result, we get a proving and verifying key. These keys are specifically for our circuit and will be used in the proving and verifying processes.
We want to hold on to these as they will be used for all future proofs and verifications related to this circuit.

```rust
let rng = &mut thread_rng();

let proof = Groth16::<Bn254>::prove(&proving_key, circuit, rng).unwrap();
```

We call prove with Groth16::<Bn254> where Groth16 is the specific ZK proof protocol used and <Bn254> says that it's using the BN254 elliptic curve.

The resulting proof is a cryptographic object that can be shared publicly. It allows anyone with the corresponding verification key to confirm that the prover knows a valid solution to the circuit, without learning anything about the solution itself beyond what is explicitly allowed by the circuit definition.
For example, in our ExampleCircuit, the proof would demonstrate that the prover knows the value of some_value that satisfies the constraint we defined, without revealing what some_value actually is.
This proof generation step would typically be performed by a party who has some secret information (in this case, the value of some_value) and wants to prove they have a valid value without revealing it. The resulting proof can then be sent to a verifier, who can check its validity using the verification key from the setup phase.

### Beyond a reasonable doubt...the burden of proof

Nice, we have a proof. Now how do we prove this thing? If you aren't in a compute constrained environment then it's relatively straight forward.
# solana-zk-proof-example

## Overview

This repository contains a concise, practical example of generating a Groth16 ZK-SNARK proof using Arkworks and verifying that proof on Solana using the chain's pairing primitives (alt_bn128 pairing).

The project demonstrates the end-to-end flow:

- Generate a circuit and produce a proof (off-chain).
- Convert and serialize proof and verifying key material to the format expected by Solana.
- Verify the proof on-chain using Solana's pairing syscall.

Goals:

- Provide a reference implementation for Groth16 proof generation and Solana-compatible verification.
- Show how to prepare keys, serialize points, and handle endianness for Solana.
- Offer a minimal on-chain verifier that uses `alt_bn128_pairing` for efficient verification.

## Repository layout

- `on-chain-program-example/` — Solana program code (Rust) that implements the verifier and example instruction handling.
- `proof-verify/` — Tools and examples for generating proofs, converting verifying keys, and preparing serialized inputs.
- `README.md` — This document.

## Requirements

- Rust (stable). Install from https://www.rust-lang.org/tools/install
- `cargo` (comes with Rust)
- Solana CLI and `solana-test-validator` — for local testing and deploying the program: https://docs.solana.com/cli
- `arkworks` crates (used in code; managed via Cargo.toml in subprojects)

Note: The commands below assume a POSIX-like shell or PowerShell on Windows. Adjust paths accordingly if required.

## Quickstart

1. Start a local Solana validator:

```powershell
solana-test-validator
```

2. Build and deploy the on-chain program (from the `proof-verify` folder in this project):

```powershell
cd proof-verify
cargo build-bpf
solana program deploy .\target\deploy\solana_zk_example.so
```

3. Update the program ID in the example `on-chain-program-example` code (if required). Then build and run tests for the on-chain example:

```powershell
cd ..\on-chain-program-example
cargo build
cargo test
```

Replace any example program IDs in source files with the ID returned by `solana program deploy`.

## Key concepts (brief)

- Circuit: A set of arithmetic constraints describing the statement being proven.
- Proving key / verifying key: Circuit-specific parameters produced in a trusted setup step for Groth16.
- Proof: A short cryptographic object (A, B, C) proving the witness satisfies the circuit.
- Verification on Solana: Convert proof and verifying key material into byte arrays (G1/G2 points) with correct endianness and call `alt_bn128_pairing`.

Important implementation details in this repo:

- Endianness conversion: Solana expects a different byte order for curve coordinates than some libraries — consistent conversion is required.
- Negating `A` (the `a` component): Many Groth16 verification implementations expect `A` to be negated as part of the standard pairing equation rearrangement.
- Ordering of pairing inputs: The pairing syscall expects inputs in the order aligned with the Groth16 verification equation — correctness depends on precise ordering.

## Where to look in the code

- `proof-verify/src/` — proof generation, key conversion, serialization helpers, and examples generating the prepared inputs for the on-chain verifier.
- `on-chain-program-example/src/` — Solana program demonstrating how to deserialize the prepared verifier and call `alt_bn128_pairing`.

Search for these symbols in the codebase if you need to follow the flow:

- `convert_arkworks_verifying_key_to_solana_verifying_key_prepared` — converting Arkworks VK to Solana-ready bytes.
- `prepare_inputs` / `prepare_public_input` — combines public inputs into a G1 point used on-chain.
- `Groth16VerifierPrepared::verify` — builds pairing input and calls `alt_bn128_pairing`.

## Development notes

- Tests: See `on-chain-program-example/src/main.rs` for pairing-related tests (for example `test_alt_bn128_pairing_custom`). These help validate byte formats and pairing logic against known-good results.
- Debugging: Log intermediate serialized points and parsed G1/G2 values when debugging mismatches between off-chain verification (Arkworks) and on-chain checks.

## Suggested next steps / improvements

- Add an automated script to generate verifying keys and write the prepared verifying key to a file that the on-chain program can read during deployment/initialization.
- Add CI to build both crates and run core tests.
- Add additional example circuits that exercise more realistic constraints.

## Resources

- Arkworks: https://github.com/arkworks-rs
- Solana developer docs: https://docs.solana.com
- Example projects using Groth16 on Solana: Lightprotocol (https://github.com/Lightprotocol/groth16-solana)

## Contributing

Contributions are welcome. Please open issues or pull requests for bugs, improvements, or documentation enhancements.

## License

This repository is licensed under the terms in `LICENSE`.
