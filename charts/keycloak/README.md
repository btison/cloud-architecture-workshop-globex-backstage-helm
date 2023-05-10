# Backstage uses Keycloak

### The [OpenShift OAuth2Proxy](https://janus-idp.io/blog/using-openshift-authentication-to-secure-access-to-backstage) implementation causes a side-effect in Backstage, where certain APIs (such as the Scaffolder API) are losing their tokens, rendering Backstage useless.
### Backstage is accessible after logging in with OpenShift credentials, the (OpenShift-) user is identified correctly, therefore SSO works on first glance. 
### However, calls to the 
```
/api/scaffolder/v2/templates/default/template/[template]/parameter-schema/api/scaffolder/v2/templates/default/template/[template]/parameter-schema
``` 
### fail with a 401 
```
{"error":{"name":"AuthenticationError","message":"No token specified"},"request":{"method":"GET","url":"/api/scaffolder/v2/templates/default/template/springboot-template/parameter-schema"},"response":{"statusCode":401}}`
```
## Solution

Using the [Upstream OAuth2Proxy](https://oauth2-proxy.github.io/oauth2-proxy/) works as it should. However, this doesn't know anything about OpenShift, so we are using Keycloak.

If OpenShift was also using Keycloak as OIDC provider, the problem would be solved at this point (OpenShift User = Keycloak User Entity = Backstage User).

However, the automatic provisioning via RHDP creates OpenShift users (= Lab Users `user1..45`).
For this, we are configuring [OpenShift as OAuth provider](https://docs.openshift.com/container-platform/4.12/authentication/using-service-accounts-as-oauth-client.html), which can be federated through Keycloak.

![](./readme.images/identityproviders.png)

Additionally, Keycloak needs to communicate with OpenShift via SSL/TLS (Keycloak forwards the authentication request to the ServiceAccount via the OCP Master API endpoint) 
At this point, Keycloak initiates SSL/TLS communication with the OpenShift API - which fails unless 

- the API endpoint uses a (Let's Encrypt) certificate from a trusted CA (included in the default Java Truststore) 

or

- we add the self-signed certificate to Keycloak's truststore

A default RHDP Cluster uses Let's Encrypt Certs only workloads, not the API.
Found that after SSL debugging with (tip for any Java workload) adding 
```
           - name: JAVA_TOOL_OPTIONS
             value: '-Djavax.net.debug=ssl:handshake:verbose:keymanager:trustmanager'
```
to the Keycloak Container.

Keycloak pukes with a

```
org.keycloak.broker.provider.IdentityBrokerException: Could not initialize oAuth metadata
[...]
Caused by: javax.net.ssl.SSLHandshakeException: PKIX path building failed: sun.security.provider.certpath.SunCertPathBuilderException: unable to find valid certification path to requested target
```
Exception, because the API is using a self-signed cert (which can also be seen in the above debug log with `-Djavax.net.debug=ssl:handshake:verbose:keymanager:trustmanager` enabled):

```
$ openssl s_client -connect api.cluster-26v9d.26v9d.sandbox2355.opentlc.com:6443 | openssl x509 -noout -text

depth=1 OU = openshift, CN = kube-apiserver-lb-signer
verify error:num=19:self-signed certificate in certificate chain
verify return:1
depth=1 OU = openshift, CN = kube-apiserver-lb-signer
verify return:1
depth=0 CN = api.cluster-26v9d.26v9d.sandbox2355.opentlc.com
verify return:1
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 4663035812764514264 (0x40b66e5daac0f7d8)
        Signature Algorithm: sha256WithRSAEncryption
        Issuer: OU = openshift, CN = kube-apiserver-lb-signer
        Validity
            Not Before: May  7 12:25:48 2023 GMT
            Not After : Jun  6 12:25:49 2023 GMT
        Subject: CN = api.cluster-26v9d.26v9d.sandbox2355.opentlc.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
                Modulus:
                    00:c9:89:1a:3f:ab:f6:1f:c2:3b:ec:5e:e6:77:2c:
                    [...]

```

## UPDATE 2023-05-10 For the Summit Demo, the Let's Encrypt Certificate has been enabled on API Endpoints as well

Therefore, we don't need to fiddle around with getting the certificate from the API endpoint and inject it into the Keycloak truststore.

Also, since RHSSO has already been enabled for some labs and is available inside our cluster, we are using RHSSO and won't have to deploy a separate Keycloak instance.

Check the comments in the [sa-rhsso-oauthredirect.yaml](../backstage/templates/sa-rhsso-oauthredirect.yaml) file.
