# MinIO on SNO (Single-Node OpenShift)

[MinIO] is an open-source object storage server with support for the S3 API.
This means that you can run your own S3 deployment from [SNO (Single-Node
OpenShift)] in a lab environment.

This repository contains instructions and Kubernetes manifest files for
deploying MinIO onto a SNO instance. **This code is for lab/development/testing
purposes only and is NOT intended for production use.** Leverage [OpenShift
Data Foundation] on a full OpenShift cluster for production use.

This repo was forked from [k3s-minio-deployment
(@sleighzy)](https://github.com/sleighzy/k3s-minio-deployment/) and modified to
work with OpenShift.

## Prerequisists

SNO is deployed out-of-the-box without a StorageClass. You will need a
filestorage-backed PersistantVolume (PV) for MinIO to use.

**Insert link to local-path-provisioner deploy repo**

## Deploying

Update `./values.yaml` with values for your environment. Then, run:

```bash
$ oc new-project minio
$ helm install minio .
```

---

# Everything below this line is from the original project

## OpenID Connect and Keycloak

MinIO supports authentication using OpenID Connect and providers such as
[Keycloak]. If you are wanting to OpenID Connect then add the below environment
variables to `./templates/deployment.yaml`.

```yaml
- name: MINIO_IDENTITY_OPENID_CONFIG_URL
  value: https://keycloak.example.com/auth/realms/home/.well-known/openid-configuration
- name: MINIO_IDENTITY_OPENID_CLIENT_ID
  value: minio
- name: MINIO_IDENTITY_OPENID_CLIENT_SECRET
  value: cad5f999-81e6-4791-afef-86ddd18ae3df
```

If integrating with Keycloak the MinIO docs on this were pretty good, see
<https://github.com/minio/minio/blob/master/docs/sts/keycloak.md>. I used a new
client named `minio` instead of modifying the existing `account` client in
Keycloak.

See the section further down in the README file for information on setting up a
policy for authorization to MinIO resources using JWT with Keycloak.

## MinIO Client

The [MinIO Client (mc)] can be easily used as to access and administer your
MinIO deployment.

Add an alias that points to your minio deployment.

```sh
$ mc alias set homelab https://minio.example.com XXXXXXXXXXXXXX xxxxXXXxxXXX/XXXXXXX/xxxxxXXXXXXX
Added `homelab` successfully.
```

The configuration and list of aliases can be viewed in the `~/.mc/config.json`
file.

```json
{
  "version": "10",
  "aliases": {
    "homelab": {
      "url": "https://minio.example.com",
      "accessKey": "XXXXXXXXXXXXXX",
      "secretKey": "xxxxXXXxxXXX/XXXXXXX/xxxxxXXXXXXX",
      "api": "s3v4",
      "path": "auto"
    }
  }
}
```

## Users and Groups

- <https://docs.min.io/docs/minio-admin-complete-guide.html#user>
- <https://docs.min.io/docs/minio-admin-complete-guide.html#group>

```sh
$ mc admin user add homelab
Enter Access Key: my-home-user
Enter Secret Key:
Added user `my-home-user` successfully.

$ mc admin group add homelab users my-home-user
Added members {my-home-user} to group users successfully.

$ mc admin user info homelab my-home-user
AccessKey: my-home-user
Status: enabled
PolicyName:
MemberOf: users
```

## Minio Policies

MinIO can be configured with policies to only allow users access to directories
within a bucket based on their username.

- <https://docs.min.io/docs/minio-admin-complete-guide.html#policy>
- <https://docs.min.io/docs/minio-multi-user-quickstart-guide.html>

The below policy will mean that the previously created `my-home-user` will only
be able to perform operations within the `my-home-user` prefix of the `users`
bucket.

```sh
$ mc admin policy add homelab readwriteusers policies/read-write-users.json
Added policy `readwriteusers` successfully.

$ mc admin policy info homelab readwriteusers
{
 "Version": "2012-10-17",
 "Statement": [
  {
   "Effect": "Allow",
   "Action": [
    "s3:ListBucket"
   ],
   "Resource": [
    "arn:aws:s3:::users"
   ],
   "Condition": {
    "StringLike": {
     "s3:prefix": [
      "${aws:username}/*"
     ]
    }
   }
  },
  {
   "Effect": "Allow",
   "Action": [
    "s3:ListMultipartUploadParts",
    "s3:PutObject",
    "s3:AbortMultipartUpload",
    "s3:DeleteObject",
    "s3:GetObject"
   ],
   "Resource": [
    "arn:aws:s3:::users/${aws:username}/*"
   ]
  }
 ]
}
```

Apply the policy to the `users` group so that it comes into effect for all users
belonging to that group.

```sh
$ mc admin policy set homelab readwriteusers group=users
Policy readwriteusers is set on group `users`

$ mc admin group info homelab users
Group: users
Status: enabled
Policy: readwriteusers
Members: my-home-user
```

## Authentication and Authorization with OpenID Connect and OAuth2

Users can be authenticated with MinIO using OpenID Connect. In my deployment I
am using Keycloak as the OpenID Connect Provider. When the MinIO login page is
displayed it will contain an additional link "Log in with OpenID". When clicking
this the user will be redirected to the Keycloak login page, once they have
provided their credentials they will be redirected back to MinIO and logged in
automatically.

A new policy needs to created for authorization to resources accessed using a
JWT that was provided during the OpenID Connect login. This is similar to the
previous `readwriteusers` policy except that the user principal is identified by
the `preferred_username` claim within the JWT.

This policy is **not** set on the `users` group. MinIO will select and enforce
this policy based on the `readwriteusersjwt` name specified in the `policy`
claim of the token.

```sh
$ mc admin policy add homelab readwriteusersjwt policies/read-write-users-jwt.json
Added policy `readwriteusersjwt` successfully.

mc admin policy info homelab readwriteusersjwt
{
 "Version": "2012-10-17",
 "Statement": [
  {
   "Effect": "Allow",
   "Action": [
    "s3:ListBucket"
   ],
   "Resource": [
    "arn:aws:s3:::users"
   ],
   "Condition": {
    "StringLike": {
     "s3:prefix": [
      "${jwt:preferred_username}/*"
     ]
    }
   }
  },
  {
   "Effect": "Allow",
   "Action": [
    "s3:AbortMultipartUpload",
    "s3:DeleteObject",
    "s3:GetObject",
    "s3:ListMultipartUploadParts",
    "s3:PutObject"
   ],
   "Resource": [
    "arn:aws:s3:::users/${jwt:preferred_username}/*"
   ]
  }
 ]
}
```

### Keycloak User Attribute and Claim Token

Keycloak will generate a JWT containing claims for the user when they are
authenticated by Keycloak. MinIO expects that this token contain a specific
claim named `policy` and that the value matches a policy that has been
configured.

On the user account details page in Keycloak for the `my-home-user` user under
the _Attributes_ tab I have added an attribute with the name `minio-policy` and
the value `readwriteusersjwt`. In the configuration for the `minio` client I
have created an attribute mapping with the name "Minio Policy". This is a
`User Attribute` _Mapper Type_ with the _User Attribute_ value of "minio-policy"
and the _Token Claim Name_ of "policy". This means that when the user is
authenticated it will retrieve "readwriteusersjwt" from the user's
"minio-policy" attribute and use it for the value of the "policy" token claim.

### MinIO Console Administrator

MinIO has a default policy named `consoleAdmin` which allows the user access to
the administration features of the console. If authenticating as an
administrative user via OpenID Connect you will need to add set `consoleAdmin`
as the value for the `minio-policy` user attribute.

### Troubleshooting JWT Authorization

The below error in the Minio logs may be seen when failing to authenticate using
JWT. This can mean that the `policy` claim is missing. Check to ensure that the
`policy` Claim Mapping has been added to the client in Keycloak. Check that this
is mapped to a user attribute belonging to the user attempting to authenticate,
and that the value of this attribute on the user matches the Minio policy name
for the JWT policy, e.g. `readwriteusersjwt`.

```sh
API: AssumeRoleWithWebIdentity()
Time: 07:55:11 UTC 12/07/2020
DeploymentID: f096ff8d-f2a5-46ac-8c74-0b9de20fe2c6
RequestID: 164E6009BEBE4A3F
RemoteHost: 10.42.0.1
Host: minio.example.com
UserAgent: Go-http-client/2.0
Error: policy claim missing from the JWT token, credentials will not be generated
       4: cmd/sts-errors.go:57:cmd.writeSTSErrorResponse()
       3: cmd/sts-handlers.go:335:cmd.(*stsAPIHandlers).AssumeRoleWithSSO()
       2: cmd/sts-handlers.go:423:cmd.(*stsAPIHandlers).AssumeRoleWithWebIdentity()
       1: net/http/server.go:2042:http.HandlerFunc.ServeHTTP()

API: WebLoginSTS()
Time: 07:55:11 UTC 12/07/2020
DeploymentID: f096ff8d-f2a5-46ac-8c74-0b9de20fe2c6
RemoteHost: 10.42.0.1
Host: minio.example.com
UserAgent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/14.0.1 Safari/605.1.15
Error: 400 Bad Request
      25: cmd/web-handlers.go:2278:cmd.toWebAPIError()
      24: cmd/web-handlers.go:2147:cmd.toJSONError()
      23: cmd/web-handlers.go:2131:cmd.(*webAPIHandlers).LoginSTS()
```

## Tips, Tricks, and Gotchas

### S3 Connection Urls

When configuring applications to connect to S3 the url format for MinIO may
differ from the standard AWS S3 url.

For example, when integrating with Loki the below format is required. This has
some important changes:

- The protocol specifies `https` and not `s3`.
- The hostname in the url contains an extra dot at the end, i.e.
  `minio.example.com./`, which forces S3 to use this as the hostname and not the
  AWS region.
- Any forward slashes contained in the access key or secret key in the url need
  to be escaped. These need to be replaced with `%2F`.

```yaml
aws:
  s3: https://XXXXXXXXXXXXXX:xxxxXXXxxXXX%2FXXXXXXX%2FxxxxxXXXXXXX@minio.example.com./loki
```

<https://github.com/grafana/loki/issues/1434>

## License

[![MIT license]](https://lbesson.mit-license.org/)

[docker registry]: https://docs.docker.com/registry/
[external hard drive for persistent storage]:
  https://github.com/sleighzy/raspberry-pi-k3s-homelab/blob/main/k3s.md#external-hard-drive-for-persistent-storage
[grafana loki]: https://grafana.com/oss/loki/
[k3s]: https://k3s.io/
[keycloak]: https://www.keycloak.org/
[local path provisioner]: https://rancher.com/docs/k3s/latest/en/storage/
[minio]: https://min.io/
[minio client (mc)]: https://docs.min.io/docs/minio-client-complete-guide.html
[mit license]: https://img.shields.io/badge/License-MIT-blue.svg
[raspberry-pi-k3s-homelab]:
  https://github.com/sleighzy/raspberry-pi-k3s-homelab/blob/main/k3s.md
[restic]: https://restic.net/
[traefik v2]: https://traefik.io/traefik/
[v1.0.0 tag]:
  https://github.com/sleighzy/k3s-minio-deployment/releases/tag/v1.0.0
