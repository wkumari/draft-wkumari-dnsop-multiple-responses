**Important:** Read CONTRIBUTING.md before submitting feedback or contributing
```




dnsop                                                          W. Kumari
Internet-Draft                                                    Google
Intended status: Standards Track                                  Z. Yan
Expires: July 7, 2015                                              CNNIC
                                                             W. Hardaker
                                                           Parsons, Inc.
                                                         January 3, 2015


             Returning multiple answers in a DNS response.
                 draft-wkumari-dnsop-multiple-responses

Abstract

   This document reintroduces the ability to provide mulltiple answers
   in a DNS response.

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

   This Internet-Draft will expire on July 7, 2015.

Copyright Notice

   Copyright (c) 2015 IETF Trust and the persons identified as the
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



Kumari, et al.            Expires July 7, 2015                  [Page 1]

Internet-Draft            DNS Multiple Answers              January 2015


Table of Contents

   1.  Introduction  . . . . . . . . . . . . . . . . . . . . . . . .   2
     1.1.  Requirements notation . . . . . . . . . . . . . . . . . .   3
   2.  Background  . . . . . . . . . . . . . . . . . . . . . . . . .   3
   3.  Terminology . . . . . . . . . . . . . . . . . . . . . . . . .   3
   4.  Returning multiple answers  . . . . . . . . . . . . . . . . .   3
   5.  Additional records pseudo-RR  . . . . . . . . . . . . . . . .   5
   6.  Signalling support  . . . . . . . . . . . . . . . . . . . . .   5
   7.  Use of Additional information . . . . . . . . . . . . . . . .   5
   8.  IANA Considerations . . . . . . . . . . . . . . . . . . . . .   6
   9.  Security Considerations . . . . . . . . . . . . . . . . . . .   6
   10. Acknowledgements  . . . . . . . . . . . . . . . . . . . . . .   6
   11. References  . . . . . . . . . . . . . . . . . . . . . . . . .   6
     11.1.  Normative References . . . . . . . . . . . . . . . . . .   6
     11.2.  Informative References . . . . . . . . . . . . . . . . .   7
   Appendix A.  Changes / Author Notes.  . . . . . . . . . . . . . .   7
   Authors' Addresses  . . . . . . . . . . . . . . . . . . . . . . .   7

1.  Introduction

   Often the name being resolved in the DNS provides information about
   why the name is being resolved, allowing the authorative name server
   operator to predict what other answers the client will soon query
   for.  By providing multiple answers in the response, the authorative
   name server operator can ensure that the recursive server that the
   client is using has all the answers in it cache.

   For example, the name server operator of Example Widgets, Inc
   (example.com) knows that the example.com web page at www.example.com
   contains various resources, including some images (served from
   images.example.com), some Cascading Style Sheets (served from
   css.example.com) and some JavaScript (data.example.com).  A client
   attempting to resolve www.example.com is very likely to be a web
   browser, rendering the page, and so will need to also resolve all of
   the other names for these other resources.  Providing all of these
   answers in response to a query for www.example.com allows the
   recursive server to populate its cache and have all of the answers
   available when the client asks for them.

   Other examples where this technique is useful include SMTP (including
   the mail server address when serving the MX record), SRV (providing
   the target information in addition to the SRV response) and TLSA
   (providing any TLSA records associated with a name).

   This is purely an optimization - by providing all of other, related
   answers that the client is likely to need along with the answer that
   they requested, users get a better experience, iterative servers need



Kumari, et al.            Expires July 7, 2015                  [Page 2]

Internet-Draft            DNS Multiple Answers              January 2015


   to perform less queries, authorative servers have to answer fewer
   queries, etc.

   [I-D.ietf-sidr-iana-objects] and this is a reference to a draft.

1.1.  Requirements notation

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
   document are to be interpreted as described in [RFC2119].

2.  Background

   The existing DNS specifications allow for additional information to
   be included in the "additional" section of the DNS response, but in
   order to defeat cache poisining attacks most implementations either
   ignore or don't trust additional information (other than for "glue").
   For some more background, see [Ref.Bellovin], [RFC1034], [RFC2181].

   Not trusting the information in the additional section was necessary
   because there was no way to authenticate it.  If you queried for
   www.example.com and got back answers for www.invalid.com you couldn't
   tell if these were actually from invalid.com or if an attacker was
   trying to get bad information for invalid.com into your cache.  In a
   world of ubiquitious DNSSEC deployment [Ed note: By the time this
   document is published, there *will* be ubiquitous DNSSEC :-) ] the
   iterative server can validate the information and trust it.

3.  Terminology

   Additional records  Additional records are records that the
      authorative nameserver has included in the Additional section.

   Primary query  A Primary query (or primary question) is a QNAME that
      the name server operator would like to return additional answers
      for.

   Supporting information  Supporting information is the DNSSEC RRSIGs
      that prove the authenticity of the Additional records.

4.  Returning multiple answers

   The authorative nameserver should include as many of the instructed
   Additional records and Supporting information as will fit in the
   resposne packet.






Kumari, et al.            Expires July 7, 2015                  [Page 3]

Internet-Draft            DNS Multiple Answers              January 2015


   In order to include Additional records in a response, certain
   conditions need to be met.  [Ed note: Some discussion on each rule is
   below]

   1.  Additional records MUST only be included when the primary name is
       DNSSEC secured.

   2.  Additional records MUST only be served over TCP connections.
       This is to mitigate Denial of Service reflection attacks.[1]

   3.  Additional records MUST be leaf records at the same node in the
       DNS tree[2]

   4.  The DNSSEC supporting information must be included.  This is the
       RRSIGs required to validate the Additional record information.

   5.  All of the records MUST be signed with the same DNSSEC keys.

   6.  The authorative namesever SHOULD include as many of the
       additional records as will fit in the response.  Each Additional
       record MUST have its matching Supporting information.  Additional
       records MUST be inserted in the order specified in the Additional
       records list.

   7.  Operators SHOULD only include Additional answers that they expect
       a client to actually need. [3]

   [Ed note 1: The above MAY be troll bait.  I'm not really sure if this
   is a good idea or not - moving folk towards TCP is probably a good
   idea, and this is somewhat of an optional record type.  Then again,
   special handing (TCP only) for a record would be unusual.  Additional
   records could cause responses to become really large, but there are
   already enough large records that can be used for reflection attacks
   that we can jsut give up on the whole "keep responses as small as
   possible" ship.  ]

   [Ed note 2: This is poorly worded.  I mumbled about bailiwick,
   subdomains, etc but nothing I could come up with was better.  Also,
   is this rule actually needed?  I *think* it would be bad for .com
   servers to be able to include Additional records for
   www.foo.bar.baz.example.com, but perhaps <handwave>public-suffix-
   list?! This rule also makes it easier to decide what all DNSSEC
   information is required.]

   [Ed note 3: This is not enforcable. ]






Kumari, et al.            Expires July 7, 2015                  [Page 4]

Internet-Draft            DNS Multiple Answers              January 2015


5.  Additional records pseudo-RR

   To allow the authorative nameserver operator to configure what
   additional records to serve when it receives a query to a label, we
   introduce the Additional pseudo Resource Record (RR).  This is a
   pseudo-record as it provides instruction to the authorative
   nameserver, and does not appear on the wire.  [Ed note: I had
   originally considered a comment, or some sort of format where we
   listed additional records under the primary one, but we a: wanted it
   to survive zone transfers, and b: not trip up zone file parsers. ]

   The format of the Additional pseudo-RR is:

   label ADD "label,type,ttl; label,type,ttl; label,type,ttl; ..."

   For example, if the operator of example.com would like to also return
   A record answers for images.example.com, css.example.com and both an
   A and AAAA for data.example.com when queried for www.example.com he
   would enter:

   www ADD "images,A,600; css,A,1800; data,A,600; data,AAA,600;"

   The entries in the ADD list are ordered.  An authorative nameserver
   MUST attempt to insert the records in the order listed when
   determining how many records it can fit in the resposne packet.

6.  Signalling support

   Iterative nameservers that support Additional records signal this by
   setting the Z bit (bit 25 of the DNS header).

   [RFC5395] Section 2.1 says:

   There have been ancient DNS implementations for which the Z bit
   being on in a query meant that only a response from the primary
   server for a zone is acceptable. It is believed that current
   DNS implementations ignore this bit.

   Assigning a meaning to the Z bit requires an IETF Standards Action.

   [ Ed note: Hey, was worth a try.  I'm fine with an EDNS0 bit instead
   ]

7.  Use of Additional information

   When receiving Additional information, an iterative server follows
   certain rules:




Kumari, et al.            Expires July 7, 2015                  [Page 5]

Internet-Draft            DNS Multiple Answers              January 2015


   1.  Additional records MUST be validated before being used.

   2.  Additional records SHOULD be annotated in the cache as having
       been received as Additional records.

   3.  Additional records SHOULD have lower priority in the cache than
       answers received because they were requested.  This is to help
       evict Additional records from the cache first, and help stop
       cache filling attacks.

   4.  Iterative servers MAY choose to ignore Additional records for any
       reason, including CPU or cache space concerns, phase of the moon,
       etc.  It may choose to only accept all, some or none of the
       Additional records.

8.  IANA Considerations

   This document contains no IANA considerations.Template: Fill this in!

9.  Security Considerations

   Additional records will make DNS responses even larger than they are
   currently, leading to more large records that can be used for DNS
   reflection attacks.  We mitigate this by only serving these over TCP.

   A malicious authorative server could include a large number of
   Additional records (and associated DNSSEC information) and attempt to
   DoS the recursive by making it do lots of DNSSEC validation.  I don't
   view this as a very serious threat (CPU for validation is cheap
   compared to bandwith), but we mitigate this by allowing the iterative
   to ignore Additional records whenever it wants.

   By requiring the ALL of the Additional records are signed, and all
   necessary DNSSEC information for validation be included we avoid
   cache poisoning (I hope :-))

10.  Acknowledgements

   The authors wish to thank some folk.

11.  References

11.1.  Normative References

   [RFC1034]  Mockapetris, P., "Domain names - concepts and facilities",
              STD 13, RFC 1034, November 1987.





Kumari, et al.            Expires July 7, 2015                  [Page 6]

Internet-Draft            DNS Multiple Answers              January 2015


   [RFC2119]  Bradner, S., "Key words for use in RFCs to Indicate
              Requirement Levels", BCP 14, RFC 2119, March 1997.

   [RFC2181]  Elz, R. and R. Bush, "Clarifications to the DNS
              Specification", RFC 2181, July 1997.

   [RFC5395]  Eastlake, D., "Domain Name System (DNS) IANA
              Considerations", RFC 5395, November 2008.

   [Ref.Bellovin]
              Bellovin, S., "Using the Domain Name System for System
              Break-Ins", 1995, <https://www.cs.columbia.edu/~smb/
              papers/dnshack.pdf>.

11.2.  Informative References

   [I-D.ietf-sidr-iana-objects]
              Manderson, T., Vegoda, L., and S. Kent, "RPKI Objects
              issued by IANA", draft-ietf-sidr-iana-objects-03 (work in
              progress), May 2011.

Appendix A.  Changes / Author Notes.

   [RFC Editor: Please remove this section before publication ]

   From -00 to -01.

   o  Nothing changed in the template!

Authors' Addresses

   Warren Kumari
   Google
   1600 Amphitheatre Parkway
   Mountain View, CA  94043
   US

   Email: warren@kumari.net


   Zhiwei Yan
   CNNIC
   No.4 South 4th Street, Zhongguancun
   Beijing  100190
   P. R. China

   Email: yanzhiwei@cnnic.cn




Kumari, et al.            Expires July 7, 2015                  [Page 7]

Internet-Draft            DNS Multiple Answers              January 2015


   Wes Hardaker
   Parsons, Inc.
   P.O. Box 382
   Davis, CA  95617
   US

   Email: ietf@hardakers.net












































Kumari, et al.            Expires July 7, 2015                  [Page 8]
```
