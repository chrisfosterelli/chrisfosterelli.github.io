---
layout: post
permalink: getting-started-with-webrtc-data-channels.html
title: Getting Started with WebRTC Data Channels
---
[WebRTC](http://www.webrtc.org/home) data channels are in a strange state where many people are excited about them, but are not sure how to approach them due to their currently volatile nature. The standard has been rapidly changing, leaving many of the examples and resources regarding WebRTC outdated or incomplete. Unfortunately, a developer looking to get started with WebRTC data channels can be having a pretty bad time right now.

Data channels have especially been shadowed by the audio and video capabilities of WebRTC. Many documentation pages and tutorials feature incomplete examples, with the full demos being too complex to easily follow and understand. This article will approach WebRTC from the data channel only view.

So, in the spirit of supporting development, here is a complete example of _working WebRTC data channels with the latest Google Chrome, version 43_ (demo below).

<!-- Content Breaker -->

## Code

{% highlight javascript %}
/* WebRTC Demo
 * Allows two clients to connect via WebRTC with Data Channels
 * Uses Firebase as a signalling server
 * http://fosterelli.co/getting-started-with-webrtc-data-channels.html 
 */

/* == Announcement Channel Functions ==
 * The 'announcement channel' allows clients to find each other on Firebase
 * These functions are for communicating through the announcement channel
 * This is part of the signalling server mechanism
 *
 * After two clients find each other on the announcement channel, they 
 * can directly send messages to each other to negotiate a WebRTC connection
 */

// Announce our arrival to the announcement channel
var sendAnnounceChannelMessage = function() {
  announceChannel.remove(function() {
    announceChannel.push({
      sharedKey : sharedKey,
      id : id
    });
    console.log('Announced our sharedKey is ' + sharedKey);
    console.log('Announced our ID is ' + id);
  });
};

// Handle an incoming message on the announcement channel
var handleAnnounceChannelMessage = function(snapshot) {
  var message = snapshot.val();
  if (message.id != id && message.sharedKey == sharedKey) {
    console.log('Discovered matching announcement from ' + message.id);
    remote = message.id;
    initiateWebRTCState();
    connect();
  }
};

/* == Signal Channel Functions ==
 * The signal channels are used to delegate the WebRTC connection between 
 * two peers once they have found each other via the announcement channel.
 * 
 * This is done on Firebase as well. Once the two peers communicate the
 * necessary information to 'find' each other via WebRTC, the signalling
 * channel is no longer used and the connection becomes peer-to-peer.
 */

// Send a message to the remote client via Firebase
var sendSignalChannelMessage = function(message) {
  message.sender = id;
  database.child('messages').child(remote).push(message);
};

// Handle a WebRTC offer request from a remote client
var handleOfferSignal = function(message) {
  running = true;
  remote = message.sender;
  initiateWebRTCState();
  startSendingCandidates();
  peerConnection.setRemoteDescription(new RTCSessionDescription(message));
  peerConnection.createAnswer(function(sessionDescription) {
    console.log('Sending answer to ' + message.sender);
    peerConnection.setLocalDescription(sessionDescription);
    sendSignalChannelMessage(sessionDescription.toJSON());
  }, function(err) {
    console.log('Could not create offer', err);
  });
};

// Handle a WebRTC answer response to our offer we gave the remote client
var handleAnswerSignal = function(message) {
  peerConnection.setRemoteDescription(new RTCSessionDescription(message));
};

// Handle an ICE candidate notification from the remote client
var handleCandidateSignal = function(message) {
  var candidate = new RTCIceCandidate(message);
  peerConnection.addIceCandidate(candidate);
};

// This is the general handler for a message from our remote client
// Determine what type of message it is, and call the appropriate handler
var handleSignalChannelMessage = function(snapshot) {
  var message = snapshot.val();
  var sender = message.sender;
  var type = message.type;
  console.log('Recieved a \'' + type + '\' signal from ' + sender);
  if (type == 'offer') handleOfferSignal(message);
  else if (type == 'answer') handleAnswerSignal(message);
  else if (type == 'candidate' && running) handleCandidateSignal(message);
};

/* == ICE Candidate Functions ==
 * ICE candidates are what will connect the two peers
 * Both peers must find a list of suitable candidates and exchange their list
 * We exchange this list over the signalling channel (Firebase)
 */

// Add listener functions to ICE Candidate events
var startSendingCandidates = function() {
  peerConnection.oniceconnectionstatechange = handleICEConnectionStateChange;
  peerConnection.onicecandidate = handleICECandidate;
};

// This is how we determine when the WebRTC connection has ended
// This is most likely because the other peer left the page
var handleICEConnectionStateChange = function() {
  if (peerConnection.iceConnectionState == 'disconnected') {
    console.log('Client disconnected!');
    sendAnnounceChannelMessage();
  }
};

// Handle ICE Candidate events by sending them to our remote
// Send the ICE Candidates via the signal channel
var handleICECandidate = function(event) {
  var candidate = event.candidate;
  if (candidate) {
    candidate = candidate.toJSON();
    candidate.type = 'candidate';
    console.log('Sending candidate to ' + remote);
    sendSignalChannelMessage(candidate);
  } else {
    console.log('All candidates sent');
  }
};

/* == Data Channel Functions ==
 * The WebRTC connection is established by the time these functions run
 * The hard part is over, and these are the functions we really want to use
 * 
 * The functions below relate to sending and receiving WebRTC messages over
 * the peer-to-peer data channels 
 */

// This is our receiving data channel event
// We receive this channel when our peer opens a sending channel
// We will bind to trigger a handler when an incoming message happens
var handleDataChannel = function(event) {
  event.channel.onmessage = handleDataChannelMessage;
};

// This is called on an incoming message from our peer
// You probably want to overwrite this to do something more useful!
var handleDataChannelMessage = function(event) {
  console.log('Recieved Message: ' + event.data);
  document.write(event.data + '<br />');
};

// This is called when the WebRTC sending data channel is offically 'open'
var handleDataChannelOpen = function() {
  console.log('Data channel created!');
  dataChannel.send('Hello! I am ' + id);
};

// Called when the data channel has closed
var handleDataChannelClosed = function() {
  console.log('The data channel has been closed!');
};

// Function to offer to start a WebRTC connection with a peer
var connect = function() {
  running = true;
  startSendingCandidates();
  peerConnection.createOffer(function(sessionDescription) {
    console.log('Sending offer to ' + remote);
    peerConnection.setLocalDescription(sessionDescription);
    sendSignalChannelMessage(sessionDescription.toJSON());
  }, function(err) {
    console.log('Could not create offer', err);
  });
};

// Function to initiate the WebRTC peerconnection and dataChannel
var initiateWebRTCState = function() {
  peerConnection = new webkitRTCPeerConnection(servers);
  peerConnection.ondatachannel = handleDataChannel;
  dataChannel = peerConnection.createDataChannel('myDataChannel');
  dataChannel.onmessage = handleDataChannelMessage;
  dataChannel.onopen = handleDataChannelOpen;
};

var id;              // Our unique ID
var sharedKey;       // Unique identifier for two clients to find each other
var remote;          // ID of the remote peer -- set once they send an offer
var peerConnection;  // This is our WebRTC connection
var dataChannel;     // This is our outgoing data channel within WebRTC
var running = false; // Keep track of our connection state

// Use Google's public servers for STUN
// STUN is a component of the actual WebRTC connection
var servers = {
  iceServers: [ {
    url : 'stun:stun.l.google.com:19302'
  } ]
};

// Generate this browser a unique ID
// On Firebase peers use this unique ID to address messages to each other
// after they have found each other in the announcement channel
id = Math.random().toString().replace('.', '');

// Unique identifier for two clients to use
// They MUST share this to find each other
// Each peer waits in the announcement channel to find its matching identifier
// When it finds its matching identifier, it initiates a WebRTC offer with
// that client. This unique identifier can be pretty much anything in practice.
// Configure, connect, and set up Firebase
sharedKey = prompt("Please enter a shared identifier");

// You probably want to replace the text below with your own Firebase URL
var firebaseUrl = 'https://amber-fire-244.firebaseio.com/';
var database = new Firebase(firebaseUrl);
var announceChannel = database.child('announce');
var signalChannel = database.child('messages').child(id);
signalChannel.on('child_added', handleSignalChannelMessage);
announceChannel.on('child_added', handleAnnounceChannelMessage);

// Send a message to the announcement channel
// If our partner is already waiting, they will send us a WebRTC offer
// over our Firebase signalling channel and we can begin delegating WebRTC
sendAnnounceChannelMessage();
{% endhighlight %}

## Demo

The best way to show this is with a demo. Open up [this link](file/getting-started-with-webrtc-data-channels/webrtc.html) in two separate tabs (or computers!) in Google Chrome, and enter the same unique shared key into each prompt box. If you have your developer console open, you should see a number of debug messages as the WebRTC handshake is done through the signalling channel, and eventually initiated. The message that is sent by the other peer will be displayed on the page!

For those of you not able to run the demo, you would have seen something like this:

  ![Picture of Chrome demo](image/getting-started-with-webrtc-data-channels/1.png)

## The Signalling Server

This demo uses [Firebase](https://www.firebase.com) as the signalling server. While WebRTC is direct peer-to-peer communication, it requires a centralized server to handle the initial delegation of the two peers. There needs to be a 3rd party that helps the two peer-to-peer clients preform the WebRTC handshake. In this case, we have used Firebase. You can use [WebSockets](https://www.websocket.org/), HTTP calls, or anything else that you'd like. Firebase works nicely for this so we've used it here.

## Using a shared key

The shared key referred to in the code above, that's the text phrase you type in to the prompt when you load the page, is how the two clients find each other. Numerous peers connect to the Firebase instance, so we need a way of determining who is trying to connect to who. Our 'shared key' is used for this, but theoretically you could use anything such as a username or URL query parameter. If your signalling server is more intelligent, you can remove the shared key / announcement channel concept entirely by having the signalling server only send messages to the appropriate peers.

## Extending this demo

This demo doesn't support Firefox, as they use a slightly different syntax for some of the operations. It's not hard at all to add this in, however! There are actually very few subtle differences if you need to support both Chrome and Firefox. You can check out more information on [the WebRTC website](http://www.webrtc.org/firefox).

This is a two-participant peer-to-peer setup. Theoretically, much more than two peers could interact over data channels. The Javascript application would simply set up data channel connections with multiple different peers and manage each connection.

## Conclusion

Data channels are awesome, and exceptionally easy to use once you have managed to actually set them up. As the web has been slightly void of useful information, hopefully this will provide a starting reference point for someone who is looking to build WebRTC support into their latest pet project!

Got a question? [Email me](mailto:chris.james.foster@gmail.com).

__Updated April 5, 2015:__ Added syntax highlighting

__Updated June 17, 2015:__ Now works with Chrome 43, thanks to Dave Compton

__Updated May 7, 2016:__ Now works with Chrome 50, thanks to Akash Paul 
