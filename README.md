# solid-link-metadata
Adds link forwarding and archive/cache information to a URL

This repository contains both a [link metadata ontology](links.ttl) and a specification on how to use it in a [solid](https://solidproject.org/) application or server.

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
@prefix lm: <https://purl.org/pdsinterop/link-metadata#> .

<#entity> lm:redirectPermanent </anotherResource#entity> .
```

Another example, the Dutch institution that develops curricula for elementary and secondary education, slo.nl, has made all the curricula data available as linked open data at https://opendata.slo.nl/. One of the design features of all entities is that they are immutable. If a property of an existing entity changes, that entity is deprecated and a new entity with the new property value is created, with a new identity url. The old entity has a property 'replacedBy' that points to the new entity. With this link metadata ontology, this can now be stated in a standard compliant way that any Solid application can understand, as a lm:redirectPermanent statement.

## Usage Examples

### Permanent Redirect of a resource

Redirecting a local resource permanently to a new URL, turtle format:
```turtle
@prefix lm: <https://purl.org/pdsinterop/link-metadata#> .
@prefix dbr: <http://dbpedia.org/resource/> .
@prefix local: <./> .

local:Bob_Marley
	lm:redirectPermanent dbr:Bob_Marley .
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
@prefix lm: <https://purl.org/pdsinterop/link-metadata#> .
@prefix dbr: <http://dbpedia.org/resource/> .
@prefix local: <./> .

local:Bob_Marley
	lm:redirectTemporary dbr:Bob_Marley .
```

### Forget a resource

This term is specifically meant to implement the GDPR right to be forgotten. If you sent a .meta file like this:

```turtle
@prefix lm: <https://purl.org/pdsinterop/link-metadata#> .
@prefix dbr: <http://dbpedia.org/resource/> .
@prefix local: <./> .

local:Bob_Harley
	lm:forget "This was a typo, it should never have been here" .
```

The third part of this triple is the reason for the instruction to forget this link. This may be empty. If uploaded to a solid server that supports it, the server will then respond to later requests for ./Bob_Harly with:

```http
HTTP/1.1 410 Gone
X-LPDL-Forget: This was a typo, it should never have been here
```

### Deleted

This term is here so you can place a tombstone marker in a linked data set. There are numerous usecases for tombstones. One of these is to create an un-delete functionality. If you simply add a 'deleted' predicate to a subject, you can remove it from the normal dataset as rendered in a user interface, or in search results. But you can still recover 'deleted' entities, untill you actually remove them.

Another usecase is in distributed datasets, which at some point may need to be merged. By adding a 'deleted' marker for a subject, the intent of the user is preserved and the entity may be removed from all distributed sets later on.


### Archive and ArchiveDate 

This term adds an archive link to a resource link.

```turtle
@prefix lm: <https://purl.org/pdsinterop/link-metadata#> .

<http://www.muze.nl/>
	lm:archive <https://web.archive.org/web/20000605230138/http://www.muze.nl/> .

<https://web.archive.org/web/20000605230138/http://www.muze.nl/>
	lm:archiveDate "2000-06-05T23:01:38" .
```

The archiveDate term is applied to the archive link. The third part must be an ISO 8601 compliant datetime string, but may choose to just supply the date, without a time.

You may add multiple archive links to a resource. And there is no requirement to also add an archiveDate for each archive, but it is strongly advised to do so.

The reason for adding a new predicate for the archive date instead of using something like dcterms:created, is that this way the semantics of the date are more clear. There is no possible confustion about the meaning of lm:archiveDate, it specifically means 'when this archive copy was created'. If we use 'dcterms:created', you might infer it means when the original dataset was created, instead of just this archive copy.

### Content Hash

```turtle
@prefix lm: <https://purl.org/pdsinterop/link-metadata#> .

<http://www.muze.nl/>
	lm:archive <https://web.archive.org/web/20000605230138/http://www.muze.nl/> .

<https://web.archive.org/web/20000605230138/http://www.muze.nl/>
	lm:archiveDate "2000-06-05T23:01:38"
	lm:contentHash "sha384-oqVuAfXRKap7fdgcCY5uykM6+R9GqQ8K/uxy9rx7HNQlGYl1kPzQho1wx4JwY8wC" .
```

The contentHash term allows you to add a [resource integrity check](https://developer.mozilla.org/en-US/docs/Web/Security/Subresource_Integrity) to an archive or other link. The application fetching the resource can then check if the returned resource has been altered in any way.

There has been some discussion about implementing content hashes, since RDF data can be represented in many different formats. Most servers have the ability to serve the same resource in different formats, based on content negotiation (accept headers). So you cannot reliably use a hash on the raw content. However there has been some progress on creating a hashing algorithm on the parsed graph, as represented in memory. This is independant of the specific representation format used. See for emaple ["Hashing of RDF Graphs and a Solution to the Blank Node Problem - Edzard H&ouml;fix and Ina Schieferdeckeer"](http://ceur-ws.org/Vol-1259/method2014_submission_1.pdf).

## More Complex Redirect Scenario's

### Permanent Redirect to a Forget Link

In this scenario an application reads a resource that contains a link to location A. Requesting this link returns a permanent redirect to location B, either through the HTTP 308 header, or in the data using the link-metadata ontology. Requesting location B returns a 410 Gone header, or the dataset has the link-metadata:forget triple.

The correct response for the application is to remove the original link to location A, if possible. Since this link is permanently redirected, it should be updated. However, the new location sends the instruction to forget that new link. This means that the instruction to forget location B should also be applied to location A.

### Temporary Redirect to a Forget Link

This scenario is not as clear-cut. A temporary redirect may be in place for any number of reasons. It does not imply that all information from the new location will always be applicable. So the requesting application should keep the original link intact.

### Temporary Redirect to a Permanent Redirect

In this scenario the original link returns a temporary redirect instruction. The new location returns a permanent redirect instruction. The requesting application should not update the original link, but keep it intact.

### Multiple Redirect statements

We found that it is possible that a single entity or resource is split into multiple new resources. The correct way to describe this is to add multiple redirect statements to a single entity, e.g.:

```turtle
@prefix lm: <https://purl.org/pdsinterop/link-metadata#> .
@prefix dbr: <http://dbpedia.org/resource/> .
@prefix local: <./> .

local:Bob_Harley
	lm:redirectPermanent dbr:Bob_Marley,
	lm:redirectPermanent dbr:Harley_Davidson .
```

It is up to the application how to process this information. One way is to ask the user which of these links is the correct one to follow.

## Trust and ambiguity

### URL based intrinsic trust

Any resource can specify any link metadata predicate for itself and for any subresource. These will be intrinsicly trusted. A subresource is defined as any resource who's URL can be written as a relative URL to the originating resource, without the need for `../`.

### Examples

A resource at `https://pod.example.org/foo/` can specify:

```turtle
</foo/bar> lm:redirectPermanent </bar/foo/>
```

or even:

```turtle
</foo> lm:redirectPermanent </bar/>
```

it cannot specify:

```turtle
</> lm:redirectPermanent </bar/>
```

or:

```turtle
<https://solidcommunity.net/foo> lm:redirectPermanent </bar/>
```

### User supplied explicit trust

A Solid user can specify that a specific resource is trusted. In that case this resource may specify link metadata predicates for any other resource, whether these are subresources or not.

How the user specifies such trust is out of scope for this specification.

### Server implementation details

If a server implements Auxiliary Resources of type "Description Resource" (colloquially known as `.meta` files), then the server must follow the URL based trust system. It must ignore any link metadata predicates that do not describe a subresource. 

It must also ignore link metadata predicates for the resource itself. For example, in `https://pod.example.org/` there is a resource named `foo`. If a user adds a `foo.meta` and in it adds a redirect predicate for the resource `https://pod.example.org/foo`, the server must ignore this.

Technically the server could handle this and send HTTP redirect headers whenever you request `foo`, but now there is no obvious way to alter the `.meta` information for `foo`. 

You could add the LINK header in the http redirect response, with the LINK header pointing to `foo.meta`, but the redirect URL may also have a `.meta` resource. Now it is unclear which information should be trusted, if there is a conflict.

It also makes it unnecessarily complex to create a user interface that allows you to access and edit the `foo.meta` file, since `foo` itself is redirected.

### Ambiguity

Whenever a resource has a link metadata predicate for a specific entity, any other predicates for that same entity should be ignored.

For example, say you delete a subject (entity) from a resource, and the application that you use, applies the `lm:deleted` predicate to it. This means that the subject no longer exists and any other predicates on it should also be ignored. Now if you 'undelete' the subject, all the application has to do is remove the `lm:deleted` predicate and all the information is back.

This should also work for redirect information. If you set a redirect (temporary or permanent) predicate on a subject, applications should only allow you to alter or remove those predicates. Any other predicates there are no longer valid and should no longer be used.

The reasoning here is that the redirect or deleted predicates explicitly invalidate the current subject URL. If this wasn't the case the exact information about the subject becomes ambiguous. Consider the following:

- If URL &lt;A> specifies &lt;A> `foaf:knows` &lt;B>, and 
- &lt;A> `lm:redirectTemporary` &lt;C>. 
- And URL &lt;C> specifies &lt;C> `foaf:knows` &lt;D>. 

Does this mean that &lt;A> knows &lt;B>,&lt;C>? Then you could no longer remove the `foaf:knows` &lt;B> information, there is no 'undo' option. 

So only by ignoring all other predicates linked to &lt;A> can we completely redefine the knowledge in &lt;C>.

