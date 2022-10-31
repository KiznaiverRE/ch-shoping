<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Tab 1</ion-title>
      </ion-toolbar>
    </ion-header>
    <ion-content :fullscreen="true">
        <ion-header collapse="condense">
            <ion-toolbar>
                <ion-title size="large">Tab 2</ion-title>
            </ion-toolbar>
        </ion-header>
        Привет! Как сам ?
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
    
    import L, { latLng } from "leaflet";
    import { defineComponent } from 'vue';
    import { IonPage, IonHeader, IonToolbar, IonTitle, IonContent } from '@ionic/vue';
    import { AndroidPermissions } from '@awesome-cordova-plugins/android-permissions';

    export default defineComponent({
        name: 'Tab1Page',
        components: { IonHeader, IonToolbar, IonTitle, IonContent, IonPage },
        data() {
            return {

            }
        },
        methods: {
            getPermissonOnLocation() {
                AndroidPermissions.checkPermission(AndroidPermissions.PERMISSION.ACCESS_FINE_LOCATION).then((result: any) => {
                    if (!result.hasPermission) {
                        AndroidPermissions.requestPermission(AndroidPermissions.PERMISSION.ACCESS_FINE_LOCATION);
                    }
                });
            }
        },
        mounted() {
            this.getPermissonOnLocation()
        }
});
</script>
