# SEP Name
STIX Signatures

## SEP Identifier
`x-newcontext-signing-ext`

## SEP Version
1

## SEP Description

Digital Signatures are needed for STIX.  It allows organizations to share data and ensure that the receiving party receives the data they originally sent.  The ability to verify the source of the data allows receiving organizations to build trust in the sending organization, and collaborate to improve the intelligence.

There are two parts to digital signatures, the first part is hashing STIX objects in a canonical JSON format down to a unique deterministic cryptographic hash.  The second is the signing of the hash of the objects and adding the signature to the STIX object.

This signing method may be applied to either an SDO or an SRO.  As the embedded references follow the same definition in both SDOs and SROs, no modifications are needed between the two.  Being able to sign an SRO is necessary for asserting that a course of action mitigates a malware.  You want assurances that the course of action is correct and trusted (signed COA SDO) and that it really does mitigate (signed SRO) the malware (signed malware SDO).

Notes:
* Third party signatures should probably use a new signature SDO object.
* First party signatures are part of the object.  This means if you publish an object w/o a signature and want to add a signature, then you need to update the modified date.  That is you cannot add a signature to a previously published object.  You could use a third party SDO to add the signature.  (I think this make sense, but I'm open for discussion.  I think this makes the most sense, as it prevents the issue where an implementation simply compares the modifed date on the two objects, and rejects the one w/ the signature since it already exists in the database.)

## SEP Use Cases
* Verify that STIX object is authored by party even when received through untrusted channels.
* Build trust in sources, some that may be anonymous.
* Additional guarantees that things like COAs are valid and safe to execute.

## SEP Extension Context
This is an extension to all STIX objects (SDOs and SROs, but not Cyber Oberservable objects).

## SEP Definition
Note: Use of JCS requires that the STIX `integer` data type range be restricted to [ 2\**53, -2\**53 ].  This is because JavaScript, and the ECMAScript standard mandates the use of floating point values for formating numbers, and integer numbers outside this range will be not be redered properly.  If it was allowed, `integer` values could be changed w/o knowledge by the verifier.  For example, 2\**53+1 will print the same number as 2\**53, and so, the two values will result in the same hash.

Signing necessitates that there not be a cycle in the graph of nodes.  If there is a cycle, then it is not possible to generate a valid signature.  The requirement that any referenced object's modfiied date is before the current object's modified date, ensures that a cycle cannot happen.  Even if the set of objects points to the same UUID, it is guaranteed that you will reach an end, as if each modified date is strickly before, there will be a time when we reach BEFORE STIX existed, and therefor no more objects can exist that require a signature.

NOTE: The reason JWS or JOSE is not used, is that it does not define a canonical format for JSON, it only defines how to represent encrypted/signed data IN JSON, but not how sign JSON.

The hash function that **MUST** be used for all hashes is SHA-512 as specified by [RFC6234](https://tools.ietf.org/html/rfc6234).  The signature function **MUST** be Ed448 as specified in [RFC8032](https://tools.ietf.org/html/rfc8032).

### Hashing

The hash **MUST** be calculated on the following specified JSON serialization of the object, even if the original serialization is in a different format.  This is to ensure that hashes are the same no mater what the serialization format is.  The JSON canonical serialization **MUST** follow specification as laid out in [JSON Cleartext Signature (JCS)](https://cyberphone.github.io/doc/security/jcs.html).

When calculating the hash of an object for signing, each property that ends in _ref, the properties in [Ref Properties](#ref-properties) **MUST** be added to the object, both for signing of the object, and **MUST** be included when publishing the object.  These properties are **NOT** added for referenced objects, but **MAY** be present due to the referenced object having been signed.

If there are any `external-reference` data types in the object to be hashed, it is **highly** recommended that a hash value w/ the same algorithm (SHA-512) be present.  If it is not present, then the party verifying the object may be unable to validate the contents of the external reference.

When calculating an object's hash for signing, (that is, the object that will have a signature property added to it), the **signature** property **MUST NOT** be included.  If a **signatureid_ref** property is included, it, and the additional properties for it (defined in [Ref Properties](#ref-properties), **MUST** be included in the hash calculation.

If the hash is being calculated for a referenced object, all the properties of the published object **MUST** be included.  This includes the **signature** property, if present.

Care must be taken when signing properties w/ marking definitions.  An object of a higher level (more restrictive) **SHOULD NOT** be included when signing.  In some cases, even including the UUID is a violation as it acknowledges the existance of the object, and that is a violation of the marking definition.  When removing the reference to the restricted object, a new version or object **MUST** be created per the STIX specification, w/ either the reference removed, or the reference replaced with a reference to an object of the same or lower marking level.  Only objects of the same level, or lower (more permissive) **SHOULD** be signed.  If a more restrictive object needs to be included, a new version of the object that has been declassified, or downgraded, **SHOULD** be used.

### Signatures

The properties added when signing are defined by [Signature Properties](#signature-properties).

When signing an object, the property **signatureid_ref** is added if necessary, and any other properties as desribed in the section [Hashing](#hashing).  The hash is calculated over this object, and then signed using either the **created_by_ref**'s key, or the key on the Identity object referenced by **signatureid_ref**.  The **signature** property **MUST NOT** be present when calculating the hash for signing.

Verification of the signature is done by removing the **signature** property, calculating the hash of the object, and verifying the signature matches the one in the property.

As it is possible to sign an object that references an object that is unsigned, and does not have the various _ref_hash properties, those references w/o the _ref_hash properties cannot be verified, and should be considered untrusted.  It is recommended that only signed objects are referenced from other signed objects as this ensures that all data is verifiable.  An implimentation **MAY** choose to require all referenced objects to be signed (and hence have the required _ref_hash properties for verification), and ignore any referenced objects that are not signed for simplicity.

### Identity Object

The Identity Object will have an additional field to store the public key (see [Identity Properties](#identity-properties).  This public key may be changed just like any other property by creating a new version.  Implementations may choose to store older versions of the public key to allow verification of older signatures.  The key **MUST NOT** be used to sign an object with a modified date before the modified date of the first time the key appeared on the Identity Object. When the key is changed, the previous key **MUST NOT** be used to sign any object that has a modified date that is equal to, or later than the new key's modified date.

![Key and Object signing life time](https://raw.githubusercontent.com/jmgnc/cti-sep-repository/digitalsig/seps/draft/extensions/x-newcontext-signing-ext/STIX%20Identity%20Key%20Timeline.svg?sanitize=true)

NOTE: Though it is supported to modify the Identity object to have a new key, it is current not defined how to bind the keys together to make authenticating new keys easy.  If keys are rotated, and later the older key leaks, there is nothing that prevents an attacker from generating a new Identity object w/ a new key to attempt to take over the Identity.  This may be solved via a blockchain style construction, but is reserved for future work, and to prevent this proposal from being to complicated.  An addition should be able to be built upon this proposal, but not require changes to this general structure.

It is highly recommended that the Identity object is signed with the key itself.  This ensures that the name and other properties of the Identity object are verified by the key holder.  As there is no binding from Identity UUID to public key, the true unique identifier for an Identity object is the public key if present (and the signed Identity Object).  The only thing that prevents someone for attaching a public key to another Identity Object (to attempt to pass off the intelligence as belonging to another Identity) is the signing of the Identity object.  If there is no signature, there is no assertion that the name in the Identity object is the name of the key owner.

It is recommended that all signatures be kept for the Identity object.  Keeping all the signatures allows a web of trust to be formed.  Once entity A has verified entity B's public key, they can then sign entity B's Identity object.  This gives the ability of entity C, which already trusts entity A, import entity B's key w/o needing to do an out of band verification of the key.

## SEP Sponsors
Org | Primary Contact
--- | ---------------
New Context | John-Mark Gurney

## POC Implementations
Org | GitHub Repository
--- | -----------------
New Context | TBD

## Properties

### Ref Properties

| Property Name                           | Type     | Description |
| -------------                           | ----     | ----------- |
| **<basepropertyref>_ref_date (required) | `string` | Contains the modified_date of the reference object.  The value of this field **MUST** be earlier (less-than) than the modified date of the object this property is being added to. |
| **<basepropertyref>_ref_hash (required) | `string` | Contains the base64 encoded hash byte value of the object referenced by **<basepropertyref>_ref**.  The value **MUST** only contain the base64 encoded value.  It **MUST NOT** contain any whitespace characters.

### Signature Properties

| Property Name                     | Type         | Description |
| -------------                     | ----         | ----------- |
| **signature** (required)          | `string`     | A base64 encoded value of the signature of the object. |
| **signatureid_ref** (optional)    | `identifier` | The uuid of the Identity SDO that contains the public key for the signature.  This property **MUST** be present if the key used to sign does not belong to the Identity in the **created_by_ref** property.  This property **MUST NOT** be present if the key used belongs to the Identity in the **created_by_ref** property. |

### Identity Properties

| Property Name            | Type     | Description |
| -------------            | ----     | ----------- |
| **publickey** (optional) | `string` | A base64 encoded public key that is used for signing by this Identity. |
