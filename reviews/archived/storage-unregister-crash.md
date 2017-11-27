https://bugzilla.mozilla.org/show_bug.cgi?id=1388998

## overview ##

ProxyRelease on the main thread is calling unregister and we're crashing in the
nsTArray::RemoveElementAt's ShiftData call.

It looks like this is a function of a naive call to memmove of the form
memmove(dest=memory we own, src=memory we don't own that crosses 4k page
boundaries, size=0 so we assume it's harmless.)

### code excerpts ###
**highlighted call points**

template<typename E, class Alloc>
void
nsTArray_Impl<E, Alloc>::RemoveElementsAt(index_type aStart, size_type aCount)
{
  MOZ_ASSERT(aCount == 0 || aStart < Length(), "Invalid aStart index");
  MOZ_ASSERT(aStart + aCount <= Length(), "Invalid length");
  // Check that the previous assert didn't overflow
  MOZ_ASSERT(aStart <= aStart + aCount, "Start index plus length overflows");
  DestructRange(aStart, aCount);
  **this->template ShiftData<InfallibleAlloc>(aStart, aCount, 0,
                                            sizeof(elem_type),
                                            MOZ_ALIGNOF(elem_type));**
}

template<class Alloc, class Copy>
template<typename ActualAlloc>
void
nsTArray_base<Alloc, Copy>::ShiftData(index_type aStart,
                                      size_type aOldLen, size_type aNewLen,
                                      size_type aElemSize, size_t aElemAlign)
{
  if (aOldLen == aNewLen) {
    return;
  }

  // Determine how many elements need to be shifted
  size_type num = mHdr->mLength - (aStart + aOldLen);

  // Compute the resulting length of the array
  mHdr->mLength += aNewLen - aOldLen;
  if (mHdr->mLength == 0) {
    ShrinkCapacity(aElemSize, aElemAlign);
  } else {
    // Maybe nothing needs to be shifted
    if (num == 0) {
      return;
    }
    // Perform shift (change units to bytes first)
    aStart *= aElemSize;
    aNewLen *= aElemSize;
    aOldLen *= aElemSize;
    char* baseAddr = reinterpret_cast<char*>(mHdr + 1) + aStart;
    **Copy::MoveOverlappingRegion(baseAddr + aNewLen, baseAddr + aOldLen, num, aElemSize);**
  }
}

  static void MoveOverlappingRegion(void* aDest, void* aSrc, size_t aCount,
                                    size_t aElemSize)
  {
    **memmove(aDest, aSrc, aCount * aElemSize);**
  }

  **Note** that the stacks seem to vary between memmove/memcpy/unknown, but it
  really seems like it should be memmove.
