# Namespace
The global namespace represents the full set of named repositories which can be
referenced by Docker. Any scope within the namespace may refer to a subset of
the named repos. A repository contains a set of content-addressable blobs
and tags referencing those blobs. The namespace can be used to discover the
location and certificate information of a repository based on DNS, HTTP requests
and other means.

A repository should always be referenced by its fully qualified name. If a
client presents a shortened name to a user, that name should be fully
expanded based on rules defined by the client before contacting a remote
server. When mirroring repositories, the original name for a repository must be
used. Changing a repository name is equivalent to copying or moving and will not
be referenced by the original repository.

## Terminology

- *Global Namespace* - The full set of referenceable names.
- *Name* - A fully qualified string containing both the domain and resource name
- *Repository* - A collection of objects under the same name within the
namespace.
- *Namespace* - also *Namespace Scope* - A collection of repositories with a
common name prefix and set of services including registry api, index, and trust
context.
- *Short Name* - A name which does not contain a domain and requires
expansion to a fully qualified name before resolving to a repository.

## Format
A name consists of two parts, a DNS host name plus a repository path. The host
name follows the DNS rules without change. The total length of a name is 255
characters however there is no specific limitation on DNS or path components.

### Name Grammar
~~~
<name> ::= <hostname>"/"<path>
<hostname> ::= <host-part>*["."<host-part>][":"<port>]
<path> ::= <path-part>*["/"<path-part>]
<host-part> ::= <regexp "[a-z]([-]?[a-z0-9])*">
<port> ::= <number 1 to 65535>
<path-part> ::= <regexp "[a-z0-9]([._-]?[a-z0-9])*">
~~~

## Discovery
The discovery process involves resolving a name into namespace scoped metadata.
The namespace metadata contains the full set of information needed to fetch and
verify content associated with the namespace repository. The metadata includes
list of registry api endpoints, the trust model, and search index. The discovery
process should not be considered secure and therefore certificates retrieved as
part of the discovery process should be verified before trusting.

Discovery can be defined as...
`<fully qualified name> -> scope([<registry api endpoint>, ...], <certificate>, <search index>, ...)`

or in Go as...
~~~go
type NamespaceResolver interface {
	Resolve(name string) Namespace
}
~~~

### Default Method

The first element of the namespace is extracted and used as the domain name for
resolution. The domain name should be used unmodified.

`<fully qualified name> -> <domain>/<path>`

#### HTTPS
The discovery related metadata will be fetched via HTTPS from the DNS resolved
location using the remaining namespace path elements as the HTTP path in a GET
request. A discovery request url would be in the format
`https://<domain>/<name>?docker-discovery=1`

For example, “example.com/foo/bar” would create a url
“https://example.com/foo/bar?docker-discovery=1”

##### HTML Response
~~~html
<meta name="docker-scope" content="example.com">
<meta name=“docker-registry” content=“push,pull v2 https://registry.example.com/v2/”>
<meta name=“docker-registry” content=“push,pull v1 https://registry.example.com/v1/”>
<meta name=“docker-registry” content=“pull v2 https://registry.mirror.com/v2/”>
<meta name=“docker-registry” content=“pull,notag v2 http://registry.mirror.com/v2/”>
<meta name=“docker-index” content=“v1 https://search.mirror.com/v1/”>
<meta name=“docker-trust” content=“https://trust.example.com/v2/”>
<meta name=“docker-certificate” content=“https://trust.example.com/registry.cert”>
~~~

#### Fallback (Compatibility)
If HTTPS is not implemented for a namespace, a fallback protocol may be
used when the namespace prefix is explicitly marked as insecure. The
fallback process involves attempting to ping possible registry api endpoints to
determine the set of endpoints and using no trust model.

### Extensibility
A custom method may be used to provide discovery by implementing the
`NamespaceResolver` interface.

### Scope
The namespace information produced from the discovery process may contain a
scope field. The scope field means the information may apply to any namespace
with a prefix of the given scope. The Scope prefix will always be applied with a
path separator. If the scope field is ommitted, the information may not be
applied to any other namespace. If the scope field is a prefix of the requested
name, the scope should only be respected if it has a verified certificate in
that scope.

### Endpoints
The registry api endpoints each contain a version (may be v1 or v2 registries)
and may either be pull only mirrors or full registries. The trust model applies
to each registry api endpoint and may not be overloaded by an individual
endpoint. The trust model defines the method for verifying the content retrieved
from an endpoint, not the method for authentication or authorization.  Each
registry api endpoint is responsible for specifying its authentication or
authorization method. 

It may also be possible in the future to extend these endpoints to support
direct downloads of tarballs, such as the result from a `docker save`.

## Certificates
Each namespace scope may have a certificate which can be used to sign keys for
each individual repository. The certificate must be scoped to the namespace and
be used to sign certificates using the fully qualified name for each repository.
If namespace does not contain a certificate, the namespace will be considered
untrusted for any repositories it contains. It is up to the client whether to
require these certificates to be signed by a root CA.

## Name Expansion
Before a name can be resolved, it must be expanded to its fully qualified form.
This may mean adding to the resource path as well as inserting a default
domain. The rules for expansion must be determined by the client.

Expansion can be defined as...
`<name> -> <fully qualified name>`

or in Go as...
~~~go
type Expander interface {
	Expand(name string) string
}
~~~

### Compatibility
Current Docker clients have a default expansion which must remain backwards
compatible from a user perspective. Docker clients expand short names containing
no slashes as "docker.com/library/{name}" and all other short names as 
"docker.com/{name}". Current tooling built around Docker expects to use the
Docker hub registry for all short names.

