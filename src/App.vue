<template>
  <div>
    <div class="mt-7 ml-7">
      <div class="mb-6">
        <video v-if="inCall" ref="localVideo" autoplay muted class="bg-gray-600  w-56 absolute left-[34%] top-1/3"></video>
        <video ref="remoteVideo" autoplay class="w-2/4 h-full bg-black"></video>
        
      </div>
      <button v-if="inCall" @click="disconnectCall" class="bg-red-800 p-5 rounded text-white">Desconectar</button>
      <button v-if="!inCall" @click="startCall" class="bg-black text-white p-5 rounded">Iniciar llamada</button>
    </div>
    <div class="flex flex-col justify-end items-end" v-if="inCall">
      <div class="mb-2">
        <ul>
          <li v-for="(message, index) in this.messages" :key="index">
            {{ message.data }}
          </li>
        </ul>
      </div>
      <div class="">
        <input type="text" name="messages" v-model="this.message" class="border-black border-2 mr-2 pl-4 rounded-3xl h-8 text-black">
        <input type="button" value="Enviar" @click="sendMessage()" class="bg-black w-32 h-8 text-white">
      </div>
      
    </div>
  </div>
</template>

<script>
import SimplePeer from 'simple-peer';
import {io} from 'socket.io-client'

export default {
  data() {
    return {
      socket: Object,
      peer: null,
      message: '',
      messages: [],
      inCall: false
    };
  },
  mounted() {
    this.socket = io('localhost:3000')
    this.peer = new SimplePeer({
      initiator: true, // Cambia a true si este cliente inicia la llamada
      trickle: false
    });

    
      this.peer.on('stream', stream => {
      this.$refs.remoteVideo.srcObject = stream; // Mostrar el flujo de video remoto
    });

    this.peer.on('signal', signal => {
      this.socket.emit('signal', signal); // Enviar la señal al otro extremo
    });

    this.peer.on('connect', () => {
      console.log('Conexión establecida');
    });

    this.peer.on('error', err => {
      console.error('Error en la conexión:', err);
    });

    // Recibir la señal del otro extremo y procesarla
    this.socket.on('signal', signal => {
      this.peer.signal(signal);
    });

    this.socket.on('message', (data) => {
      this.messages.push(data)
    })
    
    
  },
  methods: {
    startCall() {
      this.inCall = !this.inCall
      navigator.mediaDevices.getUserMedia({ video: true, audio: true })
        .then(stream => {
          this.$refs.localVideo.srcObject = stream;
          this.peer.addStream(stream);
          this.peer.signal(/* Señal inicial recibida del otro extremo */);
        })
        .catch(err => {
          console.error('Error al acceder a la cámara:', err);
        });
    },
    disconnectCall() {
      this.inCall = !this.inCall

      const tracks = this.$refs.localVideo.srcObject.getTracks();
      tracks.forEach(track => track.stop());

      // Asignar null al srcObject del video local
      this.$refs.localVideo.srcObject = null;

      // Asignar null al srcObject del video remoto
      this.$refs.remoteVideo.srcObject = null;

      if (typeof process !== 'undefined') {
        

      // Detener la llamada cerrando la conexión SimplePeer
      this.peer.destroy();
      
      // Detener el flujo de video local
      this.$refs.localVideo.srcObject.getTracks().forEach(track => track.stop());
      
      // Detener el flujo de video remoto
      this.$refs.remoteVideo.srcObject = null;
      }
    },
    sendMessage(){
      try{
        if(this.message !== ""){
          this.socket.emit('message',{
            data: this.message
          })

        }
      }catch(err){
        console.log('error al mandar el mensaje')
      }
    }
  }
};
</script>