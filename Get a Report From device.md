# Get a Report from Device

###### tags: `CHIP` `Matter` `Firmware` `Engineer`

> 2022/05/14
> CHIP git hash code 67b4746ad8

## Setup

- Python controller running on MAC
- Lock-app running on thunderboard sense 2
- OpenThread board router on Raspberry pi 2B

## The lock-app example

In this example, you would be able to control the locker by python controller if everything goes well. You may also see this kind of message in the device log:
`00> <info  > [ZCL] On/Off set value: 1 1 `
`00> <info  > [ZCL] Toggle on/off from 0 to 1`

As you can image, if you build up this controller and install it to your phone, you could then control your front door locker by your phone. However, you probably also want to know when your door locker is unlocked or locked(at least I want to know...).

And the document in this example doesn't show you how to do make the locker device reports the status to the controller(your phone).

Therefore, if you are not familiar with ZCL just like me. Here are some steps to show you how I set up report function and I will have a brief walk through in the code.

## Zap file
The first thing we should know about the report is the zap file. This is the output file of the [ZAP](https://github.com/project-chip/zap). In short, this program is to config clusters on both client and server side.

Boil down, if you look at the zap file, you will know that this is a readable json format file. Here is a snippet of the lock-app zap file(under examples/lock-app/lock-common):
```json
 {
          "name": "On/Off",
          "code": 6,
          "mfgCode": null,
          "define": "ON_OFF_CLUSTER",
          "side": "server",
          "enabled": 1,
          "commands": [],
          "attributes": [
            {
              "name": "OnOff",
              "code": 0,
              "mfgCode": null,
              "side": "server",
              "included": 1,
              "storageOption": "RAM",
              "singleton": 0,
              "bounded": 0,
              "defaultValue": "0x00",
              "reportable": 1,
              "minInterval": 0,
              "maxInterval": 65344,
              "reportableChange": 0
            },
            ...
          ]
        },
```

As you can see, it defines a cluster called "On/Off" on server side. What we need to check here is the "reportable." Just make sure it is 1.

Also, you could learn more about the report attribute in the ZCL specification section 2.5.

:::info
Server is the thunderboard sense 2, client is python controller
:::

## Server side(thunderboard sense 2)
You don't really have to do anything, just recompile and flash the image to the device.

## Client side(python controller)
There is a command called "zclconfig." We will use this command to set up the report interval.

```shell=
chip-device-ctrl > zclconfigure OnOff OnOff 30 1 10 20 1 
```

The `10 20` means min-max interval in SECOND.

Again, if everything goes well, you will see this kind of log on the controller side(if you open the log)
```shell=
1632434820677] [2137:31085] CHIP: [DMG] ReportData =                                                                                                                           
[1632434820677] [2137:31085] CHIP: [DMG] {                                                                                                                                      
[1632434820677] [2137:31085] CHIP: [DMG]        SubscriptionId = 0x2782b66dc1448111,                                                                                            
[1632434820677] [2137:31085] CHIP: [DMG]        AttributeDataList =                                                                                                             
[1632434820677] [2137:31085] CHIP: [DMG]        [                                                                                                                               
[1632434820677] [2137:31085] CHIP: [DMG]                AttributeDataElement =                                                                                                  
[1632434820677] [2137:31085] CHIP: [DMG]                {                                                                                                                       
[1632434820677] [2137:31085] CHIP: [DMG]                        AttributePath =                                                                                                 
[1632434820677] [2137:31085] CHIP: [DMG]                        {                                                                                                               
[1632434820677] [2137:31085] CHIP: [DMG]                                NodeId = 0x1e,                                                                                          
[1632434820677] [2137:31085] CHIP: [DMG]                                EndpointId = 0x1,                                                                                       
[1632434820677] [2137:31085] CHIP: [DMG]                                ClusterId = 0x6,                                                                                        
[1632434820677] [2137:31085] CHIP: [DMG]                                FieldTag = 0x0,                                                                                         
[1632434820677] [2137:31085] CHIP: [DMG]                        }                                                                                                               
[1632434820677] [2137:31085] CHIP: [DMG]                                                                                                                                        
[1632434820677] [2137:31085] CHIP: [DMG]                        Data = true,                                                                                                    
[1632434820677] [2137:31085] CHIP: [DMG]                        DataElementVersion = 0x0,                                                                                       
[1632434820677] [2137:31085] CHIP: [DMG]                },                                                                                                                      
[1632434820677] [2137:31085] CHIP: [DMG]                                                                                                                                        
[1632434820677] [2137:31085] CHIP: [DMG]        ],                                                                                                                              
[1632434820677] [2137:31085] CHIP: [DMG]                                                                                                                                        
[1632434820677] [2137:31085] CHIP: [DMG] }                                                                                                                                      
[1632434820678] [2137:31085] CHIP: [ZCL] ReadAttributesResponse:                                                                                                                
[1632434820678] [2137:31085] CHIP: [ZCL]   ClusterId: 0x0000_0006     
```

## How it works
In python controller, if you follow the calling stack of command zclconfig, you will find out the command type of this command is Protocols::InteractionModel::MsgType::SubscribeRequest. Thus, on server side, we have to find out which component register for this command type.

In InteractionModelEngine::Init, you can find this line of code
```cpp=
ReturnErrorOnFailure(mpExchangeMgr->RegisterUnsolicitedMessageHandlerForProtocol(Protocols::InteractionModel::Id, this));
```
It is showing you that this class(InteractionModelEngine) register for all commands with protocol ID "InteractionModel". And Protocols::InteractionModel::MsgType::SubscribeRequest is under the "InteractionModel" ID. So the handler of this command is **InteractionModelEngine**.

### InteractionModelEngine::OnMessageReceived
This is the function we are going to start with. From this function, you can easily follow the calling stack:
1. InteractionModelEngine::OnMessageReceived
2. InteractionModelEngine::OnReadInitialRequest
3. ReadHandler::OnReadInitialRequest(app/ReadHandler.cpp)
4. ReadHandler::ProcessSubscribeRequest
5. Engine::ScheduleRun(app/reporting/Engine.cpp)

In Engine::ScheduleRun, it schedules a work to the CHIP stack thread, and the work is
`Engine::Run(System::Layer * aSystemLayer, void * apAppState)`
In this function, it simply calls
`Engine::Run()`
:::info
There are three different type of system implements. I use the system/SystemLayerImplLwIP.cpp as example. In this file, you can find a function:
LayerImplLwIP::ScheduleWork(TimerCompleteCallback onComplete, void * appState).

It basically creates a timer, and the callback of this timer will call TimerCompleteCallback. And TimerCompleteCallback in our case is Engine::Run(System::Layer * aSystemLayer, void * apAppState)
:::

### Engine::Run()
Finally, you can find this block of code in this function:
```cpp=
 while ((mNumReportsInFlight < CHIP_IM_MAX_REPORTS_IN_FLIGHT) && (numReadHandled < CHIP_IM_MAX_NUM_READ_HANDLER))
    {
        if (readHandler->IsReportable())
        {
            CHIP_ERROR err = BuildAndSendSingleReportData(readHandler);
            if (err != CHIP_NO_ERROR)
            {
                return;
            }
        }
        numReadHandled++;
        mCurReadHandlerIdx = (mCurReadHandlerIdx + 1) % CHIP_IM_MAX_NUM_READ_HANDLER;
        readHandler        = imEngine->mReadHandlers + mCurReadHandlerIdx;
    }
```

This is where the device will build and send out the report.

## TBC
This is how device sends out the report, but you may have a problem which is "how the controller gets this report." I will have another note about this later.

