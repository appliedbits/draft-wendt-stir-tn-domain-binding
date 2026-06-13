---
title: "Binding a Domain Identifier to Telephone Number Authority in STIR Certificates"
abbrev: "STIR TN-Domain Binding"
category: std

docname: draft-wendt-stir-tn-domain-binding-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "Secure Telephone Identity Revisited"
keyword:
 - telephone number
 - domain
 - right-to-use
 - service provider code
 - delegate certificate

author:
 -
    fullname: Chris Wendt
    organization: Somos, Inc.
    email: "chris@appliedbits.com"
    country: US

normative:
  RFC2119:
  RFC8174:
  RFC8224:
  RFC8225:
  RFC8226:
  RFC9447:
  RFC9448:
  I-D.ietf-acme-authority-token-jwtclaimcon:
  I-D.ietf-stir-certificate-transparency:

informative:
  RFC8555:
  RFC9060:
  I-D.ietf-stir-certificates-shortlived:

--- abstract

This document defines a mechanism for binding a domain identifier to telephone number authority within a STIR certificate. A certificate produced under this mechanism carries, as a co-validated pair, the telephone numbers or service provider codes a subject is authorized for in a TNAuthList extension and a domain the subject controls in a SubjectAltName dNSName entry. The binding is established at issuance by requiring proof of domain control and validation of a TNAuthList authority token within a single certificate issuance, such that the resulting certificate attests that the same entity holds both. The mechanism applies to STIR certificates whose TNAuthList contains telephone number entries, service provider code entries, or both, allowing a domain to be bound to the right-to-use holder for a set of numbers or to the provider identified by a service provider code. This document defines the issuance conformance requirements and the relying party verification rule that together make the binding meaningful. It does not define telephone number or service provider code authorization, domain validation, or certificate transparency, all of which are specified elsewhere and referenced here.

--- middle

# Introduction

The STIR architecture ({{RFC8224}}, {{RFC8225}}, {{RFC8226}}) establishes that a signer is authorized for the telephone numbers or service provider codes carried in the TNAuthList certificate extension {{RFC8226}}, validated through the STIR certificate chain. This authority may be held by an entity to which numbers have been assigned, including through delegation {{RFC9060}}, or by a provider identified by a service provider code. However, STIR does not establish a verifiable identity for the entity that holds this authority, nor does it bind that identity to an Internet identifier that a relying party can recognize across contexts.

A domain name is a stable, globally unique Internet identifier for which well-established mechanisms exist to prove control. If a STIR certificate could attest that the entity authorized for a set of telephone numbers or service provider codes is the same entity that controls a particular domain, a relying party would gain an independent, corroborating identity signal bound to that authority. For a certificate carrying telephone numbers, the domain identifies the right-to-use holder for those numbers. For a certificate carrying a service provider code, the domain identifies the provider. In both cases the value of the binding depends entirely on the two facts being established independently and then bound together by the issuer, rather than merely asserted by the subject.

This document defines that binding. It specifies how an issuing Certification Authority co-validates the TNAuthList authority and domain control within a single issuance, what the resulting certificate contains, and how a relying party verifies the binding. The mechanism reuses existing components without modification: TNAuthList authority via authority tokens ({{RFC9447}}, {{RFC9448}}), domain control validation via existing challenge mechanisms such as those defined for ACME {{RFC8555}}, and public logging via certificate transparency {{I-D.ietf-stir-certificate-transparency}}. The contribution of this document is the requirement that these be bound together at issuance and the verification rule that relies on that binding.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses the following terms.

TNAuthList authority: the authorization for an entity to use the telephone numbers or to hold the service provider code carried in a TNAuthList extension {{RFC8226}}, established by the authority responsible for those entries, such as a numbering authority, responsible provider, or service provider code authority.

TNAuthList authority token: a signed assertion of TNAuthList authority for the entries it covers, validated during STIR certificate issuance, as defined in {{RFC9447}} and {{RFC9448}}. For telephone number entries this attests right-to-use of the numbers; for service provider code entries it attests holdership of the code.

Right-to-Use (RTU): the case of TNAuthList authority in which the entries are telephone numbers, representing authorization for an entity to use those numbers.

Domain identifier: a DNS domain name controlled by the certificate subject, carried in a SubjectAltName dNSName entry.

Bound certificate: a STIR certificate issued under this document, carrying a co-validated TNAuthList and domain identifier.

# Overview of the Binding

A bound certificate asserts a single composite fact: the entity that controls the domain identifier in the certificate is the same entity that holds the TNAuthList authority in the certificate. This composite fact is established at issuance and is not derivable from either proof alone.

The binding rests on two independent proofs:

