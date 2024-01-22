# Musings on pattern-matching Evidence

The Verifier's main function is to find patterns in Evidence that match known-good-values or known-bad-values, or some specific "state" that can be associated with metadata related to the Attester (i.e., what CoRIM calls "endorsed values").

To pattern-match Evidence, the Verifier needs:

* A way to identify which Evidence claim needs to be matched
* The comparison logic to be used in matching
* The value(s) to compare against

It makes sense to encapsulate all that into a basic _matcher_ object that can become a building block of higher-level constructs.

Given the variability of Evidence, such _matcher_ needs to be assisted by an "attestation scheme"-specific function that identifies the claim in the Evidence Claims-Set that this _matcher_ is describing.

```cddl
matcher = {
  cmp: $cmp
  values: values
}

$cmp /= "in-set"   ; any
  / "in-range" ; sortable types
  / "masked"   ; bytes
  / "regexp"   ; text

values = [ + any ]

claim-id = text / int
```

## Reference Values

```cddl
RV = { 
  ? environment-id: any
  cond: { + claim-id => matcher }
}
```

## x-Reference Values

```cddl
xRV = {
  ? environment-id: any
  cond: { + claim-id => matcher }
  reason: $reason
}

$reason /= "insecure"
  / "obsolete"
```

## Endorsed Values

```cddl
EV = {
  ? environment-id: any
  cond: { + claim-id => matcher }
  claims: named-claims
}

named-claims = {
  + claim-id => any
}
```

## Manifest

```cddl
manifest = {
  heading: heading
  ? reference-values: [ + RV ]
  ? x-reference-values: [ + xRV ]
  ? endorsed-values: [ + EV ]
}

heading = {
  author: text
  attestation-scheme: text
  ; $extns
}
```

## Examples

### `in-range` matching

```json
{
  "svn": {
  "cmp": "in-range",
    "values": [
      { "min": 0, "max": 10 }
    ]
  }
}
```

### `masked` matching

```json
{
  "raw-value": {
    "cmp": "masked",
    "values": [
      { "bytes": "AAE=", "mask": "AQE=" }
    ]
  }
}
```

### Arm CCA

Complete examples of manifests for Arm CCA platform and realm:

* [cca-platform.json](cca-platform.json)
* [cca-realm.json](cca-realm.json)

---
> **WIP**
---


```python
def match(ClaimsSet, RV, CTX)
    for rv in RV:
        tbcClaim = CTX.profile.claim_lookup(ClaimsSet, rv.cid)
        if not rv.cmp(tbcClaim, rv.vals):
            return false
    return true
```