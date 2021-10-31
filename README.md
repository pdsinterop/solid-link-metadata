# solid-link-metadata
Adds link forwarding and archive/cache information to a URL

This repository contains both a [link metadata ontology](links) and a specification on how to use it in a [solid](https://solidproject.org/) application or server.

## Why

The stated goal of the Solid project is to do for data, just what the world-wide web did for documents: build a web of data. One of the unsolved problems of the Web is that of linkrot. If we don't provide some tools to prevent linkrot in data, the web of data will become a pile of half-connected webs.

The web of data is much more granular than the web of documents. One of the design choices is to start with resources that contain many entities. You link to them using the resource link and a hash, e.g:

```
http://www.example.com/resource#entity
```

In a turtle file, this looks like:

```turtle
<#entity> dcterm:title "An entity" .
```

Or alternatively:

```turtle
@prefix local: <./#> .

local:entity dcterm:title "An entity" .
```

The problem is that if you start out like this, at some point it becomes necessary to split your single resource file into multiple resources. One reason might be that you want to give access to a subset of your resource to someone else. You can only define access levels per resource file. So you must move a number of entities from one resource file to another, and update all your internal links between entities. However, if anyone else has linked to your entities, those links will now be broken. You cannot use HTTP redirect headers for the resource, since the resource itself is still there.

By adding the ability to add redirect information inside a resource, for entities in the resource, this problem can be tackled. The external link can be resolved and even automatically updated, if you add a breadcrumb trail to the original resource. e.g.:

```turtle
@prefix ldpl: <https://purl.org/pdsinterop/link-metadata#> .

<#entity> ldpl:redirectPermanent </anotherResource#entity> .
```

Another example, the dutch institution that develops curricula for elementary and secondary education, slo.nl, has made all the curricula data available as linked open data at https://opendata.slo.nl/. One of the design features of all entities is that they are immutable. If a property of an existing entity changes, that entity is deprecated and a new entity with the new property value is created, with a new identity url. The old entity has a property 'replacedBy' that points to the new entity. With this link metadata ontology, this can now be stated in a standard compliant way that any Solid application can understand, as a ldpl:redirectPermanent statement.

## Usage Examples

### Permanent Redirect of a resource

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

### Temporary Redirect of a resource

This is very similar to the permanent redirect, but doesn't imply the requesting application should update the originating link.

```turtle
@prefix ldpl: <https://purl.org/pdsinterop/link-metadata#> .
@prefix dbr: <http://dbpedia.org/resource/> .
@prefix local: <./> .

local:Bob_Marley
	ldpl:redirectTemporary dbr:Bob_Marley .
```

### Forget a resource (Deleted)

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

This term adds an archive link to a resource link.

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

```turtle
@prefix ldpl: <https://purl.org/pdsinterop/link-metadata#> .

<http://www.muze.nl/>
	lpdl:archive <https://web.archive.org/web/20000605230138/http://www.muze.nl/> .

<https://web.archive.org/web/20000605230138/http://www.muze.nl/>
	lpdl:archiveDate "2000-06-05T23:01:38"
	lpdl:contentHash "sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC" .
```

The contentHash term allows you to add a [resource integrity check](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity) to an archive or other link. The application fetching the resource can then check if the returned resource has been altered in any way.

## More Complex Redirect Scenario's

### Permanent Redirect to a Forget Link

In this scenario an application reads a resource that contains a link to location A. Requesting this link returns a permanent redirect to location B, either through the HTTP 308 header, or in the data using the link-metadata ontology. Requesting location B returns a 410 Gone header, or in the dataset has the link-metadata:forget triple.

The correct response for the application is to remove the original link to location A, if possible. Since this link is permanently redirected, it should be updated. However the new location sends the instruction to forget that new link. This means that the instruction to forget location B should also be applied to location A.

### Temporary Redirect to a Forget Link

This scenario is not as clear-cut. A temporary redirect may be in place for any number of reasons. It does not imply that all information from the new location will always be applicable. So the requesting application should keep the original link intact.

### Temporary Redirect to a Permanent Redirect

In this scenario the original link returns a temporary redirect instruction. The new location returns a permanent redirect instruction. The requesting application should not update the original link, but keep it intact.

### Multiple Redirect statements

We found that it is possible that a single entity or resource is split into multiple new resources. The correct way to describe this is to add multiple redirect statements to a single entity, e.g.:

```turtle
@prefix ldpl: <https://purl.org/pdsinterop/link-metadata#> .
@prefix dbr: <http://dbpedia.org/resource/> .
@prefix local: <./> .

local:Bob_Harley
	ldpl:redirectPermanent dbr:Bob_Marley,
	ldpl:redirectPermanent dbr:Harley_Davidson .
```

It is up to the application on how to process this information. One way is to ask the user which of these links is the correct one to follow.