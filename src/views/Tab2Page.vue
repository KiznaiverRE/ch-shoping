<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Tab 2</ion-title>
      </ion-toolbar>
    </ion-header>
    <ion-content :fullscreen="true">
      <ion-header collapse="condense">
        <ion-toolbar>
          <ion-title size="large">Tab 2</ion-title>
        </ion-toolbar>
      </ion-header>
      <IonButton @click="create_bar()">Scan barcode</IonButton>
      <IonButton @click="create_http()">Get HTTP</IonButton>
      <IonButton @click="get_geo_location()">Get Location</IonButton>
      <ExploreContainer name="Tab 2 page" />
    </ion-content>
  </ion-page>
</template>

<script lang="ts">
import { defineComponent } from 'vue';
    import { IonPage, IonHeader, IonToolbar, IonTitle, IonContent, IonButton, alertController } from '@ionic/vue';
import ExploreContainer from '@/components/ExploreContainer.vue';
    import { BarcodeScanner }  from '@awesome-cordova-plugins/barcode-scanner';
    import { HTTP } from '@awesome-cordova-plugins/http';
    //import WonderPush from 'wonderpush-cordova-sdk';
    

export default defineComponent({
  name: 'Tab2Page',
    components: { ExploreContainer, IonHeader, IonToolbar, IonTitle, IonContent, IonPage, IonButton },
    data() {
        return {
            barcode: "",
            alert2: "open"
        }
    },
    methods: {
        create_bar() {
            BarcodeScanner.scan().then(barcodeData => {
                this.barcode = barcodeData['text'];
                this.presentAlert();
            }).catch(err => {
                console.log('Error', err);
            });

        },
        create_http() {
            
            HTTP.get('https://jihafieplie.beget.app/' + this.barcode, { }, { 'content-type': 'text/html' })
                .then(data => {
                    this.presentAlert2(data.data);
                    console.log(data.status);
                    console.log(data.data); // data received by server
                    console.log(data.headers);
                })
                .catch(error => {
                    this.presentAlert2("");
                    console.log(error.status);
                    console.log(error.error); // error message as string
                    console.log(error.headers);

                });
        },
        async presentAlert2(data: any) {
            const alert2 = await alertController.create({
                header: 'Check product ?',
                message: data,
                buttons: [
                    {
                        text: 'Cancel',
                        role: 'cancel',
                        handler: () => {
                            this.closeAlert2(alert2);
                        }
                    },
                    {
                        text: 'OK',
                        role: 'confirm',
                        handler: () => {
                            this.create_http();
                        }
                    },
                ],
            });
            await alert2.present();
            
        },
        async closeAlert2(alert2: any) {
            await alert2.dismiss();
        },
        async presentAlert() {
            const alert = await alertController.create({
                header: 'Check product ?',
                message: 'Barcode : ' + this.barcode,
                buttons: [
                    {
                        text: 'Cancel',
                        role: 'cancel',
                        handler: () => {
                            this.closeAlert2(alert);
                        }
                    },
                    {
                        text: 'OK',
                        role: 'confirm',
                        handler: () => {  
                            this.create_http();
                        }
                    },
                ],
            });

            await alert.present();

            
        }
        //create_push() {
        //    WonderPush.subscribeToNotifications();
        //}
    }
});
</script>
