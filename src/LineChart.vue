<script>
  import {Line, mixins} from 'vue-chartjs'
  const { reactiveProp } = mixins;
  export default {
    extends: Line,
    mixins: [reactiveProp],
    mounted () {
      // notice that `chartData` is provided by the mixin... which is why you don't see a local ref to it
      // ref: http://vue-chartjs.org/#/home?id=reactive-data
      this.renderChart(this.chartData, this.options)
    }
  }
</script>

<style scoped>
</style>