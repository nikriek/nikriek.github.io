---
layout: post
title:  "Sign requests using an NGINX reverse proxy"
date:   2018-07-22 12:27:46 +0200
tags: tech
summary: "NGINX is a fantastic tool for various use cases such as Load Balancing, proxying etc. Sometimes, complex use cases require custom configurations. In our case, NGINX should act a reverse proxy in a private network and sign incoming requests from a trusted source."
---

NGINX is a fantastic tool for various use cases such as Load Balancing, proxying etc. Sometimes, they require custom configurations. In our case, NGINX should act a reverse proxy in a private network and 
sign incoming requests from a trusted source. The upstream API has HMAC based authentication provided by the [Rails API Auth Gem](https://github.com/mgomes/api_auth).
It's heavily inspired by AWS V4 signature based authentication.

## First idea
The OpenResty extension for NGINX provides various tools to enhance NGINX such as GeoIP, Lua script support and cryptographics functions. 
In my first attempt, I was trying to set the authorization header using the provided functions such as `set_md5` and `set_hmac_sha1`. However, due to NGINX' request processing phases, calculating the content MD5 
was not directly possible. At the rewrite phase, the `$request_body` variable is still empty. For every input, it always returned the same hash of an empty string. Bummer.

## Custom signature using Lua 
As mentioned above, OpenResty also provides support for the Lua JIT. This allows us to use custom Lua code with access to different NGINX variables and functions. The `access_by_lua_file` configures the execution of custom code in a later NGINX phase.
```
set_by_lua $now "return ngx.http_time(ngx.time())";
set $authorization_header '';
set $content_md5_header '';
access_by_lua_file '/usr/local/authorization.lua'
```

In this phase, the request body can be read and used for proper signature calculation. In the end, the different NGINX vars can be directly set from the Lua code.

```lua
local resty_md5 = require "resty.md5"
local str = require "resty.string"

-- Setup variables
local path = ngx.var.uri
local method = ngx.var.request_method
local content_type = ngx.var.content_type
local access_id = ngx.var.access_id
local secret_key = ngx.var.secret_key
local now = ngx.var.now
ngx.req.read_body()
local body = ngx.req.get_body_data()
if body == nil then body = '' end

-- Calculate content signature
local md5 = resty_md5:new()
md5:update(body)
local digest = md5:final()
local content_md5 = ngx.encode_base64(digest)

-- Calculate request authorization header
local canonical_values = {
    method,
    content_type,
    content_md5,
    path,
    now
}
local canonical = table.concat(canonical_values, ',')
local signature = ngx.encode_base64(ngx.hmac_sha1(secret_key, canonical))

-- Set proxy headers
ngx.var.authorization_header = "APIAuth-HMAC-SHA1 " .. access_id .. ':' .. signature;
ngx.var.content_md5_header = content_md5;
```
