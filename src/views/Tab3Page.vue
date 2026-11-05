
<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Crear Post</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content class="ion-padding">

      <ion-list>
        <ion-item>
          <ion-select
            v-model="usuarioId"
            label="Usuario"
            label-placement="stacked"
            placeholder="Seleccione un usuario"
          >
            <ion-select-option
              v-for="usuario in usuarios"
              :key="usuario.id"
              :value="usuario.id"
            >
              {{ usuario.name }}
            </ion-select-option>
          </ion-select>
        </ion-item>

        <ion-item>
          <ion-input
            v-model="titulo"
            label="Título"
            label-placement="stacked"
            placeholder="Ingrese el título"
          />
        </ion-item>

        <ion-item>
          <ion-textarea
            v-model="contenidoQR"
            label="Contenido"
            label-placement="stacked"
            :auto-grow="true"
            placeholder="Escanee el código QR"
          />
        </ion-item>
      </ion-list>

      <ion-button expand="block" @click="escanearQR">
        Escanear QR
      </ion-button>

      <ion-button
        expand="block"
        color="success"
        :disabled="enviando"
        @click="crearPost"
      >
        {{ enviando ? 'Publicando...' : 'Crear' }}
      </ion-button>

      <ion-card v-if="respuesta">
        <ion-card-header>
          <ion-card-title>Respuesta del POST</ion-card-title>
        </ion-card-header>

        <ion-card-content>
          <pre>{{ JSON.stringify(respuesta, null, 2) }}</pre>
        </ion-card-content>
      </ion-card>

    </ion-content>
  </ion-page>
</template>

<script setup lang="ts">
import { ref } from 'vue';

import {
  IonPage,
  IonHeader,
  IonToolbar,
  IonTitle,
  IonContent,
  IonList,
  IonItem,
  IonSelect,
  IonSelectOption,
  IonInput,
  IonTextarea,
  IonButton,
  IonCard,
  IonCardHeader,
  IonCardTitle,
  IonCardContent,
  alertController
} from '@ionic/vue';

import {
  CapacitorBarcodeScanner,
  CapacitorBarcodeScannerTypeHint
} from '@capacitor/barcode-scanner';

import type { Usuarios } from '@/interfaces/usuarios';
import type { Posts } from '@/interfaces/posts';

const usuarios = ref<Usuarios[]>([]);
const usuarioId = ref<number | undefined>(undefined);
const titulo = ref('');
const contenidoQR = ref('');
const respuesta = ref<Posts | null>(null);
const enviando = ref(false);


async function cargarUsuarios() {
  try {
    const response = await fetch(
      'https://jsonplaceholder.typicode.com/users'
    );

    if (!response.ok) {
      throw new Error('No se pudieron cargar los usuarios');
    }

    usuarios.value = await response.json();

  } catch (error) {
    console.error(error);
  }
}


async function escanearQR() {
  try {
    const resultado =
      await CapacitorBarcodeScanner.scanBarcode({
        hint: CapacitorBarcodeScannerTypeHint.QR_CODE
      });

    if (resultado.ScanResult) {
      procesarQR(resultado.ScanResult);
    }

  } catch (error) {
    console.error('Error al escanear QR:', error);
  }
}


async function procesarQR(textoQR: string) {
  try {
    let tituloLeido = '';
    let contenidoLeido = '';
    let usuarioLeido = 0;


    if (textoQR.trim().startsWith('{')) {
      const datos = JSON.parse(textoQR);

      tituloLeido = String(datos.title ?? '');
      contenidoLeido = String(datos.body ?? '');
      usuarioLeido = Number(datos.userId);

    } else {
  

      const tituloEncontrado = textoQR.match(
        /^Title:\s*(.+)$/im
      );

      const contenidoEncontrado = textoQR.match(
        /^Body:\s*(.+)$/im
      );

      const usuarioEncontrado = textoQR.match(
        /^User:\s*(\d+)$/im
      );

      if (
        !tituloEncontrado ||
        !contenidoEncontrado ||
        !usuarioEncontrado
      ) {
        throw new Error('Formato de QR no válido');
      }

      tituloLeido = tituloEncontrado[1].trim();
      contenidoLeido = contenidoEncontrado[1].trim();
      usuarioLeido = Number(usuarioEncontrado[1]);
    }

    if (
      !tituloLeido.trim() ||
      !contenidoLeido.trim() ||
      !Number.isInteger(usuarioLeido) ||
      usuarioLeido <= 0
    ) {
      throw new Error('Datos incompletos');
    }


    titulo.value = tituloLeido;
    contenidoQR.value = contenidoLeido;
    usuarioId.value = usuarioLeido;

    respuesta.value = null;

    const alerta = await alertController.create({
      header: 'QR escaneado',
      message: 'Los campos se completaron automáticamente.',
      buttons: ['Aceptar']
    });

    await alerta.present();

  } catch (error) {
    console.error('Error al procesar QR:', error);

    const alerta = await alertController.create({
      header: 'Código QR no válido',
      message: 'El QR no contiene los datos esperados.',
      buttons: ['Aceptar']
    });

    await alerta.present();
  }
}

async function crearPost() {
  if (
    usuarioId.value === undefined ||
    !titulo.value.trim() ||
    !contenidoQR.value.trim()
  ) {
    const alerta = await alertController.create({
      header: 'Campos incompletos',
      message: 'Complete todos los campos.',
      buttons: ['Aceptar']
    });

    await alerta.present();
    return;
  }

  enviando.value = true;
  respuesta.value = null;

  try {
    const response = await fetch(
      'https://jsonplaceholder.typicode.com/posts',
      {
        method: 'POST',
        body: JSON.stringify({
          title: titulo.value,
          body: contenidoQR.value,
          userId: usuarioId.value
        }),
        headers: {
          'Content-type': 'application/json; charset=UTF-8'
        }
      }
    );

    if (!response.ok) {
      throw new Error('Error al crear la publicación');
    }

    const json: Posts = await response.json();

    console.log(json);
    respuesta.value = json;

  } catch (error) {
    console.error('Error en POST:', error);

    const alerta = await alertController.create({
      header: 'Error',
      message: 'No se pudo enviar la publicación.',
      buttons: ['Aceptar']
    });

    await alerta.present();

  } finally {
    enviando.value = false;
  }
}

cargarUsuarios();
</script>
