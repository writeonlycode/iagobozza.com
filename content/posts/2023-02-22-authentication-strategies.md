---
title: "Authentication Strategies"
author: "Iago Bozza"
categories: ["Backend", "API", "Authentication"]
date: "2023-02-22"
summary: >
  (...)
draft: true
---

Authentication is the process of determining and verifying the identity of a
client: when the server receives a request from a client, an authentication
strategy allow the server to determine whether the client is a known user.

<!-- Authentication is not to be confused with authorization: authentication is -->
<!-- about determining the identity of the client, while authorization is about -->
<!-- determining whether the client has the permission to access a certain resource. -->

A few common authentication strategies are the following:

- **Basic Authentication:** The client sends a request to the server. The
  server receiver the request and checks for the presence of the
  `Authorization` header. If the header is present, it checks the credentials
  and, if the credentials are valid, responds with the appropriate data.
  Otherwise, the server responds with a `401` with the `WWW-Authenticate`
  header set to `Basic`. When the client's browser receives this response, it
  show a sign-in pop up, so the user can enter their credentials, and the
  browser can send another request with the `Authenticate` header set with the
  given credentials.

Basic Authentication is mainly a front-end convenience: We don't have to create
a login screen and handle the submission of the credentials. Rather, the
browser automatically shows a sign in pop up, and sends the request with the
appropriate headers set with the given credentials. At the back-end, we still
have to retrieve the credentials from the request and check whether they are
valid.

- **Session Based Authentication:** The client initially sends their
  credentials to the server; the server creates, saves, and sends back an
  authentication token; the client stores the authentication token somehow
  (typically in a session or cookie), and sends it with each subsequent request
  to the server.

Session based authentication requires a server and a database, as well as a
browser that can handle sessions and cookies.

- **Token Based Authentication:** The client initially sends their credentials
  to the server; the server sends back an authentication token that is
  self-contained (i.e. it has includes all the information it needs to be
  validated); the client stores the authentication token somehow, and sends it
  with each subsequent request to the server.

The main difference between session based and token based authentication is
that session based authentication requires the server to store the
authentication token somewhere, while token based authentication doesn't.

So, token based authentication still requires a server, but no database and no
browsers that can handle sessions and cookies.

The specific ways to implement token based authentication are:

- SWT

- JWT Authentication: An specific way to implement token based authentication
  using JWT.

- OAuth

- SAML

- OpenID

Another concept is SSO:

- **SSO (Single Sign On):** The client initially sends their credentials to an
  special server; the server verifies the identity of the client using some
  strategy, and redirects the client to another service.

