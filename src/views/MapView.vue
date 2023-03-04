<template>
    <div class="layout">

        <div class="layout__info">
            <div class="layout__chart">
                <MapChart
                    v-if="pointsAreReady"
                    :points="points"
                    @sendPoints="renderMarkers"
                />
            </div>
            
            <hr>

            <div class="layout__wrapper">
                <div class="layout__text">
                    Точек: {{ points.length }}
                </div>
                
                <button
                    class="layout__btn"
                    @click="updatePoints"
                    :disabled="!pointsAreReady"
                >
                    Обновить точки
                </button>
            </div>
            
        </div>

        <div class="layout__map" id="map"></div>

    </div>
</template>

<script>
import MapChart from '@/components/MapChart.vue'
import L from 'leaflet'
import 'leaflet-draw/dist/leaflet.draw-src.js'
import districts from '@/mocks/districts.js'
import minmax from '@/utils/minmax.js'
import getRandom from '@/utils/getRandom.js'
import checkMarker from '@/utils/checkMarker.js'

export default {
    components: {
        MapChart
    },

    data: () => ({
        map: null,
        icon: null,
        drawnItems: null,
        districts,
        layerLinks: [],
        POINTS_QTY: 100,
        points: [],
        markers: []
    }),

    computed: {
        // границы координат
        getBounds() {
            const totalArr = [
                ...this.districts['north'].coords,
                ...this.districts['east'].coords,
                ...this.districts['southWest'].coords,
                ...this.districts['west'].coords
            ]

            // все значения широты
            const fullLats = []
            // все значения долготы
            const fullLons = []

            totalArr.forEach(el => {
                fullLats.push(el[0])
                fullLons.push(el[1])
            })
            
            // границы диапазона широты
            const latsBounds = minmax(fullLats)
            // границы диапазона долготы
            const lonsBounds = minmax(fullLons)

            return {
                latsBounds,
                lonsBounds
            }
        },

        // точки сгенерированы
        pointsAreReady() {
            return this.points.length === this.POINTS_QTY
        }
    },

    methods: {
        // инициализация карты
        initLMap() {
            this.map = L.map('map', {
                center: [55.7522200, 37.6155600],
                zoomControl: true,
                zoom: 10,
                minZoom: 5,
                maxZoom: 19,
                attributionControl: false
            })

            L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                maxZoom: 19,
                noWrap: true
            }).addTo(this.map)

            this.icon = new L.icon({
                iconUrl: 'https://unpkg.com/leaflet@1.0.3/dist/images/marker-icon.png',
                iconSize: [20, 30]
            })
        },

        // инициализация L Draw
        initLDraw() {
            this.drawnItems = new L.FeatureGroup()
            this.map.addLayer(this.drawnItems)
        },

        // рендеринг районов
        renderDistricts() {
            const options = {
                color: '#FF0000FF',
                weight: 2,
                fill: true,
                fillOpacity: 1.0
            }

            for(let key in this.districts) {
                const layer = L.polygon(this.districts[key].coords, {
                    ...options,
                    fillColor: this.districts[key].fill,
                }).addTo(this.drawnItems)

                this.layerLinks.push({
                    id: layer._leaflet_id,
                    title: key
                })
            }
        },

        // генерация точек
        generatePoints() {
            for(let i = 0; i < this.POINTS_QTY; i++) {
                this.makePoint(this.getBounds.latsBounds, this.getBounds.lonsBounds)
            }
        },

        // генерация одной точки
        makePoint(latsBounds, lonsBounds) {
            // генерация координат точки
            const point = [
                getRandom(latsBounds.min, latsBounds.max),
                getRandom(lonsBounds.min, lonsBounds.max)
            ]

            const marker = L.marker([point[0], point[1]], { icon: this.icon })

            const checkResults = []

            // проверка координат маркера по всем округам
            for(let key in this.drawnItems._layers) {
                let check = checkMarker(marker, this.drawnItems._layers[key])

                if(check) {
                    checkResults.push({
                        check,
                        key
                    })

                    break
                }
            }

            // координаты попали хотя бы в один округ
            let pointIsInBounds = checkResults.some(el => el.check)

            if(pointIsInBounds) {
                let result = checkResults.find(el => el.check)
                let layer = this.layerLinks.find(el => el.id == result.key)

                // запись точки
                this.points.push({
                    ...layer,
                    coords: point
                })
                
            } else {
                // запуск повторной генерации
                this.makePoint(latsBounds, lonsBounds)
            }
        },

        // рендеринг маркеров
        renderMarkers(points) {
            this.clearMarkers()

            for(let key in points) {
                let marker = L.marker([points[key].coords[0], points[key].coords[1]], { icon: this.icon }).addTo(this.map)
                this.markers.push(marker)
            }
        },

        // удаление маркеров
        clearMarkers() {
            this.markers.forEach(el => {
                this.map.removeLayer(el)
            })

            this.markers.length = 0
        },

        // перегенерировать точки
        updatePoints() {
            this.clearMarkers()
            
            this.points = []

            setTimeout(() => {
                this.generatePoints()
            })
        }
    },

    watch: {
        // все точки сгенерированы
        pointsAreReady(val) {
            if(val) {
                this.renderMarkers(this.points)
            }  
        }
    },

    mounted() {
        this.initLMap()

        this.initLDraw()

        this.renderDistricts()

        this.generatePoints()
    }
}
</script>

<style lang="scss">
.layout {
    height: 100vh;
    display: flex;
    justify-content: space-between;

    &__info {
        width: 30%;
    }

    &__chart {
        min-height: 400px;
    }

    &__wrapper {
        padding: 6px;
    }

    &__text {
        margin-bottom: 12px;
    }

    &__map {
        height: 100%;
        width: 70%;

        cursor: crosshair;

        position: relative;
        z-index: 1;
    }
}

@import '~@/../node_modules/leaflet/dist/leaflet.css';
@import '~@/../node_modules/leaflet-draw/dist/leaflet.draw.css';
</style>