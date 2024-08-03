---
title: "ProofPlus - Decentralised Prover Protocol"
date: 2024-07-15T23:39:49+02:00
description: "Potential solution to the current problem of centralized zkVM provers"
# image: "/static/images/avatar.jpg"
type: "post"
tags: ["ethereum"]
showTableOfContents: true
---
# WHY?

Imagine a world where zero-knowledge proofs (ZKPs) are accessible to everyone, where the power to generate these proofs is not confined to a few centralized entities but is spread across a vast, decentralized network. ProofPlus exists to make this vision a reality.

At the heart of ProofPlus is the belief that decentralization breeds innovation and resilience. We started with a simple question: Why should the power to verify complex computations be restricted to a few, when it could be democratized for all? Centralized zkVM provers pose a significant risk by creating single points of control, which can lead to censorship and inefficiencies. This goes against the very essence of blockchain technology, which thrives on transparency, security, and decentralization.

ProofPlus aims to disrupt this status quo by creating a permissionless, decentralized prover market. Our innovative ticketing mechanism ensures that both enterprise-grade and small-scale provers can compete on an even playing field, democratizing access to proof generation tasks. We are committed to fostering a community-aligned approach, where our open-source protocol is built, reviewed, and improved by its users.

We exist to empower developers and organizations of all sizes to leverage zero-knowledge technology without fear of centralization. By decentralizing proof generation, we enhance the security, scalability, and privacy of blockchain applications, driving greater adoption and trust in this transformative technology. ProofPlus is not just a protocol; it’s a movement towards a more inclusive, transparent, and resilient blockchain ecosystem.

# What is ProofPlus?

ProofPlus is a permissionless, decentralized prover market protocol that utilizes a ticketing mechanism to democratize proof generation tasks.

ProofPlus aims to revolutionize the zero-knowledge proof (ZKP) landscape by enabling both enterprise-grade and small-scale provers to participate and earn revenue. The protocol ensures inclusivity and fairness through its ticketing system, which balances the playing field based on self-reported proof generation capabilities.

**Permissionless:** ProofPlus allows everyone to participate in the proof market, providing equal opportunities for all participants to earn revenue, regardless of their scale.

**Community-aligned:** ProofPlus is built with the support of its community and is fully open-source. This means anyone can use, modify, and contribute to the ProofPlus source code under a permissive license. The project encourages community engagement in reviewing, auditing, testing, and improving the protocol, fostering a collaborative and transparent ecosystem.

# Introduction

Zero-knowledge technology significantly enhances blockchain by providing privacy, scalability, and efficiency. It allows transactions to be verified without revealing sensitive information, ensuring user confidentiality while maintaining trust. This is particularly valuable in financial and identity-related applications, where privacy is crucial. Additionally, zero-knowledge proofs can improve scalability by reducing the amount of data processed on-chain, leading to lower operating costs. By enabling secure and private interactions, zero-knowledge technology fosters greater adoption of blockchain across various sectors, promoting a more secure and efficient decentralized ecosystem.

Groth16 proofs, known for their succinctness, allow complex computations to be verified with minimal calldata, reducing on-chain processing and transaction costs. Compared to executing the same logic in the Ethereum Virtual Machine (EVM), these proofs offer a more cost-effective solution in terms of on-chain gas. Currently, a Groth16 proof verification on-chain costs about a constant ~280k gas without regard of the size and complexity of the underlying logic that is being proven.

## Current State of zkVMs

