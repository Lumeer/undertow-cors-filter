# undertow-cors-filter
**A filter to enable correct handling of CORS headers in undertow-based servers (Wildfly, JBOSS EAP)**

There seems to be an oversight in Java EE's filter handling, because when the container is configured with
container managed authorization and a user that is not (yet) authenticated attempts to access a protected
resource, the container intercepts that request and sends a 401 response. That response does not have CORS
headers, but for some reason cannot be filtered. Neither with a Jax-Rs ContainerResponseFilter, not with a
plain servlet filter. A container-specific solution seems to be the only way to get the job done.

This project provides that solution for JBoss containers based on Undertow.

> Obviously this filter is container specific. It should work in containers based on Undertow.
> This includes Wildfly 8/9/10, JBoss AS, JBoss EAP and Wildfly Swarm. Tested on Wildfly 10.1.0.

## Download
The module zip file can be downloaded directly from Maven Central: **[undertow-cors-filter-0.4.0-bin.zip](https://repo1.maven.org/maven2/com/stijndewitt/undertow/cors/undertow-cors-filter/0.4.0/undertow-cors-filter-0.4.0-bin.zip)**.

## Installation
To use this filter, install it as a module in WildFly / EAP.
Grab the module zip file from Maven Central and unzip it in the root of your JBoss installation folder.

If everything works as planned, it will result in a folder `modules/com/stijndewitt/undertow/cors/main`
in your WildFly / EAP installation folder containing a JAR file `undertow-cors-filter-0.4.0.jar` and a
`module.xml` file with this content:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<module xmlns="urn:jboss:module:1.0" name="com.stijndewitt.undertow.cors">
  <resources>
    <resource-root path="undertow-cors-filter-0.4.0.jar"/>
  </resources>
  <dependencies>
    <module name="io.undertow.core"/>
  </dependencies>
</module>
```

## Configuration
To make the installed filter available inside the container, add a `filter` to the `filters` section of `standalone.xml`:

```xml
<filters>
  <filter name="undertow-cors-filter" class-name="com.stijndewitt.undertow.cors.Filter" module="com.stijndewitt.undertow.cors">

  </filter>
</filters>
```

Then, add a `filter-ref` to the `host` element (still in `standalone.xml`):

```xml
<host name="default-host" alias="localhost">
  <filter-ref name="undertow-cors-filter" />
</host>
```

You can modify the default behavior of the filter by adding `<param>` elements to the `<filter>` configuration.

### A complete example configuration
For your copy-paste convenience, here is a complete filter configuration that has all possible parameters.
Remove the comments for those parameters that you need.

```xml
<filters>
  <filter name="undertow-cors-filter" class-name="com.stijndewitt.undertow.cors.Filter" module="com.stijndewitt.undertow.cors">
    <!-- which requests should be filtered? defaults to "^.*$", matching all requests -->
    <!-- param name="urlPattern" value="^http(s)?://([^/]+)(:([^/]+))?(/([^/])+)?/api(/.*)?$" /-->

    <!-- which policy should be used? defaults to AllowAll -->
    <!-- param name="policyClass"       value="com.stijndewitt.undertow.cors.AllowAll" /-->
    <!-- alternative policy: AllowMatching -->
    <!-- param name="policyClass"      value="com.stijndewitt.undertow.cors.AllowMatching" /-->
    <!-- param name="policyParam"      value="^http(s)?://(www\.)?example\.(com|org)$" /-->
    <!-- alternative policy: Whitelist -->
    <!-- param name="policyClass"      value="com.stijndewitt.undertow.cors.Whitelist" /-->
    <!-- param name="policyParam"      value="${jboss.server.data.dir}/whitelist.txt" /-->

    <!-- which CORS headers should be set? values listed are the defaults -->
    <!-- param name="exposeHeaders"    value="Accept-Ranges,Content-Length,Content-Range,ETag,Link,Server,X-Total-Count" /-->
    <!-- param name="maxAge"           value="864000" /-->
    <!-- param name="allowCredentials" value="true" /-->
    <!-- param name="allowMethods"     value="DELETE,GET,HEAD,OPTIONS,PATCH,POST,PUT" /-->
    <!-- param name="allowHeaders"     value="Authorization,Content-Type,Link,X-Total-Count,Range" /-->
  </filter>
</filters>

```

### urlPattern
The url pattern is matched agains the full URL of the request. Requests that do not match are not filtered.
If this parameter is not set, the filter will apply for all requests.

Make sure to configure the `urlPattern` to match those URLs that the filter should be applied to. In the example above,
it is configured to match any URLs starting with `"http://"` or `"https://"`, followed by a host and then optionally a port,
followed by an optional context path and then the path segment `"/api"`, optionally followed by a slash and more stuff.
The regex is a little complicated but if you have your API mounted in the root of your webapp, then replacing `"api"` in
the example with whatever name your API is mounted under will give you a regex that works for all hosts and all ports under
http and https, whether your application is running under a context path or directly under the root. Modify the regex
as you see fit.

### policyClass
The parameter `policyClass` can be used to select one of the available policies for determining whether a certain origin should
get CORS headers added. The default policy if no `policyClass` is given is `AllowAll`, which does what it's name implies.
This configuration is therefore effectively the same as the snippet we saw before:

```xml
<filters>
  <filter name="undertow-cors-filter" class-name="com.stijndewitt.undertow.cors.Filter" module="com.stijndewitt.undertow.cors">
    <param name="policyClass" value="com.stijndewitt.undertow.cors.AllowAll" />
  </filter>
</filters>
```

See the section named [Policies](https://github.com/download/undertow-cors-filter#policies) for more information.

### policyParam
A single string parameter which is used to pass configuration info to the selected policy. Ignored for `AllowAll`, but used by
the other available policies to configure the policy. For example `AllowMatching` is configured with a regex and only origins
matching the regex will be allowed:

```xml
<filters>
  <filter name="undertow-cors-filter" class-name="com.stijndewitt.undertow.cors.Filter" module="com.stijndewitt.undertow.cors">
    <param name="policyClass" value="com.stijndewitt.undertow.cors.AllowMatching" />
    <param name="policyParam" value="^http(s)?://(www\.)?example\.(com|org)$" />
  </filter>
</filters>
```

This configuration will add CORS headers to any requests from the following origins:

* http://example.com
* https://example.com
* http://www.example.com
* https://www.example.com
* http://example.org
* https://example.org
* http://www.example.org
* https://www.example.org

### allowCredentials

This configuration parameter allows you to set the value of the [Access-Control-Allow-Credentials](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS#Access-Control-Allow-Credentials) header.

### allowHeaders
This configuration parameter allows you to set the value of the [Access-Control-Allow-Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS#Access-Control-Allow-Headers) header.

### allowMethods
This configuration parameter allows you to set the value of the [Access-Control-Allow-Methods](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS#Access-Control-Allow-Methods) header.

### exposeHeaders
This configuration parameter allows you to set the value of the [Access-Control-Expose-Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS#Access-Control-Expose-Headers) header.

### maxAge
This configuration parameter allows you to set the value of the [Access-Control-Max-Age](https://developer.mozilla.org/en-US/docs/Web/HTTP/Access_control_CORS#Access-Control-Max-Age) header.

## Policies
The following policies are available out of the box. If you need something different, you can also
write [custom policies](https://github.com/Download/undertow-cors-filter#custom-policies).

### AllowAll
The simplest policy just adds CORS headers to all origins. Suitable if your API server already has strong authorization using e.g.
bearer token. CORS mainly protects agains cross site scripting attacks where a foreign website does an AJAX request to your webserver,
using the fact that the browser will add the user's session cookie to the request behind the scenes, allowing it to perform actions
under the user's account. If your API is not protected with cookies but with some other mechanism, CORS isn't all that helpful anyway.

To use this policy, either don't specify the `policyClass` at all, or set it's value to `com.stijndewitt.undertow.cors.AllowAll`

### AllowMatching
This policy performs a regex match on the origin to determine whether it is allowed access. Only origins matching the regex are
allowed. The `policyParam` is used to set the regex.

To use this policy set the value of `policyClass` to `com.stijndewitt.undertow.cors.AllowMatching` and set the value of
`policyParam` to the regular expression to match the origin against.

### Whitelist
This policy performs a series of regex matches on the origin to determine whether it is allowed access. Only origins matching one of the
regexes specified in the whitelist are allowed. The `policyParam` is used to set the absolute file path of the whitelist file.

This policy is the most advanced policy available at the moment. It can be useful if you need to whitelist a large amount of domains
that would make for a very complex regex otherwise, or if your list of domains changes during runtime.

The whitelist file should be a file of type `text/plain` encoded as Unicode UTF/8 without a byte order mark (BOM).
The file should contain a single regex per line. Regexes will be tried in turn and as soon as a match is found the origin will
be allowed access. If no match can be made the origin is denied. The file format allows empty lines and comment lines starting with
a hash (`#`) character or double slash (`//`) characters. For example a whitelist file might look like this:

```
# this is a comment
// this is also a comment

// blank lines are ignored
^http(s)?://(www\.)?example\.(com|org)$
^http://example\.net$
```

A watcher is set up that watches the file for changes, so you could write a new version of the file during the app's operation
and the changes should be picked up within a few seconds. This feature depends on the WatchService API which hooks into OS file
change notification events. Results may vary based on the OS support.

To use this policy set the value of `policyClass` to `com.stijndewitt.undertow.cors.Whitelist` and set the value of
`policyParam` to the absolute file path of the whitelist file containing the regular expressions to match the origin against.

### Custom policies

You should be able to write custom policies, package them as a JAR and install them in a JBoss Module just like this filter itself.
Add a dependency to the module in this module's `module.xml` and the class should become available for use. Have a look at the
policies available in this repo for inspiration.

#### Share your Policy
Wrote a great Policy class? Share it back! Create a PR on this project that adds your Policy. Don't forget to add
it to this README as well!

## Credits
I asked a [question on StackOverflow](https://stackoverflow.com/questions/35217936/add-cors-headers-to-response-of-j-security-check)
about some requests not being intercepted when using a ServletFilter or a JAX-RS filter and the answer I got there from user
[nfedorov](https://stackoverflow.com/users/6758880/nfedorov) lead me to create this project. So thanks again nfedorov!

## Issues
Add an issue in this project's [issue tracker](https://github.com/download/undertow-cors-filter/issues)
to let me know of any problems you find, or questions you may have.

## Copyright
Copyright 2018 by [Stijn de Witt](http://StijnDeWitt.com). Some rights reserved.

## License
Licensed under the [Creative Commons Attribution 4.0 International (CC-BY-4.0)](https://creativecommons.org/licenses/by/4.0/) Open Source license.
