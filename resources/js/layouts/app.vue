<template>
    <div class="d-flex">
        <!-- Sidebar izquierdo -->
        <div class="sidebar-left bg-light border-end">
            <h2 class="text-center py-3">Seleccione Ubicación</h2>
            <div class="p-3">
                <h5>Provincias</h5>
                <select class="form-select mb-3" v-model="selectedProvince" @change="updateProvince">
                    <option value="" disabled selected>Seleccione una provincia</option>
                    <option v-for="province in provinces" :key="province.properties.PROVINCIA" :value="province">
                        {{ province.properties.PROVINCIA }}
                    </option>
                </select>

                <h5>Distritos</h5>
                <select class="form-select" v-model="selectedDistrict" @change="updateDistrict" :disabled="!selectedProvince">
                    <option value="" disabled selected>Seleccione un distrito</option>
                    <option v-for="district in filteredDistricts" :key="district.properties.UBIGEO" :value="district">
                        {{ district.properties.DISTRITO }}
                    </option>
                </select>
            </div>
        </div>

        <!-- Mapa -->
        <div id="map" class="flex-grow-1"></div>

        <!-- Sidebar derecho -->
        <div class="sidebar-right bg-light border-start">
            <h2 class="text-center py-3">Detalles del Lugar</h2>
            <div class="p-3">
                <p v-if="selectedProvince">
                    <strong>Provincia:</strong> {{ selectedProvince.properties.PROVINCIA }}
                </p>
                <p v-if="selectedDistrict">
                    <strong>Distrito:</strong> {{ selectedDistrict.properties.DISTRITO }}
                </p>
                <p v-else-if="!selectedProvince && !selectedDistrict">
                    Seleccione una provincia o un distrito para ver detalles.
                </p>
            </div>
        </div>
    </div>
</template>

