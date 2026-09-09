<template>
  <!-- ===================== HEADER ===================== -->
  <header class="header">
    <div class="header-left">
      <div>
        <div class="header-titulo">Barberia Don Ramiro</div>
        <div class="header-subtitulo">Registro de servicios diarios</div>
      </div>
    </div>
    <button class="btn-nuevo" @click="abrirModalNuevo">+ Nuevo servicio</button>
  </header>

  <!-- ===================== RESUMEN ===================== -->
  <div class="resumen">
    <div class="resumen-card">
      <div class="numero">{{ servicios.length }}</div>
      <div class="etiqueta">Servicios</div>
    </div>
    <div class="resumen-card">
      <div class="numero">$ {{ totalCobrado() }}</div>
      <div class="etiqueta">Total cobrado</div>
    </div>
    <div class="resumen-card rojo">
      <div class="numero">{{ contarEstado('pendiente') }}</div>
      <div class="etiqueta">Pendientes</div>
    </div>
    <div class="resumen-card azul">
      <div class="numero">{{ contarEstado('abonado') }}</div>
      <div class="etiqueta">Abonados</div>
    </div>
    <div class="resumen-card">
      <div class="numero small" :class="{ small: servicioMasPopular().length > 8 }">
        {{ servicioMasPopular() }}
      </div>
      <div class="etiqueta">Mas solicitado</div>
    </div>
  </div>

  <!-- ===================== LISTA ===================== -->
  <div class="lista">
    <div v-if="serviciosFiltrados().length === 0" class="lista-vacia">
      <p>No hay servicios registrados</p>
    </div>

    <div
      v-for="s in serviciosFiltrados()"
      :key="s.id"
      class="tarjeta"
      :class="s.estadoPago"
    >
      <!-- nombre y fecha -->
      <div class="t-top">
        <div class="t-nombre">{{ s.nombre }}</div>
        <div class="t-fecha">{{ formatearFecha(s.fechaHora) }}</div>
      </div>

      <!-- servicio -->
      <div class="t-servicio">{{ s.tipoServicio }}</div>
      <div class="t-barbero">Atendido por: {{ s.barbero }}</div>

      <!-- precio y estado -->
      <div class="t-fila">
        <div>
          <div class="t-precio">$ {{ formatPrecio(s.precio) }}</div>
          <div v-if="s.estadoPago === 'abonado'" class="t-saldo">
            Saldo: $ {{ formatPrecio(calcularSaldo(s)) }}
          </div>
        </div>
        <span class="badge" :class="s.estadoPago">
          <span v-if="s.estadoPago === 'pagado'">Pagado</span>
          <span v-else-if="s.estadoPago === 'abonado'">Abonado</span>
          <span v-else-if="s.estadoPago === 'pendiente'">Pendiente</span>
        </span>
      </div>

      <!-- metodo de pago -->
      <div class="t-metodo">
        Pago:
        <span class="metodo-tag" :class="s.metodoPago">{{ s.metodoPago }}</span>
      </div>

      <!-- calificacion -->
      <div class="t-fila">
        <div class="t-estrellas">
          <span v-for="i in 5" :key="i" :class="i <= s.calificacion ? 'estrella-on' : 'estrella-off'">
            &#9733;
          </span>
        </div>
        <span v-if="s.calificacion === 0" class="t-sin-calificar">Sin calificar</span>
        <span v-else-if="s.calificacion <= 2" class="t-alerta">Servicio a revisar</span>
        <span v-else class="t-calificacion-ok">Buena atencion</span>
      </div>

      <!-- aviso destacado si el pago esta pendiente -->
      <div v-show="s.estadoPago === 'pendiente'" class="t-aviso-pendiente">
        Pago pendiente — contactar al cliente
      </div>

      <!-- panel de abonos (solo si tiene) -->
      <div v-if="s.estadoPago === 'abonado' && s.abonos && s.abonos.length > 0" class="t-abonos">
        <div class="t-abonos-titulo">Historial de abonos</div>
        <div class="t-abonos-lista">
          <div v-for="(ab, idx) in s.abonos" :key="idx" class="t-abono-item">
            <span>{{ formatearFechaCorta(ab.fecha) }}</span>
            <span class="t-abono-valor">$ {{ formatPrecio(ab.monto) }}</span>
          </div>
        </div>
        <div class="t-abonos-total">
          <span>Total abonado</span>
          <span>$ {{ formatPrecio(totalAbonado(s)) }}</span>
        </div>
      </div>

      <!-- boton abonar -->
      <button
        v-if="s.estadoPago === 'abonado'"
        class="btn-abonar"
        @click="abrirModalAbono(s)"
      >
        Registrar abono
      </button>

      <!-- observaciones -->
      <div v-if="s.observaciones" class="t-obs">Nota: {{ s.observaciones }}</div>

      <!-- acciones -->
      <div class="t-acciones">
        <button class="btn-calificar" @click="abrirModalCalificacion(s)">
          <span v-if="s.calificacion > 0">Recalificar</span>
          <span v-else>Calificar</span>
        </button>
        <button class="btn-editar" @click="abrirModalEditar(s)">Editar</button>
        <button class="btn-eliminar" @click="pedirConfirmacion(s)">Eliminar</button>
      </div>
    </div>
  </div>

  <!-- ===================== MODAL FORMULARIO ===================== -->
  <div v-if="mostrarModal" class="modal-overlay" @click.self="cerrarModal">
    <div class="modal">
      <div class="modal-cabecera">
        <div class="modal-titulo">
          <span v-if="modoEditar">Editar servicio</span>
          <span v-else>Registrar nuevo servicio</span>
        </div>
        <button class="modal-cerrar" @click="cerrarModal">&#10005;</button>
      </div>

      <!-- SECCION: datos del cliente -->
      <div class="form-seccion">Datos del cliente</div>

      <div class="form-grupo">
        <label>Nombre del cliente *</label>
        <input
          v-model="form.nombre"
          type="text"
          placeholder="Ej: Carlos Perez"
          :class="{ 'campo-error': errores.nombre }"
        />
        <span v-if="errores.nombre" class="form-error-msg">{{ errores.nombre }}</span>
      </div>

      <!-- SECCION: servicio -->
      <div class="form-seccion">Detalle del servicio</div>

      <div class="form-fila">
        <div class="form-grupo">
          <label>Tipo de servicio *</label>
          <select v-model="form.tipoServicio" :class="{ 'campo-error': errores.tipoServicio }">
            <option value="">-- Seleccionar --</option>
            <option value="Barba">Barba</option>
            <option value="Corte + Barba">Corte + Barba</option>
            <option value="Cejas">Cejas</option>
            <option value="Tinte">Tinte</option>
            <option value="Degradado">Degradado</option>
            <option value="Corte nino">Corte nino</option>
          </select>
          <span v-if="errores.tipoServicio" class="form-error-msg">{{ errores.tipoServicio }}</span>
        </div>
        <div class="form-grupo">
          <label>Barbero *</label>
          <select v-model="form.barbero" :class="{ 'campo-error': errores.barbero }">
            <option value="">-- Seleccionar --</option>
            <option v-for="b in barberos" :key="b" :value="b">{{ b }}</option>
          </select>
          <span v-if="errores.barbero" class="form-error-msg">{{ errores.barbero }}</span>
        </div>
      </div>

      <div class="form-fila">
        <div class="form-grupo">
          <label>Fecha y hora *</label>
          <input
            v-model="form.fechaHora"
            type="datetime-local"
            :min="fechaMinima()"
            :class="{ 'campo-error': errores.fechaHora }"
            @change="validarHorario"
          />
          <span v-if="errores.fechaHora" class="form-error-msg">{{ errores.fechaHora }}</span>
        </div>
        <div class="form-grupo">
          <label>Precio *</label>
          <div class="precio-wrapper">
            <span class="precio-prefijo">$</span>
            <input
              v-model="form.precio"
              type="number"
              min="0"
              step="100"
              placeholder="0"
              :class="{ 'campo-error': errores.precio }"
            />
          </div>
          <span v-if="errores.precio" class="form-error-msg">{{ errores.precio }}</span>
        </div>
      </div>

      <!-- SECCION: pago -->
      <div class="form-seccion">Informacion de pago</div>

      <div class="form-fila">
        <div class="form-grupo">
          <label>Metodo de pago *</label>
          <select v-model="form.metodoPago" :class="{ 'campo-error': errores.metodoPago }">
            <option value="">-- Seleccionar --</option>
            <option value="efectivo">Efectivo</option>
            <option value="transferencia">Transferencia</option>
            <option value="tarjeta">Tarjeta</option>
          </select>
          <span v-if="errores.metodoPago" class="form-error-msg">{{ errores.metodoPago }}</span>
        </div>
        <div class="form-grupo">
          <label>Estado del pago *</label>
          <select v-model="form.estadoPago" :class="{ 'campo-error': errores.estadoPago }">
            <option value="">-- Seleccionar --</option>
            <option value="pagado">Pagado</option>
            <option value="abonado">Abonado</option>
            <option value="pendiente">Pendiente</option>
          </select>
          <span v-if="errores.estadoPago" class="form-error-msg">{{ errores.estadoPago }}</span>
        </div>
      </div>

      <!-- SECCION: observaciones -->
      <div class="form-seccion">Notas adicionales</div>

      <div class="form-grupo">
        <label>Observaciones (opcional)</label>
        <textarea v-model="form.observaciones" placeholder="Alguna nota sobre el servicio..."></textarea>
      </div>

      <div class="modal-botones">
        <button class="btn-cancelar" @click="cerrarModal">Cancelar</button>
        <button class="btn-guardar" @click="guardarServicio">
          <span v-if="modoEditar">Guardar cambios</span>
          <span v-else>Registrar servicio</span>
        </button>
      </div>
    </div>
  </div>

  <!-- ===================== MODAL ABONO ===================== -->
  <div v-if="mostrarModalAbono" class="modal-overlay" @click.self="cerrarModalAbono">
    <div class="modal modal-abono">
      <div class="modal-cabecera">
        <div class="modal-titulo">Registrar abono</div>
        <button class="modal-cerrar" @click="cerrarModalAbono">&#10005;</button>
      </div>

      <!-- info del servicio -->
      <div v-if="servicioAbonando" class="abono-info">
        <div class="abono-info-fila">
          <span>Cliente</span>
          <span>{{ servicioAbonando.nombre }}</span>
        </div>
        <div class="abono-info-fila">
          <span>Servicio</span>
          <span>{{ servicioAbonando.tipoServicio }}</span>
        </div>
        <div class="abono-info-fila">
          <span>Valor total</span>
          <span>$ {{ formatPrecio(servicioAbonando.precio) }}</span>
        </div>
        <div class="abono-info-fila">
          <span>Total abonado</span>
          <span>$ {{ formatPrecio(totalAbonado(servicioAbonando)) }}</span>
        </div>
        <div class="abono-info-fila destacado">
          <span>Saldo pendiente</span>
          <span>$ {{ formatPrecio(calcularSaldo(servicioAbonando)) }}</span>
        </div>
      </div>

      <!-- historial de abonos -->
      <div v-if="servicioAbonando && servicioAbonando.abonos && servicioAbonando.abonos.length > 0">
        <div class="form-seccion" style="margin-bottom: 8px;">Historial</div>
        <div class="abono-lista-scroll">
          <div
            v-for="(ab, idx) in servicioAbonando.abonos"
            :key="idx"
            class="abono-registro"
          >
            <span>{{ formatearFechaCorta(ab.fecha) }}</span>
            <span>$ {{ formatPrecio(ab.monto) }}</span>
          </div>
        </div>
      </div>
      <div v-else-if="servicioAbonando" class="abono-sin-registros">
        Sin abonos registrados aun
      </div>

      <!-- nuevo abono -->
      <div class="form-seccion" style="margin-top: 4px;">Nuevo abono</div>

      <div class="form-grupo">
        <label>Monto del abono *</label>
        <div class="precio-wrapper">
          <span class="precio-prefijo">$</span>
          <input
            v-model="montoAbono"
            type="number"
            min="1"
            step="100"
            placeholder="0"
            :class="{ 'campo-error': errorAbono }"
          />
        </div>
        <span v-if="errorAbono" class="form-error-msg">{{ errorAbono }}</span>
      </div>

      <div class="modal-botones">
        <button class="btn-cancelar" @click="cerrarModalAbono">Cancelar</button>
        <button class="btn-registrar-abono" @click="registrarAbono">Registrar abono</button>
      </div>
    </div>
  </div>
  <!-- ===================== MODAL CALIFICACION ===================== -->
  <div v-if="mostrarModalCalificacion" class="modal-overlay" @click.self="cerrarModalCalificacion">
    <div class="modal modal-calif">
      <div class="modal-cabecera">
        <div class="modal-titulo">Calificacion del cliente</div>
        <button class="modal-cerrar" @click="cerrarModalCalificacion">&#10005;</button>
      </div>

      <div v-if="servicioCalificando" class="calif-cliente">
        {{ servicioCalificando.nombre }} — {{ servicioCalificando.tipoServicio }}
      </div>

      <div class="form-grupo">
        <label>Como calificas el servicio prestado?</label>
        <div class="estrellas-form estrellas-grandes">
          <span
            v-for="i in 5"
            :key="i"
            :class="i <= calificacionTemp ? 'on' : 'off'"
            @click="calificacionTemp = i"
          >&#9733;</span>
        </div>
        <div class="calif-texto" v-if="calificacionTemp === 1">Muy malo</div>
        <div class="calif-texto" v-else-if="calificacionTemp === 2">Malo</div>
        <div class="calif-texto" v-else-if="calificacionTemp === 3">Regular</div>
        <div class="calif-texto" v-else-if="calificacionTemp === 4">Bueno</div>
        <div class="calif-texto calif-texto--bueno" v-else-if="calificacionTemp === 5">Excelente</div>
        <span v-if="errorCalificacion" class="form-error-msg">{{ errorCalificacion }}</span>
      </div>

      <div class="modal-botones">
        <button class="btn-cancelar" @click="cerrarModalCalificacion">Omitir</button>
        <button class="btn-guardar" @click="guardarCalificacion">Guardar calificacion</button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref } from 'vue'
