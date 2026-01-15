---
title: Adding an Uncacheable Directory-Entry Metadata Attribute to NFSv4.2
abbrev: Uncacheable Dirents
docname: draft-ietf-nfsv4-uncacheable-directories-latest
category: std
date: {DATE}
consensus: true
ipr: trust200902
area: General
workgroup: Network File System Version 4
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping, comments]

author:
 -
    ins: T. Haynes
    name: Thomas Haynes
    organization: Hammerspace
    email: loghyr@gmail.com

normative:
  RFC2119:
  RFC4506:
  RFC4949:
  RFC7204:
  RFC7862:
  RFC7863:
  RFC8174:
  RFC8178:
  RFC8881:
  RFC9754:

informative:
  POSIX.1:
    title: The Open Group Base Specifications Issue 7
    seriesinfo: IEEE Std 1003.1, 2013 Edition
    author:
      org: IEEE
    date:  2013
  RFC1813:
  Samba:
    target: https://www.samba.org/
    title: Samba.org. Samba Project Website.
  SMB2:
    title:  Server Message Block (SMB) Protocol Versions 2 and 3
    author:
      org: Microsoft Learn

--- abstract

Network File System version 4.2 (NFSv4.2) clients commonly cache
directory entries (dirents) to improve performance. While effective
in many cases, such caching can prevent servers from enforcing
per-user access controls on directory entries and up-to-date directory
entry attributes such as size and timestamps.  This document
introduces a new uncacheable directory attribute for NFSv4.2 that
allows servers to advise clients that caching of directory entry
metadata is unsuitable.  This enables servers to present directory
contents based on user-specific access permissions while remaining
compatible with existing NFSv4.2 clients.

--- note_Note_to_Readers

