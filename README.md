# Agora Best Practice For Poor Networks 
In this guide we look at what to do when a user's network connection is not good enough to support either video upload or download or both.

## Two Person Calls
Two person calls are a special case as the publisher (sender) will adjust their outgoing bitrate in real-time to suit both the needs of their uplink and also the remote user's downlink. This is known as Full Path Feedback (FPF).
However, the outgoing bitrate will never drop below 100kps and for some very poor connections the video needs to be turned off as otherwise it will overwhelm the uplink connection and cause audio and online status to be impacted.
Note that Full Path Feedback is disengaged when the publisher is also sharing his screen.

When problems occurr during ScreenShare (SS) we want FPF to be engaged and this can be achieved by turning off the webcam. 

Agora also has a mode ENABLE_AUT_CC which will prioritise audio over video and the server not forward video when the receiver's downlink is struggling to receive it.

### Recommended Strategy
<ul>
  <li>Enable ENABLE_AUT_CC</li>
  <li>setStreamFallbackOption to audio only</li>
  <li>Blur incoming webcam video when frozen (not screenshare)</li>
  <li>When webcam video fallsback to audio only ask publisher to turn it off</li>
  <li>When screenshare video fallsback to audio only, ask publisher to turn off webcam if present otherwise turn off screenshare</li>
  <li>If publisher detects currentPacketLossRate > 0.4 on outbound video then turn it off</li>
  <li>Link your networks status indicator to the frozen and fallback/recover events below</li>
</ul>

Putting this all together


## Prioritize Audio over Video
Set this parameter after calling createClient in order for the server to drop video when the downlink is suffering high packet loss due to congestion

Video will be dropped by server
```
    agoraClient = await AgoraRTC.createClient({ mode: "rtc", codec: codec_ });
    AgoraRTC.setParameter("ENABLE_AUT_CC", true);

```

Enable fallback/recover events for the ENABLE_AUT_CC algorithm       
Call this after subscribing to remote video for each stream      
```
     agoraClient.setStreamFallbackOption(user.uid, 2);

```
## Detect and Blur Frozen Webcam 
Add a callback to detect inbound webcam video has frozen.

In the user-published callback where you subscribe to video you can add a callback for changes to video freeze state.
Only do this for the webcam video and not any screenshare
```
user.videoTrack.on('video-state-changed', (event) => {
    onVideoStateChanged(event, user.videoTrack, 'webcam');
});
```

Detect the incoming webcam video has frozen and blur it
```
async function onVideoStateChanged(vState, videostream, vtype) {
    console.log(`video-state-changed fired ${vState}`, videostream, vtype);
    if (vState == 3) {
        videostream._player.videoElement.style.filter = "blur(10px)";
        return;
    }
    videostream._player.videoElement.style.filter = "";
}
```

## Request publisher turns off Webcam video
When subscriber receives a fallback message send a stopPublish event to the publisher with the uid so the publisher can turn off webcam if publishing both screenshare and webcam. This will enable Full Path Feedback.   

```
function handleStreamFallback(uid,state) {
    console.error('handleStreamFallback ',uid,state);
    if (state == 'fallback') {
        agoraClient.sendStreamMessage({ payload: '{"stopPublish":"' + uid + '"}' }).then(() => {
        }).catch(error => {
            console.error('AgoraRTM  send failure');
        });     
    }
}
```

## Turn Off the Publisher's Webcam When Subscriber Requests It
This code shows how to receive a custom message at the publisher client in order to turn off his webcam and show a toast message 
```
function handleStreamMessage(senderId, data) {
    const textDecoder = new TextDecoder();
    const decodedText = textDecoder.decode(data);
    const jsonObject = JSON.parse(decodedText);
    if (jsonObject.stopPublish) {
        console.log('handleStreamMessage video-state-changed stopPublish', jsonObject.stopPublish);
    }
}

agoraClient.on("stream-message", handleStreamMessage);
```

## Turn Off the Publisher's Outgoing Video When Very High Packet Loss On Uplink
Monitor the currentPacketLossRate for all outbound video every 1 second.    
currentPacketLossRate is the average over a 400ms window.    
0.4 equates to 40% packet loss   
```
if (agoraClient.getLocalVideoStats().currentPacketLossRate > 0.4) {
    // stop publishing video
}
```

 
