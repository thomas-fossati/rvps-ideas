# Proposal

This is a proposal for evolving the Reference Value Provider Service (RVPS) to overcome some of the current shortcomings identified in https://github.com/confidential-containers/kbs/issues/238.

## Design assumptions

### Reference Values "shape"

* Reference values can be provided in "manifests" comprising one or more reference value.
* A reference value applies to a given "target environment" (TE) - see [§3.1 of RATS](https://www.rfc-editor.org/rfc/rfc9334.html#section-3.1).
* Each reference value comprises one or more "measurements" (e.g., digests associated with TCB's SW components, snapshots of certain configuration registers, etc.).
* Each TE has an associated name, for example, an instance or class identifier.  Sometimes, a TE can have an _ephemeral_ name, which may require some amount of creativity to work around - for example, Arm CCA Realms.
* Reference values manifests can be given a [media type](https://www.rfc-editor.org/rfc/rfc6838.html).
* Reference value manifests are signed by a reference value provider (RVP).

Pictorially:

```mermaid
flowchart LR
    subgraph M[" "]
        direction TB
        manifest["Reference Value manifest"]
        TE1["TE_1"]
        TEi["..."]
        TEn["TE_n"]

        M11["m_1_1"]
        M1i["..."]
        M1k["m_1_k"]

        Mn1["m_n_1"]
        Mni["..."]
        Mnk["m_n_j"]
    end

    subgraph SM[" "]
        direction BT
        sign["RVP Signature"]
        M
    end

    sign --- M

    manifest --- TE1
    manifest --- TEi
    manifest --- TEn

    TE1 --- M11
    TE1 --- M1i
    TE1 --- M1k

    TEn --- Mn1
    TEn --- Mni
    TEn --- Mnk

    SM -.- mt["application/vnd.XYZ"]
```

### Examples

The following is an instance of the data model for Arm CCA platform and realm reference values carried in a CoRIM manifest:

```mermaid
flowchart LR
    subgraph M[" "]
        direction TB
        manifest["CoRIM ref-val triples"]
        TE1["TE\ncca+realm:02c4d6..."]
        TE2["TE\ncca+realm:555ad2..."]
        TEn["TE\ncca+platform:ef3ec2..."]

        M11["measurements\n------\nrim=ef3ec2...,\nrem[0]=de432e...\nrem[1]=000000...\nrem[2]=000000...\nrem[3]=000000...\nperso=ae432f..."]
        M21["measurements\n------\nrim=555ad2...,\nrem[0]=fe4432...\nrem[1]=892fab...\nrem[2]=000000...\nrem[3]=000000...\nperso=aa21f4..."]

        Mn1["measurements\n------\nconfig=ead330...\nsw-component[0].signer-id=412da2...\nsw-component[0].measurement-value=25621a..."]
        Mnk["measurements\n------\nconfig=ead330...\nsw-component[0].signer-id=412da2...\nsw-component[0].measurement-value=ef33a1..."]
    end

    subgraph SM[" "]
        direction BT
        sign["RVP Signature"]
        M
    end

    sign --- M

    manifest --- TE1
    manifest --- TE2
    manifest --- TEn

    TE1 --- M11
    TE2 --- M21

    TEn --- Mn1
    TEn --- Mnk

    SM -.- mt["application/corim+cbor;\nprofile=tag:arm.com,2023:cca"]
```




### Roles, trust relations, interactions

* An RVP is a supply chain entity.  It could be a TEE vendor, a trusted firmware vendor, a confidential workload developer, or an integrator who takes sole responsibility over a complex supply chain by aggregating and re-signing reference values from trusted, downstream RVPs.
* A trust relationship exists between the RVP and the verifier (owner). Therefore, the identity of the RVP must be known and trusted by the verifier.
* A given RVP is trusted to speak in a trustworthy way about one or more  TEE(s) / confidential workload(s).  The capabilities of an RVP must be known to the verifier.
* Reference values from an unknown RVP MUST be rejected.  (The event MUST be logged.)
* Reference values about a TEE that the RVP is not authorized to "talk about" MUST be rejected.  (This event MUST be logged.)

* The RVPS sits within the same trust perimeter as the verifier, i.e., the verifier implementation trusts the RVPS implementation and vice-versa.  How this trust relationship is established and maintained is out of the scope of the current discussion.


## Architecture

### Ingest (push model)

The following describes the implementation of a "push model" ingestion of reference values.

```mermaid
flowchart TD
    subgraph RVPs
        direction TB
        SP1(SupChain\nA)
        SP2(SupChain\nB)
        SP3(SupChain\nC)
        T1(Integrator\n1)
    end
    subgraph RVP service
        direction TB
        subgraph Handlers
            FMT1[RV X\nhandler]
            FMT2[RV Y\nhandler]
        end
        Store[(Store)]
        KeyMinter
        API[Ingest\nREST API]
    end

    SP1 -.-> T1
    T1 -.-> API
    SP2 -.-> T1
    SP3 -->|"(1)\nPOST /submit\nContent-Type: mt_Y\nBody: rv[1..n]"| API
    API -.-> FMT1
    API -->|"(2)\nmt_Y, rv[1..n]"| FMT2
    FMT2 -->|"(3)\nmt_Y, Id_1, rv[i]"| KeyMinter
    KeyMinter -->|"(4)\nk_i"| FMT2
    FMT2 -->|"(5)\nk_i: {rv[i], mt_Y, Id_1, ...}"| Store
    FMT1 -.-> Store
    FMT2 -->|"(6)\nuuid: {k_i}i=1..n"| Store
    FMT2 -->|"(7)\n{k_i}i=1..n"| API
    API -->|"(8)\n201 Created\nLocation: uuid"| SP3
```

A unique identifier (UUID) is associated with a given (successful) submission, grouping all the reference values that have been ingested.  This UUID is returned to the RVP.

### Store keys

A key `k_i` identifies a given TE.

A key is deterministically synthesised for each reference value.

Note that different reference values can be associated with the same `k_i`, for example if multiple states are acceptable for a given TE (TEE or workload).

To avoid potential identifier collisions, keys are structured so to provide segregation among different attestation schemes.

They are strings with the following structure (the resemblance with [URNs](https://www.rfc-editor.org/rfc/rfc8141.html) is not accidental):
```
"rvps" ':' <attestation-scheme-id> ':' <scheme-specific-id>
```

`:` is a reserved character and MUST not be present in any of the id blocks.

Attestation schemes will need to provide their `attestation-scheme-id` name and `scheme-specific-id` format.

For this to work, the verifier and RVPS need to have a shared understanding of the syntax and semantics of the keys (i.e., the `KeyMinter` boxes in the diagrams must be aligned).

#### Examples

For CCA realm reference values the base-16 representation of the RIM value is used as `scheme-specific-id`:
```
rvps:cca+realm:02c4d6a3e8472211a45e3cc8119c4aa6a4adcfbe5271381151e917cbd4239da1
```

(As [previously](#reference-values-shape) mentioned, this is an _ephemeral_ name.)

For CCA platform reference values, the base-16 representation of the Implementation ID is used as `scheme-specific-id`:
```
rvps:cca+platform:ef3ec22395bfb57301c8363e765c90dd4898896c19acd8684b9ceedcf58fa411
```

### Query (push model)

The following describes the query interface to the RVPS for reference values that have been ingested into the system by the RVP, as [previously](#ingest-push-model) described.

```mermaid
flowchart RL
    subgraph AS
        direction TB
        TSB[TEE-specific\nbackend]
        KeyMinter
    end

    subgraph RVPS
        Store[(Store)]
        API2[Query\nREST API]
    end

    TSB -->|"(1)\nmt_E, E, scheme-specific-id"| KeyMinter
    KeyMinter -->|"(2)\nTE_id"| TSB
    TSB -->|"(3)\nGET /query?key={TE_id}"| API2
    API2 -->|"(4)\nlookup(k)"| Store
    Store -->|"(5)\nrv[i..j]"| API2
    API2 -->|"(6)\nrv[1..j]"| TSB
```

### Query (pull model)

The following describes the query interface to the RVPS for reference values that need to be fetched from a remote RVP service.

To make apparent the remote interaction, the following assumes a cache miss on the Store for the specific key:

```mermaid
flowchart LR
    subgraph AS
        direction TB
        TSB[TEE-specific\nbackend]
        KeyMinter
    end

    subgraph RVPS
        direction RL
        Store[(Store)]
        API2[Query\nREST API]
        SSRVPS[Scheme-specific\nRV fetcher]
    end

    TVRVPS["Vendor-specific\nRV service"]

    TSB -->|"(1)\nmt_E, E, scheme-specific-id"| KeyMinter
    KeyMinter -->|"(2)\nTE_id"| TSB
    TSB -->|"(3)\nGET /query?key=TE_id"| API2
    API2 -->|"(4)\nfetch(k)"| SSRVPS
    SSRVPS -->|"(5)\nGET /f(k)"| TVRVPS
    TVRVPS -->|"(6)\nrv"| SSRVPS
    SSRVPS -->|"(7)\ncache(rv)"| Store
    API2 -->|"(8)\nrv[1..j]"| TSB
```

## Known "Bad" Values

There should be a way to model revocation of previously provisioned reference values or, more generally, to define "deny lists" of explicitly prohibited measurements for a given environment.

The syntax of such a construct would be the same as the one used to define RVs with an additional "reason" (e.g., "obsolete", "insecure", etc.).

## Endorsed Values

It should be possible to attach metadata about an environment that can then be added as "augmented claims" to an attestation result if a certain RV (or also if a known bad value), is matched.  E.g., workload name, version, author, link into a supply-chain transparency log, CVE, etc.