1. Proof of domain control: the subject demonstrates control of the domain identifier, using an existing domain control validation mechanism.
2. Proof of TNAuthList authority: the subject presents a TNAuthList authority token attesting authorization for the telephone numbers or service provider codes, validated by the issuing CA against the authority responsible for that token.

Neither proof is defined by this document. What this document defines is the requirement that both be satisfied within a single issuance, and that the issued certificate carry both results as a bound pair, so that a relying party validating the certificate can rely on the composite fact.

# Binding Process

The binding is produced by a three-phase process. The first two phases each establish one proof using mechanisms that already exist; the third phase binds the two results into one certificate. The phases are described here to give an operational picture; the normative conformance requirements are stated in {{issuance-requirements}}.

Phase 1 establishes control of the domain identifier. Phase 2 establishes TNAuthList authority for the telephone numbers or service provider codes. Phase 3 is a single certificate issuance that requires both proofs and binds their results. The following diagram shows the process at a glance.

~~~
   +-------------+                              +-------------------+
   |  Subject    |                              | TNAuthList        |
   | (domain +   |                              | Authority         |
   |  TN or SPC  |                              | (numbering auth / |
   |  authority) |                              |  provider / SPC   |
   |             |                              |  authority)       |
   +------+------+                              +---------+---------+
          |   Phase 2: request authority evidence        |
          | --------------------------------------------+
          |                                              |
          | <-------- TNAuthList authority token --------+
          |          (proves TN right-to-use or SPC holdership)
          |
          | Phase 1 + Phase 3: single issuance request to CA
          | (CSR + domain challenge + authority token)
          v
   +------+-------------------------------------------------+
   |                  STI Certification Authority           |
   |                                                        |
   |  Phase 1: validate domain control (e.g. dns-01)        |
   |  Phase 2 check: validate TNAuthList authority token    |
   |  Phase 3: bind validated domain + validated TNAuthList |
   |           into one certificate, submit to CT log       |
   +------+-------------------------------------------------+
          |
          | bound certificate (domain in SAN + TN/SPC in
          | TNAuthList + embedded SCT)
          v
   +------+-------------------+
   |  Bound certificate       |
   +--------------------------+
~~~

## Phase 1: Domain control

The subject proves control of the domain identifier that will appear in the SubjectAltName. This document does not define a new mechanism for this; it reuses domain control validation as already deployed for the Web PKI. The challenge types defined for ACME {{RFC8555}} are directly applicable:

- dns-01: the subject provisions a DNS TXT record containing a key-authorization value under a well-known name in the zone of the domain. This proves control of the zone.
- http-01: the subject serves a key-authorization value at a well-known path under the domain. This proves control of content served at the domain.

Either challenge is sufficient on its own. The choice follows existing ACME practice and is operational.

## Phase 2: TNAuthList authority

The subject obtains a TNAuthList authority token from the authority responsible for the entries it covers: a numbering authority or responsible provider for telephone numbers, or a service provider code authority for an SPC. This is the same authority token construct used for ordinary STIR certificate issuance ({{RFC9447}}, {{RFC9448}}). The token is scoped to specific telephone numbers or service provider codes, is verifiable by the CA against the responsible authority's credentials, and is consumed within a single issuance rather than held as a long-lived bearer credential. The authority issuing the token and the subject are frequently different entities; binding the two proofs is what makes that separation verifiable rather than assumed.

## Phase 3: Binding issuance

The subject places a single certificate issuance request to the CA that carries both proofs: a Certificate Signing Request naming the domain identifier in its SubjectAltName, a domain control challenge response, and the TNAuthList authority token. The CA validates domain control and validates the authority token, and only if both succeed issues a certificate whose SubjectAltName carries the validated domain and whose TNAuthList carries the validated telephone numbers or service provider codes. The certificate is submitted to a transparency log and the resulting SCT is embedded before the certificate is used.

The reason both proofs are evaluated within one issuance, rather than assembled from separately obtained credentials, is that the trustworthiness of the binding depends on a single party having confirmed both at the same time. If a subject could obtain a domain credential from one source and a TNAuthList authority credential from another and combine them itself, a relying party would have no assurance that the same entity legitimately held both. Requiring the CA to validate both before issuing makes the certificate a first-party attestation that the binding was verified, not merely asserted.

The following sequence shows the issuance in full, including the transparency step.

~~~
 Subject             STI-CA              TNAuthList Auth   STI-CT Log
    |                  |                        |               |
    |  (Phase 2 done earlier)                   |               |
    |  request authority evidence ----------->  |               |
    |  <----- TNAuthList authority token -----  |               |
    |                  |                        |               |
    |  1. issuance request (CSR w/ domain SAN + token)          |
    | ---------------> |                        |               |
    |  2. domain challenge (dns-01 / http-01)   |               |
    | <--------------- |                        |               |
    |  provision TXT / serve resource           |               |
    | ---------------> |                        |               |
    |                  |  validate domain       |               |
    |                  |  3. validate token --> |               |
    |                  | <--- token valid ----- |               |
    |                  |  4. both OK? yes       |               |
    |                  |  5. submit pre-cert -----------------> |
    |                  | <----------- SCT ----------------------|
    |                  |  embed SCT             |               |
    | <----- bound certificate (domain + TN/SPC + SCT) -------  |
    |                  |                        |               |
