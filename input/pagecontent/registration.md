The requirements in this section are applicable to both consumer-facing and B2B apps and the servers that support them.

Before FHIR data requests can be made, Client applications operators **SHALL** register each of their applications with the Authorization Servers identified by the FHIR servers with which they wish to exchange data.  Client applications **SHALL** use the client_id assigned by an Authorization Server in subsequent authorization and token requests to that server.

Authorization Servers SHOULD support dynamic registration as specified in the UDAP Dynamic Client Registration profile DRAFT 2019-05-15 at <http://www.udap.org/udap-dynamic-client-registration.html> with the additional options and constraints defined in this guide. Confidential clients, i.e. conventional server-based web applications that can maintain a secret, **MAY** use this dynamic client registration protocol as discussed further below to obtain a `client_id`. Other client types SHOULD follow the manual registration processes for each Authorization Server. Future versions of this guide may add support for dynamic client registration by native device applications or public clients which cannot protect a private key.

This guide supports applications using either the client credentials or authorization code grant types. The B2B transactions in this guide occur between a requesting organization (the Requestor operating the client application) and a responding organization (the Responder operating the OAuth Server and Resource Server holding the data of interest to the Requestor). In some cases, the Requestor's client app operates in an automated manner. In other cases, there will also be a local user from the requesting organization (the User interacting with the Requestor's client application). The client credentials grant type is always used for automated (aka "headless") client apps. However, when a User is involved, either the client credentials or authorization code grant may be used, as discussed below.

For client credentials flow, any necessary User authentication and authorization is performed by the Requestor as a prerequisite, before the Requestor's client app interacts with the Responder's servers, i.e. the Requestor is responsible for ensuring that only its authorized Users access the client app and only make requests allowed by the Requestor. How the Requestor performs this is out of scope for this guide but will typically rely on internal user authentication and access controls.

For authorization code flow, the User is expected to be interacting with the Requestor's client app in real-time, at least during the initial authorization of the client app with the Responder's OAuth Server. Typically, the User must authenticate to the Responder's system at the time of initial authorization. If the local user has been issued credentials by the Responder to use for this purpose, the authorization code flow will typically involve use of those credentials. However, it is anticipated that in most circumstances, the local user will not have their own credentials on the Responder's system, but will instead have credentials on their "home" system. In these cases, the UDAP Tiered OAuth workflow is used so that the Responder's OAuth Server can interact with the Requestor's OIDC Server in an automated manner to authenticate the User.

Thus, this guide provides two different paths (client credentials grants and authorization code grants with Tiered OAuth) that a user affiliated with the Requestor without credentials on the Responder's system may use to obtain access to data held by the Responder.

### Software Statement

