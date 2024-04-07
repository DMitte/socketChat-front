<template>
  <div class="flex gap-6">
    <div v-if="!inCall" @click="EnterCall()">
      <button class="bg-black text-white mt-5 ml-5 w-auto p-5 rounded-xl text-2xl">Enter Call</button>
    </div>

    <div v-if="inCall">
      <video ref="LocalVideo" autoplay muted></video>
      <video ref="RemoteVideo" autoplay></video>
    </div>

    <div v-if="inCall" class="flex flex-col justify-end items-end">
      <div class="mb-2">
        <ul>
          <li v-for="(message, index) in this.messages" :key="index">
            {{ message.data }}
          </li>
        </ul>
      </div>
      <div class="">
        <input type="text" name="messages" v-model="this.message" class="border-black border-2 mr-2 rounded-3xl h-8 text-black">
        <input type="button" value="Enviar" @click="sendMessage()" class="bg-black w-32 h-8 text-white">
      </div>
      
    </div>
    

  </div>
</template>

<script>
import SimplePeer from 'simple-peer';
import {io} from 'socket.io-client'

export default {
  data(){
    return{
      inCall: false,
      peer: null,
      peerRemote: null,
      Socket: null,
      userid: "dany",
      message: '',
      messages: []
    }
  },
  name: 'App',
  methods: {
    EnterCall() {
      this.inCall = !this.inCall

      navigator.mediaDevices.getUserMedia({video: true, audio:true})
        .then(stream => {          
          this.$refs.LocalVideo.srcObject = stream;          
          this.peer.addStream(stream)

          this.peer.on('call', call => {
            call.answer(stream)           
            
          
          })
          this.peer.on('signal', signal =>{
            this.Socket.emit('signal', signal)
          })

          this.peer.signal
        })
        .catch(err => {
          console.error('error al acceder a la acamara:', err)
        })
    },
    sendMessage(){
      try{
        if(this.message !== ""){
          this.Socket.emit('message',{
            data: this.message
          })

        }
      }catch(err){
        console.log('error al mandal el mensaje')
      }
    }
  },
  mounted(){
    this.Socket = io('localhost:3000')
    this.peer = new SimplePeer({
      initiator: false,
      trickle: false,      
    })    

    
    this.peer.on('stream', stream => {
      this.$refs.RemoteVideo.srcObject = stream
    })
    this.peer.on('signal', signal => {
      this.Socket.emit('signal', signal)
    })

    this.peer.on('connect', () => {
      console.log('Conexion establecida');
    })

    this.peer.on('error', err => {
      console.error('errror en la conexion:' , err)
    })

    this.Socket.on('signal', signal => {
      this.peer.signal(signal)      
    })

    this.Socket.on('message', (data) => {
      this.messages.push(data)
    })
  }

}
</script>