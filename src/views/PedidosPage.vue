
<template>
  <ion-page>
    <ion-header>
      <ion-toolbar>
        <ion-title>Pedidos</ion-title>
      </ion-toolbar>
    </ion-header>

    <ion-content :fullscreen="true" class="ion-padding">

      <div v-if="!autenticado">
        <h2>Iniciar sesión</h2>
        <p>Ingrese sus credenciales para consultar los pedidos.</p>

        <ion-list>
          <ion-item>
            <ion-input
              label="Correo electrónico"
              label-placement="stacked"
              type="email"
              placeholder="Ingrese su correo"
              v-model="correo"
            ></ion-input>
          </ion-item>

          <ion-item>
            <ion-input
              label="Contraseña"
              label-placement="stacked"
              type="password"
              placeholder="Ingrese su contraseña"
              v-model="contrasena"
            ></ion-input>
          </ion-item>
        </ion-list>

        <ion-button
          expand="block"
          class="ion-margin-top"
          :disabled="cargando"
          @click="iniciarSesion"
        >
          {{ cargando ? 'Ingresando...' : 'Iniciar sesión' }}
        </ion-button>
      </div>

      <div v-else>
        <h2>Listado de pedidos</h2>

        <p v-if="nombreUsuario">
          Sesión iniciada como: {{ nombreUsuario }}
        </p>

        <ion-button
          color="primary"
          :disabled="cargando"
          @click="cargarPedidos"
        >
          Actualizar
        </ion-button>

        <ion-button
          color="danger"
          @click="cerrarSesion"
        >
          Cerrar sesión
        </ion-button>

        <p v-if="cargando">Cargando pedidos...</p>

        <ion-list v-if="!cargando">
          <ion-item
            v-for="pedido in pedidos"
            :key="pedido.id"
          >
            <ion-label>
              <h2>Pedido #{{ pedido.numero_orden }}</h2>
              <p>Cliente: {{ pedido.customer?.name ?? 'Sin cliente' }}</p>
              <p>Total: {{ pedido.total }}</p>
              <p>IVA: {{ pedido.iva }}</p>
            </ion-label>
          </ion-item>
        </ion-list>

        <p v-if="!cargando && pedidos.length === 0 && !error">
          No hay pedidos para mostrar.
        </p>

        <ion-button
          v-if="siguientePagina && !cargando"
          expand="block"
          fill="outline"
          @click="cargarPedidos(siguientePagina)"
        >
          Cargar más pedidos
        </ion-button>
      </div>

      <ion-text color="danger" v-if="error">
        <p>{{ error }}</p>
      </ion-text>

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
  IonInput,
  IonButton,
  IonLabel,
  IonText
} from '@ionic/vue';

interface Pedido {
  id: number;
  numero_orden: string;
  total: number;
  iva: number;
  customer: {
    id: number;
    name: string;
  } | null;
}

interface RespuestaPedidos {
  data: Pedido[];
  links?: {
    next: string | null;
  };
}

const API = 'https://ds4-demo-5w8lmtds.on-forge.com/api';

const correo = ref('');
const contrasena = ref('');
const autenticado = ref(false);
const cargando = ref(false);
const error = ref('');
const nombreUsuario = ref('');

const pedidos = ref<Pedido[]>([]);
const siguientePagina = ref<string | null>(null);

const token = ref(sessionStorage.getItem('token_pedidos') || '');

async function iniciarSesion() {
  error.value = '';

  if (!correo.value.trim() || !contrasena.value) {
    error.value = 'Complete todos los campos.';
    return;
  }

  cargando.value = true;

  try {
    const formulario = new FormData();

    formulario.append('email', correo.value);
    formulario.append('password', contrasena.value);
    formulario.append('device_name', 'movil');

    const respuesta = await fetch(`${API}/sanctum/token`, {
      method: 'POST',
      headers: {
        'Accept': 'application/json'
      },
      body: formulario
    });

    const datos = await respuesta.json();

    if (!respuesta.ok) {
      throw new Error(datos.message || 'Credenciales incorrectas');
    }

    if (typeof datos.token !== 'string' || !datos.token) {
      throw new Error('El servidor no devolvió un token válido.');
    }

    token.value = datos.token;
    sessionStorage.setItem('token_pedidos', token.value);

    autenticado.value = true;
    contrasena.value = '';

    await cargarUsuario();
    await cargarPedidos();

  } catch (err) {
    error.value = err instanceof Error
      ? err.message
      : 'Error al iniciar sesión';

  } finally {
    cargando.value = false;
  }
}

async function cargarUsuario() {
  try {
    const respuesta = await fetch(`${API}/user`, {
      headers: {
        'Accept': 'application/json',
        'Authorization': `Bearer ${token.value}`
      }
    });

    if (!respuesta.ok) {
      throw new Error('No se pudo consultar el usuario');
    }

    const usuario = await respuesta.json();
    nombreUsuario.value = usuario.name || usuario.email || '';

  } catch (err) {
    console.error(err);
  }
}

async function cargarPedidos(url?: string) {
  if (!token.value) return;

  cargando.value = true;
  error.value = '';

  try {
    const destino = url || `${API}/orders`;

    const respuesta = await fetch(destino, {
      headers: {
        'Accept': 'application/json',
        'Authorization': `Bearer ${token.value}`
      }
    });

    if (respuesta.status === 401) {
      cerrarSesion();
      throw new Error('La sesión expiró. Inicie sesión nuevamente.');
    }

    if (!respuesta.ok) {
      throw new Error(`Error al cargar pedidos (${respuesta.status})`);
    }

    const datos: RespuestaPedidos = await respuesta.json();

    if (url) {
      pedidos.value = [...pedidos.value, ...datos.data];
    } else {
      pedidos.value = datos.data;
    }

    siguientePagina.value = datos.links?.next ?? null;

  } catch (err) {
    error.value = err instanceof Error
      ? err.message
      : 'No se pudieron cargar los pedidos';

  } finally {
    cargando.value = false;
  }
}

function cerrarSesion() {
  sessionStorage.removeItem('token_pedidos');

  token.value = '';
  autenticado.value = false;
  nombreUsuario.value = '';
  correo.value = '';
  contrasena.value = '';
  pedidos.value = [];
  siguientePagina.value = null;
}

if (token.value) {
  autenticado.value = true;

  cargarUsuario();
  cargarPedidos();
}
</script>
