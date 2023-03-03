<template>
    <div id="chart"></div>
</template>

<script>
import districts from '@/mocks/districts.js'
import Highcharts from 'highcharts'

export default {
    props: {
        points: {
            type: Object,
            required: true
        }
    },

    emits: [
        'sendPoints'
    ],

    data: () => ({
        series: [],
        districts
    }),

    computed: {
        // точки, выбраны все районы
        allDistrictsSelected() {
            return this.series.filter(el => el.selected).length === 0
        },

        // код выбранного района
        oneDistrictSelectedCode() {
            const selected = this.series.filter(el => el.selected)

            return selected[0]?.code
        },

        // точки, выбран один район
        oneDistrictPoints() {
            const selected = this.series.filter(el => el.selected)

            const points = [...this.points]

            return points.filter(el => {
                return el.title === selected[0]?.code
            })  
        }
    },

    methods: {
        // данные для графика
        makeSeries() {
            const titles = []

            this.points.forEach(el => {
                titles.push(el.title)
            })

            const pointsInAreas = titles.reduce((acc, el) => {
                acc[el] = (acc[el] || 0) + 1
                return acc
            }, {})

            for(let key in pointsInAreas) {
                this.series.push({
                    name: this.districts[key].title,
                    code: key,
                    y: (pointsInAreas[key] / this.points.length) * 100,
                    color: this.districts[key].fill,
                    sliced: false,
                    selected: false
                })
            }
        },

        // рендеринг графика
        renderChart() {
            Highcharts.chart('chart', {
                chart: {
                    plotBackgroundColor: null,
                    plotBorderWidth: null,
                    plotShadow: false,
                    type: 'pie'
                },
                title: {
                    text: 'Распределение точек по округам',
                    align: 'center'
                },
                tooltip: {
                    pointFormat: '{series.name}: <b>{point.percentage:.1f}%</b>'
                },
                accessibility: {
                    point: {
                    valueSuffix: '%'
                    }
                },
                plotOptions: {
                    pie: {
                    allowPointSelect: true,
                    cursor: 'pointer',
                    dataLabels: {
                        enabled: true,
                        format: '<b>{point.name}</b>: {point.percentage:.1f} %'
                    }
                    }
                },
                series: [{
                    name: 'Точки',
                    colorByPoint: true,
                    allowPointSelect: true,
                    data: this.series
                }]
            })
        }
    },

    watch: {
        allDistrictsSelected(val) {
            if(val) {
                this.$emit('sendPoints', this.points)
            }
        },
        oneDistrictSelectedCode(code) {
            if(code) {
                this.$emit('sendPoints', this.oneDistrictPoints)
            }
        }
    },

    created() {
        this.makeSeries()
    },

    mounted() {
        this.renderChart()
    }
}
</script>
