<template>
    <ion-page>
        <ion-header>
            <ion-toolbar>
                <div class="header-block">
                    <ion-title>Tab 3</ion-title>
                </div>
                <div class="search-header" id="open-search">
                    <ion-item>
                        <ion-label position="floating">Поиск...</ion-label>
                        <ion-input v-model="search_word"></ion-input>
                    </ion-item>
                </div>
            </ion-toolbar>
        </ion-header>
        <ion-content :fullscreen="true">
            <ion-header collapse="condense">
                <ion-toolbar>
                    <ion-title size="large">Tab 3</ion-title>
                </ion-toolbar>
            </ion-header>
            <div v-if="products_dropdown_search" class="search-product-cover" id="product-menu">
                <div v-for="product in products_dropdown_search" class="search-product-block" :key="product.id">
                    <div class="search-product-block-name">
                        <div @click="change_selected_product(product)" class="search-product-block-name-click">   {{ product.name }} </div>
                    </div>
                </div>
            </div>
            <div v-if="one_selected_product">
                <div v-if="map_open === false">
                    <div class="select_product_without_map">
                        {{ one_selected_product.name }}
                        {{ one_selected_product.category_name }}
                        <IonButton @click="change_selected_class(one_selected_product)">Найти на карте</IonButton>
                    </div>
                </div>
                <div v-else>
                    <div  :class="[{ 'active' : one_selected_product_find_on_map }, 'select_product_with_map']" @click="change_selected_class(one_selected_product)">
                        {{ one_selected_product.name }}
                    </div>
                </div>
            </div>

            <div v-if="map_open === true" class="map-container">
                <div id="map" v-if="map_open === true"></div>
            </div>
            <div class="button-footer">
                <IonButton @click="scanning_by_barcode()">Штрихкод</IonButton>
                <IonButton @click="trigger_map()">Карта </IonButton>
                <IonButton @click="clear()">Очистить</IonButton>
            </div>
        </ion-content>
    </ion-page>
</template>

