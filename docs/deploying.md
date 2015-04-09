# Deploying

**TODO(stevvooe):** This should discuss various deployment scenarios for
production-ready deployments. These may be backed by ready-made docker images
but this should explain how they were created and what considerations were
present.


# Middleware Configuration

This section describes how to configure storage middleware in the registry to enable layers to be served via a CDN, thus reducing requests to the storage layer.  Currently [Amazon Cloudfront](http://aws.amazon.com/cloudfront/) is supported and must be used in conjunction with the S3 storage driver.

## Cloudfront

## Parameters

`name`: The name of the storage middleware.  Currently `cloudfront` is an accepted value.

`disabled`: This can be set to false to easily disable the middleware.

`options` : A set of key/value options to configure the middleware:

* `baseurl` : The cloudfront base URL
* `privatekey` : The location of your AWS private key on the filesystem 
* `keypairid` : The ID of your Cloudfront keypair.
* `duration` : The duration in minutes for which the URL is valid.  Default is 20.

Note: Cloudfront keys exist separately to other AWS keys.  See [here](http://docs.aws.amazon.com/AWSSecurityCredentials/1.0/AboutAWSCredentials.html#KeyPairs) for more information.

## Example



```
middleware:
    storage:
        - name: cloudfront
          disabled: false
          options:
             baseurl: http://d111111abcdef8.cloudfront.net
             privatekey: /path/to/asecret.pem
             keypairid: asecret
             duration: 60
```

# Configure nginx to deploy alongside v1 registry

This sections describes how to configure nginx to proxy to both a v1 and v2
registry. Nginx will handle routing of to the correct registry based on the
URL and Docker client version.

## Example configuration
With v1 registry running at `localhost:5001` and v2 registry running at
`localhost:5002`.  Add this to `/etc/nginx/conf.d/registry.conf`.
```
server {
  listen 5000;
  server_name localhost;

  ssl on;
  ssl_certificate /etc/docker/registry/certs/domain.crt;
  ssl_certificate_key /etc/docker/registry/certs/domain.key;

  client_max_body_size 0; # disable any limits to avoid HTTP 413 for large image uploads

  # required to avoid HTTP 411: see Issue #1486 (https://github.com/docker/docker/issues/1486)
  chunked_transfer_encoding on;

  location /v2/ {
    # Do not allow connections from docker 1.5 and earlier
    # docker pre-1.6.0 did not properly set the user agent on ping, catch "Go *" user agents
    if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" ) {
      return 404;
    }

    proxy_pass                       http://localhost:5002;
    proxy_set_header  Host           $http_host;   # required for docker client's sake
    proxy_set_header  X-Real-IP      $remote_addr; # pass on real client's IP
    proxy_read_timeout               900;
  }

  location / {
    proxy_pass                       http://localhost:5001;
    proxy_set_header  Host           $http_host;   # required for docker client's sake
    proxy_set_header  X-Real-IP      $remote_addr; # pass on real client's IP
    proxy_set_header  Authorization  ""; # see https://github.com/docker/docker-registry/issues/170
    proxy_read_timeout               900;
  }
}
```

## Running nginx without a v1 registry
When running a v2 registry behind nginx without a v1 registry, the `/v1/` endpoint should
be explicitly configured to return a 404 if only the `/v2/` route is proxied. This
is needed due to the v1 registry fallback logic within Docker 1.5 and 1.6 which will attempt
to retrieve content from the v1 endpoint if no content was retrieved from v2.

Add this location block to explicitly block v1 requests.
```
localhost /v1/ {
	return 404;
}
```
