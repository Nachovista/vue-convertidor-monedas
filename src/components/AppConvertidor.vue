<template>
  <section class="convertidor d-flex align-items-center py-5">
    <div class="container">
      <div class="row justify-content-center">
        <div class="col-12 col-md-10 col-lg-7">
          <div class="card shadow-sm rounded-4 border-0">
            <div class="card-body p-4 p-md-5">
              <h1 class="h4 mb-2 text-center">Convertidor de Moneda</h1>
              <p class="text-secondary text-center mb-4">
                Ingresá un monto y elegí las monedas.
              </p>

              <form @submit.prevent="convertir">
                <div class="row g-3">
                  <!-- Cantidad -->
                  <div class="col-12">
                    <div class="form-floating">
                      <input
                        v-model.number="cantidad"
                        type="number"
                        min="0"
                        step="0.01"
                        class="form-control"
                        id="cantidad"
                        placeholder="0,00"
                      />
                      <label for="cantidad">Cantidad</label>
                    </div>
                  </div>

                  <!-- Moneda origen -->
                  <div class="col-12 col-md-6">
                    <div class="form-floating">
                      <select v-model="tengo" class="form-select" id="tengo">
                        <option disabled value="">Seleccioná moneda</option>
                        <option v-for="m in monedas" :key="'t'+m.code" :value="m.code">
                          {{ m.label }}
                        </option>
                      </select>
                      <label for="tengo">Tengo</label>
                    </div>
                  </div>

                  <!-- Moneda destino -->
                  <div class="col-12 col-md-6">
                    <div class="form-floating">
                      <select v-model="quiero" class="form-select" id="quiero">
                        <option disabled value="">Seleccioná moneda</option>
                        <option v-for="m in monedas" :key="'q'+m.code" :value="m.code">
                          {{ m.label }}
                        </option>
                      </select>
                      <label for="quiero">Quiero</label>
                    </div>
                  </div>

                  <!-- Botones -->
                  <div class="col-12 d-flex gap-2">
                    <button
                      type="submit"
                      class="btn btn-primary flex-grow-1 d-flex align-items-center justify-content-center"
                      :disabled="!puedeConvertir || cargando"
                    >
                      <template v-if="cargando">
                        <span class="spinner-border spinner-border-sm me-2" role="status" aria-hidden="true"></span>
                        Convirtiendo...
                      </template>
                      <template v-else>Convertir</template>
                    </button>

                    <button type="button" class="btn btn-outline-secondary" @click="limpiar">
                      Limpiar
                    </button>
                  </div>
                </div>
              </form>

              <!-- Resultado -->
              <div v-if="resultado !== null" class="text-center mt-4">
                <h5>
                  <span class="badge bg-success me-2">{{ formato(cantidad) }} {{ tengo }}</span>
                  <span class="badge bg-dark me-2">SON</span>
                  <span class="badge bg-success">{{ formato(resultado) }} {{ quiero }}</span>
                </h5>
                <p class="text-muted small mt-2" v-if="tasaActual !== null">
                  Tasa actual: 1 {{ tengo }} = {{ formato(tasaActual) }} {{ quiero }}
                </p>
              </div>

              <!-- Error -->
              <div v-if="error" class="alert alert-danger text-center mt-4">
                {{ error }}
              </div>
            </div>
          </div>

          <p class="text-center text-muted mt-3 small">
            * Tasas desde economia.awesomeapi.com.br (pares X-BRL).
          </p>
        </div>
      </div>
    </div>
  </section>
</template>

<script>
import axios from "axios";

export default {
  name: "AppConvertidor",
  data() {
    return {
      cantidad: null,
      tengo: "",
      quiero: "",
      resultado: null,
      tasaActual: null,   // tengo -> quiero
      cargando: false,
      error: null,

      // Solo USD, EUR y BRL (Reales)
      monedas: [
        { code: "USD", label: "Dólares (USD)" },
        { code: "EUR", label: "Euros (EUR)" },
        { code: "BRL", label: "Reales (BRL)" }
      ],

      // mapa a BRL (cuántos BRL por 1 unidad de la moneda)
      toBRL: null,
      lastUpdated: null,

      // .env.local opcional: VUE_APP_AWESOME_BASE=https://economia.awesomeapi.com.br/last
      awesomeBase: process.env.VUE_APP_AWESOME_BASE || "https://economia.awesomeapi.com.br/last",
    };
  },
  computed: {
    puedeConvertir() {
      return this.cantidad > 0 && this.tengo && this.quiero;
    },
    necesitaDescarga() {
      if (!this.toBRL || !this.lastUpdated) return true;
      return Date.now() - this.lastUpdated > 30 * 60 * 1000; // 30 min
    },
  },
  methods: {
    async ensurePairs() {
      if (!this.necesitaDescarga) return;

      // Traemos USD-BRL y EUR-BRL en una sola llamada
      const url = `${this.awesomeBase}/USD-BRL,EUR-BRL`;
      this.cargando = true;
      this.error = null;
      try {
        const resp = await axios.get(url, { timeout: 10000 });
        // Respuesta típica: { USDBRL:{bid:'x'}, EURBRL:{bid:'y'} }
        const d = resp.data || {};
        const parseBid = (obj) => (obj && typeof obj.bid === "string" ? Number(obj.bid) : NaN);

        const usd = parseBid(d.USDBRL);
        const eur = parseBid(d.EURBRL);

        if (!Number.isFinite(usd) || !Number.isFinite(eur)) {
          console.log("Respuesta AwesomeAPI inesperada:", d);
          throw new Error("Pares no disponibles");
        }

        this.toBRL = {
          USD: usd,
          EUR: eur,
          BRL: 1
        };
        this.lastUpdated = Date.now();
      } catch (e) {
        console.error("Error al obtener pares BRL:", e);
        this.error = "No se pudo obtener la tasa. Intente nuevamente más tarde.";
        this.toBRL = null;
      } finally {
        this.cargando = false;
      }
    },

    // tasa cruzada: (quiero->BRL) / (tengo->BRL)
    calcular() {
      if (!this.toBRL) return;

      if (this.tengo === this.quiero) {
        this.tasaActual = 1;
        this.resultado = this.cantidad;
        return;
      }
      const rQuieroBRL = this.toBRL[this.quiero];
      const rTengoBRL  = this.toBRL[this.tengo];

      if (!Number.isFinite(rQuieroBRL) || !Number.isFinite(rTengoBRL)) {
        this.error = "Moneda no soportada por la API en este momento.";
        this.resultado = null;
        this.tasaActual = null;
        return;
      }

      this.tasaActual = rQuieroBRL / rTengoBRL;
      this.resultado = this.cantidad * this.tasaActual;
    },

    async convertir() {
      if (!this.puedeConvertir) {
        this.resultado = null;
        this.tasaActual = null;
        return;
      }
      await this.ensurePairs();
      if (this.toBRL) this.calcular();
    },

    limpiar() {
      this.cantidad = null;
      this.tengo = "";
      this.quiero = "";
      this.resultado = null;
      this.error = null;
      this.tasaActual = null;
    },

    formato(n) {
      return new Intl.NumberFormat("es-AR", {
        minimumFractionDigits: 2,
        maximumFractionDigits: 2
      }).format(n);
    }
  },
  watch: {
    cantidad: "convertir",
    tengo: "convertir",
    quiero: "convertir"
  }
};
</script>

<style scoped>
.convertidor { background: transparent; }
</style>
