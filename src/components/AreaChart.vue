<template>
  <div class='areachart'>
    <svg :width='width' :height='height'>
      <!-- MOVIE ARCS -->
      <path v-for='d in arcs' :d='d.path' :fill='d.fill' stroke='#fff' />
      <!-- AXES -->
      <g ref='xAxis' :transform='`translate(0, ${y0})`' />
      <g ref='yAxis' :transform='`translate(${margin.left}, 0)`' />
    </svg>
  </div>
</template>

<script>
import _ from 'lodash'
import * as d3 from 'd3'

const width = 1000
const height = 300
const margin = {top: 0, right: 0, bottom: 20, left: 60}

export default {
  name: 'areachart',
  props: ['movies', 'filtered'],
  data() {
    return {
      width, height, margin,
      y0: 0,
      arcs: [],
    }
  },
  watch: {
    movies() {
      this.calculateScale()
      this.calculateDate()
      this.renderAxes()
    },
    filtered() {
      this.calculateDate()
    }
  },
  methods: {
    calculateScale() {
      if (!this.movies.length) return

      const [minDate, maxDate] = d3.extent(this.movies, d => d.date)
      this.xScale = d3.scaleTime()
        .domain([d3.timeMonth.offset(minDate, -2), d3.timeMonth.offset(maxDate, 2)])
        .range([margin.left, width - margin.right])

      this.medianBox = d3.median(this.movies, d => d.boxOffice)

      const boxOfficeExtent = d3.extent(this.movies, d => d.boxOffice - this.medianBox)
      this.yScale = d3.scaleLinear()
        .domain(boxOfficeExtent)
        .range([height - margin.bottom, margin.top])
      this.y0 = this.yScale(0)

      const scoreExtent = d3.extent(this.movies, d => d.score)
      this.colorScale = d3.scaleSequential()
        .domain(scoreExtent)
        .interpolator(d3.interpolateViridis)
      
      this.area = d3.area()
        .y0(this.y0)
        .curve(d3.curveCatmullRom)
    },
    calculateDate() {
      if (!this.filtered.length) return

      this.arcs = this.filtered.map(d => {
        return {
          id: d.id,
          path: this.area([
            [this.xScale(d3.timeMonth.offset(d.date, -2)), this.y0],
            [this.xScale(d.date), this.yScale(d.boxOffice - this.medianBox)],
            [this.xScale(d3.timeMonth.offset(d.date, 2)), this.y0],
          ]),
          fill: this.colorScale(d.score),
        }
      })
    },
    renderAxes: function() {
      if (!this.xScale || !this.yScale) return

      const xAxis = d3.axisBottom().scale(this.xScale).tickFormat(this.format)
      const yAxis = d3.axisLeft().scale(this.yScale)
        .tickSizeOuter(0)
        .tickFormat(d => `${d3.format('$')(parseInt(d / 1000000))}M`)

      // call axes on group elements
      d3.select(this.$refs.xAxis).call(xAxis)
      d3.select(this.$refs.yAxis).call(yAxis)
        .selectAll('path').remove()
    },
  }
}
</script>

<style>
.areachart {
  position: relative;
}

.arc {
  cursor: pointer;
}
</style>
