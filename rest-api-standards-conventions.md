# REST API standards and conventions

The following standards and conventions are mandatory. The purpose of these rules is to enforce a uniform style on our API's and make the developers out there happier.
However, this document is not meant to be a static resource, but it'll hopefully evolve in time gathering practical use cases and solutions.

## API's documentation

We've chosen to document our API's with [Swagger](http://swagger.io/).
 * Every API's must be documented using Swagger.
 * The Swagger file (yaml or json):
   * must be updated, reflecting the actual API implementation
   * must be available on-line along with the API
   * must be versioned
   * must be in English
   * is the first 'prototype' of the API (currently, we're investing quick ways to produce API mockups directly from the Swagger definition file)
   * is the primary communication mean with all the parties involved in the API design (PO, other team members, etc.)
   * is a public document
 * Required updates to the Swagger file are an essential part of the definition of DONE.

## API's security

 * Public API's must use our OAuth2 implementation (please refer to [OAuth2 service page](https://confluence.contactlab.com/display/arch/Oauth2))
 * Each API is responsible for authorization at application level. The authorization must be enforced using OAuth2 scopes.

**Notes**
 * Today OAuth2 scopes are used as roles. Probably, this will change in the near future. A process of analysis is currently active.
 * Access to every API's must be authenticated and authorized, excluding obvious public endpoints, but a standard mechanism to properly handle the authentication and authorization propagation between services is currently under definition.

## API's REST standards and conventions

### API's versioning and compatibility

#### Versioning

 * The API version is declared in the URI path, eg. https://api.contactlab.it/hub/v1/...
 * The single application is not version aware: v2 is a different application from v1.
 * The version in the path is the major version in the context of semantic versioning. Obviously, compatibility must be preserved between minor versions.

#### Compatibility

Compatibility is preserved when:

 * a new property is added to a resource
 * a new resource is added to an API

Compatibility is broken when:

 * an property is removed from a resource
 * a resource is removed from an API
 * the type or the semantic of a property is changed

Moving from a major version to another is an expensive process. We must design a public API's thinking of multi-years life cycles.

High level suggestions:

 * Think hard of the right level of abstractions you're designing
 * Use flexible data structures for extensible features, like arrays or dictionaries.

This document is the right place to collect useful techniques.

### Resource URI's

There are no thinks such RESTful Resource URI's, but we want design nice and consistent URI's to help external developers orienting themselves.

Please respect the following conventions:

 * Use a singular noun to refer to an instance resource: eg. http://myapi.com/configuration
 * Use a plural noun to refer to a collection resource: eg. http://myapi.com/cars
 * URI's must be lowercase
 * Use dash as word separator: eg. http://myapi.com/my-cars

### Resource representation

We've chosen to represent our resources exclusively in JSON format (media type: application/json).

Do use camel casing for property names. Eg.
 * firstName is OK
 * first_name or FirstName are not OK.

Naturally, you can use other media-types when is required - eg. when returning a rendered email in text/html.

Handle the process of content negotiation using the proper headers *Accept* and *Content-Type* - eg. refusing to serve not supported content types.

### Collection resources: pagination and sorting

#### Paging

A URI representing a collection resource that can be paginated should support the following query parameters:

 * ***size***  - An optional positive integer representing the page size. The max value should be documented and a *Bad Request* error should be returned when a bad value is passed.
 * ***page*** - An optional positive integer representing the page number. The first page is 0. If page is greater or equals to the total pages, an empty collection is returned.

#### Sorting

A URI representing a collection resource that can be ordered should support the optional query parameter ***sort***. The parameter is a comma-separated list of values containing the name of the sorting field, optionally followed by the plus sign and the sorting direction (**asc**, **desc**). Eg. https://myapi.com/books?sort=title+desc,author+asc

#### Standard JSON representation of a page resource

The resource should contains the following objects and properties:

```yaml
page:
  type: object
  properties:
    size:
      type: integer
      description: Size of the page
    totalElements:
      type: integer
      description: Total elements in the collection
    totalPages:
      type: integer
      description: Total pages of the collection
    number:
      type: integer
      description: Number of the current page
elements:
  description: Elements contained in the page
  type: array
```

### Resource: reference expansion and partial representations

On specific resources, it can be helpful to implement the patterns described below.

#### Reference resource expansion

When a resource id is included in another resource, embedding the full representation of this resource can be useful in order to avoid a second request.

Eg.

GET https://myapi.com/books/1

```json
 {
  "title": "A book",
  "authorId": "jd298232h2388jd"
 }
```

GET https://myapi.com/books/1?expand=author

```json
 {
  "title": "A book",
  "author": {
    "id": "jd298232h2388jd",
    "name": "Joe The Programmer"
  }
 }
```

The parameter **expand** can be a comma-separated list of values.

#### Partial resource representation

When the full representation of a resource is too expensive in terms of payload/parsing, an alternate compact representation can be served. The client can request the alternate representation using the parameter **style**

Eg.

GET https://api.co/users/123?style=compact

The parameter can be used in collection resources as well.

The only allowed value for the parameter **style** is **compact**.  Other styles will be allowed after real use cases will be presented.

### HTTP methods

Remember that HTTP methods doesn't map to CRUD operations.

Use HTTP methods according to their semantics:

| Method  | Semantics                                               |
|---------|---------------------------------------------------------|
| GET     | safe, idempotent, cacheable                             |
| POST    | not safe, not idempotent                                |
| PATCH   | not safe, not idempotent - partial update of a resource |
| PUT     | not safe, idempotent - full update of a resource        |
| DELETE  | not safe, idempotent - resource deletion (may be soft)  |
| OPTIONS | safe - describe possible actions on the resource        |

#### Method override

Using the POST method and the header *X-HTTP-Method-Override* must be possible to override every HTTP method.  

#### Partial updates

For partial updates the API must accept *PATCH* and a partial representation of the resource. The partial representation is a JSON document without properties that don't need to be updated.

Remember that *PUT* is idempotent and hence the client must supply a full resource representation.

### HTTP status codes

Use appropriate HTTP status codes when answering requests:

| Type | Meaning           |
|------|-------------------|
| 1xx: | Hold on...        |
| 2xx: | Here you go!      |
| 3xx: | Go away!          |
| 4xx: | You messed up :-D |
| 5xx: | I messed up :-(   |

#### Some common cases
 * Use *201 Created* after a resource has been created
 * Use *204 No Content* in case like DELETE with no content
 * For Asynch requests use 202 Accepted
 * 401 “Unauthorized” really means Unauthenticated
 * 403 “Forbidden” really means Unauthorized

### API error format

On errors, the API must return a JSON representation of the error using a non HAL version of the media type [application/vnd.error+json](https://github.com/blongden/vnd.error)

TODO: provide JSON example/schema

```json
{
  "message": "Validation failed",
  "path": "/username",
  "logref": 42,
}
```

### API parameters

Common sense rules on where to put API parameters:

| Where  | When                          |
|--------|-------------------------------|
| Path   | required, resource identifier |
| Query  | optional, query collections   |
| Body   | resource specific logic       |
| Header | global, platform-wide         |

### URI resource nesting

 * Avoid nested resource paths if relationships are not by composition.
 * Provide nested paths when they are clear shortcuts for the client navigation.
 * Typically a many-to-many hides a new resource (this can be an helpful extension point).

### Caching

 * Please, handle the correct header implementation for caching when is required for performance reasons.
 * Both *If-None-Match/ETag* and *Modified-Since/Last-Modified* can be supported.

### CORS

CORS should be enabled by default in every API service.

**Access-Control-Allow-Origin: * **

A wildcard same-origin policy is appropriate. Our API's are intended to be accessible from everyone who has been authorized, including any code or site.

### Date/time format and time zones

 * Store and return time values in UTC
 * In JSON documents represent timestamps with ISO 8601/RFC 3339 standards

### Hypermedia

We won't use hypermedia formats. Even if we recognize the value of a RESTful API, there are too many open questions that must be answered before we can embrace a specific format.

Open issues:

 * The concept of hypermedia is not spread in the developer community and there's no way to enforce its correct use (eg. opaque URI's). This way, all the possible advantages would be lost, leaving us only with the burden of maintain it.
 * There's no a real standard or de-facto standards.
 * Swagger is incompatible with an hypermedia format.
 * Hypermedia has useful concepts for handling compatibility and evolution of the API, but doesn't solve every issue (eg. a property removal).
 * Payloads tend to be heavier.