To register dynamically, the client application first constructs a software statement as per [section 2](https://www.udap.org/udap-dynamic-client-registration.html#section-2) of UDAP Dynamic Client Registration.

The software statement **SHALL** contain the required header elements specified in [Section 1.2.3] of this guide and the JWT claims listed in the table below.  The software statement **SHALL** be signed by the client application operator using the signature algorithm identified in the `alg` header of the software statement and with the private key that corresponds to the public key listed in the client's X.509 certificate identified in the `x5c` header of the software statement.

<table class="table">
  <thead>
    <th colspan="3">Software Statement JWT Claims</th>
  </thead>
  <tbody>
    <tr>
      <td><code>iss</code></td>
      <td><span class="label label-success">required</span></td>
      <td>
        Issuer of the JWT -- unique identifying client URI. This <strong>SHALL</strong> match the value of a uniformResourceIdentifier entry in the Subject Alternative Name extension of the client's certificate included in the <code>x5c</code> JWT header
      </td>
    </tr>
    <tr>
      <td><code>sub</code></td>
      <td><span class="label label-success">required</span></td>
      <td>
        Same as <code>iss</code>. In typical use, the client application will not yet have a <code>client_id</code> from the Authorization Server
      </td>
    </tr>
    <tr>
      <td><code>aud</code></td>
      <td><span class="label label-success">required</span></td>
      <td>
        The Authorization Server's "registration URL" (the same URL to which the registration request  will be posted)
      </td>
    </tr>
    <tr>
      <td><code>exp</code></td>
      <td><span class="label label-success">required</span></td>
      <td>
        Expiration time integer for this software statement, expressed in seconds since the "Epoch" (1970-01-01T00:00:00Z UTC). The <code>exp</code> time <strong>SHALL</strong> be no more than 5 minutes after the value of the <code>iat</code> claim.
      </td>
    </tr>
    <tr>
      <td><code>iat</code></td>
      <td><span class="label label-success">required</span></td>
      <td>
        Issued time integer for this software statement, expressed in seconds since the "Epoch"
      </td>
    </tr>
    <tr>
      <td><code>jti</code></td>
      <td><span class="label label-success">required</span></td>
      <td>
        A nonce string value that uniquely identifies this software statement. This value <strong>SHALL NOT</strong> be reused by the client app in another software statement or authentication JWT before the time specified in the <code>exp</code> claim has passed
      </td>
    </tr>
    <tr>
      <td><code>client_name</code></td>
      <td><span class="label label-success">required</span></td>
      <td>
        A string containing the human readable name of the client application
      </td>
    </tr>
    <tr>
      <td><code>redirect_uris</code></td>
      <td><span class="label label-warning">conditional</span></td>
      <td>
        An array of one or more redirection URIs used by the client application. This claim SHALL be present if <code>grant_types</code> includes <code>"authorization_code"</code> and this claim SHALL be absent otherwise. Each URI SHALL use the https scheme.
      </td>
    </tr>
    <tr>
      <td><code>contacts</code></td>
      <td><span class="label label-success">required</span></td>
      <td>
        An array of URI strings indicating how the data holder can contact the app operator regarding the application. The array <strong>SHALL</strong> contain at least one valid email address using the <code>mailto</code> scheme, e.g.<br>
        <code>["mailto:operations@example.com"]</code>
      </td>
    </tr>
    <tr>
      <td><code>logo_uri</code></td>
      <td><span class="label label-warning">conditional</span></td>
      <td>
        A URL string referencing an image associated with the client application, i.e. a logo. If <code>grant_types</code> includes <code>"authorization_code"</code>, client applications <strong>SHALL</strong> include this field, and the authorization server <strong>MAY</strong> display this logo to the user during the authorization process. The URL <strong>SHALL</strong> use the https scheme and reference a PNG, JPG, or GIF image file, e.g. <code>"https://myapp.example.com/MyApp.png"</code>
      </td>
    </tr>
    <tr>
      <td><code>grant_types</code></td>
      <td><span class="label label-success">required</span></td>
      <td>
        Array of strings, each representing a requested grant type, from the following list: <code>"authorization_code"</code>, <code>"refresh_token"</code>, <code>"client_credentials"</code>. The array <strong>SHALL</strong> include either <code>"authorization_code"</code> or <code>"client_credentials"</code>, but not both. The value <code>"refresh_token"</code> <strong>SHALL NOT</strong> be present in the array unless <code>"authorization_code"</code> is also present.
      </td>
    </tr>
    <tr>
      <td><code>response_types</code></td>
      <td><span class="label label-warning">conditional</span></td>
      <td>
        Array of strings. If <code>grant_types</code> contains <code>"authorization_code"</code>, then this element <strong>SHALL</strong> have a fixed value of <code>["code"]</code>, and <strong>SHALL</strong> be omitted otherwise
      </td>
    </tr>
    <tr>
      <td><code>token_endpoint_auth_method</code></td>
      <td><span class="label label-success">required</span></td>
      <td>
        Fixed string value: <code>"private_key_jwt"</code>
      </td>
    </tr>
    <tr>
      <td><code>scope</code></td>
      <td><span class="label label-success">required</span></td>
      <td>
        String containing a space delimited list of scopes requested by the client application for use in subsequent requests. The Authorization Server <strong>MAY</strong> consider this list when deciding the scopes that it will allow the application to subsequently request. Client apps requesting the <code>"client_credentials"</code> grant type <strong>SHOULD</strong> request system scopes; apps requesting the <code>"authorization_code"</code> grant type <strong>SHOULD</strong> request user or patient scopes.
      </td>
    </tr>
  </tbody>
</table>

The unique client URI used for the `iss` claim **SHALL** match the uriName entry in the Subject Alternative Name extension of the client app operator's X.509 certificate, and **SHALL** uniquely identify a single client app operator and application over time. The software statement is intended for one-time use with a single OAuth 2.0 server. As such, the `aud` claim **SHALL** list the URL of the OAuth Server's registration endpoint, and the lifetime of the software statement (`exp` minus `iat`) **SHALL** be 5 minutes.

### Example

#### Client Credentials

Example software statement, prior to Base64URL encoding and signature, for a B2B app that is requesting the use of the client credentials grant type (non-normative, the "." between the header and claims objects is a convenience notation only):

```
{
  "alg": "RS256",
  "x5c": ["MIEF.....remainder omitted for brevity"]
}.{
  "iss": "http://example.com/my-b2b-app",
  "sub": "http://example.com/my-b2b-app",
  "aud": "https://oauth.example.net/register",
  "exp": 1597186041,
  "iat": 1597186341,
  "jti": "random-value-109a3bd72"
  "client_name": "Acme B2B App",
  "contacts": ["mailto:b2b-operations@example.com"],
  "grant_types": ["client_credentials"],
  "token_endpoint_auth_method": "private_key_jwt",
  "scope": "system/Patient.read system/Procedure.read"
}
```

#### Authorization Code

Example software statement, prior to Base64URL encoding and signature, for a B2B app that is requesting the use of the client credentials grant type (non-normative, the "." between the header and claims objects is a convenience notation only):

```
{
  "alg": "RS256",
  "x5c": ["MIEF.....remainder omitted for brevity"]
}.{
  "iss": "http://example.com/my-user-b2b-app",
  "sub": "http://example.com/my-user-b2b-app",
  "aud": "https://oauth.example.net/register",
  "exp": 1597186054,
  "iat": 1597186354,
  "jti": "random-value-f83f37a29"
  "client_name": "Acme B2B User App",
  "redirect_uris": ["https://b2b-app.example.com/redirect"],
  "contacts": ["mailto:b2b-operations@example.com"],
  "logo_uri": "https://b2b-app.example.com/B2BApp.png",
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "private_key_jwt",
  "scope": "user/Patient.read user/Procedure.read"
}
```

#### Request Body

The registration request for use of either grant type is submitted by the client to the Authorization Server's registration endpoint.

```
POST /register HTTP/1.1
Host: oauth.example.net
Content-Type: application/json

{
   "software_statement": "...the signed software statement JWT...",
   "certifications": ["...a signed certification JWT..."]
   "udap": "1"
}
```

The Authorization Server **SHALL** validate the registration request as per Section 4 of UDAP Dynamic Client Registration. This includes validation of the JWT payload and signature, validation of the X.509 certificate chain, and validation of the requested application registration parameters. If a new registration is successful, the Authorization Server **SHALL** return a registration response with a HTTP 201 response code as per [Section 5.1](https://www.udap.org/udap-dynamic-client-registration.html#section-5.1) of UDAP Dynamic Client Registration, including the unique `client_id` assigned by the Authorization Server to that client app. If a new registration is not successful, e.g. it is rejected by the server for any reason, the Authorization Server **SHALL** return an error response as per [Section 5.2](https://www.udap.org/udap-dynamic-client-registration.html#section-5.2) of UDAP Dynamic Client Registration.

### Inclusion of Certifications and Endorsements

Authorization Servers **MAY** support the inclusion of certifications and endorsements by client application operators using the certifications framework outlined in [UDAP Certifications and Endorsements for Client Applications](https://www.udap.org/udap-certifications-and-endorsements.html). Authorization Servers **SHALL** ignore unsupported or unrecognized certifications.

Authorization Servers **MAY** require registration requests to include one or more certifications. If an Authorization Server requires the inclusion of a certain certification, then the Authorization Server **SHALL** communicate this by including the corresponding certification URI in the `udap_certifications_required` element of its UDAP metadata.

### Modifying and Cancelling Registrations

The client URI in the Subject Alternative Name of an X.509 certificate uniquely identifies a single application and its operator over time. Thus, a previously registered client application **MAY** request a modification of its previous registration with an Authorization Server by submitting another registration request to the same Authorization Server's registration endpoint using a certificate with a Subject Alternative Name client URI entry matching the original registration request.

If an Authorization Server receives a valid registration request with a software statement containing the same `iss` value as an earlier software statement but with a different set of claims or claim values, or with a different (possibly empty) set of optional certifications and endorsements, the server **SHALL** treat this as a request to modify the registration parameters for the client application by replacing the information from the previous registration request with the information included in the new request. For example, an Application operator could use this mechanism to update a redirection URI or to remove or update a certification. If the registration modification request is accepted, the Authorization Server **SHOULD** return the same `client_id` in the registration response as for the previous registration. If it returns a different `client_id`, it **SHALL** cancel the registration for the previous `client_id`.

If an Authorization Server receives a valid registration request with a software statement that contains an empty `grant_types` array from a previously registered application, the server **SHOULD** interpret this as a request to cancel the previous registration. A client application **SHALL** interpret a registration response that contains an empty `grant_types` array as a confirmation that the registration for the `client_id` listed in the response has been cancelled by the Authorization Server.

If the Authorization Server returns the same `client_id` in the registration response for a modification request, it SHOULD also return an HTTP 200 response code. If the Authorization Server returns a new `client_id` in the registration response, the client application **SHALL** use only the new `client_id` in subsequent transactions with the Authorization Server.

{% include link-list.md %}