---
title: What is REST
date: 2019-07-07 18:52:15
tags:
- REST
- Web
categories: Web
---

# 1. Origin
REST this word is proposed by [Roy Fielding](https://en.wikipedia.org/wiki/Roy_Fielding) in his [paper](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm). He is also the originator of the HTTP specification.

This is what he said in his paper:
> My work is motivated by the desire to understand and evaluate the architectural design of network-based application software through principled use of architectural constraints, thereby obtaining the functional, performance, and social properties desired of an architecture.

# 2. Key Concepts
REST stands for Representational State Transfer. It's a architecural style or design pattern for APIs (it's not a protocol). It's a popular approach to building APIs because it emphasizes simplicity, extensibility, reliability, and performance.
<!--more-->
## Client
A client is the person or software who uses the API.

## Resource
A resource can be any object that API can provide information about. Any information that can be named can be a resource. Each resource has a URI. If you obtain one resource, you jut use its URI.

## Representation
The state of resource at any particular timestamp is known as resource representation. Representations are the way API clients see the resources. A representation consists of data, metadata describing the data and hypermedia links which can help the clients in transition to next desired state. A RESTful API never hands resources directly to a client. Interactions happen only via representations of the real resource. For example, you can store all users in a database table, but the representation of these users can be JSON or XML.

The data format of a representation is known as a media type. The media type identifies a specification that defines how a representation is to be processed. 

Resources are decoupled from their representation so that their content can be accessed in a variety of formats, such as HTML, XML, plain text, PDF, JPEG, JSON, and others.

## Actions
Clients don't have direct access to resources, so they need actions to alter the state of a resource. Roy Fielding has never mentioned any recommendation around which method to be used in which condition. All he emphasizes is that it should be uniform interface.

# 3. REST vs HTTP
It's important to mention that REST is not limited to HTTP. The author of REST and HTTP is same, thus REST and HTTP fits very well with each other.

In HTTP, URLs can be used to locate REST resources. Media Types spcifies how the data in requests and responses look like. HTTP built-in methods implement REST actions.

# 4. REST 6 Constrains

## Uniform interface
You MUST decide APIs interface for resources inside the system. A resource has to be represented by a URI. The response should provide a way to fetch related or additional data.

A single resource should not be too large and contain each and everything in its representation. Whenever relevant, a resource should contain links (HATEOAS) pointing to relative URIs to fetch related information.

>Once a developer becomes familiar with one of your API, he should be able to follow the similar approach for other APIs.

## Client - Server Spearation
The client and the server MUST be able to evolve separately without any dependency on each other. The client should know only resource URIs and that's all.

>Servers and clients may also be replaced and developed independently, as long as the interface between them is not altered.

## Stateless
The server doesn't remember anything about the user. The server treats each request as a new request. No session, no history. So the request needs to contains all the information the server needs to process the request - including authentication and authorization details.

>No client context shall be stored on the server between requests. The client is responsible for managing the state of the application.

## Layerd system
There maybe mutiple layers in the backend (security layer, caching layer, load-balancing layer, or other functionality). The client is agnostic to the layers. You deploy the APIs on server A, and store data on server B and authenticate requests in Server C

## Cacheable
The response should contain information about whether or not hte data is cacheable. These resources MUST declare themselves cacheable. Caching can be implemented on the server or client side.

>Well-managed caching partially or completely eliminates some client-server interactions, further improving scalability and performance.

## Code-on-demand
The response can return codes to client.

# HTTP status code
## 2xx - Success class
These status code are returned when the server succeeded in processing the request. 200 OK. 201 Created (ideally, contains a Location header). 204 No Content (no response body)

## 3xx - Redirection class
These are used when the server found the requested resource somewhere else. 301 Moved Permanently. The server SHOULD generate a Location header field in the response containing a preferred URI reference for the new permanent URI.

## 4xx - Client error class
The server retrieved a wrong request. 400 Bad Request, 401 Unauthorized, 403 Forbidden. 404 Not Found.

## 5xx - Server error class
Something wrong happens on the server.