<script>
    import L, { LatLng } from "leaflet";
    import { defineComponent } from 'vue'; 
    import { IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonButton, IonInput, IonItem, IonLabel } from '@ionic/vue';
    import { AndroidPermissions } from '@awesome-cordova-plugins/android-permissions';
    import { Geolocation } from '@awesome-cordova-plugins/geolocation/';
    import { HTTP } from '@awesome-cordova-plugins/http';
    import { BarcodeScanner } from '@awesome-cordova-plugins/barcode-scanner';


    export default defineComponent({
        name: 'Tab3Page',
        components: { IonHeader, IonToolbar, IonTitle, IonContent, IonPage, IonButton, IonInput, IonItem, IonLabel },
        data() {
            return {
                map_open: false,
                one_selected_product: null,
                mapZoom: null,
                user_lat: 53.01,
                user_long: 27.01,
                search_word: null, 
                map_layer_user: null,
                map_layer_shops: null,
                shops_all: null,
                shops_around: null,
                map: null,
                products_dropdown_search: null,
                one_selected_product_find_on_map: false,
                radius_closely: 0.00003, // радиус определения ближайших магазинов
                radius_big: 5.0 // радиус определения всех магазинов
            }
        },
        methods: {
            async clear() {
                this.one_selected_product = null;
                this.search_word = null;
                this.products_dropdown_search = null;
                try {
                    await this.map.removeLayer(this.map_layer_shops)
                } catch (err) { console.log(err); }
                    
            },
            find_product_on_map(product) {
                HTTP.get('https://jihafieplie.beget.app/product/' + product.id + '/' + this.radius_big + '/' + this.user_lat + '/' + this.user_long, {}, { 'content-type': 'application/json' })
                    .then(response => {
                        let data = JSON.parse(response.data);
                        if (!data || data.length <= 0) alert('Товаров не найдено')
                        this.map_paint_shops_with_price(data)
                    })
                    .catch(error => {
                        console.log(error);
                    })
            },
            scanning_by_barcode() {
                BarcodeScanner.scan().then(barcode_full => {
                    let barcode = barcode_full['text'];
                    HTTP.get('https://jihafieplie.beget.app/' + barcode, {}, { 'content-type': 'application/json' })
                        .then(data => {
                            this.change_selected_product(JSON.parse(data.data))
                        })
                        .catch(err => {
                            console.log('Error', err);
                        });
                }).catch(err => {
                    console.log('Error', err);
                });
            },
            change_selected_class(product) {
                if (this.one_selected_product_find_on_map) {
                    this.one_selected_product_find_on_map = false;
                    try {
                        this.map.removeLayer(this.map_layer_shops)
                    } catch (err) { console.log(err); }
                } else {
                    this.one_selected_product_find_on_map = true;
                    this.find_product_on_map(product);
                }
            },
            change_selected_product(product) {
                if (this.map_open) {
                    this.clear().then(() => {
                        this.one_selected_product = product;
                        this.one_selected_product_find_on_map = false;
                        this.change_selected_class(product);
                    })
                } else {
                    this.clear().then(() => {
                        this.one_selected_product = product;
                        this.one_selected_product_find_on_map = false;
                    })
                }
                
                
            },
            trigger_map() {
                this.one_selected_product_find_on_map = false;
                if (this.map_open) {
                    this.map_open = false // закрывает карту
                } else {
                    this.create_map_with_shops() // отправка на открытие карты
                }
            },
            async get_current_position() {
                try {
                    let options = { maximumAge: 0, timeout: 500, enableHighAccuracy: true }
                    await Geolocation.getCurrentPosition(options)
                        .then((resolve, reject) => {
                            this.user_lat = resolve.coords.latitude
                            this.user_long = resolve.coords.longitude
                        })
                } catch (err) {
                    console.log('Error', err)
                }
            },
            async build_map() {
                try {
                    let map = L.map('map').setView([this.user_lat, this.user_long], 13)
                    this.map_layer_user = L.featureGroup()
                    L.circleMarker([this.user_lat, this.user_long]).addTo(this.map_layer_user)
                    map.addLayer(this.map_layer_user);
                    L.tileLayer('https://tile.openstreetmap.org/{z}/{x}/{y}.png', {
                        maxZoom: 19,
                        attribution: ''
                    }).addTo(map)
                    map.on({
                        zoom: function () {
                            //this.mapZoom = map.getZoom();
                            let wi = map.getZoom() * 10;
                            console.log(wi)
                            let cols = document.getElementsByClassName('my-div-spans')
                            for (let i = 0; i < cols.length; i++) {
                                //cols[i].style.width = wi + 'px';
                            }
                        }
                    })
                    this.map = map
                } catch (err) {
                    console.log('Error', err)
                }
            },
            getMapZoom() {
                console.log(111);
            },
            get_all_shops() {

                //let data = [{ "id": 2, "shop_id": 1, "location_lat": 53.906307, "location_long": 27.58685 }, { "id": 3, "shop_id": 3, "location_lat": 53.909954, "location_long": 27.578382 }, { "id": 1, "shop_id": 2, "location_lat": 53.904343, "location_long": 27.59872 }];
                //this.shops_all = data;
                //this.map_paint_shops(data)
                HTTP.get('https://jihafieplie.beget.app/find_shops_by_coords/' + this.radius_big + '/' + this.user_lat + '/' + this.user_long, {}, { 'content-type': 'application/json' })
                    .then(response => {
                        let data = JSON.parse(response.data);
                        this.shops_all = data
                        this.map_paint_shops(data)
                    })
                    .catch(error => {
                        console.log(error)
                    });
            },
            map_paint_shops(shops) {
                this.create_map().then(() => {
                    // протестить в будущем
                    if (
                        typeof (this.map_layer_shops) != "undefined" &&
                        this.map_layer_shops !== null &&
                        typeof (this.map) != "undefined" &&
                        this.map !== null
                    )
                        this.map.removeLayer(this.map_layer_shops)

                    this.map_layer_shops = L.featureGroup();
                    for (var i = 0; i < shops.length; i++) {
                        new L.Marker([shops[i].location_lat, shops[i].location_long]).addTo(this.map_layer_shops);
                    }
                    this.map.addLayer(this.map_layer_shops);
                })
            },
            map_paint_shops_with_price(shops) {
                this.create_map().then(() => {
                    // протестить в будущем
                    if (
                        typeof (this.map_layer_shops) != "undefined" &&
                        this.map_layer_shops !== null &&
                        typeof (this.map) != "undefined" &&
                        this.map !== null
                    )
                        this.map.removeLayer(this.map_layer_shops)

                    this.map_layer_shops = L.featureGroup();
                    for (var i = 0; i < shops.length; i++) {
                        new L.Marker([shops[i].location_lat, shops[i].location_long], {
                            icon: new L.DivIcon({
                                className: 'price_with_shop_cover',
                                iconSize: [30, 50],
                                iconAnchor: [0, 50],
                                html: '<div class="price_with_shop">' + shops[i].price + 'р.</div><div class="price_with_img"><img src="marker-icon.png"></div>'
                            })
                        })
                        .addTo(this.map_layer_shops)
                    }
                    this.map.addLayer(this.map_layer_shops)
                })
            },
            async create_map() {
                if (this.map_open === false) { // если карты нет
                    this.map_open = true;
                    await this.get_current_position().then(() => {
                        this.build_map()
                    })
                }
            },
            async create_map_with_shops() {
                try {
                    this.create_map().then(() => {
                        this.get_all_shops();
                    })
                } catch (err) {
                    console.log('Error', err)
                }
            },
            async search_by_word() {
                if (this.search_word)
                await HTTP.get('https://jihafieplie.beget.app/sname/' + this.search_word, {}, { 'content-type': 'application/json' })
                    .then(data => {
                        console.log(JSON.parse(data.data))
                        this.products_dropdown_search = JSON.parse(data.data);
                    })
                    .catch(err => {
                        console.log('Error', err)
                    });
            },
            delay(fn, ms) {
                let timer = 0
                return function (...args) {
                    clearTimeout(timer)
                    timer = setTimeout(fn.bind(this, ...args), ms || 0)
                }
            }
        },
        watch: {
            search_word: function () {
                if (!this.search_word) this.products_dropdown_search = null;
                this.search_by_word()
            },
            mapZoom: function () {
                console.log(this.mapZoom)
            }
        }
    });
</script>