~~~

If either proof fails at step 4, the CA does not issue, the CT log is never contacted, and the subject receives an error rather than a certificate.

# Issuance Requirements {#issuance-requirements}

An issuing Certification Authority that issues bound certificates under this document MUST satisfy all of the following within a single certificate issuance.

The CA MUST validate control of the domain identifier that will appear in the SubjectAltName dNSName entry of the certificate. The CA MUST use an established domain control validation mechanism. The challenge-based mechanisms defined for ACME {{RFC8555}} are suitable and RECOMMENDED. This document does not define a new domain control validation mechanism.

The CA MUST validate a TNAuthList authority token covering the telephone numbers or service provider codes that will appear in the TNAuthList extension of the certificate, in accordance with {{RFC9447}} and {{RFC9448}}. The CA MUST NOT include in the TNAuthList any telephone number or service provider code not covered by a validated TNAuthList authority token.

The CA MUST treat the two validations as a single atomic condition for issuance. If either the domain control validation or the TNAuthList authority token validation fails, the CA MUST NOT issue the certificate. The CA MUST NOT issue a certificate that carries a domain identifier without a corresponding validated TNAuthList authority, nor one that carries TNAuthList authority bound to an unvalidated domain identifier.

The CA MUST submit the issued certificate to a transparency log and the certificate MUST carry an embedded Signed Certificate Timestamp as defined in {{I-D.ietf-stir-certificate-transparency}}, so that the binding is publicly auditable prior to use.

The two validations MAY be performed in any order within the issuance, and MAY reuse a recent prior domain control validation for the same domain identifier and the same subject account, provided the reused validation is within the validity window permitted by the CA's policy. Reuse of a prior TNAuthList authority token validation is NOT permitted; a token MUST be validated for each issuance.

# Certificate Profile

A bound certificate MUST conform to the STIR certificate profile in {{RFC8226}}. Where the certificate is a delegate certificate, it MUST also conform to {{RFC9060}}. A bound certificate MUST contain the following.

A SubjectAltName extension containing exactly one dNSName entry carrying the validated domain identifier. The domain identifier MUST be a DNS-resolvable domain name controlled by the subject.

A TNAuthList extension {{RFC8226}} containing one or more entries, which may be telephone number entries, service provider code entries, or both. Every entry MUST be covered by a TNAuthList authority token validated during issuance.

An embedded Signed Certificate Timestamp as defined in {{I-D.ietf-stir-certificate-transparency}}.

A bound certificate MAY additionally carry JWTClaimConstraints or EnhancedJWTClaimConstraints extensions and other elements permitted by {{RFC8226}}; these are out of scope for the binding defined here. Issuers SHOULD issue short-lived certificates as described in {{I-D.ietf-stir-certificates-shortlived}}.

This document associates no new semantics with the Subject distinguished name and does not require organizational identity to be present in the certificate. The domain identifier in the SubjectAltName is the identifier this document binds to the TNAuthList authority.

The domain identifier is carried in the SubjectAltName, not in the Subject Common Name, consistent with current PKI practice of conveying domain identity in the SubjectAltName rather than the Common Name. This binding adds a SubjectAltName dNSName entry and neither relies on nor alters the Subject distinguished name. It is therefore compatible with certificate profiles that specify Subject Common Name content for their own purposes, such as the SHAKEN profile (ATIS-1000074, ATIS-1000080, ATIS-1000084), which uses the Common Name to mark a certificate as a SHAKEN certificate. A certificate may carry such Common Name content and a bound domain identifier in its SubjectAltName without conflict.

## Examples

The following examples illustrate the two principal cases. The certificate fields are shown in the textual style produced by common certificate tooling, omitting fields not relevant to the binding. The TNAuthList extension is identified by the object identifier id-pe-TNAuthList (1.3.6.1.5.5.7.1.26) and its value is the DER encoding of the TNAuthorizationList structure defined in {{RFC8226}}; because most tooling does not decode this extension natively, both the decoded contents and the DER octets are shown. Values are illustrative.

The first example is a certificate issued to a service provider. Its TNAuthList carries a single service provider code, and its SubjectAltName carries the domain that identifies that provider. The Common Name marks it as a SHAKEN certificate, illustrating coexistence with the SHAKEN profile.

