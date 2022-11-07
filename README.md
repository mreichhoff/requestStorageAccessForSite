# requestStorageAccessForOrigin Explainer


## Authors:
* [Matt Reichhoff](https://github.com/mreichhoff) (mreichhoff@chromium.org)
* [Kaustubha Govind](https://github.com/krgovind) (kaustubhag@chromium.org)
* [Johann Hofmann](https://github.com/johannhof) (johannhof@chromium.org)

## Participate

* [Issues on this repo are welcome!](https://github.com/mreichhoff/requestStorageAccessForOrigin/issues)
* Feedback on [the issue in the storage access repo](https://github.com/privacycg/storage-access/issues/107) is also welcome!

## Introduction

Enabled-by-default cross-site cookie access is in the process of being deprecated by several major browsers. Multiple substitutes have been proposed, like [the Storage Access API](https://webkit.org/blog/8124/introducing-storage-access-api/) and the [SameParty cookie attribute](https://github.com/WICG/first-party-sets#sameparty-cookies-and-first-party-sets) in the [First-Party Sets](https://github.com/WICG/first-party-sets) proposal.

However, `SameParty` [has been abandoned](https://github.com/WICG/first-party-sets/issues/92), and the Storage Access API is primarily [intended](https://github.com/privacycg/storage-access/issues/122) for authenticated embeds, a use case which entails `<iframe>` use. This raises questions like:



*   How can legacy content directly embedded in a document rely on cross-site cookies?
*   How can top-level sites ensure their cross-site content can get the access it needs early enough in the page lifecycle to avoid user experience degradation?

Both Firefox and Safari have run into this issue before and solved it through the application of an internal-only “requestStorageAccessForOrigin” API([1](https://bugzilla.mozilla.org/show_bug.cgi?id=1724376),[2](https://github.com/WebKit/WebKit/commit/e0690e2f6c7e51bd73b66e038b5d4d86a6f30909#diff-1d194b67d50610776c206cb5faa8f056cf1063dd9743c5a43cab834d43e5434cR253)), which is applied on a case-by-case basis by custom browser scripts (Safari: [3](https://github.com/WebKit/WebKit/blob/a39a03d621e441f3b7ca3a814d1bc0e2b8dd72be/Source/WebCore/page/Quirks.cpp#L1065),[4](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/page/Quirks.cpp#L1217) Firefox: [5](https://phabricator.services.mozilla.com/D129185),[6](https://phabricator.services.mozilla.com/D124493),[7](https://phabricator.services.mozilla.com/D131643)). While such an internal API should not be treated as setting precedent, it could be evidence that such an API would be useful for developers.

This document proposes a version of this API that could be web-exposed, with a requirement of additional trust signals and security controls to ensure safety.


## Goals


*   Enable legacy use cases by allowing requests for cross-site cookies to be made from top-level browsing contexts, with access applying at the page level (rather than to a specific frame). 
    *   Note that page specificity is similar to [the prior behavior](https://github.com/privacycg/storage-access/issues/122) of `requestStorageAccess`. That behavior is being changed to focus the API on authenticated embed use cases and to improve security, which leaves top-level access a gap.
*   Ensure that the security, privacy, and abuse concerns with legacy cross-site cookie behavior remain mitigated.

## Non-goals

*   Maintaining parity with the `SameParty` cookie proposal, particularly in characteristics like synchronicity, is not a goal.
*   Re-creating legacy passive cross-site cookie behavior is not a goal; additional guardrails must be in place.
*   Although prior art like `requestStorageAccess` will inform the proposal, it is intended to be a separate API. The access it grants (which would be page-level) would be separate from that obtained by a successful `requestStorageAccess` call (which would be applicable only to the calling frame).
*   Much like the implementation-defined steps in `requestStorageAccess`, disallowing or otherwise undermining user agent-defined steps to determine whether to grant access is not a goal. Every user agent should still be free to take custom steps, including prompting users or other heuristics.

## Proposed API: requestStorageAccessForOrigin

Since the `requestStorageAccess` API was [originally designed](https://webkit.org/blog/8124/introducing-storage-access-api/) for authenticated embeds, it has requirements that are perhaps uniquely well-suited for that use-case. Specifically, it is only possible for the embeddee to request access, and only from within `<iframe>` elements that have received user interaction. However, these restrictions place adoption costs on websites that have functionality deployed across multiple sites, where cross-site subresources may include images or JavaScript files instead of `<iframe>`-embedded documents. [A similar discussion](https://github.com/privacycg/storage-access/issues/3) previously resulted in the existing `requestStorageAccess` API operating at the page level, rather than frame-only, though this decision [is being reversed](https://github.com/privacycg/storage-access/issues/122).

This document proposes a similar, but separate, API,` document.requestStorageAccessForOrigin`, which would allow the embedding site to request access it knows it needs on behalf of its embedded content. 

This new API would be somewhat similar to the existing `requestStorageAccess`. It would still require activation, though of the top-level document; would still delegate to per-browser logic, so that each browser can customize the experience to their users’ expectations (for example, user prompts like those of `requestStorageAccess` could be used, or First-Party Sets could help gate access); and would gate a permission that is similar to the prior page-level `requestStorageAccess` grant.

This API could be treated as similar in principle to browser-specific compatibility measures, implemented in [Safari](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/page/Quirks.cpp#L1131-L1163) and [Firefox](https://searchfox.org/mozilla-central/rev/287583a4a605eee8cd2d41381ffaea7a93d7b987/dom/base/Document.cpp#17051), where an internal API is invoked, based on browser-defined domain allowlists, that requests cross-site cookie access on behalf of embedded sites.

Requiring user interaction in `<iframe>` elements helps the original `requestStorageAccess` API deter spam and abuse from both embedders and embeddees, and the requirement that the embeddee call `requestStorageAccess` indicates its willingness to be embedded. To compensate for the lack of these protections with the proposed API, browsers should require additional trust signals, such as FPS membership, to grant access. This is discussed in Privacy & Security Considerations.


## Key scenarios

### Top-level Requests on Behalf of Another Party
In contrast with the existing `requestStorageAccess` API, the proposed extension would allow non-iframe use, and would afford more control to the top-level site over what access is requested when.

With implementation-defined grant logic based on First-Party Set membership, example use could be:

```
<!--
Top-level site: fps-member1.example. fps-member2.example is in the same First-Party Set. other-party.example is not.

Note that all <script> tags below would be new post-3rd party cookie deprecation.

Assume fps-member2.example has previously set two cookies:
Set-Cookie: sameSiteLax=123; SameSite=Lax 
Set-Cookie: sameSiteNone=456; SameSite=None; Secure
-->
<html>
<head>
  <script>
    document.requestStorageAccessForOrigin('https://fps-member2.example')
     .then(
           /*not called;no activation.*/)
     .catch(/*called due to top-level document lacking activation at load time*/);
  </script>
</head>

<body>
<button id='play-button'></button>
<script>
  const playButton = document.getElementById('play-button');
  playButton.addEventListener('click', function(){
    document.requestStorageAccessForOrigin('https://fps-member2.example')
     .then(
       /*
       called;has activation, same First-Party Set is sufficient.
       Cookie `sameSiteNone=456` available. Cookie `sameSiteLax=123` is not.
       Image tags or other assets could be requested: 
       */
       let img = document.createElement('img');
       
       // CORS would be required for the SameSite=None cookies to be attached.
       // This helps protect the embeddee from attacks by the embedder.
       img.crossOrigin = 'use-credentials';

       img.src='https://fps-member2.example/profile_pic.png';
       document.body.appendChild(img);
     )
     .catch(/*not called due to lacking activation*/);
  
    document.requestStorageAccessForOrigin('https://other-party.example')
     .then(/*for v1, rejected; not in the same First-Party Set*/)
     .catch(/*called due to not being in the same First-Party Set*/);
  });
</script>
<!--initial page load: would not have cookies; see event handler above-->
<img src='https://fps-member2.example/profile_pic.png'>
<!--
Would have access to its cross-site cookies, after the button is clicked and the process runs as expected. Future page-loads would remember this for some TBD period of time.
-->
<iframe src='https://fps-member2.example'></iframe>
<!--
Would require a separate call to requestStorageAccessForOrigin, because of origin, not site, scoping.
-->
<iframe src='https://sub-domain.fps-member2.example'></iframe>
</body>
</html>
```


### Authenticated Embeds

Authenticated embeds are the primary use case targeted by the `requestStorageAccess` API. See the numerous [examples available elsewhere](https://developer.mozilla.org/en-US/docs/Web/API/Document/requestStorageAccess#examples) for sample code. This proposal does not modify the behavior or intended use cases of the existing `requestStorageAccess` API.


## Detailed design discussion


### Proposed Draft Spec Addition

The proposed spec could include a set of steps for the browser to follow that are somewhat similar to those done with <code>[requestStorageAccess](https://developer.mozilla.org/en-US/docs/Web/API/Document/requestStorageAccess#conditions_for_granting_storage_access)</code>. The spec could include a function that takes a <code>string</code> as the origin:


```
function requestStorageAccessForOrigin(origin)
```


Where a draft set of steps could be:



1. If the browser is not processing a user gesture, reject.
1. If the document has a null origin, or if the requested domain is invalid or has a null origin, reject.
1. If the document's frame is not the main frame, reject.
1. If the requested origin is equal to the main frame's, resolve.
1. If the requested `origin` already has been granted access, resolve.
1. Check for embeddee opt-in:
    1. Run implementation-defined embeddee opt-in checks.
        1. FPS, excluding service domains, would be one example
        1. The hope is that, over time, such opt-in signals can be standardized.
    1. If opt-in is not found, reject the requestStorageAccessForOrigin call.
1. Check for user approval, if required:
    1. Check implementation-defined acceptance or rejection steps; if any are triggered, reject the requestStorageAccessForOrigin call or skip to step 4.
        1. Browsers could choose to accept or reject based on their own rules. Allowing sharing within the same FPS, allowing some limited number of implicit grants, or always choosing to prompt users would all be possible.
    1. Prompt the user, return their acceptance or denial.
1. If acceptance is returned, save a permission for the pair `{top-level site, requested origin}`. Note that the permission would be separate from the permission granted by `requestStorageAccess`.

Fetch could then be modified to include cross-site cookies when appropriate (though the modification may depend on [cookie layering changes](https://github.com/httpwg/http-extensions/issues/2084)). A draft of such a spec change follows:

1. At request time, if the request is cross-site and the appropriate permission for `{top-level site, requested origin}` exists, attach cookies only if all of the below checks are met:
    1. Run a modified ancestor check, like that of the site for cookies algorithm, but allowing ancestors that are same-site with either the embeddee or the top-level site, to prevent any unrelated iframes from getting access. This is recommended in . Permission policy set by the top-level document can opt trusted `<iframe>`s out of this protection.
    1. If the request is for a subresource (i.e., not a navigation), the request must be CORS-enabled. In other words, a plain `<img>` or `<script>` without a `crossorigin` attribute would not have cross-site `SameSite=None` cookies attached, regardless of whether access had been granted. Similarly, a `fetch` or `XHR` request would omit cross-site `SameSite=None` cookies unless CORS was enabled.
    1. The cookies to be included must be marked `SameSite=None`. In other words, the cookies must have been explicitly opted in by the requested domain. Cookies with any other `SameSite` option are ignored and not sent, regardless of whether a grant exists.

### Plural vs Singular API


One could imagine either a singular API:


```
// follows the existing 1x1 pattern established by requestStorageAccess
// returns a Promise similarly
requestStorageAccessForOrigin("origin.example")
```



Or a plural one:


```
// allows, for example, use of Promise.all()
requestStorageAccessForOrigins(["origin1.example","origin2.example"])
```

Given the increased complexity with potential user prompts, and the 1x1 nature of the existing `requestStorageAccess` API, the singular version is recommended. Note that it is also simpler to switch from singular to plural than from plural to singular, should that ever become necessary. 


### Site vs Origin Scope

The existing requestStorageAccess API is scoped to site for the top-level page in both Safari and Firefox. The embeddee, however, is scoped to site in Safari and origin in Firefox.

This has been [the subject of debate](https://github.com/privacycg/storage-access/issues/39). This proposal is to scope the grant similarly to Firefox, with a key like: `{top-level site, requested origin}`. This does mean repeated calls would be required for origins like www.site.example and site.example.

A previous version of this proposal suggested embedded site scoping. See [a recent security analysis](https://github.com/privacycg/storage-access/issues/113) for information about the benefits of embedded origin scoping.


## Considered alternatives


### Browser-Specific Allowlists

As discussed in the introduction, both Firefox and Safari have implemented an internal-only “requestStorageAccessForOrigin” API([1](https://bugzilla.mozilla.org/show_bug.cgi?id=1724376),[2](https://github.com/WebKit/WebKit/commit/e0690e2f6c7e51bd73b66e038b5d4d86a6f30909#diff-1d194b67d50610776c206cb5faa8f056cf1063dd9743c5a43cab834d43e5434cR253)), that is applied on a case-by-case basis by custom browser scripts (Safari: [3](https://github.com/WebKit/WebKit/blob/a39a03d621e441f3b7ca3a814d1bc0e2b8dd72be/Source/WebCore/page/Quirks.cpp#L1065),[4](https://github.com/WebKit/WebKit/blob/main/Source/WebCore/page/Quirks.cpp#L1217) Firefox: [5](https://phabricator.services.mozilla.com/D129185),[6](https://phabricator.services.mozilla.com/D124493),[7](https://phabricator.services.mozilla.com/D131643)).

This approach is not preferred, as it favors websites that have access to the corresponding browser's developers, and may not produce equitable outcomes. In addition, it does not allow site authors to proactively fix issues without interacting with browser developers.


### Forward Declaration

Forward declaration of storage access requirements [remains under discussion](https://github.com/privacycg/storage-access/issues/83). This proposal is not intended to replace (or otherwise take a stance on) that option, which may still be relevant in the future. Instead, this proposal attempts to resolve adoption considerations that aren’t directly resolved by the forward declaration-based design, which requires a top-level navigation to the embedded origin and is intended to address identity/login use-cases.


### HTTP Header Access Requests

The proposed API introduces a dependency on JavaScript for a site wanting to use cookies within a First-Party Set context. While this may be a fairly small selection of sites, the sites and their clients may not have previously required JavaScript, which increases the effort for adoption.

In a similar way to using the `allow` attribute on an `iframe` to enable specific features for a domain map to an equivalent Permissions Policy, it would be possible to provide an equivalent for a storage access call.

For example, the JavaScript call:


```
document.requestStorageAccessForOrigin('https://fps-member2.example')
```


Could be equivalent to an HTTP header, possibly using [permissions policy syntax](https://developer.chrome.com/en/docs/privacy-sandbox/permissions-policy/):


```
Permissions-Policy: storage-access=(self "https://fps-member2.example")
```


While this option may be attractive in the future, and would be doable in a First-Party Set membership-driven approval system, it is outside the scope of this document. Such an option is instead considered a potential future work item.


## Privacy and Security Considerations

By exposing a new access-granting API, especially one that relaxes the current `<iframe>` requirement of requestStorageAccess and allows for arbitrary domains to be passed in, care must be taken not to open additional security issues or abuse vectors. It is easy to imagine an untrusted top-level domain requesting access on behalf of an unrelated origin. Such access (or even asking for such access) could be reputation-damaging, or enable CSRF, clickjacking, or other attacks against the embeddee.

Generally, there are two separate issues that must both be addressed: abuse and security concerns.


### Abuse Prevention

There is a risk of abuse of the API by top-level documents, for example by attempting to associate an embeddee with an unrelated embedder (e.g., showing a prompt that would link `we-hate-puppies.example` with `reputable-news-site.example` could harm the news site’s reputation). Excessive prompting must also be avoided; this is especially true because of the ubiquity of third-party scripts included in top-level contexts. 

To mitigate abuse concerns, browsers must seek additional trust signals. Gating access on First-Party Sets is one mechanism by which this concern can be mitigated. Note that First-Party Sets guarantee mutual exclusivity, preventing a single domain from linking data across sets, and that there [are policy checks](https://github.com/WICG/first-party-sets#abuse-mitigation-measures) that should ensure a valid relationship between the domains in each set. The service domain subset can also be used to disallow less-privileged domains from requesting access.

Other potential embeddee opt-in mechanisms, especially for those user agents that do not support First-Party Sets, could include:

*   Specification of a `.well-known` configuration that can be checked to ensure embeddee opt-in.
*   Checking the passed-in origin against the origin of the script making the call.

Additional prompt spam abuse mechanisms could be:

*   Limiting the number of calls to the API on a given page load.
*   Monitoring usage in order to quiet permission prompts or auto-reject requests by disruptive sources.
*   Consuming (rather than merely requiring) user activation on the top-level website, which would prevent repeated attempts at gaining access without additional user activity.

Over time, standardization of such signals is desired, though it may also be important to allow user agents latitude to implement such requirements as they see fit, much like is done with `requestStorageAccess` grant logic.

### Security Protections

Besides abuse concerns, security issues must also be addressed; browsers must ensure that SameSite=None cookies are not sent when they shouldn’t be, and that the scope of access is not overly broad.

Note that a much-more-detailed analysis is available in [a recent security analysis](https://github.com/privacycg/storage-access/issues/113). Its recommendations are summarized here:



*   Only cookies marked `SameSite=None` should be granted by the API. This indicates explicit intent on the part of the embeddee to allow cross-site use. By ensuring a default `SameSite` setting of at least `Lax`, browsers can ensure that the embedded resources opted into cross-domain sharing by setting `SameSite` to `None`. 
*   The permission should be scoped to `{top-level site, embedded origin}`. In other words, a grant for `jokes.example.com` should not imply a grant for `auth.example.com`. While this does not stop the top-level site from later requesting `auth.example.com`, it does ensure that the permission is scoped to avoid accidental leakage.
*   For nested resource loads, [a variant of the site for cookies algorithm](https://datatracker.ietf.org/doc/html/draft-ietf-httpbis-rfc6265bis-10#section-5.2.1) should be used to avoid unrelated iframes from using a grant, with a permission policy opt-out, as suggested in.
*   `SameSite=None` cookies granted via requestStorageAccessForOrigin on subresources should only be attached on CORS-enabled requests. For example, an `<img>` without a <code>[crossorigin attribute](https://developer.mozilla.org/en-US/docs/Web/HTML/Attributes/crossorigin)</code> would not have <code>SameSite=None</code> cookies attached, even with a valid grant. This ensures the server is aware of the caller and can react accordingly. Note that CORS is not possible on navigations; this protection is not intended to protect `<iframe>` elements, which have other mechanisms for forbidding embedding (e.g., `x-frame-options` or `CSP`).


#### CSRF Considerations

A side effect of disabling `SameSite=None` cookies is that attacks like CSRF become significantly harder to carry out. While the existing `requestStorageAccess` API already allows a mechanism to opt a specific frame out of this protection, `requestStorageAccessForOrigin` could be used more broadly due to its relaxation of the `<iframe>` requirement. Additionally, `requestStorageAccessForOrigin` is invoked by the embedder, as opposed to the embedded origin which gets access to cross-site cookies. This makes additional opt-in requirements for embedded resources, like those described above, more attractive.


## Stakeholder Feedback / Opposition

*   TBD

## References & acknowledgements

Many thanks for valuable feedback and advice from:

* [Artur Janc](https://github.com/arturjanc)
* The existing [Storage Access API spec](https://privacycg.github.io/storage-access/), the [MDN Storage Access API documentation](https://developer.mozilla.org/en-US/docs/Web/API/Storage_Access_API) and the [Safari documentation](https://webkit.org/blog/8124/introducing-storage-access-api/) were all instrumental in authoring this document.