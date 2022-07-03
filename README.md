# REST-like API Design Tips from Experience

> We are all apprentices in a craft where no one ever becomes a master.



**A working guide of API design tips and trend evaluations.**

Over the years I've built many HTTP/web-based software APIs, many of which adopt common elements of the REST design pattern. This repo serves as a kind of "living" document, bringing together --into a single resource-- many concrete design considerations and architectural perspectives I've come to appreciate and actively implement in present and future projects.



#### Contributing

If you have any comments, questions, or suggestions, I encourage you to [open an issue](https://github.com/ptboyer/restful-api-design-tips/issues/new) or a [pull request](https://github.com/ptboyer/restful-api-design-tips/pulls)! After initially publishing this article many years ago ([Reddit](https://www.reddit.com/r/programming/comments/6edt2t/consistent_and_beautiful_restful_api_design_tips/di9smxn/), [Medium](https://medium.com/@peterboyer/learn-restful-api-design-ideals-c5ec915a430f)) many threads of discussion have been incredibly helpful in adjusting and tweaking my explanations and opinions on HTTP/API design to make this resource as robust and useful as possible.



- Interfaces
- Versioning
- CORS
- Methods
- Routes
- Responses



# Interfaces

In most cases:

- you will be creating an **Application Programming *Interface*** (API)
- with the intention connecting it to a **User *Interface*** (UI).



Almost all web-**applications** adopt an architecture with:

- a web-**server** (*back*-end, API) communicating with:
- a web-**client** (*front*-end, UI),

forming a "Full-Stack" Web Application.



**Interfaces provide structural and behavioural guarantees to interface consumers (clients).**

- When receiving a particular "request", a "server" is **expected** to respond with a particular "response".
- If this interface is broken/violated/dishonoured, **unexpected** (and potentially dangerous) behaviour occurs.



**Any consumer/client of any interface will become coupled (dependent) to that interface in some way.**

This means you must take great care in how you develop your own APIs for consumption by clients over its *entire* lifetime.



# REST

[REpresentational State Transfer](https://en.wikipedia.org/wiki/Representational_state_transfer) (REST) one of many architectural design patterns for internet applications.

A RESTful API is an API that adopts some key patterns and constraints of the REST architecture.

Many RESTful APIs that you'll come across on the internet only make use of [Methods](#methods) and [Routes](#routes) as means of interacting with the State of the web service.



# Versioning

When developing your API from now and into the future, not only must you consider **the *shapes* of the data** returned and received, you must also consider **the *behaviours* of the interfaces** you create to handle these requests.

- e.g. A "create blog post" endpoint may have **identical data inputs and outputs** between v1 and v2, however v2 introduces **validation behaviour** that requires a blog post MUST have a "title" whereas it used to be optional. -- The data shapes are the same but the behaviour is now a **breaking change** in which older consumers/clients may fail unexpectedly if they don't properly handle this new requirement. 



**Public**: If you make your API available to the **public** (consumers that *you do not control*), you are creating a *commitment* to your consumers that they will be able to request and receive data to/from your interface EXACTLY as you promise.

**Private**: Even if you make your API unavailable to the public, i.e. **private** (consumers that only *you do control*, i.e. your own frontend), you still have a commitment to your own clients and will want to minimise the effects of your API changes:

- e.g. Users with older/outdated versions of your downloadable mobile app (which is a consumer/client of your "private" API).
- e.g. References to your API in your website's code that would otherwise need to be updated in tandem with your server.



### Namespace Versioning

A **naive** approach to versioning is to simply prefix your web-based service API with a "version-namespace" like `/v1`.

```
GET foobar.com/api/v1/posts
```

If you're developing a *downloadable* software **package** then versioning your published releases is a great strategy (particularly with [Semantic Versioning](https://semver.org/) (SemVer)). However, a software **service** is *not downloadable* and only has one publicly accessible version: the version that is currently deployed.



Attempting to version a software service in this way introduces new problems to solve:

- **When do you increment** the version from v1 to v2 and beyond?
- **What counts as a major-enough update** to warrant this whole new namespace?
- **How long do we need to maintain the earlier/now-legacy namespaces** and its endpoints?



For example: if we have two endpoints, `/users` and `/posts`, and (unfortunately) decide we MUST add important but breaking security functionality to `/users`: do we now increment from `/v1` to `/v2`?

If yes, do we need to also maintain an insecure --but non-breaking-- `/v1` for older consumers? Surely not!

And even if we do end up on `/v2`, what if later we need to critically update the behaviour of the `/posts` endpoint, do we now increment from `/v2` to `/v3`?

If yes, then yikes: we just did 2 MAJOR version bumps in what could've been a single day!

**This isn't a sustainable way to develop, let alone maintain, an API**; prefixing everything with a `/v*` "version-namespace" creates more problems than it solves and increases maintenance stress and, most certainly, technical debt.





### Progressive Versioning

A robust approach is to accept that your API is an evolving, non-permanent service, in which:

- **we must carefully and sparingly add new interfaces and behaviours**,
  - i.e. consider new and potentially future properties and their names,
- **we must be disciplined in maintaining previously added interfaces and behaviours**,
  - i.e. write automated tests to ensure your system is functions how you expect it to (after development/refactoring),
- **we must only disrupt existing behaviours if we can't add a new interface or behaviour solve a problem**,
  - i.e. there is a serious security bug or exploit that affects your service,
- and **allow all interfaces the possibility of failing** -- such that new validation requirements and security considerations may be gracefully added and reflected into consumers' own expectations of those behaviours.



*These strategies and rules may be far more relaxed if your API is private*, because your own clients (e.g. UI/web-pages/etc.) are the only intended consumers of your API and you are not making any public guarantees/commitments to clients you don't control.



#### Carefully add new properties

Example: I want to add a property on an item that exposes its current "star" rating.

```diff
GET /items/dkajvh3k24jh
{
+  "rating": 3  		// simple/agnostic value
-  "rating": "3 Stars"  // unnecessarily formatted value
}
```

- Consider: Return the rating in it's most simple form so that the client can format it how it wants, otherwise the client may need to parse and reformat the value to use it.



#### Add properties rather than modifying existing ones

Example: I want to express ratings as a number instead of a pre-formatted string.

```diff
GET /items/dkajvh3k24jh
{
  "rating": "3 Stars",
+ "ratingRaw": 3
}
```

- Consider: Don't change `rating` to be a number, but instead consider adding a new property like `ratingRaw` so that any client relying on `rating` to be a formatted string will continue to work, and new clients can opt to use `ratingRaw` as a better alternative.



#### Disrupt existing behaviours only if we can't solve it any other way

Example: I accidentally have been returning a user's private email and password hash for `/users/:id` requests.

```diff
GET /users/joe.bloggs
{
  "given_names": "Joe",
  "family_name": "Bloggs",
- "email": "joe.bloggs@secretemail.com",
+ "email": "", 			// redact info, preserve the shape with blank
- "password": "******************************",
+ "password": "",       // redact info, preserve the shape with blank
}
```

- Consider: Even if there was a service or feature that relied on this unintentionally exposed information from the API to be able to work correctly, we can still do our best to maintain our public commitments of our API.



#### Allow for flexible and parse-able failure

Example: I have input validation rules for the given data.

```diff
POST /books { "title": "" }
{
  "error": {
-   "type": "TitleInvalid",
+	"type": "DataInvalid",
+   "meta": {
+	  "fields": { "title": ["must not be empty"] }
+   }
  }
}
```

- Consider: Rather than an endpoint that has a rigid set of error types, consider error types and responses that are more open to potentially new data and in a predicable/growable format.
- Consider: Even if your endpoint doesn't have any errors to begin with, allow for the possibility of some kind of "UnhandledError" to be returned to allow the client/consumer to be aware of the possible error interface.





# CORS

**Cross-Origin Resource Sharing (CORS) is a web-browser enforced security mechanism** to protect users from malicious website scripts.

Your API will probably be primarily accessed by via the web using a hostname like `foobar.com`.

If your API "backend" is served by a separate web service to how your "frontend" is served, you may consider one of two main strategies:

- (1) Serving your entire API from a route like `/api`: e.g. `foobar.com/api/...`
- (2) Serving your entire API from a sub-domain like `api.`: e.g. `api.foobar.com/...`



If your API is private and intended to only serve requests from your own website, like (1) you don't need to add a CORS HTTP--header to your server requests because your webpage and webserver are accessible via the same hostname: e.g. `foobar.com`.

That said, when developing your UI and API simultaneously, you'll probably need to [add a CORS middleware](https://enable-cors.org/) anyway because your web-page (e.g. `localhost:3000`) and web-server (e.g. `localhost:5000`) have different hostnames (due to the different ports!).







# Methods

Use HTTP methods such as:

- `GET` for fetching data.
- `POST` for adding data.
- `PUT` for updating data (as a whole object).
- `PATCH` for updating data (with partial information for the object).
- `DELETE` for deleting data.

I would like to add that I think `PATCH` is great way to cut down the size of requests to change parts of bigger objects, but also that it fits well with commonly implemented auto-submit/auto-save fields.

A nice example of this is with Tumblr’s "Dashboard Settings" screen, where non-critical options about the user experience of the service can be edited and saved, per item, without the need of a final form submission button. It is simply a much more organic way to interact with the user’s preference data.

![img](https://cdn-images-1.medium.com/max/800/1*DUQ4vVh0faIReP2Wqr9V_w.png)

*The "Saved" tag appears and then disappears shortly after modifying the option.*



# Routes

When building your routes you need to think of your endpoints as groups of resources from which you may read, add, edit and delete from and these actions are encapsulated as HTTP methods.



## Use Plurals

It makes semantic sense that you request many posts from `/posts`.

And for goodness sake don’t consider mixing `/post/all` with `/post/:id`!

Although some resource names can't be pluralised, you should try to get as close to plural as you can.

```js
// DO: plurals are consistent and make sense
GET /v1/posts/:id/attachments/:id/comments

// DON'T: is it just one comment? is it a form? etc.
GET /v1/post/:id/attachment/:id/comment
```



## Use Nesting for Relationship Filtering

*Query strings should be used for further filtering results beyond the initial grouping of a logical set offered by a relationship*.

Aim to design endpoint paths that avoid unnecessary query string parameters as they are generally harder to read and to work with when compared to paths whose structure promotes an initial relationship-based filtering and grouping of such items the deeper it goes.

> This `/posts/x/attachments` is better than `/attachments?postId=x`.
>
> And this `/posts/x/attachments/y/comments` is so much better than `/comments?postId=x&attachmentId=y`.



## Use More of your "Route-Space"

You should aim to **keep your API as flat as possible** and not crowd your resource paths. Allow yourself to provide root-level endpoints for all of your resources for when you eventually must update/delete them. For example, the case of a post having comments, allow `GET /posts/:id/comments` to fetch the comments for a post based on that relationship, but also offer `PATCH /comments/:id` to allow editing of comments without needing to track the id  for that post for every single route.

- **Longer paths for creating/fetching** nested resources by relationship
- **Shorter paths for updating/deleting** resources based on their `id`.



### Use the Authorisation context to Filter

When it comes to providing an endpoint to access to all of a user's own resources (e.g. all my own posts) you may end up with many ways to serve that information; it's up to you what best suits your application.

1. Nest a `/posts` relationship under `/me` with `GET /me/posts`, or
2. Use the existing `/posts` endpoint but filter with query string, `GET /posts?user=<id of self>`, or
3. Reuse `/posts`  to show only your own posts, and expose public posts with `GET /feed/posts`.



### Use a "Me" Endpoint

As hinted above, implement a `GET /me` endpoint to deliver basic data about the user as distinguished through the `Authorisation` header. This can include info about the user's permissions/scopes/groups/posts/sessions etc. that allow the client to show/hide elements and routes based on your permissions.

When it comes to providing endpoints for updating user preferences allow `PATCH /me` to change those intrinsic values.



## Cursor-based Pagination

Pagination is necessary as your dataset grows and fetching potentially thousands of items from a collection is incredibly expensive for both the server and the client. One of the best ways to do handle pagination is by using query parameters such as `from_<PROPERTY>` to indicate the beginning of your result set, with an optional `limit` parameter that defaults to something like `20` with a maximum of `50` (depending on your use case). In your responses' `meta` key you can provide information such as `total` count, and a uniquely generated `cursor` used for [cursor-based pagination](https://www.prisma.io/docs/concepts/components/prisma-client/pagination#cursor-based-pagination) (used by most applications, particularly [Google’s Places API](https://developers.google.com/places/web-service/search#PlaceSearchResults) and [Twitter](https://dev.twitter.com/ads/basics/pagination)).



# Responses

## Use Envelopes

It is good to maintain a clean response structure for your APIs in the event that you need to add metadata (e.g. pagination information) to your responses. An easy way to do this is to envelope your application response data under a `data` key, similar to GraphQL responses. From there you can freely add other keys, most likely an `errors` key in which to return any errors that are not to be conflated with correct `data`, and perhaps even a `meta` key, in which you may include pagination information like `total` and `cursor` which, once again, are not to be conflated with `data` items or indicative of any `errors`.

If you choose to build your endpoints to allow for partial submission success (in which your transaction will still operate on and return correct data, but will also accumulate errors from incorrect data to respond afterwards) you will be able to return both successful response data with `data` **AND** any problems with `errors`. In the case of HTTP status codes, [`207` seems the most appropriate](https://httpstatuses.com/207).

Separating your responses' data types into `data`, `errors`, `meta`, etc. helps remove validation headache when clients inspect your response, whereby you may easily check for the existence of an `errors` key in the response rather than using keys like `code` to differentiate an `ITEM` from an `ERROR`.

```js
// DO: enveloped
{
  data: ITEM or [ ITEM, ITEM, ... ],
  meta: { total: 1000, cursor: "0794405882026854" },
  errors: [
    { code: "MY_ERROR_CODE", extensions?: { ... } },
    ERROR,
    ERROR,
    ...
  ],
}

// DON'T non-enveloped
[
  ITEM,
  ITEM,
  // ...
]
```



## Responses and Requests

Most of the time you'll be communicating in JSON for both requests and responses.

If your API responses are in JSON, your server must include a `Content-Type: application/json` header expressing the MIME-type in use.

For all other response formats such as CSVs/images/etc. you must define and return those respective MIME-types in your headers.

Most modern frameworks and tools for APIs and clients-alike will automatically handle these headers for you.

### `Accept` header

This header is set by the client making a request to the server and is used to inform the server as to what format it wants in return.

If you're only supporting JSON responses (instead of something like XML), then your API should return a `415` Unsupported Media Type HTTP status code for the resource that is trying to be accessed if the API is unable to respond in the format specified by the `Accept` header.

I've seen some API's that use a `/json` or `/xml` suffix as part of a URL to access a resource, however this isn't exactly RESTful, and instead the `Accept` header should be used.



### Return the Updated Object

When updating any resource through a `PUT` or `PATCH` it’s good practice to return the updated resource in response to a successful `POST` , `PUT` , or `PATCH` request!



### Use 204 for Deletions

Support the `204 — No Content` response status code in cases where the request was successful but has no content to return. The envelope of the response, coupled with a `2XX` HTTP success code is enough to indicate a successful response without arbitrary "information".

```js
DELETE /v1/posts/:id
// response - 204
{
  "data": null
}
```



## Use HTTP Status Codes and Error Responses

Because we are using HTTP methods, we should use HTTP status codes. Although a challenge here is to select a distinct slice of these codes, and then depend on response data to detail any response errors. Keeping a small set of codes helps you consume and handle errors consistently.

I like to use:

### for Data Errors

- `400` for when the requested information is incomplete or malformed.
- `422` for when the requested information is okay, but invalid.
- `404` for when everything is okay, but the resource doesn’t exist.
- `409` for when a conflict of data exists, even with valid information.

### for Auth Errors

- `401` for when an access token isn’t provided, or is invalid.
- `403` for when an access token is valid, but requires more privileges.

### for Standard Statuses

- `200` for when everything is okay.
- `204` for when everything is okay, but there’s no content to return.
- `500` for when the server throws an error, completely unexpected.

Furthermore, returning responses after these errors is also very important. I want to consider not only the presentation of the status itself, but also a reason behind it.

In the case of trying to create a new account, imagine we provide an `email` and `password`. Of course we would like to have our client app prevent any requests with an invalid email, or password that is too short, but outsiders have as much access to the API as we do from our client app when it’s live.

- If the `email` field is missing, return a `400`.
- If the `password` field is too short, return a `422`.
- If the `email` field isn’t a valid email, return a `422`.
- If the `email` is already taken, return a `409`.

> "It’s much better to specify a more specific 4xx series code than just plain 400. I understand that you can put whatever you want in the response body to break down the error but codes are much easier to read at a glance." ([source](https://www.reddit.com/r/programming/comments/6edt2t/consistent_and_beautiful_restful_api_design_tips/diafp5l/))

Now from these cases, two errors returned `422`s regardless of their reasons being different. This is why we need an error code, and maybe even an error description. It’s important to make a distinction between code and description as I intend to have `code` as a machine consumable constant, and `message` as a human consumable string that may change.

In the case of per-field errors, the presence of the field as a key in the error is enough of a "code" to indicate that it is a target of a validation error.



## Field Validation Errors

For returning those per field errors, it may be returned as:

```js
POST /v1/register
// request
{
  "email": "end@@user.comx"
  "password": "abc"
}

// response - 422
{
  "errors": [{
    "code": "FIELDS_VALIDATION_ERROR",
    "extensions": {
      "email": "Invalid email address.",
      "password": "Password too short."
    }
  }]
}
```



## Operational Validation Errors

And for returning operational validation errors:

```js
POST /v1/register
// request
{
  "email": "end@user.com",
  "password": "password"
}
// response - 409
{
  "error": {
    "code": "EMAIL_ALREADY_EXISTS",
    "message": "An account already exists with this email."
  }
}
```

This way, your fetch logic watches out for non-200 errors, and can then straight-up check the `error` key from the response and then compare it to any further logic in the client app.

The `message` can act as a fallback human-readable error message to help understand the request when developing, and also in the case an appropriate localisation string implementation cannot be used.



# Authentication

Modern stateless, RESTful APIs implement authentication with tokens most commonly provided through the `Authorization` header (or even an `access_token` query param).



## Use Self-extending Session Tokens

Originally I thought that issuing JWTs for regular API requests was a great way to handle authentication—*until I wanted to invalidate those tokens*.

In my last revision of this post (and detailed [in a separate post](https://medium.com/studioarmix/expiring-jwts-with-refresh-tokens-cf54057fe727)) I offered a way for JWTs to be reissued through an additionally stored client secret "Refresh Token" (RT) which was to be exchanged for new JWTs. However in order to expire these JWTs they each contained a reference to the issuing RT so if the RT was invalided/deleted so would the JWT. However this mechanism defeats the statelessness of the JWT itself...

My solution now is to simply use a `/sessions` resource endpoint to exchange login credentials for a single unique session token (using `uuid4`) which is hashed and stored as a database row. Just like many moderns apps, the token doesn't need to be reissued unless there is a long period of inactivity (similar to session timeout, but to the scale of weeks). After initial authentication, every future request bumps the life of the token in a self-extending manner as long as it hasn't expired.



### Session Creation – Logging In

A normal login process would look like:

2. Receive email/password combination with `POST /sessions`, treating `sessions` as just another resource.
2. Check the email/password-hash against the database.
3. Create a new `session` database row that contains a hashed `uuid4()` as a `token`.
4. Return the non-hashed `token` string to the client.



### Session Renewal

In this flow tokens don't need to be explicitly renewed or reissued. That's because the API extends the life of the token if its still valid every request, saving regular users from ever having a session expire for them.

Whenever a token is received by the API i.e. through an `Authorization` header:

1. Receive the `token` i.e. from the `Authorization` header.
2. Compare against the `token`'s hash, if there is no matching `session` row, raise an authentication error.
3. Check the `updated_at` property of the `session`, if now if greater than `updated_at + session_life` the session is considered expired, delete the `session` row, raise an authentication error.
4. If it exists and is still valid from `updated_at` time, set the `updated_at` time to `now()` to renew the token.



### Session Management

Because all sessions are tracked as database rows mapped to a user, a user can see all their active sessions similar to Facebook's account security sessions view. You can also chose to include any associated metadata you have chosen to collected when initially creating a session such as the browser's User Agent, IP address etc. And

Fetching all your sessions is as simple as:

1. `GET /sessions` to return all sessions associated with your user via the `Authorization` header.



### Session Termination – Logging Out

And because you have handles to your sessions you can terminate them to invalidate unauthorised or unwanted access to your account. And logging out would simply be terminating the client's session and purging the session from the client.

1. Receive the `token` as part of a `DELETE /sessions/id` request.
2. Compare against the `token`'s hash, delete the matching session row.



## Avoid Password Composition Rules

After doing a lot of research into password rules, I’ve come to agree that [password rules are bullshit](https://blog.codinghorror.com/password-rules-are-bullshit/) and [are part of NIST’s "don’ts"](https://nakedsecurity.sophos.com/2016/08/18/nists-new-password-rules-what-you-need-to-know/), especially considering that **password composition rules help narrow down valid passwords** based on their validity rules.

I’ve collated some of the best points (from the above links) for password handling:

1. Only enforce a minimum *unicode* password length (min 8-10).
2. Check against common passwords ("password12345")
3. Check for basic entropy (don’t allow "aaaaaaaaaaaaa").
4. Don’t use password composing rules (at least one "!@#$%&").
5. Don’t use password hints (rhymes with "assword").
6. Don’t use knowledge-based authentication ("security" questions).
7. Don’t expire passwords without reason.
8. Don’t use SMS for two-factor authentication.
9. Use a password salt of 32-bits or more.

These "don’ts" should make password validation much easier!



# Meta

## Use a "Health-Check" Endpoint

Through developing with AWS, it been necessary to provide a way to output a simple response that can indicate that the API instance is alive and does not need to be restarted. It’s also useful for easily checking what version of the API is on any machine at any time, without authentication.

```js
GET /v1
// response - 200
{
  "status": "running",
  "version": "fdb1d5e"
}
```

I provide `status` and `version` (which refers to the git commit ref of the API at the time it was built). It’s also worth mentioning that this value is not derived from an active `.git` repo being bundled with the APIs container for EC2. Instead, it is read (and stored in memory) on initialisation from a `version.txt` file (which is generated from the build process), and defaults to `__UNKNOWN__` in case of a read error, or the file does not exist.
