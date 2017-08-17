# API Gateway for PCF

This is a very simple setup for a reverse proxy / API gateway using nginx that is easy to configure and deploy into a PCF environment using the [Staticfile Buildpack](https://docs.cloudfoundry.org/buildpacks/staticfile/).

It assumes all HTTPS all the time for inbound connections. The HTTPS may be decrypted by the PCF router and passed through as HTTP to the gateway, but the `nginx.conf` assumes responses should be HTTPS and includes an HSTS header.

## Deployment

`cf push api-gateway-name`

## Adding a Microservice

In `nginx.conf` copy one of the existing `location` nodes, update the `location` regex to match your endpoint, and deploy.

The initial configuration shows a route to a one service that hosts two resources - `/api/accounts` and `/api/users`. If the request doesn't match one of those, there's a "fallback host" that handles any other API requests that don't match other microservices.

The values in `proxy_pass` should be the root URL to the server hosting the API with no trailing slash or path info. If you need to modify path info, put it in the `rewrite` directive.

## Swaggregator Integration

The `/swagger/` route in the reverse proxy needs to point at a deployed [Swaggregator](https://github.com/tillig/swaggregator). It is currently hardcoded to a URL. If multiple environments need to use this reverse proxy a future enhancement might be to use `VCAP_SERVICES` or environment variables to attach Swaggregator.

## Location Matching

The `location` matching syntax [is sort of hard to find](http://nginx.org/en/docs/http/ngx_http_core_module.html#location) other than in [tutorials](https://www.digitalocean.com/community/tutorials/understanding-nginx-server-and-location-block-selection-algorithms), so here it is:

```
location optional_modifier location_match {
    ...
}
```

`optional_modifier` can be one of these things:
- Empty: If no modifiers are present, the location is interpreted as a prefix match. This means that the location given will be matched against the beginning of the request URI to determine a match.
- `=`: If an equal sign is used, this block will be considered a match if the request URI exactly matches the location given.
- `~`: If a tilde modifier is present, this location will be interpreted as a case-sensitive regular expression match.
- `~*`: If a tilde and asterisk modifier is used, the location block will be interpreted as a case-insensitive regular expression match.
- `^~`: If a carat and tilde modifier is present, and if this block is selected as the best non-regular expression match, regular expression matching will not take place.

## Tips and Notes

- **Avoid using `^~` for microservice location matching.** As soon as you put a `^~` location match, if it even _remotely_ matches location, it'll stop trying to do any of the other regex matches. This causes some requests to go to the wrong location. Use `~*` for non-case-sensitive regex matches instead.
- **Debug using outbound headers.** You may not be able to use `echo` in your PCF environment. If you don't have that module you can add headers with debug messages to outbound requests. `add_header X-Debug-Message "my debug message" always;`
- **Watch for missing semicolons.** This'll get you every time.
- **Rewrite logging requires error logging at `notice` level.** The `debug` level works, too. Higher than that and you won't get rewrite logs.
- **Regex location matches need `rewrite` for `proxy_pass`.** If you match your location on a regex, you need to use a `rewrite` directive to "solidify" the URL. Add a `break` directive at the end to ensure you don't get an infinite redirect loop. Without the `rewrite` you'll get an `nginx.conf` error at app startup. Also, make sure the `proxy_pass` has no trailing slash or you'll get an error.

## References

- [Staticfile Buildpack](https://docs.cloudfoundry.org/buildpacks/staticfile/)
- [nginx http Core Module](http://nginx.org/en/docs/http/ngx_http_core_module.html): Includes directives found in the `http` element of `nginx.conf` including things that go in the `server` element.
- [nginx http Proxy Module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html): Includes directives like `proxy_pass` and things valuable in reverse proxy setup.
- [nginx http Rewrite Module](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html): Explains how `rewrite` works to use regular expressions to rewrite URLs. You can do this before doing a `proxy_pass` to manipulate the inbound URL.
- [nginx Full Example Configuration](https://www.nginx.com/resources/wiki/start/topics/examples/full/)
- [nginx Variable List](http://nginx.org/en/docs/varindex.html): The set of variables you can use in rewrites.
