**Important:** Read CONTRIBUTING.md before submitting feedback or contributing
```




Network Working Group                                          J. Vcelak
Internet-Draft                                                    CZ.NIC
Intended status: Standards Track                             S. Goldberg
Expires: March 23, 2017                                  D. Papadopoulos
                                                       Boston University
                                                      September 19, 2016


            NSEC5, DNSSEC Authenticated Denial of Existence
                         draft-vcelak-nsec5-03

Abstract

   The Domain Name System Security (DNSSEC) Extensions introduced the
   NSEC resource record (RR) for authenticated denial of existence and
   the NSEC3 for hashed authenticated denial of existence.  The NSEC RR
   allows for the entire zone contents to be enumerated if a server is
   queried for carefully chosen domain names; N queries suffice to
   enumerate a zone containing N names.  The NSEC3 RR adds domain-name
   hashing, which makes the zone enumeration harder, but not impossible.
   This document introduces NSEC5, which provides an cryptographically-
   proven mechanism that prevents zone enumeration.  NSEC5 has the
   additional advantage of not requiring private zone-signing keys to be
   present on all authoritative servers for the zone.

Status of This Memo

   This Internet-Draft is submitted in full conformance with the
   provisions of BCP 78 and BCP 79.

   Internet-Drafts are working documents of the Internet Engineering
   Task Force (IETF).  Note that other groups may also distribute
   working documents as Internet-Drafts.  The list of current Internet-
   Drafts is at http://datatracker.ietf.org/drafts/current/.

   Internet-Drafts are draft documents valid for a maximum of six months
   and may be updated, replaced, or obsoleted by other documents at any
   time.  It is inappropriate to use Internet-Drafts as reference
   material or to cite them other than as "work in progress."

   This Internet-Draft will expire on March 23, 2017.

Copyright Notice

   Copyright (c) 2016 IETF Trust and the persons identified as the
   document authors.  All rights reserved.

   This document is subject to BCP 78 and the IETF Trust's Legal
   Provisions Relating to IETF Documents
   (http://trustee.ietf.org/license-info) in effect on the date of
   publication of this document.  Please review these documents
   carefully, as they describe your rights and restrictions with respect
   to this document.  Code Components extracted from this document must
   include Simplified BSD License text as described in Section 4.e of
   the Trust Legal Provisions and are provided without warranty as
   described in the Simplified BSD License.

Table of Contents

   1.  Introduction
     1.1.  Rationale
     1.2.  Requirements
     1.3.  Terminology
   2.  Backward Compatibility
   3.  How NSEC5 Works
   4.  NSEC5 Algorithms
   5.  The NSEC5KEY Resource Record
     5.1.  NSEC5KEY RDATA Wire Format
     5.2.  NSEC5KEY RDATA Presentation Format
   6.  The NSEC5 Resource Record
     6.1.  NSEC5 RDATA Wire Format
     6.2.  NSEC5 Flags Field
     6.3.  NSEC5 RDATA Presentation Format
   7.  The NSEC5PROOF Resource Record
     7.1.  NSEC5PROOF RDATA Wire Format
     7.2.  NSEC5PROOF RDATA Presentation Format
   8.  NSEC5 Proofs
     8.1.  Name Error Responses
     8.2.  No Data Responses
       8.2.1.  No Data Response, Opt-Out Not In Effect
       8.2.2.  No Data Response, Opt-Out In Effect
     8.3.  Wildcard Responses
     8.4.  Wildcard No Data Responses
   9.  Authoritative Server Considerations
     9.1.  Zone Signing
     9.2.  Zone Serving
     9.3.  NSEC5KEY Rollover Mechanism
     9.4.  Secondary Servers
     9.5.  Zones Using Unknown Hash Algorithms
     9.6.  Dynamic Updates
   10. Resolver Considerations
   11. Validator Considerations
     11.1.  Validating Responses
     11.2.  Validating Referrals to Unsigned Subzones
     11.3.  Responses With Unknown Hash Algorithms
   12. Special Considerations
     12.1.  Transition Mechanism
     12.2.  NSEC5 Private Keys
     12.3.  Domain Name Length Restrictions
   13. Performance Considerations
     13.1.  Performance of Cryptographic Operations
       13.1.1.  NSEC3 Hashing Performance
       13.1.2.  NSEC5 Hashing Performance
     13.2.  Authoritative Server Startup
     13.3.  NSEC5 Answer Generating and Processing
     13.4.  Precomputed NSEC5PROOF Values
     13.5.  Response Lengths
     13.6.  Summary
   14. Security Considerations
     14.1.  Zone Enumeration Attacks
     14.2.  Hash Collisions
     14.3.  Compromise of the Private NSEC5 Key
     14.4.  Key Length Considerations
     14.5.  Transitioning to a New NSEC5 Algorithm
   15. IANA Considerations
   16. Contributors
   17. References
     17.1.  Normative References
     17.2.  Informative References
   Appendix A.  RSA Full Domain Hash Algorithm
     A.1.  FDH signature
     A.2.  FDH verification
   Appendix B.  Elliptic Curve VRF
     B.1.  ECVRF Hash To Curve
     B.2.  ECVRF Auxiliary Functions
       B.2.1.  ECVRF Hash Points
       B.2.2.  ECVRF Proof To Hash
       B.2.3.  ECVRF Decode Proof
     B.3.  ECVRF Signing
     B.4.  ECVRF Verification
   Appendix C.  Change Log
   Appendix D.  Open Issues
   Authors' Addresses

1.  Introduction

1.1.  Rationale

   The DNS Security Extensions (DNSSEC) provides data integrity
   protection using public-key cryptography, while not requiring that
   authoritative servers compute signatures on-the-fly.  The content of
   the zone is usually pre-computed and served as is.  The evident
   advantages of this approach are reduced performance requirements per
   query, as well as not requiring private zone-signing keys to be
   present on nameservers facing the network.

   With DNSSEC, each resource record (RR) set in the zone is signed.
   The signature is retained as an RRSIG RR directly in the zone.  This
   enables response authentication for data existing in the zone.  To
   ensure integrity of denying answers, an NSEC chain of all existing
   domain names in the zone is constructed.  The chain is made of RRs,
   where each RR represents two consecutive domain names in canonical
   order present in the zone.  The NSEC RRs are signed the same way as
   any other RRs in the zone.  Non-existence of a name can be thus
   proven by presenting a NSEC RR which covers the name.

   As side-effect, however, the NSEC chain allows for enumeration of the
   zone's contents by sequentially querying for the names immediately
   following those in the most-recently retrieved NSEC record; N queries
   suffice to enumerate a zone containing N names.  As such, the NSEC3
   hashed denial of existence was introduced to prevent zone
   enumeration.  In NSEC3, the original domain names in the NSEC chain
   are replaced by their cryptographic hashes.  While NSEC3 makes zone
   enumeration more difficult, offline dictionary attacks are still
   possible and have been demonstrated; this is because hashes may be
   computed offline and the space of possible domain names is restricted
   [nsec3walker][nsec3gpu].

   Zone enumeration can be prevented with NSEC3 if having the
   authoritative server compute NSEC3 RRs on-the-fly, in response to
   queries with denying responses.  Usually, this is done with Minimally
   Covering NSEC Records or NSEC3 White Lies [RFC7129].  The
   disadvantage of this approach is a required presence of the private
   key on all authoritative servers for the zone.  This is often
   undesirable, as the holder of the private key can tamper with the
   zone contents, and having private keys on many network-facing servers
   increases the risk that keys can be compromised.

   To prevent zone content enumeration without keeping private keys on
   all authoritative servers, NSEC5 replaces the unkeyed cryptographic
   hash function used in NSEC3 with a public-key hashing scheme.
   Hashing in NSEC5 is performed with a separate NSEC5 key.  The public
   portion of this key is distributed in an NSEC5KEY RR, and is used to
   validate NSEC5 hash values.  The private portion of the NSEC5 key is
   present on all authoritative servers for the zone, and is used to
   compute hash values.

   Importantly, the NSEC5KEY key cannot be used to modify the contents
   of the zone.  Thus, any compromise of the private NSEC5 key does not
   lead to a compromise of zone contents.  All that is lost is privacy
   against zone enumeration, effectively downgrading the security of
   NSEC5 to that of NSEC3.  NSEC5 comes with a cryptographic proof of
   security, available in [nsec5].

   The NSEC5 is not intended to replace NSEC or NSEC3.  It is designed
   as an alternative mechanism for authenticated denial of existence.

1.2.  Requirements

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
   document are to be interpreted as described in [RFC2119].

1.3.  Terminology

   The reader is assumed to be familiar with the basic DNS and DNSSEC
   concepts described in [RFC1034], [RFC1035], [RFC4033], [RFC4034],
   [RFC4035], and subsequent RFCs that update them: [RFC2136],
   [RFC2181], [RFC2308], and [RFC5155].

   The following terminology is used through this document:

   Base32hex:  The "Base 32 Encoding with Extended Hex Alphabet" as
      specified in [RFC4648].  The padding characters ("=") are not used
      in NSEC5 specification.

   Base64:  The "Base 64 Encoding" as specified in [RFC4648].

   NSEC5 proof:  A signed hash of a domain name (hash-and-sign
      paradigm).  A holder of the private key (e.g., authoritative
      server) can compute the proof.  Anyone knowing the public key
      (e.g., client) can verify it's validity.

   NSEC5 hash:  A cryptographic hash (digest) of an NSEC5 proof.  If the
      NSEC5 proof is known, anyone can compute and verify it's NSEC5
      hash.

   NSEC5 algorithm:  A pair of algorithms used to compute NSEC5 proofs
      and NSEC5 hashes.

2.  Backward Compatibility

   The specification describes a protocol change that is not backward
   compatible with [RFC4035] and [RFC5155].  NSEC5-unaware resolver will
   fail to validate responses introduced by this document.

   To prevent NSEC5-unaware resolvers from attempting to validate the
   responses, new DNSSEC algorithms identifiers are introduced, the
   identifiers alias with existing algorithm numbers.  The zones signed
   according to this specification MUST use only these algorithm
   identifiers, thus NSEC5-unaware resolvers will treat the zone as
   insecure.

   The new algorithm identifiers defined by this document are listed in
   Section 15.

3.  How NSEC5 Works

   To prove non-existence of a domain name in a zone, NSEC uses a chain
   built from domain names present in the zone.  NSEC3 replaces the
   original domain names by their cryptographic hashes.  NSEC5 is very
   similar to NSEC3, except that the cryptographic hash is replaced by
   hashes computed using a verifiable random function (VRF).  A VRF is
   essentially the public-key version of a keyed cryptographic hash.  A
   VRF comes with a public/private key pair, and only the holder of the
   private key can compute the hash, but anyone with public key can
   verify the hash.

   In NSEC5, the original domain name is hashed twice:

   1.  First, the domain name is hashed using a VRF keyed with the NSEC5
       private key; the result is called the NSEC5 proof.  Only an
       authoritative server that knows the private NSEC5 key can compute
       the NSEC5 proof.  Any client that knows the public NSEC5 key can
       validate the NSEC5 proof.

   2.  Second, the NSEC5 proof is hashed.  The result is called the
       NSEC5 hash value.  This hash can be computed by any party that
       knows the input NSEC5 proof.

   The NSEC5 hash determines the position of a domain name in an NSEC5
   chain.  That is, all the NSEC5 hashes for a zone are sorted in their
   canonical order, and each consecutive pair forms an NSEC5 RR.

   To prove an non-existence of a particular domain name in response to
   a query, the server computes the NSEC5 proof (using the private NSEC5
   key) on the fly.  Then it uses the NSEC5 proof to compute the
   corresponding NSEC5 hash.  It then identifies the NSEC5 RR that
   covers the NSEC5 hash.  In the response message, the server returns
   the NSEC5 RR, it's corresponding signature (RRSIG RRset), and
   synthesized NSEC5PROOF RR containing the NSEC5 proof it computed on
   the fly.

   To validate the response, the client first uses the public NSEC5 key
   (stored in the zone as an NSEC5KEY RR) to verify that the NSEC5 proof
   corresponds with the domain name to be disproved.  Then, the client
   computes the NSEC5 hash from the NSEC5 proof and checks that it is
   covered by the NSEC5 RR.  Finally, it checks that the signature on
   the NSEC5 RR is valid.

4.  NSEC5 Algorithms

   The algorithms used for NSEC5 authenticated denial are independent of
   the algorithms used for DNSSEC signing.  An NSEC5 algorithm defines
   how the NSEC5 proof and the NSEC5 hash is computed and validated.

   The input for the NSEC5 proof computation is an RR owner name in the
   canonical form in the wire format and an NSEC5 private key; the
   output is an octet string.

   The input for the NSEC5 hash computation is the corresponding NSEC5
   proof; the output is an octet string.

   This document defines RSAFDH-SHA256-SHA256 NSEC5 algorithm as
   follows:

   o  NSEC5 proof is computed using an RSA based Full Domain Hash (FDH)
      signature with SHA-256 hash function used internally for input
      preprocessing.  The signature and verification is formally
      specified in Appendix A.

   o  NSEC5 hash is computed by hashing the NSEC5 proof with the SHA-256
      hash function as specified in [RFC6234].

   o  The public key format to be used in NSEC5KEY RR is defined in
      Section 2 of [RFC3110] and thus is the same as the format used to
      store RSA public keys in DNSKEY RRs.

   This document defines EC-P256-SHA256 NSEC5 algorithm as follows:

   o  NSEC5 proof is computed using an Elliptic Curve VRF with FIPS
      186-3 P-256 curve.  The proof computation and verification is
      formally specified in Appendix B.  The curve parameters are
      specified in [FIPS-186-3] (Section D.1.2.3) and [RFC5114]
      (Section 2.6).

   o  NSEC5 hash is x-coordinate of the group element gamma from the
      NSEC5 proof (specified in Appendix B), encoded as a fixed-width
      32-octet unsigned integer in network byte order.  In practice, the
      hash is a substring of the proof ranging from 2nd to 33th octet of
      the proof inclusive.

   o  The public key format to be used in NSEC5KEY RR is defined in
      Section 4 of [RFC6605] and thus is the same as the format used to
      store ECDSA public keys in DNSKEY RRs.

   This document defines EC-ED25519-SHA256 NSEC5 as follows:

   o  NSEC5 proof is the same as with EC-P256-SHA256 but using Ed25519
      elliptic curve with parameters defined in [RFC7748] (Section 4.1).

   o  NSEC5 hash is the same as with EC-P256-SHA256.

   o  The public key format to be used in NSEC5KEY RR is defined in
      Section 3 of [I-D.ietf-curdle-dnskey-ed25519] and thus is the same
      as the format used to store Ed25519 public keys in DNSKEY RRs.

5.  The NSEC5KEY Resource Record

   The NSEC5KEY RR stores an NSEC5 public key.  The key allows clients
   to verify a validity of NSEC5 proof sent by a server.

5.1.  NSEC5KEY RDATA Wire Format

   The RDATA for NSEC5KEY RR is as shown below:

                        1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2 2 3 3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |   Algorithm   |                  Public Key                   /
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

   Algorithm is a single octet identifying NSEC5 algorithm.

   Public Key is a variable sized field holding public key material for
   NSEC5 proof verification.

5.2.  NSEC5KEY RDATA Presentation Format

   The presentation format of the NSEC5KEY RDATA is as follows:

   The Algorithm field is represented as an unsigned decimal integer.

   The Public Key field is represented in Base64 encoding.  Whitespace
   is allowed within the Base64 text.

6.  The NSEC5 Resource Record

   The NSEC5 RR provides authenticated denial of existence for an RRset.
   One NSEC5 RR represents one piece of an NSEC5 chain, proving
   existence of RR types present at the original domain name and also
   non-existence of other domain names in a part of the hashed domain
   name space.

6.1.  NSEC5 RDATA Wire Format

   The RDATA for NSEC5 RR is as shown below:

                        1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2 2 3 3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |            Key Tag            |     Flags     |  Next Length  |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                     Next Hashed Owner Name                    /
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   /                         Type Bit Maps                         /
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

   Key Tag field contains the key tag value of the NSEC5KEY RR that
   validates the NSEC5 RR, in network byte order.  The value is computed
   from the NSEC5KEY RDATA using the same algorithm, which is used to
   compute key tag values for DNSKEY RRs.  The algorithm is defined in
   [RFC4034].

   Flags field is a single octet.  The meaning of individual bits of the
   field is defined in Section 6.2.

   Next length is an unsigned single octet specifying the length of the
   Next Hashed Owner Name field in octets.

   Next Hashed Owner Name field is a sequence of binary octets.  It
   contains an NSEC5 hash of the next domain name in the NSEC5 chain.

   Type Bit Maps is a variable sized field encoding RR types present at
   the original owner name matching the NSEC5 RR.  The format of the
   field is equivalent to the format used in NSEC3 RR, described in
   [RFC5155].

6.2.  NSEC5 Flags Field

   The following one-bit NSEC5 flags are defined:

    0 1 2 3 4 5 6 7
   +-+-+-+-+-+-+-+-+
   |           |W|O|
   +-+-+-+-+-+-+-+-+

     O - Opt-Out flag

     W - Wildcard flag

   All the other flags are reserved for future use and MUST be zero.

   The Opt-Out flag has the same semantics as in NSEC3.  The definition
   and considerations in [RFC5155] are valid, except that NSEC3 is
   replaced by NSEC5.

   The Wildcard flag indicates that a wildcard synthesis is possible at
   the original domain name level (i.e., there is a wildcard node
   immediately descending from the immediate ancestor of the original
   domain name).  The purpose of the Wildcard flag is to reduce a
   maximum number of RRs required for authenticated denial of existence
   proof.

6.3.  NSEC5 RDATA Presentation Format

   The presentation format of the NSEC5 RDATA is as follows:

   The Key Tag field is represented as an unsigned decimal integer.

   The Flags field is represented as an unsigned decimal integer.

   The Next Length field is not represented.

   The Next Hashed Owner Name field is represented as a sequence of
   case-insensitive Base32hex digits without any whitespace and without
   padding.

   The Type Bit Maps representation is equivalent to the representation
   used in NSEC3 RR, described in [RFC5155].

7.  The NSEC5PROOF Resource Record

   The NSEC5PROOF record is synthesized by the authoritative server on-
   the-fly.  The record contains the NSEC5 proof, proving a position of
   the owner name in an NSEC5 chain.

7.1.  NSEC5PROOF RDATA Wire Format

   The RDATA for NSEC5PROOF is as as shown below:

                        1 1 1 1 1 1 1 1 1 1 2 2 2 2 2 2 2 2 2 2 3 3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |            Key Tag            |        Owner Name Hash        /
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

   Key Tag field contains the key tag value of the NSEC5KEY RR that
   validates the NSEC5PROOF RR, in network byte order.

   Owner Name Hash is a variable sized sequence of binary octets
   encoding the NSEC5 proof of the owner name of the RR.

7.2.  NSEC5PROOF RDATA Presentation Format

   The presentation format of the NSEC5PROOF RDATA is as follows:

   The Key Tag field is represented as an unsigned decimal integer.

   The Owner Name Hash is represented in Base64 encoding.  Whitespace is
   allowed within the Base64 text.

8.  NSEC5 Proofs

   This section summarizes all possible types of authenticated denial of
   existence.  For each type the following lists are included:

   1.  Facts to prove.  The minimum amount of information an
       authoritative server must provide to a client to assure the
       client that the response content is valid.

   2.  Authoritative server proofs.  NSEC5 RRs an authoritative server
       must include in a response to prove the listed facts.

   3.  Validator checks.  Individual checks a validating server is
       required to perform on a response.  The response content is
       considered valid only if all the checks pass.

   If NSEC5 is said to match a domain name, the owner name of the NSEC5
   RR has to be equivalent to an NSEC5 hash of that domain name.  If an
   NSEC5 RR is said to cover a domain name, the NSEC5 hash of the domain
   name must lay strictly between that NSEC5 RR's Owner Name and Next
   Hashed Owner Name.

8.1.  Name Error Responses

   Facts to prove:

   No RRset matching the QNAME exactly exists.

   No RRset matching the QNAME via wildcard expansion exists.

   The QNAME does not fall into a delegation.

   The QNAME does not fall into a DNAME redirection.

   Authoritative server proofs:

   Closest encloser.

   Next closer name.

   Validator checks:

   Closest encloser belongs to the zone.

   Closest encloser has the Wildcard flag cleared.

   Closest encloser does not have NS without SOA in the Type Bit Map.

   Closest encloser does not have DNAME in the Type Bit Maps.

   Next closer name is derived correctly.

8.2.  No Data Responses

   The processing of a No Data response for DS QTYPE differs if the Opt-
   Out is in effect.  For DS QTYPE queries, the validator has two
   possible checking paths.  The correct path can be simply decided by
   inspecting if the NSEC5 RR in the response matches the QNAME.

   Note that the Opt-Out is valid only for DS QTYPE queries.

8.2.1.  No Data Response, Opt-Out Not In Effect

   Facts to prove:

   An RRset matching the QNAME exists.

   No QTYPE RRset matching the QNAME exists.

   No CNAME RRset matching the QNAME exists.

   Authoritative server proofs:

   QNAME.

   Validator checks:

   The NSEC5 RR exactly matches the QNAME.

   The NSEC5 RR does not have QTYPE in the Type Bit Map.

   The NSEC5 RR does not have CNAME in the Type Bit Map.

8.2.2.  No Data Response, Opt-Out In Effect

   Facts to prove:

   The delegation is not covered by the NSEC5 chain.

   Authoritative server proofs:

   Closest provable encloser.

   Validator checks:

   Closest provable encloser is in zone.

   Closest provable encloser covers (not matches) the QNAME.

   Closest provable encloser has the Opt-Out flag set.

8.3.  Wildcard Responses

   Facts to prove:

   No RRset matching the QNAME exactly exists.

   No wildcard closer to the QNAME exists.

   Authoritative server proofs:

   Next closer name.

   Validator checks:

   Next closer name is derived correctly.

   Next closer name covers (not matches).

8.4.  Wildcard No Data Responses

   Facts to prove:

   No RRset matching the QNAME exactly exists.

   No QTYPE RRset exists at the wildcard matching the QNAME.

   No CNAME RRset exists at the wildcard matching the QNAME.

   No wildcard closer to the QNAME exists.

   Authoritative server proofs:

   Source of synthesis (i.e., wildcard at closest encloser).

   Next closer name.

   Validator checks:

   Source of synthesis matches exactly the QNAME.

   Source of synthesis does not have QTYPE in the Type Bit Map.

   Source of synthesis does not have CNAME in the Type Bit Map.

   Next closer name is derived correctly.

   Next closer name covers (not matches).

9.  Authoritative Server Considerations

9.1.  Zone Signing

   Zones using NSEC5 MUST satisfy the same properties as described in
   Section 7.1 of [RFC5155], with NSEC3 replaced by NSEC5.  In addition,
   the following conditions MUST be satisfied as well:

   o  If the original owner name has a wildcard label immediately
      descending from the original owner name, the corresponding NSEC5
      RR MUST have the Wildcard flag set in the Flags field.  Otherwise,
      the flag MUST be cleared.

   o  The zone apex MUST include an NSEC5KEY RRset containing a NSEC5
      public key allowing verification of the current NSEC5 chain.

   The following steps describe one possible method to properly add
   required NSEC5 related records into a zone.  This is not the only
   such existing method.

   1.  Select an algorithm for NSEC5.  Generate the public and private
       NSEC5 keys.

   2.  Add a NSEC5KEY RR into the zone apex containing the public NSEC5
       key.

   3.  For each unique original domain name in the zone and each empty
       non-terminal, add an NSEC5 RR.  If Opt-Out is used, owner names
       of unsigned delegations MAY be excluded.

       1.  The owner name of the NSEC5 RR is the NSEC5 hash of the
           original owner name encoded in Base32hex without padding,
           prepended as a single label to the zone name.

       2.  Set the Key Tag field to be the key tag corresponding to the
           public NSEC5 key.

       3.  Clear the Flags field.  If Opt-Out is being used, set the
           Opt-Out flag.  If there is a wildcard label directly
           descending from the original domain name, set the Wildcard
           flag.  Note that the wildcard can be an empty non-terminal
           (i.e., the wildcard synthesis does not take effect and
           therefore the flag is not to be set).

       4.  Set the Next Length field to a value determined by the used
           NSEC5 algorithm.  Leave the Next Hashed Owner Name field
           blank.

       5.  Set the Type Bit Maps field based on the RRsets present at
           the original owner name.

   4.  Sort the set of NSEC5 RRs into canonical order.

   5.  For each NSEC5 RR, set the Next Hashed Owner Name field by using
       the owner name of the next NSEC5 RR in the canonical order.  If
       the updated NSEC5 is the last NSEC5 RR in the chain, the owner
       name of the first NSEC5 RR in the chain is used instead.

   The NSEC5KEY and NSEC5 RRs MUST have the same class as the zone SOA
   RR.  Also the NSEC5 RRs SHOULD have the same TTL value as the SOA
   minimum TTL field.

   Notice that a use of Opt-Out is not indicated in the zone.  This does
   not affect the ability of a server to prove insecure delegations.
   The Opt-Out MAY be part of the zone-signing tool configuration.

9.2.  Zone Serving

   This specification modifies DNSSEC-enabled DNS responses generated by
   authoritative servers.  In particular, it replaces use of NSEC or
   NSEC3 RRs in such responses with NSEC5 RRs and adds on-the-fly
   computed NSEC5PROOF RRs.

   The authenticated denial of existence proofs in NSEC5 are almost the
   same as in NSEC3.  However, due to introduction of Wildcard flag in
   NSEC5 RRs, the NSEC5 proof consists from (up to) two NSEC5 RRs,
   instead of (up to) three.

   According to a type of a response, an authoritative server MUST
   include NSEC5 RRs in a response as defined in Section 8.  For each
   NSEC5 RR in the response a matching RRSIG RRset and a synthesized
   NSEC5PROOF MUST be added as well.

   A synthesized NSEC5PROOF RR has the owner name set to a domain name
   exactly matching the name required for the proof.  The class and TTL
   of the RR MUST be the same as the class and TTL value of the
   corresponding NSEC5 RR.  The RDATA are set according to the
   description in Section 7.1.

   Notice, that the NSEC5PROOF owner name can be a wildcard (e.g.,
   source of synthesis proof in wildcard No Data responses).  The name
   also always matches the domain name required for the proof while the
   NSEC5 RR may only cover (not match) the name in the proof (e.g.,
   closest encloser in Name Error responses).

   If NSEC5 is used, an answering server MUST use exactly one NSEC5
   chain for one signed zone.

   NSEC5 MUST NOT be used in parallel with NSEC, NSEC3, or any other
   authenticated denial of existence mechanism that allows for
   enumeration of zone contents.

   Similarly to NSEC3, the owner names of NSEC5 RRs are not represented
   in the NSEC5 chain and therefore NSEC5 records deny their own
   existence.  The desired behavior caused by this paradox is the same
   as described in Section 7.2.8 of [RFC5155].

9.3.  NSEC5KEY Rollover Mechanism

   Replacement of the NSEC5 key implies generating a new NSEC5 chain.
   The NSEC5KEY rollover mechanism is similar to "Pre-Publish Zone
   Signing Key Rollover" as specified in [RFC6781].  The NSEC5KEY
   rollover MUST be performed as a sequence of the following steps:

   1.  A new public NSEC5 key is added into the NSEC5KEY RRset in the
       zone apex.

   2.  The old NSEC5 chain is replaced by a new NSEC5 chain constructed
       using the new key.  This replacement MUST happen as a single
       atomic operation; the server MUST NOT be responding with RRs from
       both the new and old chain at the same time.

   3.  The old public key is removed from the NSEC5KEY RRset in the zone
       apex.

   The minimal delay between the steps 1. and 2.  MUST be the time it
   takes for the data to propagate to the authoritative servers, plus
   the TTL value of the old NSEC5KEY RRset.

   The minimal delay between the steps 2. and 3.  MUST be the time it
   takes for the data to propagate to the authoritative servers, plus
   the maximum zone TTL value of any of the data in the previous version
   of the zone.

9.4.  Secondary Servers

   This document does not define mechanism to distribute NSEC5 private
   keys.  See Section 14.3 for discussion on the security requirements
   for NSEC5 private keys.

9.5.  Zones Using Unknown Hash Algorithms

   Zones that are signed with unknown NSEC5 algorithm or by an
   unavailable NSEC5 private key cannot be effectively served.  Such
   zones SHOULD be rejected when loading and servers SHOULD respond with
   RCODE=2 (Server failure) when handling queries that would fall under
   such zones.

9.6.  Dynamic Updates

   A zone signed using NSEC5 MAY accept dynamic updates.  The changes to
   the zone MUST be performed in a way, that the zone satisfies the
   properties specified in Section 9.1 at any time.

   It is RECOMMENDED that the server rejects all updates containing
   changes to the NSEC5 chain (or related RRSIG RRs) and performs itself
   any required alternations of the NSEC5 chain induced by the update.

   Alternatively, the server MUST verify that all the properties are
   satisfied prior to performing the update atomically.

10.  Resolver Considerations

   The same considerations as described in Section 9 of [RFC5155] for
   NSEC3 apply to NSEC5.  In addition, as NSEC5 RRs can be validated
   only with appropriate NSEC5PROOF RRs, the NSEC5PROOF RRs MUST be all
   together cached and included in responses with NSEC5 RRs.

11.  Validator Considerations

11.1.  Validating Responses

   The validator MUST ignore NSEC5 RRs with Flags field values other
   than the ones defined in Section 6.2.

   The validator MAY treat responses as bogus if the response contains
   NSEC5 RRs that refer to a different NSEC5KEY.

   According to a type of a response, the validator MUST verify all
   conditions defined in Section 8.  Prior to making decision based on
   the content of NSEC5 RRs in a response, the NSEC5 RRs MUST be
   validated.

   To validate a denial of existence, zone NSEC5 public keys are
   required in addition to DNSSEC public keys.  Similarly to DNSKEY RRs,
   the NSEC5KEY RRs are present in the zone apex.

   The NSEC5 RR is validated as follows:

   1.  Select a correct NSEC5 public key to validate the NSEC5PROOF.
       The Key Tag value of the NSEC5PROOF RR must match with the key
       tag value computed from the NSEC5KEY RDATA.

   2.  Validate the NSEC5 proof present in the NSEC5PROOF Owner Name
       Hash field using the NSEC5 public key.  If there are multiple
       NSEC5KEY RRs matching the key tag, at least one of the keys must
       validate the NSEC5 proof.

   3.  Compute the NSEC5 hash value from the NSEC5 proof and check if
       the response contains NSEC5 RR matching or covering the computed
       NSEC5 hash.  The TTL values of the NSEC5 and NSEC5PROOF RRs must
       be the same.

   4.  Validate the signature of the NSEC5 RR.

   If the NSEC5 RR fails to validate, it MUST be ignored.  If some of
   the conditions required for an NSEC5 proof is not satisfied, the
   response MUST be treated as bogus.

   Notice that determining closest encloser and next closer name in
   NSEC5 is easier than in NSEC3.  NSEC5 and NSEC5PROOF RRs are always
   present in pairs in responses and the original owner name of the
   NSEC5 RR matches the owner name of the NSEC5PROOF RR.

11.2.  Validating Referrals to Unsigned Subzones

   The same considerations as defined in Section 8.9 of [RFC5155] for
   NSEC3 apply to NSEC5.

11.3.  Responses With Unknown Hash Algorithms

   A validator MUST ignore NSEC5KEY RRs with unknown NSEC5 algorithms.
   The practical result of this is that zones sighed with unknown
   algorithms will be considered bogus.

12.  Special Considerations

12.1.  Transition Mechanism

   TODO: Not finished.  Following information will be covered:

   o Transition from NSEC or NSEC3.

   o Transition from NSEC5 to NSEC/NSEC3

   o Transition to new algorithms within NSEC5

   Quick notes on transition from NSEC/NSEC3 to NSEC5:

   1.  Publish NSEC5KEY RR.

   2.  Wait for data propagation to slaves and cache expiration.

   3.  Instantly switch answering from NSEC/NSEC3 to NSEC5.

   Quick notes on transition from NSEC5 to NSEC/NSEC3:

   1.  Instantly switch answering from NSEC5 to NSEC/NSEC3.

   2.  Wait for NSEC5 RRs expiration in caches.

   3.  Remove NSEC5KEY RR from the zone.

12.2.  NSEC5 Private Keys

   This document does not define format to store NSEC5 private key.  Use
   of standardized and adopted format is RECOMMENDED.

   The NSEC5 private key MAY be shared between multiple zones, however a
   separate key is RECOMMENDED for each zone.

12.3.  Domain Name Length Restrictions

   The NSEC5 creates additional restrictions on domain name lengths.  In
   particular, zones with names that, when converted into hashed owner
   names exceed the 255 octet length limit imposed by [RFC1035], cannot
   use this specification.

   The actual maximum length of a domain name depends on the length of
   the zone name and used NSEC5 algorithm.

   All NSEC5 algorithms defined in this document use 256-bit NSEC5 hash
   values.  Such a value can be encoded in 52 characters in Base32hex
   without padding.  When constructing the NSEC5 RR owner name, the
   encoded hash is prepended to the name of the zone as a single label
   which includes the length field of a single octet.  The maximal
   length of the zone name in wire format is therefore 202 octets (255 -
   53).

13.  Performance Considerations

   This section compares NSEC, NSEC3, and NSEC5 with regards to the size
   of denial-of-existence responses and performance impact on the DNS
   components.

13.1.  Performance of Cryptographic Operations

   Additional performance costs depend on the costs of cryptographic
   operations to a great extent.  The following results were retrieved
   with OpenSSL 1.0.2g, running an ordinary laptop on a single-core of a
   CPU manufactured in 2016.  The parameters of cryptographic operations
   were chosen to reflect the parameters used in the real-world
   application of DNSSEC.

13.1.1.  NSEC3 Hashing Performance

   NSEC3 uses multiple iterations of the SHA-1 function with an
   arbitrary salt.  The input of the first iteration is the domain name
   in wire format together with binary salt; the input of the subsequent
   iterations is the binary digest and the salt.  We can assume that the
   input size will be smaller than 32 octets for most executions.

   The measured implementation gives a stable performance for small
   input blocks up to 32 octets.  About 4e6 hashes per second can be
   computed given these parameters.

   The number of additional iterations in NSEC3 parameters will probably
   vary between 0 and 20 in reality.  Therefore we can expect the NSEC3
   hash computation performance to be between 2e5 and 4e6 hashes per
   second.

13.1.2.  NSEC5 Hashing Performance

   The NSEC5 hash is computed in two steps: NSEC5 proof computation
   followed by hashing of the result.  As the proof computation involves
   relatively expensive RSA/EC cryptographic operations, the final
   hashing will have insignificant impact on the overall performance.
   We can also expect difference between NSEC5 hashing (signing) and
   verification time.

   The measurement results for various NSEC5 algorithms and key sizes
   are summarized in the following table:

   +----------------------+--------+-----------+----------+------------+
   | NSEC5 algorithm      |    Key |     Proof |    Perf. |      Perf. |
   |                      |   size |      size | (hash/s) | (verify/s) |
   |                      | (bits) |  (octets) |          |            |
   +----------------------+--------+-----------+----------+------------+
   | RSAFDH-SHA256-SHA256 |   1024 |       128 |     9500 |     120000 |
   | RSAFDH-SHA256-SHA256 |   2048 |       256 |     1500 |      46000 |
   | RSAFDH-SHA256-SHA256 |   4096 |       512 |      200 |      14000 |
   | EC-P256-SHA256       |    256 |        81 |     4700 |       4000 |
   +----------------------+--------+-----------+----------+------------+

   Picking a moderate key size of 2048-bits for RSAFDH-SHA256-SHA256,
   the NSEC5 hash computation performance will be in the order of 10^3
   issued hashes per second and 10^4 validated hashes per second.

   EC-P256-SHA256 trades off verification performance for shorter proof
   size and faster query processing at the nameserver.  In that case,
   both hash computation and verification performance will be in the
   order of 10^3 hashes per second.

   For completeness, the following table sumarrizes the signing and
   verification performance for different DNSSEC signing algorithms:

   +------------------+--------+-----------+-------------+-------------+
   | Algorithm        |    Key | Signature | Performance | Performance |
   |                  |   size |      size |    (sign/s) |  (verify/s) |
   |                  | (bits) |  (octets) |             |             |
   +------------------+--------+-----------+-------------+-------------+
   | RSASHA256        |   1024 |       128 |        9000 |      140000 |
   | RSASHA256        |   2048 |       256 |        1500 |       47000 |
   | RSASHA256        |   4096 |       512 |         200 |       14000 |
   | ECDSAP256SHA256  |    256 |        64 |        7400 |        4000 |
   | ECDSAP384SHA384  |    384 |        96 |        5000 |        1000 |
   | ECDSAP256SHA256* |    256 |        64 |       24000 |       11000 |
   +------------------+--------+-----------+-------------+-------------+

   * highly optimized implementation

   The retrieved values are important primarily for the purpose of
   evaluating performance of response validation.  The signing
   performance is usually not that important because the zone is signed
   offline.  However, when online signing is used, signing performace is
   also important.

13.2.  Authoritative Server Startup

   This section compares additional server startup cost based on the
   used authenticated denial mechanism.

   NSEC  There are no special requirements on processing of a NSEC-
      signed zone during an authoritative server startup.  The server
      handles the NSEC RRs the same way as any other records in the
      zone.

   NSEC3  The authoritative server can precompute NSEC3 hashes for all
      domain names in the NSEC3-signed zone on startup.  With respect to
      query answering, this can speed up inclusion of NSEC3 RRs for
      existing domain names (i.e., Closest provable encloser and QNAME
      for No Data response).

   NSEC5  Very similar considerations apply for NSEC3 and NSEC5.  There
      is a strong motivation to store the NSEC5PROOF values for existing
      domain names in order to reduce query processing time.  A possible
      way to do this, without inceasing the zone size, is to store
      NSEC5PROOF values in a persistent storage structure, as explained
      in Section 13.4.

   The impact of NSEC3 and NSEC5 on the authoritative server startup
   performance is greatly implementation specific.  The server software
   vendor has to seek balance between answering performance, startup
   slowdown, and additional storage cost.

13.3.  NSEC5 Answer Generating and Processing

   An authenticated denial proof is required for No Data, Name Error,
   Wildcard Match, and Wildcard No Data answer.  The number of required
   records depends on used authenticated denial mechanism.  Their size,
   generation cost, and validation cost depend on various zone and
   signing parameters.

   In the worst case, the following additional records authenticating
   the denial will be included into the response:

   o  Up to two NSEC records and their associated RRSIG records.

   o  Up to three NSEC3 records and their associated RRSIG records.

   o  Up to two NSEC5 records and their associated NSEC5PROOF and RRSIG
      records.

   The following list summarizes additional cryptographic operations
   performed by the authoritative server for authenticated denial worst-
   case scenario:

   o  NSEC:

      *  No cryptographic operations required.

   o  NSEC3:

      *  NSEC3 hash for Closest provable encloser

      *  NSEC3 hash for Next closer name

      *  NSEC3 hash for wildcard at the closest encloser

   o  NSEC5:

      *  NSEC5 proof and hash for Closest provable encloser (possibly
         precomputed)

      *  NSEC5 proof and hash for Next closer name

13.4.  Precomputed NSEC5PROOF Values

   As we dicussed in the previous section, the worst-case authenticated
   denial scenario with NSEC5 entails the computation of two NSEC5 proof
   and hash values, one for the Closest provable encloser and one for
   the Next closer name.  For the latter, these values must be computed
   from scratch at query time.  However, the proof value for the former
   had been computed during startup, without being stored, as part of
   the NSEC5 hash computation.

   The query processing time can be drastically reduced if the NSEC5
   proof values for all existing names in the zone are stored by the
   authoritative.  In that case, the authoritative identifies the
   Closest provable encloser name for the given query and looks up both
   the NSEC5 proof and hash value.  This limits the necessary
   computation during query processing to just one NSEC5 proof and hash
   value (that of the Next closer name).  For both variants of NSEC5,
   the proof computation time strongly dominates the final NSEC5 hash
   computation.  Therefore, by storing NSEC5 proof values query
   processing time is almost halved.

   On the other hand, this slightly increases the used storage space at
   the authoritative.  It should be noted that these values should not
   be part of the zone explicitly.  They can be stored at an additional
   data structure.

13.5.  Response Lengths

   [nsec5ecc] precisely measured response lengths for NSEC, NSEC3 and
   NSEC5 using empirical data from a sample second-level domain
   containing about 1000 names.  The sample zone was signed several
   times with different DNSSEC signing algorithms and different
   authenticated denial of existence mechanisms.

   For DNSKEY algorithms, RSASHA256 (2048-bit) and ECDSAPSHA256 were
   considered.  For authenticated denial, NSEC, NSEC3, NSEC5 with
   RSAFDH-SHA256-SHA256 (2048-bit), and NSEC5 with EC-P256-SHA256 were
   considered.  (Note that NSEC5 with EC-ED25519-SHA256 is identical to
   EC-P256-SHA256 as for response size.)

   For each instance of the signed zone, Name Error responses were
   collected by issuing DNS queries with a random five-character label
   prepended to each actual record name from the zone.  The average and
   standard deviation of the length of these responses are shown below.

   +-----------+--------------+------------------+---------------------+
   |           | DNSKEY       |   Average length |  Standard deviation |
   |           | algorithm    |         (octets) |            (octets) |
   +-----------+--------------+------------------+---------------------+
   | NSEC      | RSA          |             1116 |                  48 |
   | NSEC      | ECDSA        |              543 |                  24 |
   | NSEC3     | RSA          |             1440 |                 170 |
   | NSEC3     | ECDSA        |              752 |                  84 |
   | NSEC5/RSA | RSA          |             1767 |                   7 |
   | NSEC5/EC  | ECDSA        |              839 |                   7 |
   +-----------+--------------+------------------+---------------------+

13.6.  Summary

   As anticipated, NSEC and NSEC3 are the most efficient authenticated
   denial mechanisms, in terms of computation for authoritative server
   and resolver.  NSEC also has the shortest response lengths.  However,
   these mechanisms do not prevent zone enumeration.

   Regarding mechanisms that do prevent zone enumeration, NSEC5 should
   be examined in contrast with Minimally Covering NSEC Records or NSEC3
   White Lies [RFC7129].  The following table summarizes their
   comparison in terms of response size, performance at the
   authoritative server, and performance at the resolver.  For NSEC3
   White Lies, RSASHA256 (2048-bit) and ECDSAPSHA256 were considered,
   and for NSEC5, RSAFDH-SHA256-SHA256 (2048-bit) and EC-P256-SHA256
   were considered.

   +---------------+-----------------+------------------+--------------+
   | Algorithm     | Response length |    Authoritative |     Resolver |
   |               |        (octets) |        (ops/sec) |    (ops/sec) |
   +---------------+-----------------+------------------+--------------+
   | NSEC3WL/RSA   |            1440 |             1500 |        47000 |
   | NSEC3WL/ECDSA |             752 |             7400 |         4000 |
   | NSEC5/RSA     |            1767 |             1500 |        46000 |
   | NSEC5/EC      |             839 |             4700 |         4000 |
   +---------------+-----------------+------------------+--------------+

   NSEC5 responses lengths are only slighly longer than NSEC3 response
   lengths: NSEC5/RSA has responses that are about 22% longer than
   NSEC3/RSA, while NSEC5/EC has responses that are about 13% longer
   than NSEC3/ECDSA.  For both NSEC3 and NSEC5, it is clear that EC-
   based solutions give much shorter responses.

   Regarding the computation performance, with RSA the difference is
   negligible for both nameserver and resolver, whereas with the EC-
   based schemes there is no slowdown for the resolver, and a slowdown
   of 1.5x for the server.  Importantly, we expect the slowdown to be
   smaller in practice because NSEC3 entails three signing/verifying
   computations per query in the worst case (closest encloser, next
   closer, wildcard at closest encloser) whereas NSEC5 requires only
   two.  The table above does not capture this issue, it just measures
   number of signing/verifying operations per second.  Future versions
   of this draft will more accurately measure and compare NSEC5
   performance.

   Note also that while NSEC3 White Lies outperforms NSEC5 for certain
   cases, NSEC3 White Lies require authoratitive nameserver to store the
   private zone-signing key, making each nameserver a potential point of
   compromise.

14.  Security Considerations

14.1.  Zone Enumeration Attacks

   NSEC5 is robust to zone enumeration via offline dictionary attacks by
   any attacker that does not know the NSEC5 private key.  Without the
   private NSEC5 key, that attacker cannot compute the NSEC5 proof that
   corresponds to a given name; the only way it can learn the NSEC5
   proof value for a given name is by sending a queries for that name to
   the authoritative server.  Without the NSEC5 proof value, the
   attacker cannot learn the NSEC5 hash value.  Thus, even an attacker
   that collects the entire chain of NSEC5 RR for a zone cannot use
   offline attacks to "reverse" that NSEC5 hash values in these NSEC5 RR
   and thus learn which names are present in the zone.  A formal
   cryptographic proof of this property is in [nsec5].

14.2.  Hash Collisions

   Hash collisions between QNAME and the owner name of an NSEC5 RR may
   occur.  When they do, it will be impossible to prove the non-
   existence of the colliding QNAME.  However, with SHA-256, this is
   highly unlikely (on the order of 1 in 2^128).  Note that DNSSEC
   already relies on the presumption that a cryptographic hash function
   is collision resistant, since these hash functions are used for
   generating and validating signatures and DS RRs.  See also the
   discussion on key lengths in [nsec5].

14.3.  Compromise of the Private NSEC5 Key

   NSEC5 requires authoritative servers to hold the private NSEC5 key,
   but not the private zone-signing keys or the private key-signing keys
   for the zone.

   The private NSEC5 key needs only be as secure as the DNSSEC records
   whose the privacy (against zone-enumeration attacks) that NSEC5 is
   protecting.  This is because even an adversary that knows the private
   NSEC5 key cannot modify the contents of the zone; this is because the
   zone contents are signed using the private zone-signing key, while
   the private NSEC5 key is only used to compute NSEC5 proof values.
   Thus, a compromise of the private NSEC5 keys does not lead to a
   compromise of the integrity of the DNSSEC record in the zone;
   instead, all that is lost is privacy against zone enumeration, if the
   attacker that knows the private NSEC5 key can compute NSEC5 hashes
   offline, and thus launch offline dictionary attacks.  Thus, a
   compromise of the private NSEC5 key effectively downgrades the
   security of NSEC5 to that of NSEC3.  A formal cryptographic proof of
   this property is in [nsec5].

   If a zone owner wants to preserve this property of NSEC5, the zone
   owner SHOULD choose the NSEC5 private key to be different from the
   private zone-signing keys or key-signing keys for the zone.

14.4.  Key Length Considerations

   The NSEC5 key must be long enough to withstand attacks for as long as
   the privacy of the zone is important.  Even if the NSEC5 key is
   rolled frequently, its length cannot be too short, because zone
   privacy may be important for a period of time longer than the
   lifetime of the key.  (For example, an attacker might collect the
   entire chain of NSEC5 RR for the zone over one short period, and
   then, later (even after the NSEC5 key expires) perform an offline
   dictionary attack that attempt to "reverse" the NSEC5 hash values
   present in the NSEC5 RRs.)  This is in contrast to zone-signing and
   key-signing keys used in DNSSEC; these keys, which ensure the
   authenticity and integrity of the zone contents need to remain secure
   only during their lifetime.

14.5.  Transitioning to a New NSEC5 Algorithm

   Although the NSEC5KEY RR formats include a hash algorithm parameter,
   this document does not define a particular mechanism for safely
   transitioning from one NSEC5 algorithm to another.  When specifying a
   new hash algorithm for use with NSEC5, a transition mechanism MUST
   also be defined.  It is possible that the only practical and
   palatable transition mechanisms may require an intermediate
   transition to an insecure state, or to a state that uses NSEC or
   NSEC3 records instead of NSEC5.

15.  IANA Considerations

   This document updates the IANA registry "Domain Name System (DNS)
   Parameters" in subregistry "Resource Record (RR) TYPEs", by defining
   the following new RR types:

   NSEC5KEY value XXX.

   NSEC5 value XXX.

   NSEC5PROOF value XXX.

   This document creates a new IANA registry for NSEC5 algorithms.  This
   registry is named "DNSSEC NSEC5 Algorithms".  The initial content of
   the registry is:

   0 is Reserved.

   1 is RSAFDH-SHA256-SHA256.

   2 is EC-P256-SHA256.

   3 is EC-ED25519-SHA256.

   4-255 is Available for assignment.

   This document updates the IANA registry "DNS Security Algorithm
   Numbers" by defining following aliases:

   XXX is NSEC5-RSASHA256, alias for RSASHA256 (8).

   XXX is NSEC5-RSASHA512, alias for RSASHA512 (10).

   XXX is NSEC5-ECDSAP256SHA256 alias for ECDSAP256SHA256 (13).

   XXX is NSEC5-ECDSAP384SHA384 alias for ECDSAP384SHA384 (14).

16.  Contributors

   This document would not be possible without help of Moni Naor
   (Weizmann Institute), Sachin Vasant (Cisco Systems), Leonid Reyzin
   (Boston University), and Asaf Ziv (Weizmann Institute) who
   contributed to the design of NSEC5, and Ondrej Sury (CZ.NIC Labs) who
   provided advice on its implementation.

17.  References

17.1.  Normative References

   [FIPS-186-3]
              National Institute for Standards and Technology, "Digital
              Signature Standard (DSS)", June 2009.

   [I-D.ietf-curdle-dnskey-ed25519]
              Sury, O. and R. Edmonds, "Ed25519 for DNSSEC", draft-ietf-
              curdle-dnskey-ed25519-01 (work in progress), February
              2016.

   [RFC1034]  Mockapetris, P., "Domain names - concepts and facilities",
              STD 13, RFC 1034, DOI 10.17487/RFC1034, November 1987,
              <http://www.rfc-editor.org/info/rfc1034>.

   [RFC1035]  Mockapetris, P., "Domain names - implementation and
              specification", STD 13, RFC 1035, DOI 10.17487/RFC1035,
              November 1987, <http://www.rfc-editor.org/info/rfc1035>.

   [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
              Requirement Levels", BCP 14, RFC 2119,
              DOI 10.17487/RFC2119, March 1997,
              <http://www.rfc-editor.org/info/rfc2119>.

   [RFC2136]  Vixie, P., Ed., Thomson, S., Rekhter, Y., and J. Bound,
              "Dynamic Updates in the Domain Name System (DNS UPDATE)",
              RFC 2136, DOI 10.17487/RFC2136, April 1997,
              <http://www.rfc-editor.org/info/rfc2136>.

   [RFC2181]  Elz, R. and R. Bush, "Clarifications to the DNS
              Specification", RFC 2181, DOI 10.17487/RFC2181, July 1997,
              <http://www.rfc-editor.org/info/rfc2181>.

   [RFC2308]  Andrews, M., "Negative Caching of DNS Queries (DNS
              NCACHE)", RFC 2308, DOI 10.17487/RFC2308, March 1998,
              <http://www.rfc-editor.org/info/rfc2308>.

   [RFC3110]  Eastlake 3rd, D., "RSA/SHA-1 SIGs and RSA KEYs in the
              Domain Name System (DNS)", RFC 3110, DOI 10.17487/RFC3110,
              May 2001, <http://www.rfc-editor.org/info/rfc3110>.

   [RFC3447]  Jonsson, J. and B. Kaliski, "Public-Key Cryptography
              Standards (PKCS) #1: RSA Cryptography Specifications
              Version 2.1", RFC 3447, DOI 10.17487/RFC3447, February
              2003, <http://www.rfc-editor.org/info/rfc3447>.

   [RFC4033]  Arends, R., Austein, R., Larson, M., Massey, D., and S.
              Rose, "DNS Security Introduction and Requirements",
              RFC 4033, DOI 10.17487/RFC4033, March 2005,
              <http://www.rfc-editor.org/info/rfc4033>.

   [RFC4034]  Arends, R., Austein, R., Larson, M., Massey, D., and S.
              Rose, "Resource Records for the DNS Security Extensions",
              RFC 4034, DOI 10.17487/RFC4034, March 2005,
              <http://www.rfc-editor.org/info/rfc4034>.

   [RFC4035]  Arends, R., Austein, R., Larson, M., Massey, D., and S.
              Rose, "Protocol Modifications for the DNS Security
              Extensions", RFC 4035, DOI 10.17487/RFC4035, March 2005,
              <http://www.rfc-editor.org/info/rfc4035>.

   [RFC4648]  Josefsson, S., "The Base16, Base32, and Base64 Data
              Encodings", RFC 4648, DOI 10.17487/RFC4648, October 2006,
              <http://www.rfc-editor.org/info/rfc4648>.

   [RFC5114]  Lepinski, M. and S. Kent, "Additional Diffie-Hellman
              Groups for Use with IETF Standards", RFC 5114,
              DOI 10.17487/RFC5114, January 2008,
              <http://www.rfc-editor.org/info/rfc5114>.

   [RFC5155]  Laurie, B., Sisson, G., Arends, R., and D. Blacka, "DNS
              Security (DNSSEC) Hashed Authenticated Denial of
              Existence", RFC 5155, DOI 10.17487/RFC5155, March 2008,
              <http://www.rfc-editor.org/info/rfc5155>.

   [RFC6234]  Eastlake 3rd, D. and T. Hansen, "US Secure Hash Algorithms
              (SHA and SHA-based HMAC and HKDF)", RFC 6234,
              DOI 10.17487/RFC6234, May 2011,
              <http://www.rfc-editor.org/info/rfc6234>.

   [RFC6605]  Hoffman, P. and W. Wijngaards, "Elliptic Curve Digital
              Signature Algorithm (DSA) for DNSSEC", RFC 6605,
              DOI 10.17487/RFC6605, April 2012,
              <http://www.rfc-editor.org/info/rfc6605>.

   [RFC7748]  Langley, A., Hamburg, M., and S. Turner, "Elliptic Curves
              for Security", RFC 7748, DOI 10.17487/RFC7748, January
              2016, <http://www.rfc-editor.org/info/rfc7748>.

   [SECG1]    Standards for Efficient Cryptography Group (SECG), "SEC 1:
              Elliptic Curve Cryptography",
              <http://www.secg.org/sec1-v2.pdf>.

17.2.  Informative References

   [nsec3gpu]
              Wander, M., Schwittmann, L., Boelmann, C., and T. Weis,
              "GPU-Based NSEC3 Hash Breaking".

   [nsec3walker]
              Bernstein, D., "Nsec3 walker",
              <http://dnscurve.org/nsec3walker.html>.

   [nsec5]    Goldberg, S., Naor, M., Papadopoulos, D., Reyzin, L.,
              Vasant, S., and A. Ziv, "NSEC5: Provably Preventing DNSSEC
              Zone Enumeration".

   [nsec5ecc]
              Goldberg, S., Naor, M., Papadopoulos, D., and L. Reyzin,
              "NSEC5 from Elliptic Curves".

   [RFC6781]  Kolkman, O., Mekking, W., and R. Gieben, "DNSSEC
              Operational Practices, Version 2", RFC 6781,
              DOI 10.17487/RFC6781, December 2012,
              <http://www.rfc-editor.org/info/rfc6781>.

   [RFC7129]  Gieben, R. and W. Mekking, "Authenticated Denial of
              Existence in the DNS", RFC 7129, DOI 10.17487/RFC7129,
              February 2014, <http://www.rfc-editor.org/info/rfc7129>.

Appendix A.  RSA Full Domain Hash Algorithm

   The Full Domain Hash (FDH) is a RSA-based scheme that allows
   authentication of hashes using public-key cryptography.

   In this document, the notation from [RFC3447] is used.

   Used parameters:

     (n, e) - RSA public key

     K - RSA private key

     k - length of the RSA modulus n in octets

   Fixed options:

     Hash - hash function to be used with MGF1

   Used primitives:

   I2OSP -  Coversion of a nonnegative integer to an octet string as
      defined in Section 4.1 of [RFC3447]

   OS2IP -  Coversion of an octet string to a nonnegative integer as
      defined in Section 4.2 of [RFC3447]

   RSASP1 -  RSA signature primitive as defined in Section 5.2.1 of
      [RFC3447]

   RSAVP1 -  RSA verification primitive as defined in Section 5.2.2 of
      [RFC3447]

   MGF1 -  Mask Generation Function based on a hash function as defined
      in Section B.2.1 of [RFC3447]

A.1.  FDH signature

   FDH_SIGN(K, M)

   Input:

     K - RSA private key

     M - message to be signed, an octet string

   Output:

     S - signature, an octet string of length k

   Steps:

   1.  EM = MGF1(M, k - 1)

   2.  m = OS2IP(EM)

   3.  s = RSASP1(K, m)

   4.  S = I2OSP(s, k)

   5.  Output S

A.2.  FDH verification

   FDH_VERIFY((n, e), M, S)

   Input:

     (n, e) - RSA public key

     M - message whose signature is to be verified, an octet string

     S - signature to be verified, an octet string of length k

   Output:

     "valid signature" or "invalid signature"

   Steps:

   1.  s = OS2IP(S)

   2.  m = RSAVP1((n, e), s)

   3.  EM = I2OSP(m, k - 1)

   4.  EM' = MGF1(M, k - 1)

   5.  If EM and EM' are the same, output "valid signature"; else output
       "invalid signature".

Appendix B.  Elliptic Curve VRF

   The Elliptic Curve Verifiable Random Function (VRF) is a EC-based
   scheme that allows authentication of hashes using public-key
   cryptography.

   Fixed options:

     G - EC group

   Used parameters:

     g^x - EC public key

     x - EC private key

     q - primer order of group G

     g - generator of group G

   Used primitives:

   "" -  empty octet string

   || -  octet string concatenation

   p^k -  EC point multiplication

   p1*p2 -  EC point addition

   SHA256 -  hash function SHA-256 as specified in [RFC6234]

   ECP2OS -  EC point to octet string conversion with point compression
      as specified in Section 2.3.3 of [SECG1]

   OS2ECP -  octet string to EC point conversion with point compression
      as specified in Section 2.3.4 of [SECG1]

B.1.  ECVRF Hash To Curve

   ECVRF_hash_to_curve(m)

   Input:

     m - value to be hashed, an octet string

   Output:

     h - hashed value, EC point

   Steps:

   1.  c = 0

   2.  C = I2OSP(c, 4)

   3.  xc = SHA256(m || C)

   4.  p = 0x02 || xc

   5.  If p is not a valid octet string representing encoded compressed
       point in G:

       A.  c = c + 1

       B.  Go to step 2.

   6.  h = OS2ECP(p)

   7.  Output h

B.2.  ECVRF Auxiliary Functions

B.2.1.  ECVRF Hash Points

   ECVRF_hash_points(p_1, p_2, ..., p_n)

   Input:

     p_x - EC point in G

   Output:

     h - hash value, integer between 0 and 2^128-1

   Steps:

   1.  P = ""

   2.  for p in [p_1, p_2, ... p_n]: P = P || ECP2OS(p)

   3.  h' = SHA256(P)

   4.  h = OS2IP(first 16 octets of h')

   5.  Output h

B.2.2.  ECVRF Proof To Hash

   ECVRF_proof_to_hash(gamma)

   Input:

     gamma - VRF proof, EC point in G with coordinates (x, y)

   Output:

     beta - VRF hash, octet string (32 octets)

   Steps:

   1.  beta = I2OSP(x, 32)

   2.  Output beta

   Note: Because of the format of compressed form of an elliptic curve,
   the hash can be retrieved from an encoded gamma simply by omitting
   the first octet of the gamma.

B.2.3.  ECVRF Decode Proof

   ECVRF_decode_proof(pi)

   Input:

     pi - VRF proof, octet string (81 octets)

   Output:

     gamma - EC point

     c - integer between 0 and 2^128-1

     s - integer between 0 and 2^256-1

   Steps:

   1.  let gamma', c', s' be pi split after 33-rd and 49-th octet

   2.  gamma = OS2ECP(gamma')

   3.  c = OS2IP(c')

   4.  s = OS2IP(s')

   5.  Output gamma, c, and s

B.3.  ECVRF Signing

   ECVRF_sign(g^x, x, alpha)

   Input:

     g^x - EC public key

     x - EC private key

     alpha - message to be signed, octet string

   Output:

     pi - VRF proof, octet string (81 octets)

     beta - VRF hash, octet string (32 octets)

   Steps:

   1.  h = ECVRF_hash_to_curve(alpha)

   2.  gamma = h^x

   3.  choose a nonce k from [0, q-1]

   4.  c = ECVRF_hash_points(g, h, g^x, h^x, g^k, h^k)

   5.  s = k - c*q mod q

   6.  pi = ECP2OS(gamma) || I2OSP(c, 16) || I2OSP(s, 32)

   7.  beta = h2(gamma)

   8.  Output pi and beta

B.4.  ECVRF Verification

   ECVRF_VERIFY(g^x, pi, alpha)

   Input:

     g^x - EC public key

     pi - VRF proof, octet string

     alpha - message to verify, octet string

   Output:

     "valid signature" or "invalid signature"

     beta - VRF hash, octet string (32 octets)

   Steps:

   1.  gamma, c, s = ECVRF_decode_proof(pi)

   2.  u = (g^x)^c * g^s

   3.  h = ECVRF_hash_to_curve(alpha)

   4.  v = gamma^c * h^s

   5.  c' = ECVRF_hash_points(g, h, g^x, gamma, u, v)

   6.  beta = ECVRF_proof_to_hash(gamma)

   7.  If c and c' are the same, output "valid signature"; else output
       "invalid signature".  Output beta.

   [[CREF1: TODO: check validity of gamma before hashing --Jan]]

Appendix C.  Change Log

   Note to RFC Editor: if this document does not obsolete an existing
   RFC, please remove this appendix before publication as an RFC.

   pre 00 -  initial version of the document submitted to mailing list
      only

   00 -  fix NSEC5KEY rollover mechanism, clarify NSEC5PROOF RDATA,
      clarify inputs and outputs for NSEC5 proof and NSEC5 hash
      computation

   01 -  added Performance Considerations section

   02 -  Elliptic Curve based VRF for NSEC5 proofs; response sizes based
      on empirical data

Appendix D.  Open Issues

   Note to RFC Editor: please remove this appendix before publication as
   an RFC.

   1.  Consider alternative way to signalize NSEC5 support.  The NSEC5
       could use only one DNSSEC algorithm identifier, and the actual
       algorithm to be used for signing can be the first octet in DNSKEY
       public key field and RRSIG signature field.  Similar approach is
       used by PRIVATEDNS and PRIVATEOID defined in [RFC4034].

   2.  How to add new NSEC5 hashing algorithm.  We will need to add new
       DNSSEC algorithm identifiers again.

   3.  NSEC and NSEC3 define optional steps for hash collisions
       detection.  We don't have a way to avoid them if they really
       appear (unlikely).  We would have to drop the signing key and
       generate a new one.  Which cannot be done instantly.

   4.  Write Special Considerations section.

   5.  Contributor list has to be completed.

Authors' Addresses

   Jan Vcelak
   CZ.NIC
   Milesovska 1136/5
   Praha  130 00
   CZ

   Email: jan.vcelak@nic.cz


   Sharon Goldberg
   Boston University
   111 Cummington St, MCS135
   Boston,  MA 02215
   USA

   Email: goldbe@cs.bu.edu


   Dimitrios Papadopoulos
   Boston University
   111 Cummington St, MCS135
   Boston,  MA 02215
   USA

   Email: dipapado@bu.edu

```
