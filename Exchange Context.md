# Exchange Context

###### tags: `CHIP` `Matter` `Firmware` `Engineer`

> 2022/07/30
> CHIP git hash code 67b4746ad8

## What is it

I would like to consider Exchange Context(EC) as a "channel" in each seassion.

Every device/controller would hold a peer's session if they are connected. However, there are many different types of messages would be sent between controller/device, so we need a specified channel to handle each type of message.

![](https://i.imgur.com/nllpfKh.jpg)

In addition, these ECs have been running in a session, so they could use all the resource of the session. That is, EC is not like session which has extreme complicated establishment process. A lot of time, CHIP stack would create a new EC just for one message. Thus, EC is also kind of a temporary channel.

## How EC works

Assume we are using python controller to commit a device by BLE. Let's start with the very first message: SendPBKDFParamRequest.

### Send PBKDFParamRequest

At the beginning of PASE, there is a PASE session to handle the PASE handshaking. So, the EC would be created in this session. Here is how the controller creates the EC.

In DeviceCommissioner::EstablishPASEConnection, we can find that the controller creates the EC with the current session, which is PASE session.
```cpp=845
/* the second parameter is the delegate, which would be used by the EC later */
exchangeCtxt = mSystemState->ExchangeMgr()->NewContext(session.Value(), &device->GetPairing()); 
```
After creating this EC, controller would call Pair with the pointer of this EC.
```cpp=854
err = device->GetPairing().Pair(params.GetPeerAddress(), params.GetSetupPINCode(), keyID,Optional<ReliableMessageProtocolConfig>::Value(mMRPConfig), exchangeCtxt, this); 
```

In the Pair function, it will prepare many things for PASE, and call SendPBKDFParamRequest. In SendPBKDFParamRequest, we have this:
```cpp=398
mExchangeCtxt->SendMessage(MsgType::PBKDFParamRequest, std::move(req), SendFlags(SendMessageFlags::kExpectR...
```

The mExchangeCtxt is exactly the exchangeCtxt the controller created in EstablishPASEConnection.

And if we go along with the calling stack, we will find that the CHIP stack would add the ID of this EC to the message in ExchangeMessageDispatch::SendMessage:

```cpp=51
payloadHeader.SetExchangeID(exchangeId).SetMessageType(protocol, type).SetInitiator(isInitiator);
```

Remember that there is an exchange ID in the message, we are going to use this later.


### Receive PBKDFParamRequest

On the device side, we can first look at ExchangeManager::OnMessageReceived. In this function, it will
1. Try to find an existed EC which has the same ID in the message.
2. If not, create a new EC to handle this message if nedded.(with the current session, this ExchangeManager knows the current session)

And, this is the first message this device ever gets, so it would of course create a new EC to handle this PBKD param request. But, this device will **create the EC with the ID in the message.** 

```cpp=289
mContextPool.CreateObject(this, payloadHeader.GetExchangeID(), session, !payloadHeader.IsInitiator(), delegate);
```

At this moment, controller and device have a EC established by a common EC ID.

After that, the device will hand over the message to the EC and send out SendPBKDFParamResponse.

### Recive SendPBKDFParamResponse

On the controller side, we also look at the ExchangeManager::OnMessageReceived. This time, the controller will **find an existed EC with the same ID from device**, and bypass the message to the found EC.



### The log
If you open the detail option in the makefile on the controller or device project, you will find the EC ID is the same during the PASE. For example, on the contoller side, 
```shell
Prepared plaintext message 0x70000d319ae0 to 0x0000000000000000 (0)  of type 0x20 and protocolId (0, 0) on exchange 35571i with 
...
Received message of type 0x21 with protocolId (0, 0) and MessageCounter:856765985 on exchange 35571i 
...
Prepared plaintext message 0x70000d296f60 to 0x0000000000000000 (0)  of type 0x22 and protocolId (0, 0) on exchange 35571i
...
Received message of type 0x23 with protocolId (0, 0) and MessageCounter:856765986 on exchange 35571i
...
```
But later on, it created another EC to handle the certification exchange:
```shell=
Prepared encrypted message 0x70000d31a0a0 to 0x000000000000001F (1)  of type 0x8 and protocolId (0, 1) on exchange 35572i with Mes...
```

### when does EC get freed?

Currently, we just keep creating ECs. But there must be somewhere these ECs are deleted from heap. And this is how the CHIP stack frees the ECs in ExchangeContext::HandleMessage:

```cpp=425
auto deferred = MakeDefer([&]() {
        // The alreadyHandlingMessage check is effectively a workaround for the fact that SendMessage() is not calling
        // MessageHandled() yet and will go away when we fix that.
        if (alreadyHandlingMessage)
        {
            // Don't close if there's an outer HandleMessage invocation.  It'll deal with the closing.
            return;
        }
        // We are the outermost HandleMessage invocation.  We're not handling a message anymore.
        mFlags.Clear(Flags::kFlagHandlingMessage);

        // Duplicates and standalone acks are not application-level messages, so they should generally not lead to any state
        // changes.  The one exception to that is that if we have a null mDelegate then our lifetime is not application-defined,
        // since we don't interact with the application at that point.  That can happen when we are already closed (in which case
        // MessageHandled is a no-op) or if we were just created to send a standalone ack for this incoming message, in which case
        // we should treat it as an app-level message for purposes of our state.
        if ((isStandaloneAck || isDuplicate) && mDelegate != nullptr)
        {
            return;
        }

        MessageHandled();
    });
```

This lambda would be executed after the control flow goes out of this function(HandleMessage).


```cpp=495
void ExchangeContext::MessageHandled()
{
#if CONFIG_DEVICE_LAYER && CHIP_DEVICE_CONFIG_ENABLE_SED
    UpdateSEDPollingMode();
#endif

    if (mFlags.Has(Flags::kFlagClosed) || IsResponseExpected() || IsSendExpected())
    {
        return;
    }

    Close();
}
```
And in the MessageHandled, the EC would be released if no one needs this EC anymore.

## Conclusion

The above is how controller and device talk to each other on the same EC.

Although, most of time, EC would just bypass all the works to the delegate object and actually do nothing to the message, it is still important key factor to handle all the messages independently between controller and device.
