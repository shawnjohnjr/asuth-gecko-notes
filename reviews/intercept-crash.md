
## Diverted channel setup
The child-intercepted channel involves the external helper app logic and gets diverted which triggers the child channel to have to open a corresponding parent channel.
[3179->3011] [PNeckoChild] Sending  PNecko::Msg_PHttpChannelConstructor
[3179->3011] [PBackgroundChild] Sending  PBackground::Msg_PHttpBackgroundChannelConstructor
[3011<-3179] [PBackgroundParent] Received  PBackground::Msg_PHttpBackgroundChannelConstructor
[3179->3011] [PNeckoChild] Sending  PNecko::Msg_PChannelDiverterConstructor
[3179->3011] [PExternalHelperAppChild] Sending  PExternalHelperApp::Msg_DivertToParentUsing
[3011<-3179] [PNeckoParent] Received  PNecko::Msg_PHttpChannelConstructor
[3011<-3179] [PNeckoParent] Received  PNecko::Msg_PChannelDiverterConstructor
[3011<-3179] [PExternalHelperAppParent] Received  PExternalHelperApp::Msg_DivertToParentUsing
This diverter deletion is normal; ExternalHelperAppParent::RecvDivertToParentUsing just needed the diverter to get a reference to the HttpChannelParent so it could invoke DivertTo on it, targeting the ExternalHelperAppParent.
[3011->3179] [PChannelDiverterParent] Sending  PChannelDiverter::Msg___delete__
[3179<-3011] [PChannelDiverterChild] Received  PChannelDiverter::Msg___delete__

## Diversion happens.
HttpChannelParent::DivertTo() calls HttpChannelParent::StartDiversion() calls HttpBackgroundChannelParent::OnDiversion which does SendFlushedForDiversion() and SendDivertMessages().
[3011->3179] [PHttpBackgroundChannelParent] Sending  PHttpBackgroundChannel::Msg_FlushedForDiversion
[3011->3179] [PHttpBackgroundChannelParent] Sending  PHttpBackgroundChannel::Msg_DivertMessages
HttpBackgroundChannelChild::RecvFlushedForDiversion() calls HttpChannelChild::ProcessFlushedForDiversion() which schedules FlushedForDiversion() to be invoked on the neckoTarget via mEventQ.  The mEventQ is suspended because of the ongoing diversion.
[3179<-3011] [PHttpBackgroundChannelChild] Received  PHttpBackgroundChannel::Msg_FlushedForDiversion
HttpBackgroundChannelChild::RecvDivertMessages() calls HttpChannelChild::ProcessDivertMessages() which dispatches a Resume() to the neckoTarget without involving the mEventQ, which makes sense because the mEventQ won't be processed until that Resume() call happens.  But before mEventQ->Resume() is invokes, Resume() will also invoke mSynthesizedResponsePump->Resume(), which will cause SyntheticDiversionListener to start receiving OnDataAvailableEvents.  Those OnDataAvailableEvents will invoke SendDivertOnDataAvailable.  When FlushedForDiversion() is invoked, we expect it to invoke SendDivertComplete().
[3179<-3011] [PHttpBackgroundChannelChild] Received  PHttpBackgroundChannel::Msg_DivertMessages
Yeah, it looks like starting the pump allowed a DivertOnDataAvailable message to get sent before a DivertComplete message.  This is the point where I mention that this ordering is a red herring.
[3179->3011] [PHttpChannelChild] Sending  PHttpChannel::Msg_DivertOnDataAvailable
[3179->3011] [PHttpChannelChild] Sending  PHttpChannel::Msg_DivertComplete
[3011<-3179] [PHttpChannelParent] Received  PHttpChannel::Msg_DivertOnDataAvailable
[3011<-3179] [PHttpChannelParent] Received  PHttpChannel::Msg_DivertComplete
[3011->3179] [PHttpChannelParent] Sending  PHttpChannel::Msg_DeleteSelf
[3011->3179] [PHttpBackgroundChannelParent] Sending  PHttpBackgroundChannel::Msg___delete__

[3011<-3179] [PHttpChannelParent] Received  PHttpChannel::Msg_DivertOnDataAvailable
Assertion failure: mParentListener, at /home/visbrero/rev_control/hg/mozilla-central/netwerk/protocol/http/HttpChannelParent.cpp:1179
[3179<-3011] [PHttpBackgroundChannelChild] Received  PHttpBackgroundChannel::Msg___delete__
[3179<-3011] [PHttpChannelChild] Received  PHttpChannel::Msg_DeleteSelf
[3179->3011] [PHttpChannelChild] Sending  PHttpChannel::Msg___delete__



GECKO(3011) | [time: 1511939926274904][3011<-3179] [PHttpChannelParent] Received  PHttpChannel::Msg_DivertOnDataAvailable
GECKO(3011) | Assertion failure: mParentListener, at /home/visbrero/rev_control/hg/mozilla-central/netwerk/protocol/http/HttpChannelParent.cpp:1179