Discussion of this draft takes place
on the NFSv4 working group mailing list (nfsv4@ietf.org),
which is archived at
[](https://mailarchive.ietf.org/arch/search/?email_list=nfsv4). Source
code and issues list for this draft can be found at
[](https://github.com/ietf-wg-nfsv4/uncacheable-directories).

Working Group information can be found at [](https://github.com/ietf-wg-nfsv4).

--- middle

# Introduction

Clients of remote filesystems commonly cache directory entries
(dirents) to improve performance. This caching is typically shared
across users on the client and assumes that directory contents and
access permissions are uniform across users.

In this document, the term directory is used to describe the context
in which directory entries are retrieved. The uncacheable directory
attribute applies to the caching of directory entry (dirent) metadata,
including names and associated file object metadata such as size
and timestamps. It does not prohibit caching of the directory object
itself, nor does it affect caching of file data.

Access Based Enumeration (ABE), as used in the SMB protocol, restricts
directory visibility based on the access permissions of the requesting
user. Implementing similar behavior in NFSv4.2 requires server
involvement, as clients may not have sufficient information to
evaluate permissions based on identity mappings, ACLs, or server-local
policy.

Even in the absence of Access Based Enumeration, caching of directory
entry metadata can result in stale size and timestamp information
when files are modified concurrently, reducing the effectiveness
of uncacheable file data semantics when directory entry metadata
is stale.

This document introduces the uncacheable directory attribute to
NFSv4.2 to implement ABE. As such, it is an OPTIONAL to implement
attribute for NFSv4.2. If both the client and the server support
this attribute, the client is advised to bypass caching of directory
entries for directories marked as uncacheable.

The uncacheable directory entry attribute is read-write and per
directory object. The data type is bool.

Allowing clients to set this attribute provides a portable mechanism
for establishing directory access semantics at creation time without
requiring out-of-band administrative configuration. The server
remains authoritative for the attribute value, and existing NFSv4
authorization mechanisms apply.

A client can determine whether the uncacheable directory attribute
is supported for a given directory by issuing a GETATTR request and
examining the returned attribute list.

The only way that the server can determine that the client supports
the attribute is if the client sends either a GETATTR or a SETATTR
with the uncacheable directory attribute.

Using the process detailed in {{RFC8178}}, the revisions in this document
become an extension of NFSv4.2 {{RFC7862}}. They are built on top of the
external data representation (XDR) {{RFC4506}} generated from
{{RFC7863}}.

## Definitions

Access Based Enumeration (ABE)

: When servicing a READDIR or GETATTR operation, the server provides
results based on the access permissions of the user making the request.

dirent

: A directory entry representing a file or subdirectory and its
associated attributes.

dirent caching

: A client-side cache of directory entry names and associated file
object metadata, used to avoid repeated directory lookup and attribute
retrieval.

This document assumes familiarity with NFSv4.2 operations, attributes,
and error handling as defined in {{RFC8881}} and {{RFC7862}}.

## Requirements Language

{::boilerplate bcp14-tagged}

# Caching of Dirents

With a remote filesystem, the client typically caches directory
entries (dirents) locally to improve performance. This cooperation
succeeds because both the server and client operate under POSIX
semantics ([POSIX.1]) and agree to interpretation of mode bits with
respect to the uid and gid in NFSv3 {{RFC1813}}. For NFSv4.2, these
would respectively be the mode, owner, and owner_group attributes
defined in {{Section 5 of RFC8881}}.  Note that this cooperation
does not apply to Access Control List (ACLs) entries as NFSv4.2
does not implement a strict POSIX style ACL.

NFSv4.2 does implement NFSv4.1 ACLs, which are enforced on the
server and not the client. As such, ACL enforcement requires the
client to bypass the dirent cache to have checks done when a new
user attempts to access the dirent.

Another consideration is that not all server implementations natively
support the SMB {{SMB2}}. Instead, they layer Samba {{Samba}} on
top of the NFSv4.2 service. The attributes of hidden, system, and
offline have already been introduced in the NFSv4.2 protocol to
support Samba.  The Samba implementation can utilize these attributes
to provide SMB semantics. While private protocols can supply these
features, it is better to drive them into open standards.

Another concept that can be adapted from SMB is that of Access Based
Enumeration (ABE). If a share or a folder has ABE enabled, then the
user can only see the files and sub-folders for which they have
permissions.

Under the POSIX model, this can be done on the client and not the
server. However, that only works with uid, gid, and mode bits.  If
we consider identity mappings, ACLs, and server local policies,
then the determination of ABE and directory entry visibility is
best performed on the server.

Since cached dirents are shared by all users on a client, and the
client cannot determine access permissions for individual dirents,
all users are presented with the same set of attributes. To address
this, this document introduces the new uncacheable directory
attribute. This attribute advises the client not to cache directory
entry metadata for a file or directory object. Consequently, each
time a client queries for these attributes, the server's response
can be tailored to the specific user making the request.

## Uncacheable Dirents {#sec_dirents}

If a file object or directory has the uncacheable directory attribute
set, the client is advised not to cache directory entry metadata.
In such cases, the client retrieves directory entry attributes from
the server for each request, allowing the server to evaluate access
permissions based on the requesting user.  Clients are advised not
to share cached dirent attributes between different users.

# XDR for Uncacheable Dirents Attribute

~~~ xdr
///
/// typedef bool            fattr4_uncacheable_directory;
///
/// const FATTR4_UNCACHEABLE_DIRECTORY   = 88;
///
~~~

# Extraction of XDR

This document contains the external data representation (XDR)
{{RFC4506}} description of the uncacheable directory attribute.  The XDR
description is presented in a manner that facilitates easy extraction
into a ready-to-compile format. To extract the machine-readable XDR
description, use the following shell script:

~~~ shell
#!/bin/sh
grep '^ *///' $* | sed 's?^ */// ??' | sed 's?^ *///$??'
~~~

For example, if the script is named 'extract.sh' and this document is
named 'spec.txt', execute the following command:

~~~ shell
sh extract.sh < spec.txt > uncacheable_prot.x
~~~

This script removes leading blank spaces and the sentinel sequence '///'
from each line. XDR descriptions with the sentinel sequence are embedded
throughout the document.

Note that the XDR code contained in this document depends on types from
the NFSv4.2 nfs4_prot.x file (generated from {{RFC7863}}).  This includes
both nfs types that end with a 4, such as offset4, length4, etc., as
well as more generic types such as uint32_t and uint64_t.

While the XDR can be appended to that from {{RFC7863}}, the code snippets
should be placed in their appropriate sections within the existing XDR.

# Security Considerations

For a given user A, a client MUST NOT make access decisions for
uncacheable dirents retrieved for another user B. These decisions
MUST be made by the server.  If the client is Labeled NFS aware
({{RFC7204}}), then the client MUST locally enforce the MAC security
policies.

The uncacheable directory attribute allows dirents to be annotated
such that attributes are presented to the user based on the server's
access control decisions.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

Trond Myklebust, Mike Snitzer, Jon Flynn, Keith Mannthey, and Thomas
Haynes all worked on the prototype at Hammerspace.

Rick Macklem, Chuck Lever, and Dave Noveck reviewed the document.

Chris Inacio, Brian Pawlowski, and Gorry Fairhurst helped guide
this process.