<script>
export default {
    data() {
        return {
            map: null,
            provinces: [],
            districts: [],
            filteredDistricts: [],
            selectedProvince: null,
            selectedDistrict: null,
            provinceLayer: null,
            districtLayer: null,
            markers: [],
            labels: [],
            highlightedLayers: [] // Almacenamos los elementos resaltados aquí
        };
    },
    mounted() {
        this.initMap();
        this.loadGeoJSON();
    },
    methods: {
        initMap() {
            this.map = L.map('map').setView([-14.0915, -73.2004], 8);
            L.tileLayer('https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png', {
                maxZoom: 19,
                attribution: '© OpenStreetMap'
            }).addTo(this.map);
        },
        loadGeoJSON() {
            fetch('provincias_apurimac.geojson')
                .then(response => response.json())
                .then(data => {
                    this.provinces = data.features;
                    this.renderProvinces(); 
                });

            fetch('distritos_apurimac.geojson')
                .then(response => response.json())
                .then(data => {
                    this.districts = data.features;
                });
        },
        renderProvinces() {
            if (this.provinceLayer) this.map.removeLayer(this.provinceLayer);
            if (this.districtLayer) this.map.removeLayer(this.districtLayer);
            this.clearMarkers();
            this.clearLabels();
            this.clearHighlightedLayers();

            this.provinceLayer = L.geoJSON(this.provinces, {
                style: { color: 'green', weight: 2 },
                onEachFeature: (feature, layer) => {
                    const provinceName = feature.properties.PROVINCIA;
                    const center = layer.getBounds().getCenter();
                    const marker = L.marker(center)
                        .bindPopup(`<strong>${provinceName}</strong>`)
                        .addTo(this.map);
                    this.markers.push(marker);
                    const label = L.tooltip({
                        permanent: true,
                        direction: 'center',
                        className: 'province-label',
                    })
                        .setContent(provinceName)
                        .setLatLng(center)
                        .addTo(this.map);
                    this.labels.push(label);
                },
            }).addTo(this.map);

            const bounds = this.provinceLayer.getBounds();
            this.map.fitBounds(bounds);
        },
        renderSelectedProvince() {
            if (this.selectedProvince) {
                if (this.provinceLayer) this.map.removeLayer(this.provinceLayer);
                if (this.districtLayer) this.map.removeLayer(this.districtLayer);
                this.clearMarkers();
                this.clearLabels();
                this.clearHighlightedLayers();

                this.provinceLayer = L.geoJSON([this.selectedProvince], {
                    style: { color: 'green', weight: 3, fillOpacity: 0.4 },
                    onEachFeature: (feature, layer) => {
                        const provinceName = feature.properties.PROVINCIA;
                        const center = layer.getBounds().getCenter();
                        const marker = L.marker(center)
                            .bindPopup(`<strong>${provinceName}</strong>`)
                            .addTo(this.map);
                        this.markers.push(marker);
                        const label = L.tooltip({
                            permanent: true,
                            direction: 'center',
                            className: 'province-label',
                        })
                            .setContent(provinceName)
                            .setLatLng(center)
                            .addTo(this.map);
                        this.labels.push(label);
                    },
                }).addTo(this.map);

                const bounds = this.provinceLayer.getBounds();
                this.map.fitBounds(bounds);
            }
        },
        renderDistricts() {
            if (this.selectedProvince && this.filteredDistricts.length > 0) {
                if (this.districtLayer) this.map.removeLayer(this.districtLayer);
                this.clearMarkers();
                this.clearLabels();
                this.clearHighlightedLayers();

                this.districtLayer = L.geoJSON(this.filteredDistricts, {
                    style: { color: 'green', weight: 2 },
                    onEachFeature: (feature, layer) => {
                        const districtName = feature.properties.DISTRITO;
                        const center = layer.getBounds().getCenter();
                        const marker = L.marker(center)
                            .bindPopup(`<strong>${districtName}</strong>`)
                            .addTo(this.map);
                        this.markers.push(marker);
                        const label = L.tooltip({
                            permanent: true,
                            direction: 'center',
                            className: 'district-label',
                        })
                            .setContent(districtName)
                            .setLatLng(center)
                            .addTo(this.map);
                        this.labels.push(label);
                    },
                }).addTo(this.map);

                const bounds = this.districtLayer.getBounds();
                this.map.fitBounds(bounds);
            }
        },
        updateProvince() {
            this.selectedDistrict = null;
            if (this.selectedProvince) {
                const provinceName = this.selectedProvince.properties.PROVINCIA;
                this.filteredDistricts = this.districts.filter(
                    district => district.properties.PROVINCIA === provinceName
                );
                this.renderSelectedProvince();
                this.renderDistricts();
            } else {
                this.filteredDistricts = [];
                this.renderProvinces();
            }
        },
        updateDistrict() {
            // Limpiar los resaltados anteriores, los marcadores y las etiquetas de los distritos no seleccionados
            this.clearHighlightedLayers();
            this.clearMarkers();
            this.clearLabels();

            if (this.selectedDistrict) {
                const districtName = this.selectedDistrict.properties.DISTRITO;
                const bounds = L.geoJSON(this.selectedDistrict).getBounds();
                this.map.fitBounds(bounds);

                // Eliminar resalte de color rojo y no aplicar ningún estilo a las capas del distrito seleccionado

                // Agregar marcador y etiqueta
                const center = bounds.getCenter();
                const label = L.tooltip({
                    permanent: true,
                    direction: 'center',
                    className: 'district-label',
                })
                    .setContent(districtName)
                    .setLatLng(center)
                    .addTo(this.map);
                this.labels.push(label);

                const layer = L.geoJSON(this.selectedDistrict).addTo(this.map);
                this.highlightedLayers.push(layer);
            }
        },
        clearMarkers() {
            this.markers.forEach(marker => {
                this.map.removeLayer(marker);
            });
            this.markers = [];
        },
        clearLabels() {
            this.labels.forEach(label => {
                this.map.removeLayer(label);
            });
            this.labels = [];
        },
        clearHighlightedLayers() {
            // Limpiar todas las capas resaltadas
            this.highlightedLayers.forEach(layer => {
                this.map.removeLayer(layer);
            });
            this.highlightedLayers = []; // Vaciar la lista de capas resaltadas
        },
    },
};
</script>

<style>
.d-flex {
    display: flex;
    height: 100vh;
}

.sidebar-left,
.sidebar-right {
    width: 300px;
    overflow-y: auto;
}

.sidebar-left {
    position: relative;
}

.sidebar-right {
    position: relative;
}

#map {
    height: 100%;
}

.province-label,
.district-label {
    font-size: 10px;
    color: black;
    text-shadow: 1px 1px 5px white;
    background-color: rgba(255, 255, 255, 0);
    padding: 2px 5px;
    border-radius: 4px;
    border: 1px solid rgba(128, 128, 128, 0);
    font-weight: bold;
    text-align: center;
}
</style>
