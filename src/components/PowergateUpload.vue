<template>
  <h1>persist a file with powergate</h1>
  <div class='chapter'>
    <h3>powd build info</h3>
    <p><i>(always check the console for errors)</i></p>
    <div class="json">{{buildInfo}}</div>
  </div>
  <div class='chapter' v-if="buildInfo != ''">
    <h2>set or create user</h2>
    <h3>set user</h3>
    <p> user token: <input ref="userTokenInput" type="text"/> 
      <button @click="setUser"> set user </button> 
    </p>
    <h3>create new user</h3>
    <button @click="createUser">create user</button>
  </div>
  <div class='chapter' v-if="userToken !== ''">
    <h2>user info </h2>
    <p>token: {{userToken}}</p>
    <h3>addresses</h3>
    <p v-for="(addr, i) in addresses" :key="i">{{addr.name}}: {{addr.address}}</p>
    <h3>config</h3>
    <div ref="defaultConfig" class="json">{{defaultConfig}}</div>
  </div>
  <div class='chapter' v-if="userToken !== ''">
    <h2>persist a file</h2>
    <h3>choose a file from your computer</h3>
    <p>choose file: <input ref="file" type="file" /></p>
    <p>
      When connecting to a powergate localnet, 
      make sure the file has a valid size.
      Too small and too big files won't work.
      There is an explanation
      <a href="https://docs.textile.io/powergate/localnet/#localnet-features">
        in the docs.
      </a>
    </p>
    <button @click="persist" :disabled="userToken == ''">persist file</button>
  </div>
  <div class='chapter' v-if="jobs.length > 0">
    <h2>jobs</h2>
    <div class="job" v-for="job in jobs" :key="job.id">
      <div class="side left">
        <p>{{job.id}}: </p>
      </div> 
      <div class="side right"> 
        <p>"{{job.filename}}", cid:</p>
        <p>{{job.cid}}</p>
        <p><b>{{job.status}}</b> <span v-if="job.details !== ''">({{job.details}})</span></p>
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { defineComponent, reactive } from 'vue';
import { createPow, powTypes } from "@textile/powergate-client";

interface Job {
  cid : string
  filename : string
  id: string
  status: string
  details: string
}

export default defineComponent({
  name: 'HelloWorld',
  components: {
  },
  props: {
    grpcWebProxyAddress: {
      default: "http://0.0.0.0:6002"
    }, 
  },
  data() {
    return {
      pow: createPow({host: this.grpcWebProxyAddress}), 
      userToken: '',
      addresses: [] as Object[],
      buildInfo: "",
      defaultConfig: "",
      jobs: [] as Job[],
      watchingJobs: false, 
    }
  },
  mounted() {
    this.loadBuildInfo()
  },
  methods: {
    // powd build info
    loadBuildInfo() {
      this.pow.buildInfo().then(
        v => {
          this.buildInfo = JSON.stringify(v, null, 2)
        },
        console.error
      )
    },
    // set or create user
    setUser(){
      let input = this.$refs.userTokenInput as HTMLInputElement
      this.setUserToken(input.value)
    },
    createUser() {
      this.pow.admin.users.create().then(
        res => {
          if(res.user) {
            this.setUserToken(res.user.token)
          } 
        }, 
        console.error
      )
    },
    setUserToken(token: string) {
      this.userToken = token
      this.pow.setToken(token)
      this.loadUserInfo()
    },
    // user info
    loadUserInfo() {
      this.addresses = []
      this.defaultConfig = ''
      this.pow.wallet.addresses().then(
        v => {
          this.addresses = v.addressesList
        }, 
        console.error
      )
      this.pow.storageConfig.default().then( 
        v => {
          this.defaultConfig = JSON.stringify(v.defaultStorageConfig, null, 2)
        }
      )
    },
    // persist a file
    persist() {
      let files = (this.$refs.file as HTMLInputElement).files
      if(files !== null && files.length >= 1) {
        const file = files[0]
        this.stageFile(file).then(
          cid => {
            this.applyConfig(cid, file.name)
          }, 
          err => {
            console.error("failed to stage file:", err)
          }
        )
      } else {
        console.error("could not persist, no file selected")
      }
    }, 
    stageFile(file: File) {
      return new Promise<string>((resolve, reject) => {
        const reader = new FileReader(); 
        reader.readAsArrayBuffer(file)
        reader.onload = () => {
          if(reader.result instanceof ArrayBuffer) {
            const buffer = new Uint8Array(reader.result)
            this.pow.data.stage(buffer).then(
              v => {
                resolve(v.cid)
              }, 
              reject
            )
          } else {
            reject(Error(`reader.result is no ArrayBuffer`))
          }
        }
        reader.onerror = () => {
          reject(Error(`reading file as array buffer failed ${reader.error}`))
        }
      })
    }, 
    applyConfig(cid: string, filename: string) {
      this.pow.storageConfig.apply(cid).then(
        v => {
          let job = reactive({
            cid, 
            filename, 
            id: v.jobId, 
            status: 'created', 
            details: '',
          })
          this.jobs.push(job)
          this.watchJobs()
        }, 
        console.error
      )
    }, 
    watchJobs() {
      if(!this.watchingJobs) {
        this.watchingJobs = true
        this.pow.storageJobs.watch(v => {
          this.jobs.forEach(job => {
            if(job.id == v.id){
              this.updateJob(job, v)
            }
          })
        })
      }
    },
    updateJob(job: any, v: any){
      console.log("job update", v)
      switch(v.status){
        case powTypes.JobStatus.JOB_STATUS_EXECUTING:
          job.status = "executing"
          break
        case powTypes.JobStatus.JOB_STATUS_QUEUED:
          job.status = "queued"
          break
        case powTypes.JobStatus.JOB_STATUS_CANCELED:
          job.status = "cancelled"
          break
        case powTypes.JobStatus.JOB_STATUS_FAILED:
          job.status = "failed"
          job.details = v.errorCause
          break
        case powTypes.JobStatus.JOB_STATUS_SUCCESS:
          job.status = "succeeded"
          break
      }
    }
  }
});
</script>

<style scoped>
.json{
  display: inline-block;
  white-space: pre-wrap;
  background-color: #444; 
  color: #eee; 
  text-align: left;
  font-family: monospace;
  font-size: 0.9em;
  padding: 0.4em;
  border-radius: 0.4em;
}
.job .side {
  box-sizing: border-box;
  display: inline-block;
  vertical-align: top;
  width:49%; 
  padding: 0.5em; 
  margin: 0;
}
.job .side p {
  margin: 0em; 
}
.job .side.right {
  text-align: left;
}
.job .side.left {
  text-align: right;
}
input, button, div {
  margin: 0.4em; 
}
.chapter {
  margin: 1rem; 
  padding: 0.5rem; 
  border-radius: 0.5rem;
  background: linear-gradient(-50deg,#dbe7f4,#e9dbf4);
}
</style>
