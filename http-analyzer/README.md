# HTTP request analyzer part of WAF for WordPress

Examines HTTP headers in all (including login) requests.
Bans commonly used user names (hardcoded in a property),
blocks non-static requests from CDN, prevents author sniffing.

Then triggers Fail2ban accordingly.

Executes typically under 0.1 millisecond.

### List of HTTP request parts checked

- Custom CDN headers `*`
- Request URI length (2500 bytes)
- User Agent length (472 bytes)
- HTTP methods
- Two forward slashes in URI
- URI encoding
- URI blacklist
- HTTP protocol
- Request for non-existent PHP file
- Request for robots.txt in a subdirectory
- Request with `author` query field (author sniffing)
- PHP and Shockwave Flash file upload
- HTTP/POST without User Agent
- Accept header
- Content-Length header
- Content-Type header
- Accept-Language header
- Referer header `*`
- Request size for logins `*`
- Login username blacklist (`log` POST variable) `*`
- Accept-Encoding header
- IE8 and modern browser (Mozilla/5.0) login
- Test cookie (`wordpress_test_cookie`) `*`
- Connection header `*`
- Login from Tor exit nodes `*`
- Many more, altogether 34 checks

The list is in order of appearance, `*` means it can be disabled by a constant below.

```bash
grep -o "return '.*';" waf4wordpress-http-analyzer.php
```

### Usage

To install copy `waf4wordpress-http-analyzer.php`
beside your `wp-config.php` and copy these two lines in top of `wp-config.php`:

```php
require_once __DIR__ . '/waf4wordpress-http-analyzer.php';
new \Waf4WordPress\Http_Analyzer();
```

It [executes very early](https://wordpress.org/plugins/whats-running/) when `require`-d from `wp-config.php`.
In a worse case install it as an mu-plugin.
Mu-plugins are loaded by WordPress before it executes.

You can find out normal plugin execution order by this simple WP-CLI command:

```
wp option get "active_plugins"
```

### Options

You can set these options by defining constants in your `wp-config.php`.
E.g. to allow Connection header other than `keep-alive` use this:

```php
define( 'W4WP_ALLOW_CONNECTION_CLOSE', true );
```

To blocks non-static requests from Amazon CloudFront copy this to your `wp-config.php`:

```php
define( 'W4WP_CDN_HEADERS', 'HTTP_X_FORWARDED_FOR:HTTP_X_AMZ_CF_ID:HTTP_VIA' );
```

To blocks non-static requests from CloudFlare copy this to your `wp-config.php`:

```php
define( 'W4WP_CDN_HEADERS', 'HTTP_X_FORWARDED_FOR:HTTP_CF_RAY:HTTP_CF_CONNECTING_IP' );
// mod_remoteip removes HTTP_CF_CONNECTING_IP
define( 'W4WP_CDN_HEADERS', 'HTTP_X_FORWARDED_FOR:HTTP_CF_RAY:HTTP_CF_VISITOR' );
```

Restrict access to CloudFlare only: `mod_remoteip`

- `HTTP_CF_CONNECTING_IP` - mod_remoteip removes this
- `HTTP_X_FORWARDED_FOR`
- `HTTP_CF_RAY`
- `HTTP_CF_VISITOR`

Constant list

- (boolean) `W4WP_POST_LOGGING` enable logging of all POST requests, even normal ones
- (integer) `W4WP_COUNT` fail2ban trigger limit, `maxretry`
- (integer) `W4WP_MAX_LOGIN_REQUEST_SIZE` maxumim size of the login request
- (string) `W4WP_CDN_HEADERS` a colon separated list of HTTP headers your CDN is recognized by
- (boolean) `W4WP_ALLOW_REG` allow WP registration, disabled referer and test cookie checks
- (boolean) `W4WP_ALLOW_IE8` allow login with IE8 too (IE8 is not a `Mozilla/5.0` browser)
- (boolean) `W4WP_ALLOW_OLD_PROXIES` allow `HTTP/1.0` login requests
- (boolean) `W4WP_ALLOW_CONNECTION_EMPTY` allow requests without HTTP Connection header
- (boolean) `W4WP_ALLOW_CONNECTION_CLOSE` allow other HTTP Connection headers than `keep-alive`
- (boolean) `W4WP_ALLOW_TWO_CAPS` allow user names like `JohnDoe`
- (boolean) `W4WP_DISALLOW_TOR_LOGIN` to block logins from Tor exit nodes

Detect client IP address

- HTTP_CF_CONNECTING_IP (Cloudflare)
- HTTP_X_SUCURI_CLIENTIP (Sucuri)
- HTTP_INCAP_CLIENT_IP (Incapsula)
- HTTP_X_FORWARDED_FOR (Amazon CloudFront, can be comma delimited list of IP addresses)
- HTTP_X_FORWARDED
- HTTP_X_REAL_IP
- HTTP_CLIENT_IP
- HTTP_FORWARDED_FOR
- HTTP_FORWARDED
- REMOTE_ADDR

### Experimental upload traffic analysis

Insert this code at the end of `__construct()`.

```php
        if ( ! empty( $_FILES ) ) {
            $this->enhanced_error_log( sprintf( 'bad_request_upload: %s, %s',
                $this->esc_log( $_FILES ),
                $this->esc_log( $_REQUEST )
            ), 'notice' );
        }
```

To learn attack internals define `W4WP_POST_LOGGING` and edit conditions in `if ( $this->debug ...`.

### XMLRPC request and response

```xml
<?xml version="1.0" encoding="UTF-8"?>
<methodCall>
  <methodName>wp.getUsersBlogs</methodName>
  <params>
   <param><value>username</value></param>
   <param><value>password</value></param>
  </params>
</methodCall>
```

```xml
<?xml version="1.0" encoding="UTF-8"?>
<methodResponse>
  <params>
    <param>
      <value>
      <array><data>
  <value><struct>
  <member><name>isAdmin</name><value><boolean>1</boolean></value></member>
  <member><name>url</name><value><string>http://domain.wp/</string></value></member>
  <member><name>blogid</name><value><string>1</string></value></member>
  <member><name>blogName</name><value><string>Blog name</string></value></member>
  <member><name>xmlrpc</name><value><string>http://domain.wp/xmlrpc.php</string></value></member>
</struct></value>
</data></array>
      </value>
    </param>
  </params>
</methodResponse>
```

### HTTP2 and SPDY

All connections with HTTP2 and SPDY are persistent connections.

```php
define( 'W4WP_ALLOW_CONNECTION_EMPTY', true );
```

### Exclude a specific request

```apache
# Barion login request
<Limit "OPTIONS">
    RewriteCond %{QUERY_STRING} "=loginHint="
    RewriteRule "^Home/AsyncLogin$" "https://example.com/checkout/onepage/?" [L]
</Limit>
```

### Set up the Fail2ban filters

See: https://github.com/szepeviktor/debian-server-tools/tree/master/security/fail2ban-conf

Please examine the latest filter `failregexp`-s in
[Fail2ban GitHub repository](https://github.com/fail2ban/fail2ban/blob/master/config/filter.d).
It worth to combine every webserver related regexp-s in one custom filter.
You can customize the Fail2ban trigger string in the `$prefix` property of `Http_Analyzer` class.

### Other notes

You can customize the Fail2ban trigger string in the `$prefix` property.

Tests are included as a shell script: `bad-request-test.sh`

[GitHub repository](https://github.com/szepeviktor/wordpress-fail2ban)
