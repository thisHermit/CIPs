---
CIP: 54
Title: Plutus Contract Blueprint
Authors: KtorZ <matthias.benkort@cardanofoundation.org>, scarmuega <santiago@carmuega.me>, satran004
Discussions-To: https://github.com/cardano-foundation/CIPs/pull/258
Status: Draft 
Type: Process
Created: 2022-05-15
License: CC-BY-4.0
---

## Abstract

This document specifies a language for documenting Plutus contracts in a machine-readable manner. This is akin to what [OpenAPI](https://swagger.io/specification) or [AsyncAPI](https://www.asyncapi.com/docs/specifications/v2.4.0) are for, documenting HTTP services and asynchronous services respectively. In a similar fashion, Plutus contracts have an interface which is mostly defined by its datums and redeemers structures. Plus, contracts evolving around state-machines can be represented as a serie of state transitions. This document is therefore a meta-specification fixing the vocabulary by which one can specify a Plutus contract interface.

## Motivation

While publicly accessible, on-chain contracts are currently inscrutable. Ideally, one would want to get an understanding of transactions surrounding certain script executions. This is both useful to visualize and control the evolution of a contract life-cycle; but also, as a user interacting with a contract, to ensure that I am authorizing a transaction to do what I intend to. Having a machine-readable specification in the form of a JSON-schema makes it easier (possible) to create a wide variety of use-cases around a single concise document, such as:

- Contract API Reference / Documentation
- Code generators for serialization/deserialization of Contract's elements
- Extra transaction validation layers 
- Better wallet UI / integration with DApps
- Automated Plutus-code scaffolding 

## Specification

To be written, for now, exploring what the result could look like. Ultimately, we want to top-level specification to be have its own meta-schema to disambiguate any doubt and allow validating blueprints automatically. 

Generally speaking, we also want the specification to match the following requirements:

- [x] 1. Must be machine-readable 

- [x] 2. Should be JSON/YAML based 

- [x] 3. Must provide ways of specifying **at least** datums and redeemers of a _"contract"_. These are ultimately Plutus 'Data', which are fully captured by the following types:
  - sum
  - list
  - map 
  - bytes
  - integer

  Those represent our primitive types and are used to construct any higher-level type definition.

- [x] 4. For state-machine based validators, should allow specifying the state transitions assuming that states are captured as datums and transitions as redeemers.

- [x] 5. Should come with few types builtins higher-level types to allow easily defining a contract's blueprint. 

    **TODO**: specify those types in a meta-schema in terms of data's primitives (constr, map, list, integer, bytes).

    <details>
      <summary>List of provided high-level types</summary>

    - Unit
    - Bool
    - Rational
    - Transaction
    - TransactionId
    - TransactionInput
    - TransactionOutput
    - TransactionOutputReference
    - Address
    - Value
    - PolicyId
    - AssetName
    - AssetId
    - Credential
    - StakeCredential
    - DelegationCertificate
    - ScriptPurpose
    - ScriptTag
    - RedeemerPointer
    - TimeRange
    - DatumHash
    - ScriptHash
    - ValidatorHash
    - Ed25519VerificationKeyHash
    - Ed25519VerificationKey
    - Ed25519SigningKey
    - Ed25519Signature
    - Data
    </details>

    > Few examples:
    > 
    > ```yaml
    > definitions:
    >   Unit:
    >     type: sum
    >     cases: {}
    > 
    >   Bool:
    >     type: sum
    >     cases:
    >       True: []
    >       False: []
    >
    >   Rational: 
    >     type: sum
    >     cases: 
    >       Rational:
    >         - type: integer
    >         - type: integer
    > ```

- [x] 6. Should have keywords to combine smaller types together. We'll borrow `oneOf` from JSON-schema, and add some more as first-class citizens:
    - oneOf 
    - maybe
    - either
    - these 

## Rationale

To be written.

## Backward Compatibility

N/A

## Path to active

- [ ] Few major DApps adopting the schema and specifying their contracts with it.
- [ ] Explorer / Wallets leveraging the format to display enhanced information for transactions carrying Plutus scripts.
- [ ] A command-line tool for validating plutus-blueprint definitions (based on a meta-schema)

## Example


```yaml
plutus-blueprint: 1.0.0

$schema: _ # URI to the meta-schema for plutus-blueprint, once it exists.

info:
  title: "Hydra: Head protocol"
  description: "A specification of the Hydra: Head protocol contract."
  version: 0.5.0
  license: Apache-2.0

# Object containing at least one but possibly many keys. Contracts may be made up 
# multiple on-chain validators which can be named and specified independently.
#
# For example, the Hydra contract evolves around 4 on-chain scripts: 
#    initial, commit, head and split.
validators:
  # Each validator specifies 3 keys:
  #
  #  - compiled-code, which contains information to reconstruct the raw compiled script
  #    and consequently, calculate its hash if needs be. 
  #
  #  - datum, which specifies the script's datum. 
  #
  #  - redeemer, which specifies the script's redeemer.
  initial:
    compiled-code:
      # Arguments taken by the script at compile-time. They are named for templating.
      arguments: 
        foo: 
          type: integer

      # Some base16-encoded representation of the compiled-code, where possible arguments 
      # have been templated out using some basic templating syntax à la mustache 
      # (https://mustache.github.io/mustache.5.html).
      # For example `8238a87e4bb54a{{ foo }}76bba7a6a`
      template: _

    # DatumType = ()
    datum:
      $ref: $schema/definitions/Unit

    # RedeemerType = Abort | Commit { committedRef :: Maybe TxOutRef }
    redeemer:
      type: sum
      cases: 
        Abort: []
        Commit:
          - maybe: 
              $ref: $schema/definitions/TxOutRef

  head: 
    compiled-code:
      template: _ # Would be nice to get support from the Plutus' team for that.
      arguments: 
        commit-script-address: 
          $ref: $schema/definitions/Address
        initial-script-address: 
          $ref: $schema/definitions/Address

    datum:
      type: sum
      cases:
        Initial:
          - $ref: "#/definitions/ContestationPeriod"
          - $ref: "#/definitions/Parties"

        Open:
          - $ref: "#/definitions/Parties"
          - type: bytes

        Closed:
          - $ref: "#/definitions/Parties"
          - type: integer
          - type: bytes

    redeemer:
      type: sum
      cases:
        CollectCom: []
        Close:
          - type: integer
          - type: bytes
          - $ref: "#/definitions/Multisig"
        Contest:
          - type: integer
          - type: bytes
          - $ref: "#/definitions/Multisig"
        Abort: []
        Fanout: 
          - type: integer

    # State-machine-like scripts can specify state transition directly 
    # by referencing constructors for their datum and redeemers.
    transitions:
      - from: null
        to: Initial
        via: null

      - from: Initial
        to: Open
        via: CollectCom

      - from: Initial
        to: null
        via: Abort

      - from: Open
        to: Closed
        via: Close

      - from: Closed
        to: Closed
        via: Contest

      - from: Closed
        to: null
        via: Fanout

# An object to store some local definitions referenced by the schema above.
definitions: 
  ContestationPeriod:
    type: integer
    description: Foo

  Parties:
    type: list
    items: 
      $ref: $schema/definitions/VerificationKeyHash

  Multisig:
    type: list
    items: 
      $ref $schema/definitions/Ed25519Signature
```

## Copyright

CC-BY-4.0