import { useLocalStorage } from '@vueuse/core'
import Swal from 'sweetalert2'

export default {
  setup() {
    // Barberos del negocio
    const barberos = ['Don Ramiro', 'Luis', 'Andres']

    // Datos persistidos en localStorage
    const servicios = useLocalStorage('barberia-servicios-v2', [])

    // Control modal formulario
    const mostrarModal = ref(false)
    const modoEditar   = ref(false)
    const idEditando   = ref(null)

    // Control modal abono
    const mostrarModalAbono = ref(false)
    const servicioAbonando  = ref(null)
    const montoAbono        = ref('')
    const errorAbono        = ref('')

    // Control modal calificacion
    const mostrarModalCalificacion = ref(false)
    const servicioCalificando      = ref(null)
    const calificacionTemp         = ref(0)
    const errorCalificacion        = ref('')

    // Formulario en blanco
    const formVacio = {
      nombre:       '',
      tipoServicio: '',
      barbero:      '',
      fechaHora:    '',
      precio:       '',
      metodoPago:   '',
      estadoPago:   '',
      calificacion: 0,
      observaciones: '',
      abonos:       []
    }

    const form    = ref({ ...formVacio })
    const errores = ref({})

    // ── Abrir / cerrar modal formulario ──────────────────
    function abrirModalNuevo() {
      form.value    = { ...formVacio, abonos: [] }
      errores.value = {}
      modoEditar.value  = false
      idEditando.value  = null
      mostrarModal.value = true
    }

    function abrirModalEditar(servicio) {
      form.value    = { ...servicio, abonos: servicio.abonos ? [...servicio.abonos] : [] }
      errores.value = {}
      modoEditar.value  = true
      idEditando.value  = servicio.id
      mostrarModal.value = true
    }

    function cerrarModal() {
      mostrarModal.value = false
      errores.value = {}
    }

    // ── Fecha minima: solo hoy en adelante ───────────────
    function fechaMinima() {
      const hoy = new Date()
      const pad = n => String(n).padStart(2, '0')
      return (
        hoy.getFullYear() + '-' +
        pad(hoy.getMonth() + 1) + '-' +
        pad(hoy.getDate()) + 'T00:00'
      )
    }

    // ── Validar horario 8am - 6pm al cambiar el input ────
    function validarHorario() {
      if (!form.value.fechaHora) return
      const hora = new Date(form.value.fechaHora).getHours()
      if (hora < 8 || hora >= 18) {
        Swal.fire({
          icon: 'warning',
          title: 'Fuera del horario de atencion',
          text: 'La barberia atiende de 8:00 AM a 6:00 PM. Por favor selecciona una hora dentro de ese rango.',
          background: '#ffffff',
          color: '#1e1740',
          confirmButtonColor: '#7c3aed',
          confirmButtonText: 'Entendido'
        })
        form.value.fechaHora = ''
        errores.value.fechaHora = 'Selecciona una hora entre 8:00 AM y 6:00 PM'
      } else {
        errores.value.fechaHora = ''
      }
    }

    // ── Validaciones ─────────────────────────────────────
    function validarForm() {
      const e = {}

      if (!form.value.nombre.trim())
        e.nombre = 'El nombre del cliente es obligatorio'

      if (!form.value.tipoServicio)
        e.tipoServicio = 'Selecciona un tipo de servicio'

      if (!form.value.barbero)
        e.barbero = 'Selecciona el barbero'

      if (!form.value.fechaHora) {
        e.fechaHora = 'La fecha y hora son obligatorias'
      } else {
        const hora = new Date(form.value.fechaHora).getHours()
        if (hora < 8 || hora >= 18) {
          e.fechaHora = 'Selecciona una hora entre 8:00 AM y 6:00 PM'
        }
      }

      if (!form.value.precio || Number(form.value.precio) <= 0)
        e.precio = 'Ingresa un precio valido mayor a 0'

      if (!form.value.metodoPago)
        e.metodoPago = 'Selecciona el metodo de pago'

      if (!form.value.estadoPago)
        e.estadoPago = 'Selecciona el estado del pago'

      errores.value = e
      return Object.keys(e).length === 0
    }

    // ── Guardar (nuevo o edicion) ────────────────────────
    function guardarServicio() {
      if (!validarForm()) {
        Swal.fire({
          icon: 'warning',
          title: 'Campos incompletos',
          text: 'Revisa los campos marcados en rojo antes de continuar.',
          background: '#1c1c1c',
          color: '#f0f0f0',
          confirmButtonColor: '#c9a84c',
          confirmButtonText: 'Entendido'
        })
        return
      }

      const datos = {
        ...form.value,
        precio: Number(form.value.precio),
        abonos: form.value.abonos || []
      }

      if (modoEditar.value) {
        const idx = servicios.value.findIndex(s => s.id === idEditando.value)
        if (idx !== -1) {
          servicios.value[idx] = { ...datos, id: idEditando.value }
        }
        Swal.fire({
          icon: 'success',
          title: 'Servicio actualizado',
          toast: true,
          position: 'top-end',
          showConfirmButton: false,
          timer: 2200,
          background: '#1c1c1c',
          color: '#f0f0f0'
        })
        cerrarModal()
      } else {
        const nuevoId = Date.now()
        servicios.value.push({ ...datos, id: nuevoId })
        cerrarModal()
        // Abrir modal de calificacion despues de guardar
        const recienCreado = servicios.value.find(s => s.id === nuevoId)
        abrirModalCalificacion(recienCreado)
      }
    }

    // ── Eliminar ─────────────────────────────────────────
    function pedirConfirmacion(servicio) {
      Swal.fire({
        title: 'Eliminar servicio',
        html: 'Vas a eliminar el registro de <strong>' + servicio.nombre + '</strong>.<br>Esta accion no se puede deshacer.',
        icon: 'warning',
        showCancelButton: true,
        confirmButtonText: 'Si, eliminar',
        cancelButtonText: 'Cancelar',
        confirmButtonColor: '#c0392b',
        cancelButtonColor: '#2a2a2a',
        background: '#1c1c1c',
        color: '#f0f0f0'
      }).then(function(resultado) {
        if (resultado.isConfirmed) {
          servicios.value = servicios.value.filter(s => s.id !== servicio.id)
          Swal.fire({
            icon: 'success',
            title: 'Servicio eliminado',
            toast: true,
            position: 'top-end',
            showConfirmButton: false,
            timer: 2000,
            background: '#1c1c1c',
            color: '#f0f0f0'
          })
        }
      })
    }

    // ── Modal abono ───────────────────────────────────────
    function abrirModalAbono(servicio) {
      servicioAbonando.value = servicio
      montoAbono.value = ''
      errorAbono.value = ''
      mostrarModalAbono.value = true
    }

    function cerrarModalAbono() {
      mostrarModalAbono.value = false
      servicioAbonando.value = null
      montoAbono.value = ''
      errorAbono.value = ''
    }

    function registrarAbono() {
      const monto = Number(montoAbono.value)
      const saldo = calcularSaldo(servicioAbonando.value)

      if (!monto || monto <= 0) {
        errorAbono.value = 'Ingresa un monto valido mayor a 0'
        return
      }

      if (monto > saldo) {
        errorAbono.value = 'El abono no puede superar el saldo pendiente ($ ' + formatPrecio(saldo) + ')'
        return
      }

      errorAbono.value = ''

      const idx = servicios.value.findIndex(s => s.id === servicioAbonando.value.id)
      if (idx === -1) return

      // Agregar abono al historial
      const abonosActuales = servicios.value[idx].abonos || []
      const nuevoAbono = { monto: monto, fecha: new Date().toISOString() }
      const abonosActualizados = [...abonosActuales, nuevoAbono]

      // Si con este abono queda saldo 0, marcar como pagado
      const nuevoTotalAbonado = abonosActualizados.reduce((acc, a) => acc + a.monto, 0)
      const nuevoEstado = nuevoTotalAbonado >= servicios.value[idx].precio ? 'pagado' : 'abonado'

      servicios.value[idx] = {
        ...servicios.value[idx],
        abonos: abonosActualizados,
        estadoPago: nuevoEstado
      }

      // Actualizar referencia del modal
      servicioAbonando.value = servicios.value[idx]
      montoAbono.value = ''

      if (nuevoEstado === 'pagado') {
        Swal.fire({
          icon: 'success',
          title: 'Pago completado',
          text: 'El servicio quedo marcado como pagado.',
          background: '#1c1c1c',
          color: '#f0f0f0',
          confirmButtonColor: '#4caf50',
          confirmButtonText: 'Aceptar'
        }).then(function() {
          cerrarModalAbono()
        })
      } else {
        Swal.fire({
          icon: 'success',
          title: 'Abono registrado',
          text: 'Saldo pendiente: $ ' + formatPrecio(calcularSaldo(servicios.value[idx])),
          toast: true,
          position: 'top-end',
          showConfirmButton: false,
          timer: 2500,
          background: '#1c1c1c',
          color: '#f0f0f0'
        })
      }
    }

    // ── Modal calificacion ────────────────────────────────
    function abrirModalCalificacion(servicio) {
      servicioCalificando.value    = servicio
      calificacionTemp.value       = servicio.calificacion || 0
      errorCalificacion.value      = ''
      mostrarModalCalificacion.value = true
    }

    function cerrarModalCalificacion() {
      mostrarModalCalificacion.value = false
      servicioCalificando.value    = null
      calificacionTemp.value       = 0
      errorCalificacion.value      = ''
      // Toast de servicio registrado al cerrar (solo si venimos de nuevo registro)
    }

    function guardarCalificacion() {
      if (!calificacionTemp.value || calificacionTemp.value < 1) {
        errorCalificacion.value = 'Selecciona entre 1 y 5 estrellas'
        return
      }
      const idx = servicios.value.findIndex(s => s.id === servicioCalificando.value.id)
      if (idx !== -1) {
        servicios.value[idx] = { ...servicios.value[idx], calificacion: calificacionTemp.value }
      }
      Swal.fire({
        icon: 'success',
        title: 'Servicio registrado',
        toast: true,
        position: 'top-end',
        showConfirmButton: false,
        timer: 2200,
        background: '#1c1c1c',
        color: '#f0f0f0'
      })
      cerrarModalCalificacion()
    }

    // ── Lista completa sin filtros ────────────────────────
    function serviciosFiltrados() {
      return servicios.value
    }

    // ── Calculos de resumen ───────────────────────────────
    function totalCobrado() {
      const total = servicios.value
        .filter(s => s.estadoPago === 'pagado')
        .reduce((acc, s) => acc + Number(s.precio), 0)
      return formatPrecio(total)
    }

    function contarEstado(estado) {
      return servicios.value.filter(s => s.estadoPago === estado).length
    }

    function servicioMasPopular() {
      if (servicios.value.length === 0) return '--'
      const conteo = {}
      servicios.value.forEach(s => {
        conteo[s.tipoServicio] = (conteo[s.tipoServicio] || 0) + 1
      })
      let max = 0
      let popular = '--'
      for (const tipo in conteo) {
        if (conteo[tipo] > max) {
          max = conteo[tipo]
          popular = tipo
        }
      }
      return popular
    }

    // ── Calculos de abonos ────────────────────────────────
    function totalAbonado(servicio) {
      if (!servicio.abonos || servicio.abonos.length === 0) return 0
      return servicio.abonos.reduce((acc, a) => acc + Number(a.monto), 0)
    }

    function calcularSaldo(servicio) {
      return Number(servicio.precio) - totalAbonado(servicio)
    }

    // ── Formato de precio en pesos colombianos ────────────
    function formatPrecio(valor) {
      return Number(valor).toLocaleString('es-CO')
    }

    // ── Formato de fechas ─────────────────────────────────
    function formatearFecha(fechaHora) {
      if (!fechaHora) return ''
      const d = new Date(fechaHora)
      return d.toLocaleString('es-CO', {
        day:    '2-digit',
        month:  'short',
        hour:   '2-digit',
        minute: '2-digit'
      })
    }

    function formatearFechaCorta(fechaIso) {
      if (!fechaIso) return ''
      const d = new Date(fechaIso)
      return d.toLocaleString('es-CO', {
        day:    '2-digit',
        month:  'short',
        hour:   '2-digit',
        minute: '2-digit'
      })
    }

    return {
      barberos,
      servicios,
      mostrarModal,
      modoEditar,
      mostrarModalAbono,
      servicioAbonando,
      montoAbono,
      errorAbono,
      form,
      errores,
      abrirModalNuevo,
      abrirModalEditar,
      cerrarModal,
      guardarServicio,
      pedirConfirmacion,
      abrirModalAbono,
      cerrarModalAbono,
      registrarAbono,
      mostrarModalCalificacion,
      servicioCalificando,
      calificacionTemp,
      errorCalificacion,
      abrirModalCalificacion,
      cerrarModalCalificacion,
      guardarCalificacion,
      serviciosFiltrados,
      totalCobrado,
      contarEstado,
      servicioMasPopular,
      totalAbonado,
      calcularSaldo,
      formatPrecio,
      formatearFecha,
      formatearFechaCorta,
      fechaMinima,
      validarHorario
    }
  }
}
</script>
