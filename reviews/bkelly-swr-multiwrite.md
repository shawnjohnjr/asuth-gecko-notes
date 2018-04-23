### Pipeline

* ServiceWorkerRegistrar::MaybeScheduleSaveData dispatches
  ServiceWorkerRegistrarSaveDataRunnable unless:
  - mShuttingDown: Obvs.
  - mRunnableDispatched: DataSaved() will handle queueing up another if
    necessary.
  - mDataTimeStamp <= mFileTimeStamp: So the big issue here is these are using
    the same timeline...
* ServiceWorkerRegistrarSaveDataRunnable invokes
  ServiceWorkerRegistrar::SaveData which synchronously invokes WriteData, with
  aFileTimeStampOut being written to at the same time we sample mData


### mooted comments:

Suggest comments like:

// Moment mData was last mutated (on PBackground).
TimeStamp mDataTimeStamp;
// Moment mData was last sampled on an IO thread with mMonitor held and subsequently successfully written to disk.  (In other words, explicitly unrelated to when the write completed, so we won't get confused if the write takes a long time and mData is mutated while the write is in-progress.)
// Only updated on PBackground in DataSaved().
TimeStamp mFileTimeStamp;

