<template>
  <div id="app">
    <dynamic-chart-grouping />
  </div>
</template>

<script>
  import DynamicChartGrouping from './components/DynamicChartGrouping.vue'
  export default {
    name: 'app',
    components: {
      DynamicChartGrouping
    }
  }
</script>

<style scoped>
  .charts-container {
    display: flex;
    flex-direction: column;
  }

  .chart-wrapper {
    width: 100%;
    /*max-height: 400px;
    margin-bottom: 50px;*/
  }

  .chart {
    width: 100%;
    height: 400px;
    border: 1px solid lightgray;
  }

  @media (max-width: 100%) {
    body {
      text-align: center;
    }

    .charts-container {
      flex-direction: column;
    }

    .chart-wrapper {
      width: 100%;
    }
  }

    i.linkActive {
        color: green;
    }
  div.linkActive {
    cursor: pointer;
  }

</style>