<script setup>
import { ref } from 'vue';

const sdp = ref(""); // Peer 1: Local SDP
const remoteSdp = ref(""); // Peer 2: Remote SDP (from Peer 1)
const file = ref(null); // This will hold the selected file
let dataChannel = null;
let peerConnection = null;
let candidates = []; // This will hold the ICE candidates of Peer 1
const jsonCandidates = ref("");
const remoteJsonCandidates = ref("");
const connectionStatus = ref(""); // This will hold the connection status



const sendProgress = ref(0); // This will hold the send progress
const receiveProgress = ref(0); // This will hold the receive progress
const receivedFile = ref(null); // This will hold the received file

// Peer 1: Create channel and offer
const start = async () => {
  try {
    const configuration = {
      iceServers: [
        {
            urls: ["stun:stun.relay.metered.ca:80"],
        },
        // {
        //     urls: "turn:a.relay.metered.ca:80",
        //     username: "",
        //     credential: "",
        // },
        
      ]
    };
    
    peerConnection = new RTCPeerConnection(configuration);
    dataChannel = peerConnection.createDataChannel('fileTransfer');

    dataChannel.onopen = () => {
      connectionStatus.value = "Connected";
    };

    dataChannel.onclose = () => {
      connectionStatus.value = "Disconnected";
    };

    peerConnection.onicecandidate = (event) => {
      if (event.candidate) {
        candidates.push(event.candidate);
        jsonCandidates.value = JSON.stringify(candidates);
      }

      // Listen for connection state changes
      peerConnection.oniceconnectionstatechange = function() {
        console.log('ICE connection state change: ', peerConnection.value.iceConnectionState);
      };
    };

    const offer = await peerConnection.createOffer();
    await peerConnection.setLocalDescription(offer);
    sdp.value = JSON.stringify(peerConnection.localDescription);
  } catch (error) {
    console.error(error);
  }
};

// Peer 1: Send file
const sendFile = () => {
  if (dataChannel && dataChannel.readyState === 'open' && file.value) {
    const fileReader = new FileReader();
    fileReader.readAsArrayBuffer(file.value);
    fileReader.onload = (event) => {
      dataChannel.send(event.target.result);
    };
    fileReader.onprogress = (event) => {
      if (event.lengthComputable) {
        sendProgress.value = (event.loaded / event.total) * 100;
      }
    };
  } else {
    console.error('Cannot send file because data channel is not open');
  }
};

// Peer 2: Connect to Peer 1 and create answer
const connect = async () => {
  try {
    const configuration = {
        iceServers: [
          {
            urls: ["stun:stun.relay.metered.ca:80"],
          },
               // {
        //     urls: "turn:a.relay.metered.ca:80",
        //     username: "",
        //     credential: "",
        // },
        ]
      };
  
    peerConnection = new RTCPeerConnection(configuration);
  
    peerConnection.onicecandidate = (event) => {
      if (event.candidate) {
        candidates.push(event.candidate);
        jsonCandidates.value = JSON.stringify(candidates);
      }
    }
    
    peerConnection.ondatachannel = (event) => {
      dataChannel = event.channel;

      dataChannel.onopen = () => {
        connectionStatus.value = "Connected";
      };

      dataChannel.onclose = () => {
        connectionStatus.value = "Disconnected";
      };

      dataChannel.onmessage = (event) => {
        const arrayBuffer = event.data;
        const blob = new Blob([arrayBuffer]);
        receivedFile.value = URL.createObjectURL(blob);
        receiveProgress.value = 100; // Assuming the entire file is received in one message
      };
    };

    await peerConnection.setRemoteDescription(new RTCSessionDescription(JSON.parse(remoteSdp.value)));
    const answer = await peerConnection.createAnswer();
    await peerConnection.setLocalDescription(answer);
    sdp.value = JSON.stringify(peerConnection.localDescription);
    const candidatesArray = JSON.parse(remoteJsonCandidates.value);
    for (const candidate of candidatesArray) {
      await peerConnection.addIceCandidate(new RTCIceCandidate(candidate));
    }

  } catch (error) {
    console.error(error);
  }
};

const addCandidates = async () => {
  await peerConnection.setRemoteDescription(new RTCSessionDescription(JSON.parse(remoteSdp.value)));
  const candidatesArray = JSON.parse(remoteJsonCandidates.value);
    for (const candidate of candidatesArray) {
      await peerConnection.addIceCandidate(new RTCIceCandidate(candidate));
    }
};

const selectFile = (event) => {
  file.value = event.target.files[0];
};

</script>


<template>
  <h1>Tranferfile Demo</h1>

  <div>
    <p>Connection Status: {{ connectionStatus }}</p>
  </div>

  <div>
    <button @click="start">Create channel</button>
    <button @click="connect">Connect to peer</button>
    <button @click="addCandidates">Add Candidates</button>
  </div>

  <div>
    <p>SPD:</p>
    <textarea v-model="sdp"></textarea>
  </div>
  <div>
    <p>This Candidates:</p>
    <textarea v-model="jsonCandidates"></textarea>
  </div>
  <div>
    <p>Remote SDP:</p>
    <textarea v-model="remoteSdp"></textarea>
  </div>
  <div>
    <p>Remote Candidates:</p>
    <textarea v-model="remoteJsonCandidates"></textarea>
  </div>
  <div>
    <p>Tranferfile:</p>
    <input type="file" @change="selectFile">
    <button @click="sendFile">Send File</button>
  </div>
  <div>
    <p>Send Progress: {{ sendProgress }}%</p>
    <progress max="100" :value="sendProgress"></progress>
  </div>
  <div>
    <p>Receive Progress: {{ receiveProgress }}%</p>
    <progress max="100" :value="receiveProgress"></progress>
  </div>
  <div v-if="receivedFile">
    <p>Received File:</p>
    <a :href="receivedFile" download>Download File</a>
  </div>
</template>