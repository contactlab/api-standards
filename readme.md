# REST API standards and conventions

This document provides guidelines for [Contactlab](https://contactlab.com/en/) REST APIs, and represents our commitment to encourage consistency, maintainability, and best practices across applications. 

The rules are intended to enforce a uniform style in our APIs and, as a result, make the developers out there happier. However, this document is not meant to be a static resource. Instead, it will hopefully evolve over time, by gathering practical use cases and solutions.

On our side, the following standards and conventions are mandatory: feel free to use them as a guidance for your own development. 
If you have any problems or requests, please feel free to open an issue on this repo. To contribute, please make a pull request for review.


* [**API documentation**](#api-documentation)
* [**API security**](#api-security)
* [**REST API standards and conventions**](#rest-api-standards-and-conventions)
  * [API versioning and compatibility](#api-versioning-and-compatibility)
  * [Resource URIs](#resource-uris)
  * [Resource representation](#resource-representation)
  * [Collection resources: Pagination and sorting](#collection-resources-pagination-and-sorting)
  * [Resource: Reference expansion and partial representations](#resource-reference-expansion-and-partial-representations)
  * [HTTP methods](#http-methods)
  * [HTTP status codes](#http-status-codes)
  * [API error format](#api-error-format)
  * [API parameters](#api-parameters)
  * [URI resource nesting](#uri-resource-nesting)
  * [Caching](#caching)
  * [CORS](#cors)
  * [Date/time format and time zones](#datetime-format-and-time-zones)
  * [Correlation ID](#correlation-id)
  * [Hypermedia](#hypermedia)


## API documentation

We've chosen to document our APIs with [Swagger](http://swagger.io/) / [OpenAPI](https://www.openapis.org).

 * Every API must be documented using Swagger
 * The Swagger file (YAML or JSON):
   * Must be updated, reflecting the actual API implementation
   * Must be available on-line, along with the API
   * Must be versioned
   * Must be in English
   * Is the first 'prototype' of the API (currently, we're investing in quick ways of producing API mockups directly from the Swagger definition file)
   * Is the primary means of communication with all parties involved in the API design (PO, other team members, and similar)
   * Is a public document
 * Required Swagger file updates are an essential part of the definition of DONE

## API security

 * Public APIs must use our OAuth2 implementation (please see [OAuth2 service page](https://confluence.contactlab.com/display/arch/Oauth2), _authorized access only_)
 * Each API is responsible for authorization at the application level. The authorization must be enforced using OAuth2 scopes.

**Note:**

 * Today, OAuth2 scopes are used as roles. This will probably change in the near future. An analysis process is currently being undertaken.
 * Access to every API must be authenticated and authorized, excluding obvious public endpoints. A standard mechanism to properly handle authentication and authorization propagation between services, is currently being defined.

## REST API standards and conventions

### API versioning and compatibility

#### Versioning

 * The API version is declared in the URI path, for example, https://api.contactlab.it/hub/v1/...
 * The single application is not version aware: v2 is a different application to v1
 * The version in the path is the major one, in the context of semantic versioning. Obviously, compatibility must be preserved between minor versions.

#### Compatibility

Compatibility is preserved when:

 * A new property is added to a resource
 * A new resource is added to an API

Compatibility is broken when:

 * A property is removed from a resource
 * A resource is removed from an API
 * A property type or its semantic is changed
 * A URI structure is changed

Moving from one major version to another is an expensive process. We must design public APIs with multiple year life cycles in mind.

High level suggestions:

 * Think hard about the right level for any abstractions that you're designing
 * Use flexible data structures for extensible features, such as arrays or dictionaries

This document is the right place to collect useful techniques.

### Resource URIs

There is no such thing as RESTful Resource URIs, but we want to design proper and consistent URIs, to help external developers orientate themselves.

Please respect the following conventions:

 * Use a singular noun to refer to an instance resource: For example, http://myapi.com/configuration
 * Use a plural noun to refer to a collection resource: For example, http://myapi.com/cars
 * URIs must be lowercase
 * Use a dash as a word separator: For example, http://myapi.com/my-cars

### Resource representation

We've chosen to represent our resources exclusively in JSON format (media type: application/json).

Use camel case for property names. For example:

 * firstName is OK
 * first_name or FirstName are not OK.

Naturally, you can use other media-types when required, for example, when returning a rendered email in text/html.

Handle the content negotiation process using the proper headers *Accept* and *Content-Type*, for example, to refuse to serve unsupported content types.

### Collection resources: Pagination and sorting

#### Paging

A URI representing a collection resource that can be paginated should support the following query parameters:

 * ***size*** - An optional positive integer representing the page size. The maximum value should be documented, and a *Bad Request* error should be returned, when a bad value is passed.
 * ***page*** - An optional positive integer representing the page number. The first page is 0. If page is greater than or equal to the total pages, an empty collection is returned.

#### Sorting

A URI representing a collection resource that can be ordered, should support multiple instances of the query parameter ***sort***. The parameter value is the name of the property that you want to use to sort the results. Optionally, the sorting property can be followed by the sorting direction (**asc**, **desc**).

For example, https://myapi.com/books?sort=author,asc&sort=title,desc

Please describe the sorting options in the resource collection documentation.

#### Standard JSON representation of a page resource

The resource should contain the following objects and properties:

```yaml
CollectionPageResource:
  title: Resource collection page
  type: object
  properties:
    page:
      type: object
      properties:
        size:
          type: integer
          description: Size of the page
        totalElements:
          type: integer
          description: Total elements in the collection
        totalUnfilteredElements:
          type: integer
          description: Total elements in the unfiltered collection
        totalPages:
          type: integer
          description: Total pages of the collection
        number:
          type: integer
          description: Number of the current page
      required: [size, number]            
    elements:
      description: Elements contained in the page
      type: array
      items:
        type: object
  required: [elements]
```
For a non-paginated resource, use only the elements property.

### Resource: Reference expansion and partial representations

For specific resources, it can be useful to implement the patterns described below.

#### Reference resource expansion

When a resource ID is included in another resource, embedding its full representation can be useful, to avoid a second request.

For example:

GET https://myapi.com/books/1

```json
 {
  "title": "A book",
  "authorId": "550e8400-e29b-41d4-a716-446655440000"
 }
```

GET https://myapi.com/books/1?expand=author

```json
 {
  "title": "A book",
  "author": {
    "id": "550e8400-e29b-41d4-a716-446655440000",
    "name": "Joe The Programmer"
  }
 }
```

The **expand** parameter can be a comma-separated list of values.

#### Partial resource representation

When the full representation of a resource is too expensive in terms of payload/parsing, a partial representation of the resource can be requested.

Please use this pattern on specific resources and document it.

The following methods can also be used in collection resources.

##### Method 1: Style

The client can request a compact representation using the parameter **style**

For example:

GET https://api.co/users/123?style=compact

The only allowable value for the **style** parameter is **compact**.  Other styles will be allowed once real use cases are available.

##### Method 2: Fields

The client can filter resource properties using the parameter **fields**

For example:

GET https://api.co/users/123?fields=email,name

The parameter is a comma-separated list of strings

### HTTP methods

Remember that HTTP methods do not map to CRUD operations.

Use HTTP methods according to their semantics:

| Method  | Semantics                                               |
|---------|---------------------------------------------------------|
| GET     | safe, idempotent, cacheable                             |
| POST    | not safe, not idempotent                                |
| PATCH   | not safe, not idempotent - partial update of a resource |
| PUT     | not safe, idempotent - full update of a resource        |
| DELETE  | not safe, idempotent - resource deletion (may be soft)  |
| OPTIONS | safe - describe possible actions on the resource        |

#### Partial updates

For partial updates, the API must accept *PATCH* and a partial representation of the resource. The partial representation should be a JSON document, where properties which doesn't need to be updated are excluded.

Remember that *PUT* is idempotent and, as a result, the client must supply a full resource representation.

#### Method override

It must be possible to override every HTTP method using the *POST* method and the *X-HTTP-Method-Override* header.

### HTTP status codes

Use the appropriate HTTP status codes when answering requests:

| Type | Meaning           |
|------|-------------------|
| 1xx: | Hold on...        |
| 2xx: | Here you go!      |
| 3xx: | Go away!          |
| 4xx: | You messed up :-D |
| 5xx: | I messed up :-(   |

Reference: https://httpstatusdogs.com/

#### Some common cases

 * Use *201 Created* after a resource has been created
 * Use *204 No Content* in a case such as DELETE with no content
 * For Asynch requests, use 202 Accepted
 * 401 “Unauthorized” really means Unauthenticated
 * 403 “Forbidden” really means Unauthorized

### API error format

On errors, the API must return the following JSON object:

```yaml
ErrorResource:
  title: Error resource
  type: object
  properties:
    message:
      type: string
      description: Descriptive error message (English)
    logref:
      type: string
      description: Unique identifier that is reported in the application log
    data:
      type: object
      description: Specific error data
    errors:
      description: Sub-errors array
      type: array
      items:
        type: object
        properties:
          path:
            type: string
            description: JSON pointer to the invalid property
          message:
            type: string
            description: Descriptive error message (English)
          code:
            type: integer
            description: Custom error code related to the path
          data:
            type: object
            description: Specific error data
        required: [path, message]
  required: [message, logref]
```

### API parameters

Common sense rules apply to where to put API parameters:

| Where  | When                          |
|--------|-------------------------------|
| Path   | required, resource identifier |
| Query  | optional, query collections   |
| Body   | resource specific logic       |
| Header | global, platform-wide         |

### URI resource nesting

 * Avoid nested resource paths if relationships are not by composition
 * Provide nested paths when they are clear shortcuts for client navigation
 * Typically, a many-to-many hides a new resource (this can be a helpful extension point)

### Caching

 * Please use the correct header implementation for caching, when it is required for performance reasons
 * Both *If-None-Match/ETag* and *Modified-Since/Last-Modified* can be supported

### CORS

CORS should be enabled by default in every API service.

**Access-Control-Allow-Origin: * **

A wildcard same-origin policy is appropriate. Our APIs are intended to be accessible by everyone who has been authorized, including any code or site.

### Date/time format and time zones

 * Store and return time values in UTC
 * In JSON documents, represent timestamps using ISO 8601/RFC 3339 standards

### Correlation ID

Please make your API propagate or generate the **X-Tracing-ID** header in down-stream requests. It should contain a unique value (typically a UID) which can be used to track and troubleshoot issues in the call chains.

The following apps already support it:

  * OAuth Server
  * Configuration Service
  * Ruby Engine

### Hypermedia

We will not use hypermedia formats. Even if we recognize the value of a RESTful API, there are too many open questions that must be answered before we can embrace a specific format.

Open issues:

 * The concept of hypermedia is not widely spread throughout the developer community, and there is no way to enforce its correct use (for example, opaque URIs). As a result, all possible advantages would be lost, only leaving us with the burden of maintaining it.
 * There is no real standard or de-facto standards
 * Swagger is incompatible with a hypermedia format
 * Hypermedia has useful concepts for handling compatibility and the evolution of the API, but doesn't solve every issue (for example, a property removal)
 * Payloads tend to be heavier
