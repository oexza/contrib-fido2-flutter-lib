# contrib-fido2-flutter-lib
A flutter plugin for using FIDO/WebAuthN APIs. Supports Android and Web.

## Quick Start

```bash
pub get ...

```

## Supported Platforms

| Platform | Supported     |
| -------- | ------------- |
| Android  | `YES`         |
| iOS      | `NO` - see [#5](./issues/5) |
| Web      | `IN PROGRESS` |

> ⚠️ __Warning__
>
> _The Fido2 client only supports Android + Web currently!_
> 
> Since Apple has only joined the FIDO alliance in Feb 2020, it is not expected that an iOS Fido2 client will be ready like the ones available for Android.
> 
> However, we are open to contributors interested in working on the iOS side.

## Introduction

Fido2Client is a Flutter plugin that allows you to use your Flutter app as an authenticator in the Fido2 process. With this plugin, your Flutter app can create and use public key based credentials to authenticate users.

The Fido2Client supports 2 main operations:

1. **Registration** - Registration is done once per authenticator per account. It is performed when linking a credential to a user.

2. **Signing** - Signing is performed to verify a user's identity.

### How does FIDO2 verify a user's identity? (Brief overview)

The FIDO2 Authentication process is based on public key cryptography. 
When the user is in the registration phase, the client generates an asymmetric keypair (1 public key and 1 private key). 
The private key pair is stored somewhere secure on device while the public key pair is sent to the server and is associated to a particular user.

The next time that the server wants to authenticate a user, they send a challenge - usually a randomly 
generated string with a fixed, predetermined length. 
The FIDO2 client uses the private key it previously stored to sign this string, producing a signature. 
Using the previously registered public key, the server can check whether or not the signature produced was a result of using 
the associated private key to sign the particular challenge. The identity of the user is assumed from their ownership of the private key.

Read more about FIDO [here]](#see-also)

## API Calls
### `initiateRegistration`

Initiates the registration process.


#### Arguments:

| variable          | type     | description                                                                  |
|-------------------|----------|------------------------------------------------------------------------------|
| `challenge`       | `String` | The string given by the server                                               |
| `userId`          | `String` | The identifier of the user you are registering a credential for              |
| `username`        | `String` | The name of the user you are registering a credential for                    |
| `rpDomain`        | `String` | The domain of the Relying Party*                                             |
| `rpName`          | `String` | The name of the Relying Party                                                |
| `coseAlgoValue**` | `int`    | The unique COSE identifier for the algorithm to be used by the authenticator |

#### Example:

```dart
import ...
import ...

//TODO!
var result = initiateRegistration()
```

> \* A Relying Party refers to the party on whose behalf the authentication ceremony is being performed. 
> You can view the formal definition [here](https://www.w3.org/TR/webauthn/#webauthn-relying-party)
> For example, if you were using this for a mobile app with a web server backend, then the web server would be the Relying Party.

> \*\* See the supported algorithms: [EC2 algorithms](https://developers.google.com/android/reference/com/google/android/gms/fido/fido2/api/common/EC2Algorithm) and [RSA algorithms](https://developers.google.com/android/reference/com/google/android/gms/fido/fido2/api/common/RSAAlgorithm)
> These 2 links will give you the supported descriptions of the supported algorithms e.g. 'ECDSA w/ SHA-256'.
> You can search for the algorithm identifier using the following links: [COSE registry](https://www.iana.org/assignments/cose/cose.xhtml#algorithms) and [WebAuthn registry](https://www.w3.org/TR/webauthn/#sctn-cose-alg-reg).
> You will find that 'ECDSA w/ SHA-256' has a COSE identifier of -7.

#### Return Values:

The output will be in the form of a `RegistrationResult` model object with the following fields:

| variable       | type   | encoding                | description                                                                                             |
|----------------|--------|-------------------------|---------------------------------------------------------------------------------------------------------|
| keyHandle      | String | Base64URL               | A string identifier for the credential generated.                                                       |
| clientData     | String | Base64URL               | [WebAuthn spec](https://www.w3.org/TR/webauthn/#dom-authenticatorresponse-clientdatajson)               |
| attestationObj | String | CBOR and then Base64URL | [WebAuthn spec](https://www.w3.org/TR/webauthn/#dom-authenticatorattestationresponse-attestationobject) |

This corresponds to the `AuthenticatorAttestationResponse` in the WebAuthn spec.

### `initiateSigning`

Inputs:

| variable  | type   | description                                                                                                                                   |
|-----------|--------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| keyHandle | String | The string identifier for the credential you are authenticating.  This should be the same as the output of the initial `initiateRegistration` |
| challenge | String | The challenge string from the server to be signed by the FIDO client.                                                                         |
| rpDomain  | String | The domain of the Relying Party. Same as the variable in `initiateRegistration`                                                               |

Outputs:

The output will be in the form of a `SigningResult` model object with the following fields:

| variable   | type   | encoding  | description                                                                                                                                                                                            |
|------------|--------|-----------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| keyHandle  | String | Base64URL | The string identifier for the credential                                                                                                                                                               |
| clientData | String | Base64URL | [WebAuthn spec](https://www.w3.org/TR/webauthn/#dom-authenticatorresponse-clientdatajson)                                                                                                              |
| authData   | String | Base64URL | [WebAuthn spec](https://www.w3.org/TR/webauthn/#authenticator-data)                                                                                                                                    |
| signature  | String | Base64URL | The signature is to be sent to the server for verification of identity. <br/> It provides proof that the authenticator possesses the private key associated with the public key previously registered. |
| userHandle | String | Base64URL | An opaque identifier for the user being authenticated.                                                                                                                                                 |

This corresponds to the `AuthenticatorAssertionResponse` in the WebAuthn spec.


## Usage

### Android

There are 2 functions that are exposed to the user, each corresponding to a phase of the FIDO2 process:

(1) `initiateRegistration`

(2) `initiateSigning`

It is fairly straightforward to understand the purpose of these functions. Calling these functions brings up a flow that guides the user through the processes of FIDO credential registration and signing respectively and the functions return futures of generated authenticator results which can be sent to the server for registration and authentication purposes.

However, the inputs and outputs of these functions may be confusing. 

Here is an explanation of the inputs and outputs of the above functions:

#### Dependencies

The plugin uses the native Android library: Fido2ApiClient, specifically `com.google.android.gms:play-services-fido:18.1.0`.

TODO: example here

#### Hosting assetlinks.json (VERY IMPORTANT!)

This step is very important! Without this, the plugin will not work. By hosting the file that these instructions will teach you, your server is making a public statement about sharing credentials with your Flutter app.

1. Generate your app's SHA256 fingerprint by following the steps [here](https://developers.google.com/android/guides/client-auth)
2. In the JSON file below, replace "app sha256 fingerprint" with your app's SHA256 fingerprint.

`assetlinks.json`

```
[
  {
    "relation" : [
      "delegate_permission/common.handle_all_urls",
      "delegate_permission/common.get_login_creds"
    ],
    "target" : {
      "namespace" : "android_app",
      "package_name" : "com.example.android",
      "sha256_cert_fingerprints" : [
         "app sha256 fingerprint"
      ]
    }
  }
]
```

3. Host the JSON file at https://example.com/.well-known/assetlinks.json, replacing example with your domain.

#### Getting everything to work

1. While the user is logged in via traditional login processes, when the user needs to register a FIDO credential, request registration options from the server - these will be provided as inputs to `initiateRegistration`.
2. Prompt the user to begin the registration phase by calling `initiateRegistration` with the registration options retrieved in the previous step.
3. Format the `RegistrationResult` into something that your web server understands and send the results - the server will save the keyHandle(credential identifier) and public key and associate it to the user.
4. The next time the user needs to verify their identity (e.g. for login), request signing options from the server - these will be provided as inputs to `initiateSigning`.
5. Prompt the user to authenticate themselves by calling `initiateSigning` with the signing options retrieved in the previous step.
5. Once again, format the `SigningResult` into something that your web server understands and send the results for verification. If the server deems that this is indeed a valid signature produced using the private key of the key pair previously registered, then the user has been authenticated.

If you want to see a working example, feel free to reference the [example fido flow](#example-fido-flow).
If there are any issues, you may refer to the section on [common issues](#common-issues).

#### Example usage on Android

If you wish to see a working example, you can take a look at this [repo](https://github.com/kkzeng/fido2-client-example-flutter)


### Web

Web support is currently a work in progress. Take a look at 
`./lib/Fido2ClientPlugin_web.dart` and `./assets/js/fido2_client.js` for reference.

In your `pubspec.yaml`, 

```yaml
...
dependencies:
  ...
  fido2_client:
    git:
      url: git://github.com/mojaloop/fido2-client-plugin
      ref: <some commit>
```

> Note:
>
> We aren't quite ready to make releases for this library yet, but it's on the radar.
> For now you can pin to a git commit


And in your `web/index.html`:

```html
...
<body>
  <!-- fido2_client -->
  <script src="assets/packages/fido2_client/assets/js/fido2_client.js"></script>
  <script src="assets/packages/fido2_client/assets/js/cbor.js"></script>
  ...
</body>

```

## FAQ

1. __Q: I am getting a white screen when I call `initiateRegistration` or `initiateSigning`. How do I fix this?__
    Solution: Please check that you have hosted the assetlinks file correctly. Make sure you follow the steps for that correctly.

## See Also

- [W3 WebAuthn Spec](https://www.w3.org/TR/webauthn/#webauthn-relying-party)
- [Mozilla Web Authentication Docs](https://developer.mozilla.org/en-US/docs/Web/API/Web_Authentication_API)
- [Fido2ApiClient API Reference](https://developers.google.com/android/reference/com/google/android/gms/fido/fido2/Fido2ApiClient)
- [Introduction to WebAuthn API](https://medium.com/@herrjemand/introduction-to-webauthn-api-5fd1fb46c285)



## TODO:

- publish to pub.dev
- make sure APIs line up for web and android implementations