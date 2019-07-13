<template>
  <div class="content">
      <DataExportComp>

      </DataExportComp>
  </div>
</template>

<script>
  import SystemInformation from './LandingPage/SystemInformation'
  import DataExportComp from "./DataExportComp";

  export default {
    name: 'landing-page',
    components: {DataExportComp, SystemInformation },
    methods: {
      open (link) {
        this.$electron.shell.openExternal(link)
      }
    }
  }
</script>

<style>
  @import url('https://fonts.googleapis.com/css?family=Source+Sans+Pro');

  * {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }

  body { font-family: 'Source Sans Pro', sans-serif; }
  .content{
    width: 100%;
    height: 100%;
  }
  #wrapper {
    background:
      radial-gradient(
        ellipse at top left,
        rgba(255, 255, 255, 1) 40%,
        rgba(229, 229, 229, .9) 100%
      );
    height: 100vh;
    padding: 60px 80px;
    width: 100vw;
  }
</style>
