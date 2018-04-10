# Link::AsValueToContentPolicy changes
AUDIO/TRACK/VIDEO are no longer being returned as TYPE_MEDIA, but instead as
TYPE_INTERNAL

Comment on the TYPE_MEDIA mapping change:
```
It seems like this change in mapping needs the following related changes made or commented upon.  It's very regrettable that nsContentPolicyType does double-duty as both our internal and external type, as it makes things super ambiguous everywhere.  It's made worse by the Link "as" attribute mapping a content-controlled type into the same type-space as authored by privileged system code.  (I don't think that's a security problem, though, since the preload code just seems to use the type for mapping, but it's super sketchy from a typing perspective.  And the "as' values can't mint INTERNAL constant values, but, ugh.)

- CSP_ContentTypeToDirective at https://searchfox.org/mozilla-central/rev/57bbc1ac58816dc054df242948f3ecf75e12df5f/dom/security/nsCSPUtils.cpp#228
  - There's a case for TYPE_MEDIA right now, but no INTERNAL_{AUDIO,TRACK,VIDEO}.  Seems like we either want to add those or call out why we aren't.  Based on the other INTERNAL constants, my guess would be add.

- nsMixedContentBlocker::ShouldLoad at https://searchfox.org/mozilla-central/rev/57bbc1ac58816dc054df242948f3ecf75e12df5f/dom/security/nsMixedContentBlocker.cpp#544
  -



The following cases aren't a problem, but it would be nice if there were comments explaining why.  It's not your problem/not needed for this patch, though:
- ChannelWrapper.cpp's GetContentPolicyType: only operates on external types
```

## Propagation of Link::AsValueToContentPolicy

Callers:
- Link::GetContentPolicyMimeTypeMedia -
  - Link::TryDNSPrefetchOrPreconnectOrPrefetchOrPreloadOrPrerender - Propagates
    it fowrward to:
    - nsPrefetchService::PreloadURI bounces to nsPrefetchService::Preload which:
      - Uses the policy type in an equality test to see if a node is already
        loading.
      - Creates a new nsPrefetchNode if not, where it becomes its mPolicyType.
        - This is propagated into the channel and from there it's propagated
          into the new LoadInfo *WHERE IT IS SAVED AS mInternalContentPolicy*.
          - mInternalContentPolicy is dealt with nicely by:
            - nsContentUtils::IsNonSubresourceInternalPolicyType which doesn't
              care about these changed types, but clearly is meant to understand
              internal types.
            -
  - Link::UpdatePreload
  (MORE)
- nsStyleLinkElement::CheckPreloadAttrs

## TYPE_MEDIA existing uses, do they care?
- NS_CP_ContentTypeName(uint32_t contentType): no, just a stringifier and
  already has the relevant mapping.
- nsContentUtils::IsPreloadType: boolean preload type check, no exposure of
  underlying type, only cares about preload internal types, so no change there.
- nsContentUtils::InternalContentPolicyTypeToExternal: This maps the types back
  down to TYPE_MEDIA.
  - nsContentPolicy::CheckPolicy uuses to map internal `contentType` to
    `externalType`.  Regrettably, both use the same type of nsContentPolicyType.
  - used for a number of nice assertions like in the top of
    nsDataDocumentContentPolicy::ShouldLoad that assert that the type passed is
    already an external type because invoking this function on the type results
    in the same type being emitted.