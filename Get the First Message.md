# Get The First Message

###### tags: `CHIP` `Matter` `Firmware` `Engineer`

> 2022/04/23
> CHIP git hash code 67b4746ad8

I would like to follow the calling stack to understand how CHIP stack receives message. Bascially, it's like a reverse calling stack of "Send Message."

Just give you a heads-up, there is an engine to handle *CHIP-over-Bluetooth Low Energy (CHIPoBLE)* message and I skipped this module because I think it would be easier to understand this kind of code if we got the specification first.

## Setup

- Python controller running on MAC
- Lock-app running on thunderboard sense 2
- OpenThread(Not going to use this, but still)


## Start from the EFR32/BLEManagerImpl.cpp

In this file, we can know that there is a thread to handle BLE events. Also, we can find this code block in the event-handle loop in the same file:
```cpp=274
/* This event indicates that a remote GATT client is attempting to write a value of an
 * attribute in to the local GATT database, where the attribute was defined in the GATT
 * XML firmware configuration file to have type="user".  */
case sl_bt_evt_gatt_server_attribute_value_id: {
    sInstance.HandleWriteEvent(bluetooth_evt);
}
```

This is how the device handles the incoming BLE write-event and also where we are going to start.

### BLEManagerImpl::HandleRXCharWrite
From the HandleWriteEvent, you can easily reach this function. In this function, it basically puts the incoming data to a PacketBufferHandle and posts an event to CHIP stack with the PacketBufferHandle.

That being said, we will swtich to the CHIP thread from BLE thread.

### DispatchEvent
:::info
This is actually:
`GenericPlatformManagerImpl<ImplClass>::_DispatchEvent` in include/platform/internal/GenericPlatformManagerImpl.cpp.
:::

In this dispatch, you can find this:
`Impl()->DispatchEventToDeviceLayer(event);`

### DispatchEventToDeviceLayer
This function simply calls this:
`BLEMgr().OnPlatformEvent(event);`

The BLEMgr is in the file we start with:
platform/EFR32/BLEManagerImpl.cpp

So, let's go back to that file.

### OnPlatformEvent
:::info
Actually, we are at `_OnPlatformEvent`
:::

From here, it's quite straightforward to follow the calling stack, and let's go to this function directly:
`BLEEndPoint::Receive`

This is a very complicated function because it's the engine I mentioned above. For now, just go directly to the end of the function and find this:
```cpp=1388
mBleTransport->OnEndPointMessageReceived(this, std::move(full_packet));
```

This is where the message will go up to the upper level of CHIP stack. If you follow this function, you can easily reach this function:
`TransportMgrBase::HandleMessageReceived`

### TransportMgrBase::HandleMessageReceived
This is a very short function, but we have to find out what the mSessionManager is.


First, in app/server/Server.cpp, we would see this in the Init function:
```cpp=142
err = mSessions.Init(&DeviceLayer::SystemLayer(), &mTransports, &mMessageCounterManager);
```

Secondly, in transport/SessionManager.cpp, we find this in the Init function:
```cpp=69
mTransportMgr->SetSessionManager(this);
```

So, the mSessionManager in TransportMgrBase = mSession in server.cpp.

Another important thing is this is not encrypted message, so please go to the `MessageDispatch` in SessionManager::OnMessageReceived.

And in SessionManager::OnMessageReceived, you would find this:
`mCB->OnMessageReceived(packetHeader, payloadHeader, SessionHandle(session), peerAddress, isDuplicate, std::move(msg));`

Again, we can find out that the mCB is from Server.cpp.

In Server.cpp, we have this:
`err = mExchangeMgr.Init(&mSessions);`

In messaging/ExchangeMgr.cpp, we can find this in the Init:
`sessionManager->SetMessageDelegate(this);`

And that's how the session manager set up mCB. In short, the mCB is mExchangeMgr(ExchangeManager).

### ExchangeManager::OnMessageReceived
In this function, it would try to do two things.
1. Try find an existing ec(exchange context) matching the incoming ec ID.
2. It fails at step1, creates a new ec to handle this message.

Either way, this function will have an ec to handle the message. In our case, it goes to step2. In step2, there are another two steps:
1. Try to find if there is a registered handler for the incoming message.
2. Use the handler(if any) to create the ec.

#### How ExchangeManager find handler
The code itself is easy to understand.
```cpp=249
  if (!msgFlags.Has(MessageFlagValues::kDuplicateMessage) && payloadHeader.IsInitiator())
    {
        // Search for an unsolicited message handler that can handle the message. Prefer handlers that can explicitly
        // handle the message type over handlers that handle all messages for a profile.
        matchingUMH = nullptr;

        for (auto & umh : UMHandlerPool)
        {
            if (umh.IsInUse() && payloadHeader.HasProtocol(umh.ProtocolId))
            {
                if (umh.MessageType == payloadHeader.GetMessageType())
                {
                    matchingUMH = &umh;
                    break;
                }

                if (umh.MessageType == kAnyMessageType)
                    matchingUMH = &umh;
            }
        }
    }
```

The problem is when and how the handler gets into the UMHandlerPool.

First, we can find this in Server::Init.
`SuccessOrExit(err = mCommissioningWindowManager.OpenBasicCommissioningWindow());`

And in OpenBasicCommissioningWindow, we got
`ReturnErrorOnFailure(mServer->GetExchangeManager().RegisterUnsolicitedMessageHandlerForType(Protocols::SecureChannel::MsgType::PBKDFParamRequest, &mPairingSession))`

By the above function call, mPairingSession(PASESession) just registered itself as a handler for message with protocols::SecureChannel::MsgType::PBKDFParamRequest type.

:::info
If we take a look at how the contoller sends the first message(PBKDFParamRequst), we will find this:
`mExchangeCtxt->SendMessage(MsgType::PBKDFParamRequest, std::move(req), SendFlags(SendMessageFlags::kExpectResponse))`

If you don't know how we find the this, please check this post
[The First Message](/UPMS1_9FQHySkZvv0v0_sg)
:::

Finally, we know that the matchingUMH is PASESession in mCommissioningWindowManager.cpp and we use the delegate of PASESession to create the ec. Thus, we can go to `ec->HandleMessage(packetHeader.GetMessageCounter(), payloadHeader, source, msgFlags, std::move(msgBuf));` to find out how ec handle the message.


### ExchangeContext::HandleMessage
Just like ExchangeContext::SendMessage, this function would hand over the incoming message to mDispatch.
`mDispatch->OnMessageReceived(messageCounter, payloadHeader, peerAddress, msgFlags, GetReliableMessageContext()));`. After that, it will bypass the message to the delegate
`mDelegate->OnMessageReceived(this, payloadHeader, std::move(msgBuf));`

Let's take a look at the mDispatch first.

#### ExchangeContext::mDispatch
The mDispatch is set up in the constructors of ExchangeContext, and it comes from delegate(by calling `delegate->GetMessageDispatch`). We already know the delegate comes from matchingUMH, which is PASESession. Therefore, we should find the mDispatch from PASESession.

We can easily find this in PASESession.h
`SessionEstablishmentExchangeDispatch mMessageDispatch`
And that's all we want.

### SessionEstablishmentExchangeDispatch::OnMessageReceived
The main goal of dispatch is actually handling ack, but I haven't really read this part of code. So, TBD...

### PASESession::OnMessageReceived
In this function, it will handle the PBKDFParamRequst message and prepare PBKDFParamResponse.

## In conclusion
This very first meesage won't actually reach the top level such as application level. PASE seeesion will be done by the CHIP stack itself.

However, when it comes to regular message instead of PASE session message, the receive process would be different. I will have another post about it later.


