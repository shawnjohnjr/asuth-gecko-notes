Abandoned notes.

For example, in that call stack, we eventually get to code that does:
  aSurface->AddSizeOfExcludingThis(aMallocSizeOf, heap, nonHeap, handles);
  @ https://searchfox.org/mozilla-central/rev/46292b1212d2d61d7b5a7df184406774727085b8/image/FrameAnimator.cpp#566
Whose code looks like:
  @ https://searchfox.org/mozilla-central/rev/46292b1212d2d61d7b5a7df184406774727085b8/image/imgFrame.cpp#931
```
void
imgFrame::AddSizeOfExcludingThis(MallocSizeOf aMallocSizeOf,
                                 size_t& aHeapSizeOut,
                                 size_t& aNonHeapSizeOut,
                                 size_t& aExtHandlesOut) const
{
  MonitorAutoLock lock(mMonitor);

  if (mPalettedImageData) {
    aHeapSizeOut += aMallocSizeOf(mPalettedImageData);
  }
  if (mLockedSurface) {
    aHeapSizeOut += aMallocSizeOf(mLockedSurface);
  }
  if (mOptSurface) {
    aHeapSizeOut += aMallocSizeOf(mOptSurface);
  }
  if (mRawSurface) {
    aHeapSizeOut += aMallocSizeOf(mRawSurface);
    mRawSurface->AddSizeOfExcludingThis(aMallocSizeOf, aHeapSizeOut,
                                        aNonHeapSizeOut, aExtHandlesOut);
  }
}
```

Where aMallocSizeOf is `MOZ_DEFINE_MALLOC_SIZE_OF(ImagesMallocSizeOf)`
  https://searchfox.org/mozilla-central/rev/a80651653faa78fa4dfbd238d099c2aad1cec304/image/imgLoader.cpp#72
which ends up as calls to moz_malloc_size_of/moz_malloc_usable_size/malloc_usable_size_impl
  https://searchfox.org/mozilla-central/rev/46292b1212d2d61d7b5a7df184406774727085b8/memory/mozalloc/mozalloc.cpp#143

I think that explains how control flow gets into the function in question.

That does beg the question of how stuff gets so messed up, but if we look back at FrameAnimator::CollectSizeOfCompositingSurfaces
 @ https://searchfox.org/mozilla-central/rev/46292b1212d2d61d7b5a7df184406774727085b8/image/FrameAnimator.cpp#576
It calls DoCollectSizeOfCompositingSurfaces() with either a `const RawAccessFrameRef& aSurface` to either its mCompositingFrame or mCompositingPrevFrame, both of which are RawAccessFrameRef instances that are nested structures on the FrameAnimator which I posit might end up freed
 @ types https://searchfox.org/mozilla-central/rev/46292b1212d2d61d7b5a7df184406774727085b8/image/FrameAnimator.h#427
