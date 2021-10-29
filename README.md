# solid-link-metadata
Adds link forwarding and archive/cache information to a URL

This repository contains both a [link metadata ontology](links) and a specification on how to use it in a [solid](https://solidproject.org/) application or server.

## Usage Examples

### Permanent Redirect

Redirecting a local resource permanently to a new URL, turtle format:
```turtle
@prefix ldpl: <https://purl.org/pdsinterop/link-metadata#> .
@prefix dbr: <http://dbpedia.org/resource/> .
@prefix local: <./> .

local:Bob_Marley
	ldpl:redirectPermanent dbr:Bob_Marley .
```

If you upload this turtle file as http://www.example.com/.meta, supposing that it is a solid data pod, the server should interpret this file so that a subsequent request for http://www.example.com/Bob_Marley will result in a permanent redirect response, like this:

```http
HTTP/1.1 308 Moved Permanently
Location: http://dbpedia.org/resource/Bob_Marley
```

A Solid application getting this response should update the link to the new URL, if possible.

### Temporary Redirect

This is very similar to the permanent redirect, but doesn't imply the requesting application should update the originating link.

```turtle
@prefix ldpl: <https://purl.org/pdsinterop/link-metadata#> .
@prefix dbr: <http://dbpedia.org/resource/> .
@prefix local: <./> .

local:Bob_Marley
	ldpl:redirectTemporary dbr:Bob_Marley .
```

### Forget (Deleted)

This term is specifically meant to implement the GDPR right to be forgotten. If you sent a .meta file like this:

```turtle
@prefix ldpl: <https://purl.org/pdsinterop/link-metadata#> .
@prefix dbr: <http://dbpedia.org/resource/> .
@prefix local: <./> .

local:Bob_Harley
	ldpl:forget "This was a typo, it should never have been here" .
```

The third part of this triple is the reason for the instruction to forget this link. This may be empty. If uploaded to a solid server that supports it, the server will then respond to later requests for ./Bob_Harly with:

```http
HTTP/1.1 410 Gone
X-LPDL-Forget: This was a typo, it should never have been here
```

### Not found

There is no reason to explicitly add this instruction to a .meta file or other data, but is defined here just to be complete.

### Archive and ArchiveDate 

This term adds an archive link to a resource.

```turtle
@prefix ldpl: <https://purl.org/pdsinterop/link-metadata#> .

<http://www.muze.nl/>
	lpdl:archive <https://web.archive.org/web/20000605230138/http://www.muze.nl/> .

<https://web.archive.org/web/20000605230138/http://www.muze.nl/>
	lpdl:archiveDate "2000-06-05T23:01:38" .
```

The arciveDate term is applied to the archive link. The third part must be an ISO 8601 compliant datetime string, but may choose to just supply the date, without a time.

You may add multiple archive links to a resource. And there is no requirement to also add an archiveDate for each archive, but it is strongly advised to do so.

### Content Hash



## More Complex Redirect Scenario's

### Permanent Redirect to a Forget Link

In this scenario an application reads a resource that contains a link to location A. Requesting this link returns a permanent redirect to location B, either through the HTTP 308 header, or in the data using the link-metadata ontology. Requesting location B returns a 410 Gone header, or in the dataset has the link-metadata:forget triple.

The correct response for the application is to remove the original link to location A, if possible. Since this link is permanently redirected, it should be updated. However the new location sends the instruction to forget that new link. This means that the instruction to forget location B should also be applied to location A.

### Temporary Redirect to a Forget Link

This scenario is not as clear-cut. A temporary redirect may be in place for any number of reasons. It does not imply that all information from the new location will always be applicable. So the requesting application should keep the original link intact.

### Temporary Redirect to a Permanent Redirect

In this scenario the original link returns a temporary redirect instruction. The new location returns a permanent redirect instruction. The requesting application should not update the original link, but keep it intact.