~~~
Certificate:
    Data:
        Subject: CN=SHAKEN, O=Example Communications Inc
        X509v3 extensions:
            X509v3 Subject Alternative Name:
                DNS:example-comms.net
            id-pe-TNAuthList (1.3.6.1.5.5.7.1.26):
                TNAuthorizationList:
                    spc [0]: "1234"
            CT Precertificate SCTs:
                Signed Certificate Timestamp: ...
~~~

The TNAuthorizationList DER, as it appears in the extension value, with the EXPLICIT context tag on the spc CHOICE alternative:

~~~
30 08 A0 06 16 04 31 32 33 34
  30 08            SEQUENCE (TNAuthorizationList), 8 octets
     A0 06         [0] EXPLICIT (spc), 6 octets
        16 04      IA5String, 4 octets
           31 32 33 34    "1234"
~~~

In this case a relying party that validates the certificate learns that the entity holding service provider code 1234 is the same entity that controls example-comms.net, because the issuing CA validated both within a single issuance.

The second example is a certificate issued to a business entity that holds the right-to-use for a telephone number. Its TNAuthList carries that telephone number, and its SubjectAltName carries the business's domain. While a TNAuthorizationList may contain more than one entry, a certificate scoped to a single telephone number is RECOMMENDED. Telephone numbers in the TNAuthorizationList are IA5Strings restricted to the digits and symbols permitted by {{RFC8226}}, and do not include a leading "+".

~~~
Certificate:
    Data:
        Subject: O=Example Retail LLC
        X509v3 extensions:
            X509v3 Subject Alternative Name:
                DNS:exampleretail.com
            id-pe-TNAuthList (1.3.6.1.5.5.7.1.26):
                TNAuthorizationList:
                    one [2]: "12015550100"
            CT Precertificate SCTs:
                Signed Certificate Timestamp: ...
~~~

The corresponding TNAuthorizationList DER, with the EXPLICIT context tag on the telephone number CHOICE alternative:

~~~
30 0F A2 0D 16 0B 31 32 30 31 35 35 35 30 31 30 30
  30 0F            SEQUENCE (TNAuthorizationList), 15 octets
     A2 0D         [2] EXPLICIT (one), 13 octets
        16 0B      IA5String, 11 octets
           31 32 30 31 35 35 35 30 31 30 30    "12015550100"
~~~

In this case a relying party learns that the entity that controls exampleretail.com is the same entity that holds the right-to-use for the telephone number 12015550100. A relying party MUST NOT extend that conclusion to any telephone number not present in the TNAuthList.

A certificate MAY carry both telephone number entries and service provider code entries in a single TNAuthorizationList, in which case the domain identifier is bound to all of them, as permitted by {{RFC8226}}.

# Relying Party Verification

A relying party that validates a bound certificate, in addition to the STIR certificate validation procedures defined in {{RFC8224}} and {{RFC8226}}, MUST apply the following.

The relying party MUST validate the certificate chain to a trusted STIR trust anchor and MUST validate the embedded Signed Certificate Timestamp.

The relying party MUST treat the domain identifier in the SubjectAltName dNSName entry as the identity of the entity holding the TNAuthList authority in the certificate only when the certificate is valid under the above checks. The domain identifier carries this meaning by virtue of the issuance binding; a relying party MUST NOT infer TNAuthList authority from the domain identifier independently of the TNAuthList, nor infer domain association for any telephone number or service provider code not present in the TNAuthList.

When a relying party obtains the certificate by retrieving it from a location under the domain identifier, the relying party MAY treat successful retrieval over an authenticated channel for that domain as reinforcing the domain control evidence, but this retrieval is not a substitute for the issuance binding and SCT validation above.

# Security Considerations

The security of the binding depends on the issuing CA performing both validations within a single issuance. The central guarantee is that a relying party need not trust the subject's self-assertion of either domain control or TNAuthList authority; it relies instead on the CA having validated both and bound them in a transparency-logged certificate. An issuer that issued a certificate carrying a domain identifier without a correspondingly validated TNAuthList authority, or vice versa, would defeat the guarantee; the requirements in this document prohibit such issuance.

Domain control validation establishes control at the time of issuance and does not by itself attest to the legal identity of the organization controlling the domain. Where higher assurance of organizational identity is desired, it is established through the certificate policy under which the CA operates and is out of scope for this document.

Certificate transparency makes the binding publicly auditable, enabling monitors to detect a certificate that binds a domain to telephone numbers or service provider codes without authorization. Short-lived certificates limit the window during which a mistaken or compromised binding remains usable.

The reuse of a recent domain control validation permitted in the issuance requirements widens, by the reuse window, the interval between domain control being demonstrated and the binding being issued. CAs SHOULD keep this window short consistent with their policy.

# IANA Considerations

This document has no IANA actions. It relies on certificate extensions and token mechanisms registered by the documents it references.
