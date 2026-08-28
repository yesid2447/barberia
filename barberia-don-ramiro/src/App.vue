<script setup>
import { ref } from 'vue'
import { useLocalStorage } from '@vueuse/core'

const barberos = ['Don Ramiro', 'Kevin', 'Andrés']
const tiposServicio = ['Corte clásico', 'Corte moderno', 'Barba', 'Corte + Barba', 'Cejas', 'Tinte']
const metodosPago = ['Efectivo', 'Transferencia', 'Tarjeta']
const estadosPago = ['Pagado', 'Pendiente', 'Fiado']

const servicios = useLocalStorage('servicios-barberia-don-ramiro', [])

const modalAbierto = ref(false)
const modoEdicion = ref(false)
const idEdicion = ref(null)
const errorFormulario = ref('')

const fCliente = ref('')
const fTipo = ref('')
const fBarbero = ref('')
const fFecha = ref('')
const fHora = ref('')
const fPrecio = ref('')
const fMetodoPago = ref('')
const fEstadoPago = ref('Pagado')
const fCalificacion = ref(5)
const fObservaciones = ref('')

const idAEliminar = ref(null)

function limpiarFormulario() {
  fCliente.value = ''
  fTipo.value = ''
  fBarbero.value = ''
  fFecha.value = ''
  fHora.value = ''
  fPrecio.value = ''
  fMetodoPago.value = ''
  fEstadoPago.value = 'Pagado'
  fCalificacion.value = 5
  fObservaciones.value = ''
  errorFormulario.value = ''
}

function abrirModalNuevo() {
  modoEdicion.value = false
  idEdicion.value = null
  limpiarFormulario()
  modalAbierto.value = true
}

function abrirModalEditar(servicio) {
  modoEdicion.value = true
  idEdicion.value = servicio.id
  fCliente.value = servicio.cliente
  fTipo.value = servicio.tipo
  fBarbero.value = servicio.barbero
  fFecha.value = servicio.fecha
  fHora.value = servicio.hora
  fPrecio.value = servicio.precio
  fMetodoPago.value = servicio.metodoPago
  fEstadoPago.value = servicio.estadoPago
  fCalificacion.value = servicio.calificacion
  fObservaciones.value = servicio.observaciones
  errorFormulario.value = ''
  modalAbierto.value = true
}

function cerrarModal() {
  modalAbierto.value = false
  errorFormulario.value = ''
}

function elegirEstrella(numero) {
  fCalificacion.value = numero
}

function guardarServicio() {
  if (fCliente.value.trim() === '') {
    errorFormulario.value = 'Escribe el nombre del cliente'
    return
  }
  if (fTipo.value === '') {
    errorFormulario.value = 'Selecciona el tipo de servicio'
    return
  }
  if (fBarbero.value === '') {
    errorFormulario.value = 'Selecciona quién atendió'
    return
  }
  if (fFecha.value === '') {
    errorFormulario.value = 'Selecciona la fecha'
    return
  }
  if (fHora.value === '') {
    errorFormulario.value = 'Selecciona la hora'
    return
  }
  if (fPrecio.value === '' || Number(fPrecio.value) <= 0) {
    errorFormulario.value = 'El precio debe ser mayor a 0'
    return
  }
  if (fMetodoPago.value === '') {
    errorFormulario.value = 'Selecciona el método de pago'
    return
  }

  if (modoEdicion.value) {
    for (let i = 0; i < servicios.value.length; i++) {
      if (servicios.value[i].id === idEdicion.value) {
        servicios.value[i].cliente = fCliente.value.trim()
        servicios.value[i].tipo = fTipo.value
        servicios.value[i].barbero = fBarbero.value
        servicios.value[i].fecha = fFecha.value
        servicios.value[i].hora = fHora.value
        servicios.value[i].precio = Number(fPrecio.value)
        servicios.value[i].metodoPago = fMetodoPago.value
        servicios.value[i].estadoPago = fEstadoPago.value
        servicios.value[i].calificacion = Number(fCalificacion.value)
        servicios.value[i].observaciones = fObservaciones.value.trim()
      }
    }
  } else {
    servicios.value.push({
      id: Date.now(),
      cliente: fCliente.value.trim(),
      tipo: fTipo.value,
      barbero: fBarbero.value,
      fecha: fFecha.value,
      hora: fHora.value,
      precio: Number(fPrecio.value),
      metodoPago: fMetodoPago.value,
      estadoPago: fEstadoPago.value,
      calificacion: Number(fCalificacion.value),
      observaciones: fObservaciones.value.trim()
    })
  }

  cerrarModal()
}

function pedirConfirmacionEliminar(id) {
  idAEliminar.value = id
}

function cancelarEliminar() {
  idAEliminar.value = null
}

function eliminarServicio(id) {
  const nuevaLista = []
  for (let i = 0; i < servicios.value.length; i++) {
    if (servicios.value[i].id !== id) {
      nuevaLista.push(servicios.value[i])
    }
  }
  servicios.value = nuevaLista
  idAEliminar.value = null
}
</script>

