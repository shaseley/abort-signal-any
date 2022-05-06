# [Self-Review Questionnaire: Security and Privacy](https://w3ctag.github.io/security-questionnaire/)

This questionnaire has [moved](https://w3ctag.github.io/security-questionnaire/).

For your convenience, a copy of the questionnaire's questions is quoted here in Markdown, so you can easily include your answers in an [explainer](https://github.com/w3ctag/w3ctag.github.io/blob/master/explainers.md).

> 01.  What information might this feature expose to Web sites or other parties,
>      and for what purposes is that exposure necessary?

This API does not expose any user information. The only information it exposes
is the relationship between `AbortSignal`s on a page.

> 02.  Do features in your specification expose the minimum amount of information
>      necessary to enable their intended uses?

Yes.

> 03.  How do the features in your specification deal with personal information,
>      personally-identifiable information (PII), or information derived from
>      them?

This feature does not deal with any personal information or PII.

> 04.  How do the features in your specification deal with sensitive information?

This feature does not deal with any sensitive information.

> 05.  Do the features in your specification introduce new state for an origin
>      that persists across browsing sessions?

No.

> 06.  Do the features in your specification expose information about the
>      underlying platform to origins?

If engines do different optimizations for pruning signals that can no longer
cause downstream aborts, engine differentiation might be possible by timing GC
(via `FinalizationRegistry`). But, this information can already be obtained in
easier ways.

> 07.  Does this specification allow an origin to send data to the underlying
>      platform?

No.

> 08.  Do features in this specification enable access to device sensors?

No.

> 09.  Do features in this specification enable new script execution/loading
>      mechanisms?

No.

> 10.  Do features in this specification allow an origin to access other devices?

No.

> 11.  Do features in this specification allow an origin some measure of control over
>      a user agent's native UI?

No.

> 12.  What temporary identifiers do the features in this specification create or
>      expose to the web?

It does not create or expose any such identifiers.

> 13.  How does this specification distinguish between behavior in first-party and
>      third-party contexts?

It does not make such a distinction.

> 14.  How do the features in this specification work in the context of a browser’s
>      Private Browsing or Incognito mode?

This features works the same in those contexts.

> 15.  Does this specification have both "Security Considerations" and "Privacy
>      Considerations" sections?

No. The intention is for this feature to be added to the DOM spec.

> 16.  Do features in your specification enable origins to downgrade default
>      security protections?

No.

> 17.  How does your feature handle non-"fully active" documents?

The behavior of the API does not change depending on a document's active state.
`AbortSignal` objects that are associated with a window whose document is not
fully active can still be aborted and affect the composite signal.

> 18.  What should this questionnaire have asked?
