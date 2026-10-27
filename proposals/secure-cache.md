# Secure Storage API

WebExtension proposal to allow the secure storage of small amounts of data in platform specific locations.

## Champions

- Christian R. (1Password) - [@simpleplaces](https://github.com/simpleplaces/), [email](mailto:smp@pengin.systems)
- Ben Greenberg (Aglide) - [@nebbles](https://github.com/nebbles/), [email](mailto:github@greenberg.io)

## Motivation

Extensions such as credential managers, messaging apps, and cryptocurrency wallets handle sensitive user data. Therefore, a common choice is to only persist encrypted versions of the data to places like localStorage, IndexedDB, or a server. The decryption keys needed to use this data at runtime are stored only in extension memory, meaning that to access the data after a browser restart, the user must be prompted for whatever the key is derived from, usually a long and hard to type password.

Users understandably prefer not to type this, so in desktop applications, a common pattern is to store the decryption key in a system store (e.g. macOS Keychain or Windows Credential Manager). The system can be told to [only reveal the secret after user verification have been performed](https://developer.apple.com/documentation/security/keychain_services/keychain_items/restricting_keychain_item_accessibility?language=objc#2974973), meaning the application can request the secret but access is only granted if the user that stored it is physically present. This pattern is good for users but currently only possible in extensions by communicating with a native application over [Native Messaging](https://developer.mozilla.org/en-US/docs/Mozilla/Add-ons/WebExtensions/Native_messaging) to leverage platform-specific secure storage solutions. This is because access to both hardware-based key stores and user verification prompts are not exposed by existing web APIs.

However, this approach requires users to install a separate native application, which can be a barrier to adoption and adds complexity to the extension's architecture.

In a similar scenario, enterprises often want to use Single Sign-On (SSO) for the managed users in their organisation, rather than a password. This means there is no option for the user to provide a key-derivable password. In this case, device-specific encryption keys must generated after authorisation (e.g. "device approval") and can be used going forwards to decrypt the user's data on-device (in combination with an assertion from an identity provider).

There are other less demanding scenarios, such as storing authentication tokens, that could also benefit from dedicated secure storage.

## Proposal

We propose a new namespace, `browser.secureStorage`, which would implement the same interface (defined below) on both `browser.secureStorage.persistent` and `browser.secureStorage.cache` and would use platform-dependent APIs for storing sensitive data, when available:

- macOS/iOS: [Keychain](https://developer.apple.com/documentation/security/keychain_services)
- Windows: [Trusted Platform Module (TPM)](https://docs.microsoft.com/en-us/windows/security/information-protection/tpm/trusted-platform-module-overview)
- Android: [Keystore](https://source.android.com/security/keystore)
- Linux: See FAQ

Data stored using this API must be strongly bound to the extension's identifier (origin) and browser identity to prevent unexpected contexts from obtaining sensitive values. It also will not be persisted in an insecure form persistently (such as on-disk) by explicit browser action.

### Developer Story

We intend to position the Secure Storage API as the recommended solution for all extension developers who need to store sensitive or secret data. Even in browsers that only implement limited storage support (i.e. only implementing `browser.secureStorage.cache`), its use enhances all users' baseline security. Reasons include:

- `browser.storage.session` does not have a specific threat model and treats all values identically regardless of a developer's sensitivity classification.
- In-memory implementations can implement OS-specific memory hardening protections that extension developers cannot.
- All extensions using the secure storage APIs will receive costless security and UX enhancements when the underlying browser improves feature support.

In the expected use cases, there are no downsides to using `browser.secureStorage.cache` as a replacement to existing APIs like `browser.storage.session` for all secrets management. Where developers need stronger assurances of data durability, they can use only the `browser.secureStorage.persistent` namespace.

As a result, developers offload a large burden of tracking platform-specific security concerns and end users benefit from improvements during their regular browser updates. For example, this widely applicable pattern could be encouraged:

```js
async function storeSecret(id, value, mustPersist) {
  const supportsPersistent = Boolean(browser.secureStorage?.persistent);
  if (mustPersist && !supportsPersistent) {
    throw new Error("Persistent storage not available");
  }

  const store =
    browser.secureStorage?.persistent ?? browser.secureStorage?.cache;

  if (store) {
    await store.set(id, value);
  } else {
    // Old browser version
    await browser.storage.session.set(id, value);
  }
}
```

When persistence is required but not available, web extensions can fall back to the existing mechanisms that they use today for the long-standing limitation. Otherwise, the best location is selected automatically.

### Threat Model

Data written through the Secure Storage API should be resistant to malicious threats operating with the same level of permissions as the browser or user agent itself on an otherwise properly configured OS without default platform safeguards disabled. This rules out attackers with elevated permissions ("admin-level attacks") and constrains the scope to what is commonly referred to as "same-user" attacks.

The only spaces trusted to handle the sensitive data are the browser process itself and the operating system/platform the browser is required to trust and interact with. Other processes and other extensions inside the same browser should be assumed dishonest. Once data leaves one of these spaces (for example, to be persisted for an extended period) it must be encrypted (or similar) and tamper-resistant to observers outside the small trusted scope.

These examples, while non-exhaustive, shouldn't allow compromise of stored data:

- A malicious entity scanning the browser profile's persistent data on-disk.
- A malicious entity installing a malicious browser extension with an identifier spoofed to look like a real extension which has stored data in the past.
- An extension requesting a secret with the same identifier as one stored by a completely different extension earlier.

### Durability

Extension developers can select which namespace they require based on data durability guarantees, and can check for namespace availability in the usual ways.

The `browser.secureStorage.persistent` namespace makes equivalent guarantees to other Storage APIs (e.g. `browser.storage.local`) on the durability of data persistence, with the added security guarantees associated with this API.

For the `browser.secureStorage.cache` namespace, extension developers should not assume that data will be persisted for longer than the browser session (similar to the `browser.storage.session` API). The browser, operating system, or underlying platform hardware may invalidate or destory stored data at any point.

It remains general guidance to all developers, as with other Storage APIs, that developers have alternative storage locations for data stored with the Secure Storage APIs. For example, backup key-wrapping keys for decrypting the data.

Since `browser.secureStorage.cache` provides limited durability guarantees, browser agents can implement this API quickly by ignoring quirks and other problems of the underlying system the browser executes on, such as TPM resets and hardware component swaps, while remaining specification compliant.

Important: Values may be removed sooner than durabilty permits, if an extension uses the `expiry` option when storing a secret.

### Fallbacks

In the event the OS platform doesn't provide key storage APIs which enable greater security or more durable persistence (such as Linux), a browser would not offer the `browser.secureStorage.persistent` API, however it could keep data written through `browser.secureStorage.cache` inside of its own process memory (wrapped in any OS-specific guards available) until the browser is closed.

For developers who need to rely on more durable secure storage, feature discovery of the `browser.secureStorage.persistent` namespace will determine whether they have suitable storage available.

This provides several compounding advantages over exclusively using platform storage:

- APIs can be made available over the widest set of platform possible immediately, allowing extensions to adopt it faster without their own detection logic.
- The new `secureStorage.cache` will still be more durable then an extension's own memory, especially in the context of MV3 extensions which may be restarted at any point.
- The new APIs will have higher ambient security assurances compared to session storage, which lets `browser.secureStorage` always be the go-to recommendation for keys.

### API

```ts
interface Browser {
  secureStorage: SecureStorage;
}

interface SecureStorage {
  cache: SecureStorageArea;
  persistent: SecureStorageArea;
}

interface SecureStorageAreaInfo {
  uvAvailable: boolean;
}

interface SecureStorageArea {
  /**
   * Gets information about the API implementation.  This can also
   * be used to determine whether the API is available.
   */
  getInfo(): Promise<SecureStorageAreaInfo>;

  /**
   * Stores a key-value pair in secure storage area.
   *
   * If the ID has already been used, the value will be overwritten. The ID
   * must only be unique within the extension.
   *
   * If `uvRequired` is set, the user will be required to verify their identity
   * before get(id) resolves.
   */
  set(
    id: string,
    value: BufferSource,
    options?: {
      uvRequired?: boolean;
      expiry?: Temporal.Duration;
    }
  ): Promise<void>;

  /**
   * Retrieves IDs of all stored values.
   *
   * If no values are stored, this will return an empty array.
   */
  getIds(): Promise<string[]>;

  /**
   * Retrieves a value from the secure storage area. If the value does not
   * exist, returns undefined.  If `uvRequired` was set when the value was
   * stored, this will prompt the user for verification before returning the
   * value.
   *
   * If the value was stored with `uvRequired: true`, user verification will be
   * will be applied, regardless of the `options.uvRequired` value supplied.
   *
   * If user verification fails, this will reject with an error.
   */
  get(
    id: string,
    options?: {
      uvRequired?: boolean;
    }
  ): Promise<ArrayBuffer | undefined>;

  /**
   * Deletes one or more values from secure storage area.
   *
   * If the value does not exist, the promise will resolve successfully.
   */
  remove(id: string | string[]): Promise<void>;

  /**
   * Deletes all values from the secure storage area.
   *
   * If there are no values stored, this will resolve successfully without
   * doing anything.
   */
  clear(): Promise<void>;
}
```

Example usage:

```ts
const keyMaterial = crypto.getRandomValues(new Uint8Array(32));

// Persistent will outlast browser restarts,
// but developer-provided `expiry` will limit the lifetime of the key
await browser.secureStorage.persistent.set("tempEncryptionKey", keyMaterial, {
  uvRequired: true,
  expiry: Temporal.Duration.from({ days: 14 }),
});

// This will await until the user is verified by the platform
const rawKey = await browser.secureStorage.persistent.get("tempEncryptionKey");
if (!rawKey) throw new Error("Encryption key missing or expired");

// Imported for use in a WebCrypto operation
const encryptionCryptoKey = await crypto.subtle.importKey(
  "raw",
  rawKey,
  "AES-GCM",
  false,
  ["encrypt", "decrypt"]
);

// etc...
```

#### SecureStorageArea.getInfo

Enables an extension to determine what capabilities the current platform
implementation its running on provides.

This may be important to extensions that need to rely on the availability of user verification (`uvAvailable`).

#### SecureStorageArea.set

Stores the provided string `value` in the specified storage location.

If `value` is larger then 1KB, an error may be thrown. Large amounts of data should not be stored with this API, as it may encounter platform-specific slowdowns or limitations (i.e. `mlock` limits). In order to improve cross-platform compatibility, a lower bound may enforced by the browser.

If `set()` is called with the same `id` all `options` are replaced with the latest call.

Additional options:

- `uvRequired` boolean (optional, defaults `false`): when omitted, the value can be retrieved without any user verification prompt. When set to `true` the browser must first perform user verification. If user verification fails, or is not available, the API throws an `Error`. This value does not cause user verification to be triggered when setting the value.

- `expiry` [Duration](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Temporal/Duration) (optional): when omitted, the secret will remain available as long as the underlying implementation can uphold. Calling `set()` with the same `id` will overwrite the `expiry` even if left undefined (no expiry).

#### SecureStorageArea.getIds

Returns an array of all stored IDs. Any ID where the value has expired is omitted from the array.

#### SecureStorageArea.get

Retrieves the stored value, if available. Otherwise returns `undefined` on no value, or throws error on failure.

If `uvRequired` was set to `true` on the most recent `set(id)` call, the browser will only provide it if the user successfully verifies themself through a strong implementation-specific authentication ceremomy. If user verification fails an error is thrown. User verification can take time, and so callers should expect and handle a returned Promise taking a while to settle.

If the the user verification prompt or method presented by the platform is cancelled, the call will throw an error informing the caller of such. The extension may retry the retrieval as soon as it needs to.

If `expiry` was set on the most recent `set(id)` call and the duration has elapsed, the call returns `undefined` (i.e. no value).

The browser agent may invalidate and remove the secret at the following points:

- when get() is called and the value has expired
- at the point the value is due to expire
- due to durability limitations (based on secure storage area used)

#### SecureStorageArea.remove

Removes an entry from the secure storage area given an ID. No user verification is required regardless of the secret's original configuration.

#### SecureStorageArea.clear

Deletes all values from the specified secure storage area.

If there are no values stored, this will resolve successfully without doing anything.

### New Permissions

The `secureStorage` API uses the existing "storage" permission, as the sematic meaning to an end-user is the same, whereas the API difference is a technical one.

Introducing a new permission scope (e.g. "secureStorage") was considered, but was discarded to avoid confusing users about the difference which is largely technical.

### Manifest File Changes

No new manifest fields are required for this API.

## Security and Privacy

### Exposed Sensitive Data

The API does not expose any new sensitive data or personally-identifiable information to the extension, that wasn't already previously known since this is a storage API. The API does not allow for cross-extension storage.

### Abuse Mitigations

As with standard storage APIs, there could be a limit imposed by browsers on the maximum amount of data that can be stored on a per-ID basis (see [#SecureStorageArea.set] above, or _How large could the stored data be?_ in the FAQ below), and/or a maximum total storage limit for the API.

### Additional Security Considerations

This API is designed to enable extensions to store the most sensitive information they may handle, such as encryption keys. As such, it is critical that browser implementations handle this data with the utmost care, and the best guarantees available. Browsers should not implement this API if they cannot provide strong guarantees around the security of the stored data, and the user verification process (e.g. where platform/hardware-backed options are not available).

## Alternatives

### Open Web API

This API could belong on the open web. However, the most common use cases in practice today are for applications that need to store an encryption key.

Since these keys are particularly sensitive, application developers are likely to continue using an extension-based approach to minise the exposure of these keys to more hostile environments.

Additionally, Secure Storage is modelled off the existing Storage APIs that are only available to extension developers.

Practically, a Web Extension API is the highest leverage point to introduce such an API. Web Extensions can also make the API available to trusted web origins if desirable, without exposing the API to all websites.

## FAQ

### Are there any extensions that would use this?

The [1Password extension](https://chrome.google.com/webstore/detail/1password-%E2%80%93-password-mana/aeblfdkhhhdcdjpifhhbdiojplfjncoa?hl=en) is available for Chrome, Firefox, Safari, and Edge and has more than 2M weekly active users in the Chrome Web Store alone. They would be quick to implement this in conjunction with or in place of our current Native Messaging solution. Additionally, it may provide a better durable store for client-side secrets such as the Secret Key across 1Password's web apps.

The [Aglide](https://aglide.com/) browser extension, used for handling credentials, encryption keys, and web sessions, would immediately use this API to enable a new product architecture that no longer relies on a desktop application being installed just to access platform APIs for secure storage.

The [Dashlane browser extension](https://chromewebstore.google.com/detail/dashlane-%E2%80%94-password-manag/fdjamakpfbbddfjaooikfcpapjohcfmg), also a password manager with over 1M weekly active users in the Chrome Web Store, currently provides a remember me and biometric authentication feature in its extension to make repeated authentication easier. They would also be very quick to implement this in place of their solution which requires a network connection, as they don’t leverage a native application to interface with native storage.

The Keeper Security [browser extension](https://chrome.google.com/webstore/detail/keeper%C2%AE-password-manager/bfogiafebfohielmmehodmfbbebbbpei?hl=en&authuser=0) would also immediately use this API for storage of client-side generated encryption keys. Even though a desktop app is optionally provided to users, Keeper does not wish to use native app dependencies for biometric authentication and key storage. Keeper would also prefer to prompt users in the browser extension directly for biometric authentication instead of relying on a desktop app.

The [MetaMask](https://metamask.io/) [browser extension](https://chromewebstore.google.com/detail/metamask/nkbihfbeogaeaoehlefnkodbefgpgknn), a cryptocurrency wallet and a gateway to blockchain applications with over 12 million monthly active users, would also greatly benefit from using this API as a more secure way to store a user's private keys while their wallet is locked. They would also be interested in offering users the choice to use biometrics as a simpler way to unlock their wallet.

We expect that the API would benefit a wide range of extensions, providing value far beyond credential management. If you'd like to add your extension or use case, feel free to open a PR.

### Why not use a Web API?

There have been proposals in the past which are now stale/abandoned, including:

- A [WebKit feature request (October 2020)](https://bugs.webkit.org/show_bug.cgi?id=217929) for the ability to store secrets protected by Face ID and Touch ID.
- A “[Web API For Accessing Secure Element (Global Platform, September 2016)](https://globalplatform.github.io/WebApis-for-SE/doc/)”, which seems to have been abandoned.
- A [Hardware Based Secure Service](https://github.com/w3c/websec/blob/gh-pages/hbss.html) proposal, which is [no longer active](https://lists.w3.org/Archives/Public/public-hb-secure-services/2018Mar/0001.html).

We believe that introducing this capability as a Web Extension API is the most pragmatic approach as it would be adopted nearly immediately by numerous extensions as a direct alternative to improve the product experience and reduce complexity for their users.

Additionally, the Web Extension API surface is more appropriate for the use cases of securely storing sensitive data such as encryption keys. Extensions are much easier to secure, and remain a reasonably low barrier to entry to users of these types of applications.

Application developers would probably choose to use an extension-based approach event if these capabilities were available on the web, to minimise the exposure of these keys to more hostile environments.

### Why not use WebAuthn?

Whilst WebAuthn with the addition of the PRF extension does provide a faster mechanism for authenticating a user and retrieving a secret, than passwords, and support for PRF is improving (see [WebAuthn PRF Support by Corbado]). It is insufficient for two reasons:

[WebAuthn PRF Support by Corbado]: https://www.corbado.com/blog/passkeys-prf-webauthn

1.  It relies on a more complicated user flow with no control over user verification. Passkeys trigger the browser, or operating system, to request the user verifies themselves. The application developer has no ability to avoid user verification where it is not necessary (e.g. for less sensitive data, or background tasks).

2.  In some use cases, application developers generate client-specific keys that are then used during a pairing process to become 'trusted'. From that point on, the key can be used as a form of encryption and authentication deliberately to avoid other forms of user authentication.

    For example, a product that relies on the OIDC/SAML protocol for authentication requires the user has a 'trusted device' (with a unique durable encryption key). For this model to work, there must be:

    (a) strong guarantees that the key material that the 'trusted client' is storing is not accessible to other applications (as such, standard web storage APIs are not suitable for this use case since they are not encrypted at rest), and

    (b) no dependence on deriving a key from other material provided by the user, since they can only provide an assertion which is unsuitable for key derivation.

    In these systems, after the user has completed SAML authentication, and the assertion has been verified, the client-unique encryption key can be used to decrypt the end-to-end encrypted data.

Additionally, it is worth noting that certain enterprises ban the use of passkeys as an authentication method.

### Could Device-Bound Session Credentials (DBSC) be used?

Device Bound Session Credentials (DBSC) ([DBSC Draft]; [DBSC, Google]) is a proposed API that would allow the browser to handle secure storage and refreshing of cookies on behalf of websites.

Unfortunately, DBSC is not a suitable alternative as it does not expose the primitives necessary for application developers to implement the use cases outlined in this proposal, specifically the combination of authentication and encryption at the application layer.

[DBSC Draft]: https://w3c.github.io/webappsec-dbsc/
[DBSC, Google]: https://developer.chrome.com/docs/web-platform/device-bound-session-credentials

### What’s the plan for durability on Linux?

Linux is one of the platforms where this API would be most beneficial. However, it’s less clear which API would be appropriate here. APIs such as the [FreeDesktop Secure Storage API] exist, but not all of the available APIs are guaranteed to be hardware backed, restricted by user verification, and other options have different limitations. Feedback is welcome on what options may be suitable to increase usability on Linux.

As per the specification, browsers should not implement this API on platforms where they cannot provide the security guarantees of the API.

[FreeDesktop Secure Storage API]: https://www.gnu.org/software/emacs/manual/html_node/auth/Secret-Service-API.html

### Why is it useful to allow storing secrets without biometric authentication?

Millions of users use devices without the hardware for biometric authentication, but install extensions which would be likely to implement this API. User verification is not strictly a requirement for all the use cases of securely storing secrets, and that determination falls with the application developer to decide based on their requirements.

### Are there any alternatives? How do vendors like 1Password implement biometrics today?

1Password currently use a solution based on Native Messaging where the extension communicates with a native app. This pattern is also used by other credential management apps. These native apps are able to use system APIs to request biometrics and access hardware based storage. This has limitations, however. Not all users have the native application (users of Chromebooks and iOS Safari, to name a few) and implementing reliable cross-process messaging is a challenge compared to simple JavaScript API patterns that don't require bridging processes from different app vendors.

### Could secrets be stored in localStorage, IndexedDB, or Web Extension Storage instead?

Credential managers have historically chosen not to do this because it makes the secrets too easy to access. While a compromised machine is hard to protect against, data in these APIs can be trivially accessed with a terminal and no authentication, or by another application without any user interaction. These other storage mechanisms allow an “evil maid” attack if a computer is left unlocked, or for another application to read the secret from disk independently of the browser.

While the secret must be in memory while the extension is in use, extensions like password managers aggressively enter a “locked state” after inactivity where secrets are cleared from memory. This means that after only a short period of idle time such an attack would require biometric authentication.

`storage.session` does not provide the durability guarantees necessary to be a viable option for securely storing secrets in all use cases, particularly ones go through a pairing process to be recognised as trusted to avoid further authentication in future.

### Why not use the existing StorageArea interface for the schema specification?

There are concerns specific to secure storage that are not supported by the more generic `StorageArea` interface, (e.g. user verification).

Additionally, the StorageArea interface carries baggage unnecessary for the bespoke requirements of handling secure materials (e.g. getting/setting multiple keys at once, support for callbacks).

It is also helpful and educational for application developers to see the distinction between the two.

### Why use Secure Storage 'Areas'?

Following the existing Storage API’s pattern of separate storage areas with a shared interface leaves room for additional secure storage areas in the future. For example, a `browser.secureStorage.sync` area could support synchronisation across devices, potentially using secure platform-specific services such as iCloud Keychain, whilst remaining explicit for the developer to opt into the behaviour.

### How large could the stored data be?

The expectation is that implementation specific limits would be set. Only a very small allowance per value (e.g. ~5 KB for ML-DSA private key) is needed for the known use cases.

### Why bytes instead of string data types?

Whilst a string may be more widely recognised for a general developer audience, the primary use case for this API is to store sensitive material, such as encryption keys.

The [Web Crypto API], which is most similarly related in use case, routinely handle an `ArrayBuffer`, a `TypedArray` or a `DataView` object that contain the data for their parameter types. E.g. https://developer.mozilla.org/en-US/docs/Web/API/SubtleCrypto/importKey#keydata

Using binary storage formats has the additional benefits of being closer aligned to underlying platform primitives for handling sensitive material, so we would alleviate serialisation costs.

[Web Crypto API]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Crypto_API

## References

- The issue that initiated this proposal:
  https://github.com/w3c/webextensions/issues/154
- Discussion at TPAC November 2025:
  https://github.com/w3c/webextensions/blob/main/_minutes/2025-11-13-wecg-tpac.md#secure-storage
- Discussion at WECG Face-to-Face Meeting 7 April 2026:
  https://github.com/w3c/webextensions/blob/main/_minutes/2026-04-07-london-f2f.md#secure-storage-api

## Remaining Work

Remaining work is tracked in the WECG [issue tracker](https://github.com/w3c/webextensions/issues).