<template>
  <div>
    <div class="barra-superior">
      <div class="encabezado">
        <h1>Barbería Don Ramiro</h1>
        <p>Registro de servicios</p>
      </div>
      <button class="boton-nuevo" @click="abrirModalNuevo">+ Nuevo servicio</button>
    </div>

    <p v-if="servicios.length === 0" class="sin-resultados">
      Todavía no hay servicios registrados.
    </p>

    <div v-for="servicio in servicios" :key="servicio.id" class="tarjeta-servicio" :class="{ 'sin-pagar': servicio.estadoPago !== 'Pagado' }">
      <div class="tarjeta-encabezado">
        <h3>{{ servicio.cliente }}</h3>
        <span
          class="estado"
          :class="{ pagado: servicio.estadoPago === 'Pagado', pendiente: servicio.estadoPago === 'Pendiente', fiado: servicio.estadoPago === 'Fiado' }"
        >{{ servicio.estadoPago }}</span>
      </div>

      <p class="tipo-servicio">{{ servicio.tipo }}</p>

      <p class="fila-datos"><b>Barbero:</b> {{ servicio.barbero }}</p>
      <p class="fila-datos"><b>Fecha:</b> {{ servicio.fecha }} — {{ servicio.hora }}</p>
      <p class="fila-datos"><b>Precio:</b> ${{ servicio.precio }}</p>

      <p class="fila-datos">
        <span v-if="servicio.metodoPago === 'Efectivo'" class="icono-pago">💵</span>
        <span v-else-if="servicio.metodoPago === 'Transferencia'" class="icono-pago">🏦</span>
        <span v-else-if="servicio.metodoPago === 'Tarjeta'" class="icono-pago">💳</span>
        {{ servicio.metodoPago }}
      </p>

      <div class="estrellas" :class="{ baja: servicio.calificacion <= 2 }">
        <span v-for="n in 5" :key="n">
          <span v-if="n <= servicio.calificacion" class="llena">★</span>
          <span v-else>★</span>
        </span>
      </div>
      <p v-if="servicio.calificacion <= 2" class="texto-calificacion-baja">Calificación baja</p>

      <p v-if="servicio.observaciones" class="observaciones">{{ servicio.observaciones }}</p>

      <div v-if="idAEliminar !== servicio.id" class="tarjeta-botones">
        <button @click="abrirModalEditar(servicio)">Editar</button>
        <button class="boton-eliminar" @click="pedirConfirmacionEliminar(servicio.id)">Eliminar</button>
      </div>

      <div v-else class="confirmar-eliminar">
        <p>¿Seguro que quieres eliminar este servicio?</p>
        <button @click="eliminarServicio(servicio.id)">Sí, eliminar</button>
        <button @click="cancelarEliminar">Cancelar</button>
      </div>
    </div>

    <div v-if="modalAbierto" class="fondo-modal">
      <div class="caja-modal">
        <h2 v-if="modoEdicion">Editar servicio</h2>
        <h2 v-else>Nuevo servicio</h2>

        <p v-if="errorFormulario" class="mensaje-error">{{ errorFormulario }}</p>

        <form @submit.prevent="guardarServicio">
          <div class="campo-formulario">
            <label>Nombre del cliente</label>
            <input type="text" v-model="fCliente" placeholder="Ej: Carlos Pérez">
          </div>

          <div class="campo-formulario">
            <label>Tipo de servicio</label>
            <select v-model="fTipo">
              <option value="">Selecciona...</option>
              <option v-for="tipo in tiposServicio" :key="tipo" :value="tipo">{{ tipo }}</option>
            </select>
          </div>

          <div class="campo-formulario">
            <label>Barbero</label>
            <select v-model="fBarbero">
              <option value="">Selecciona...</option>
              <option v-for="barbero in barberos" :key="barbero" :value="barbero">{{ barbero }}</option>
            </select>
          </div>

          <div class="campo-formulario">
            <label>Fecha</label>
            <input type="date" v-model="fFecha">
          </div>

          <div class="campo-formulario">
            <label>Hora</label>
            <input type="time" v-model="fHora">
          </div>

          <div class="campo-formulario">
            <label>Precio cobrado</label>
            <input type="number" v-model="fPrecio" placeholder="Ej: 25000">
          </div>

          <div class="campo-formulario">
            <label>Método de pago</label>
            <select v-model="fMetodoPago">
              <option value="">Selecciona...</option>
              <option v-for="metodo in metodosPago" :key="metodo" :value="metodo">{{ metodo }}</option>
            </select>
          </div>

          <div class="campo-formulario">
            <label>Estado del pago</label>
            <select v-model="fEstadoPago">
              <option v-for="estado in estadosPago" :key="estado" :value="estado">{{ estado }}</option>
            </select>
          </div>

          <div class="campo-formulario">
            <label>Calificación del cliente</label>
            <div class="selector-estrellas">
              <button
                type="button"
                v-for="n in 5"
                :key="n"
                :class="{ activa: n <= fCalificacion }"
                @click="elegirEstrella(n)"
              >★</button>
            </div>
          </div>

          <div class="campo-formulario">
            <label>Observaciones (opcional)</label>
            <textarea v-model="fObservaciones" placeholder="Ej: pidió que no le rebajaran mucho a los lados"></textarea>
          </div>

          <div class="botones-modal">
            <button type="submit" class="boton-guardar">Guardar</button>
            <button type="button" class="boton-cancelar" @click="cerrarModal">Cancelar</button>
          </div>
        </form>
      </div>
    </div>
  </div>
</template>