Recently, user-friendly zero-knowledge virtual machines (zkVMs) like [SP1](https://docs.succinct.xyz/getting-started/quickstart.html) and [Risc Zero](https://dev.risczero.com/api) have emerged, streamlining the integration of zero-knowledge proofs into various platforms. These zkVMs simplify the development process, enabling developers to leverage advanced cryptography without extensive expertise, thus broadening the adoption of secure, privacy-preserving technologies across industries.

## Challenges zkVM projects are facing

Zero-knowledge virtual machines (zkVMs) face challenges, particularly with the slow generation of Groth16 proofs. While these proofs are efficient for verification, their creation can be computationally intensive. For instance, generating a proof to validate a single BLS signature can take over eight hours on a high-end GPU virtual machine. 

In addition to proof generation time, current zkVM provers face centralization challenges. Relying on a single entity to provide valid proofs is suboptimal, as it introduces the risk of censorship and control over request processing. Decentralizing the proof generation process is crucial to ensuring transparency and resilience, allowing for a more secure and trustworthy blockchain ecosystem where no single point of failure or control exists.

## Terminology

The following is a list of terms and their corresponding definitions

**Zero-Knowledge Proof (ZKP)**: A cryptographic method that allows one party to prove the validity of a statement without revealing any underlying information. This ensures privacy and security in digital transactions by confirming facts without exposing sensitive data.

**RISC-V ELF Binary (Binary)**: An executable file format for programs compiled on the RISC-V architecture, using the Executable and Linkable Format (ELF). It contains machine code and metadata, enabling programs to run on RISC-V processors.

## Actors

**User** - requests proof generation tasks

**Prover** - Generates on-demand ZKPs. Sophisticated entity tasked with fast proof generation. It may use hardware acceleration such as FPGAs, enterprise grade GPUs or ASICs.

**Orchestrator network** - orchestrates proof generation tasks and resolves conflicts in case of malicious behaviour.

## Flow

1. The user requests proof generation from the On-chain protocol using an SDK.
2. The user listens for a specific event from the Orchestrator network to get the server URL of the assigned prover.
3.  The user requests a Zero-Knowledge Proof (ZKP) from the prover.
4. The prover generates the ZKP and simultaneously:
    1. Submits the task result to the Orchestrator network.
    2. Returns the ZKP to the user.
5. The user listens for a task completion event from the Orchestrator network, then validates the received proof and data against those on-chain.
If there is a mismatch or a liveness fault, the user triggers a slashing method.

![architecture](/images/decentralized-network-architecture.png)

## Orchestrator Network

The  Orchestrator network - an on-chain protocol - is responsible for conflict resolution and orchestration of proof requests.

It will delegate proof generation request tasks, binding a task to a prover. Instead of relying on the reputation of the prover, It will rank provers based on objective metrics, such as proof generation success rate (liveness) and proof generation time.

In order for provers to participate in the protocol, they will need to register by depositing pre-defined collateral, serving as stake, and report their capabilities and public endpoint Based on the self-reported proof generation time, the protocol will expect given tasks to be completed on-time, otherwise the prover will be subject to slashing conditions.

### DTO

The Orchestrator Network keeps track of task structs with the following structure:

```jsx
    struct TaskStatus {
        address requester;
        address prover;
        uint256 reward;
        bytes32 imageId;
        bytes32 publicInputsHash;
        bytes32 proofHash;
    }
```

`publicInputsHash` and `proofHash` are reported by the Prover SDK on proof task submission and the `taskId` is internally generated by the protocol contracts.

## Prover

A lightweight SDK, on-chain listener and server are implemented to aid the prover:

- ProverServer - that exposes a POST endpoint `/proof`, allowing the User to pass all constituents necessary for proof generation - the ELF binary and the private inputs.
- An on-chain listener that fetches the proof generation request metadata and verifies correctness of the downloaded binary and inputs against the hashes.
- It then starts the proof generation process locally. Upon successful proof generation, it returns the proof back to the requester off-chain. It also submits a task submission response to the on-chain protocol with the `taskId , proofHash and binaryHash`.

## ProofPlus Client SDK

An SDK module is implemented for the User that serves as a client to the protocol - `ProofPlusProver` - that:

- Initiates an on-chain request to the Orchestator network.
- listens for an event containing the `endpoint` of the server of the Prover tasked with the proof generation
- long-polls the prover for the ZKP response at the fetched `endpoint`
- verifies its received proof against the `proofHash` , as the Prover is also required to submit the task completion transaction to the Orchestration network

## Slashing

The SDK also serves as a Slasher. In case of mismatch of the locally computed proof hash and the one on-chain, or in case of liveness fault, the SDK calls the `slash` method, passing the whole proof, it’s local public inputs, the `taskId`, and the whole binary. If the proof with the public inputs (passed as calldata) verifies correctly on-chain and the binary, when hashed, don’t correspond to the `publicInputsHash` , then the Prover was mallicious and gets slashed.

## Roadmap

- Add SP1 support
- Add ProofPlus token EIP-712 Permits