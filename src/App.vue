<script setup lang="ts">
import { computed, onMounted, ref } from 'vue'

const getID = (id => (() => ++id))(0);

const rows = ref<number[]>(Array.from({ length : 3 }, getID));
const percent = computed(() => (100 / rows.value.length).toFixed(2));


const addRow = () => rows.value.push(getID());

const delRow = function (i: number) {
  if (rows.value.length <= 1)
    return;

  rows.value = rows.value.filter(e => e !== i);
}

const evTypes = ['control de lectura', 'trabajo parcial', 'examen parcial'];

const e_accScore = ref<HTMLOutputElement | null>(null);
const e_canAprov = ref<HTMLOutputElement | null>(null);
const e_msgFinal = ref<HTMLOutputElement | null>(null);

const execute = function (e: Event) {
  e.preventDefault();

  const entries = new FormData(e.target as HTMLFormElement);

  
  const reqScore = Number(entries.get('req-score'));
  type t_record = [string, number, number];
  const [graded, ungraded, sum] = rows.value.reduce<[t_record[],t_record[], number]>((p, _, i) => {

    const record : t_record = [
      entries.get(`${i}-0`) as string,
      Number(entries.get(`${i}-1`)) * .01,
      Number(entries.get(`${i}-2`)),
    ];

    p[entries.get(`${i}-2`) !== '' ? 0 : 1].push(record);
    p[2] += record[1];

    return p
  }, [[],[], 0])


  if (Math.abs(sum - 1.0) > 0.01)
    console.warn('suma de porcentajes no es igual a 100%');

  // const accScore = graded.reduce((p, [,w,s]) => p + w * s, 0);
  // const accValue = graded.reduce((p, [,w]) => p + w, 0) * 20.0;
  
  const [accScore, accValue] = graded.reduce<[number, number]>(([ps,pw], [,w,s]) => [ps + w * s, pw + w], [0,0]);

  const canAprov = reqScore - accScore <= ungraded.reduce((p, [,w]) => p + w * 20.0, 0);

  const msgFinal = reqScore <= accScore ? 
    'Ya has pasado el curso!' : 
    ungraded.length === 1 ? 
      (function(){
        const [t,w] = ungraded[0];

        return `${ canAprov ? 'necesitas' : 'necesitarías' } ${ ((reqScore - accScore) / w).toFixed(2) } en el ${t || 'examen final'} para pasar`
      })() : 
      (function(){
        return 'deja el puntaje de UNA sola evaluación en blanco para calcular cuanto necesitas (o necesitarías, si ya jalaste 😢) para aprobar';
      })();

  e_accScore.value!.value = accScore.toFixed(2);
  e_accScore.value!.nextElementSibling!.textContent = '/' + (accValue * 20.0).toFixed(2);

  e_canAprov.value!.value = canAprov ? 'Sí' : 'No';
  e_msgFinal.value!.value = msgFinal;
}

onMounted(() => {
  e_accScore.value!.value = '0.00';
  e_canAprov.value!.value = 'Tal vez';
  e_msgFinal.value!.value = 'la fé es lo más bonito de la vida';
})
</script>

<template>
  <main class="base">
    <h1>Calcula promedios</h1>

    <section>
      <h2>Instrucciones</h2>
      <p>
        Ingresa las notas de todas tus evaluaciones de modo que la suma del <code>peso</code> de todas sea <code>100%</code>.
        Puedes agregar un nombre a cada evaluación (o dejarlas en blanco) para que sea te sea más fácil organizarte.

        Debes llenar
      </p>
    </section>

    <form style="width: 100%" novalidate @submit="execute">


      <section aria-label="configuración">
        <table class="table-grades">
          <colgroup>
            <col width="auto" />
            <col width="112" />
            <col width="64" />
          </colgroup>

          <tbody>
            <tr>
              <th scope="row" style="text-align: left;">Puntaje requerido para aprobar</th>
              <td colspan="2">
                <div class="input">
                  <input name="req-score" required type="number" step=".05" min="0" max="20" value="12.5" />
                  <span>/20</span>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </section>

      <hr />

      <section aria-label="datos">
        <table class="table-grades">
          <colgroup>
            <col width="32" class="index" />
            <col width="auto" />
            <col width="112" />
            <col width="112" />
            <col width="64" />
          </colgroup>
          <thead>
            <tr>
              <th>#</th>
              <th>evaluación</th>
              <th>peso</th>
              <th>nota</th>
              <th></th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="(e, i) in rows" :key="e">
              <td>{{ i + 1 }}</td>
              <td>
                <div class="input">
                  <input tabindex="1" :name="`${i}-0`" required type="text" :placeholder="evTypes[Math.floor(Math.random() * 3)]" />
                </div>
              </td>
              <td>
                <div class="input">
                  <input @change="e => (e.target as HTMLInputElement).value = Number((e.target as HTMLInputElement).value).toFixed(2)" tabindex="2" :name="`${i}-1`" required type="number" step=".5" max="100" min="0" :placeholder="percent" />
                  <span>%</span>
                </div>
              </td>
              <td>
                <div class="input">
                  <input tabindex="3" :name="`${i}-2`" required type="number" step=".05" max="20" min="0"
                    :placeholder="(12.5).toFixed(2)" />
                  <span>/20</span>
                </div>
              </td>
              <td>
                <button required type="button" @click="delRow(e)">&#215;</button>
              </td>
            </tr>
            <tr>
              <td>{{ rows.length + 1 }}</td>
              <td colspan="3" style="text-align: center">
                <button type="button" @click="addRow">agregar</button>
              </td>
            </tr>
          </tbody>
        </table>
      </section>

      <div style="text-align: center; margin-block: 2rem;">
        <button style="padding: .25rem 2rem; border-radius: 4px; line-height: 1; background-color: #43f; border: none; color: white;">calcular</button>
      </div>

      <section>
        <h2>Resultados</h2>

        <table class="table-grades">
          <colgroup>
            <col width="320" />
            <col width="auto" />
            <col width="176" />
          </colgroup>

          <tbody>
            <tr>
              <th scope="row" colspan="2" style="text-align: left;">Tienes acumulado hasta ahora: / lo que podrías tener si sacabas 20:</th>
              <td>
                <div class="input">
                  <output ref="e_accScore" style="text-align: right; padding-inline: .5rem;"></output>
                  <span>/20</span>
                </div>
              </td>
            </tr>
            <tr>
              <th scope="row" colspan="2" style="text-align: left;">Tienes oportunidad de aprovar?:</th>
              <td>
                <div class="input">
                  <output ref="e_canAprov" ></output>
                </div>
              </td>
            </tr>
            <tr>
              <th scope="row" colspan="1" style="text-align: left;">Mensaje final:</th>
              <td colspan="2" >
                <div class="input">
                  <output ref="e_msgFinal" ></output>
                </div>
              </td>
            </tr>
          </tbody>
        </table>
      </section>

    </form>
  </main>
</template>

<style>
.base {
  max-width: min(56rem, 100%);
  margin-inline: auto;
  padding-inline: .5rem;
}

.table-grades {
  & {
    background-color: #8882;
    border-collapse: collapse;
    border: 1px solid black;
    line-height: 1.25;

    font-size: 14px;
    font-family: Arial, Helvetica, sans-serif;

    width: 100%;
  }

  & :where(input, output) {
    & {
      display: block;
      color: inherit;

      padding: 0;

      width: 100%;

      border: none;
      background-color: white;

      padding-left: .5rem;

      text-align: inherit;
      font-size: inherit;
      font-family: inherit;
      font-weight: inherit;
    }

    &[type="number"] { text-align: right; }
    &[type="text"] { text-align: left; }

    &:focus { outline: none; }

    &:is(output) {
      line-height: 1.75;
    }

    &::placeholder {
      color: #888d;
    }
  }

  & div.input:only-child:has(> :where(input, output)) {
    & {
      display: flex;
      align-items: stretch;

      margin: .25rem;
      height: calc(100% - .5rem);

      border: 1px solid currentColor;
      border-radius: 4px;
      overflow: clip;

      color: #457;
      background-color: #bbc;
    }

    &:has(:focus, :valid, output) {
      outline: none;
      color: black;
    }

    &>*:not(:last-child) {
      border-right: 1px solid currentColor;
      margin: 0;
    }

    > span {
      padding-inline: .25rem;
      font-size: small;
      font-weight: 500;
      align-self: center;
    }
  }

  &> :where(tbody, thead)>tr> :where(td, th) {
    & {
      padding: 0;
      height: 2.25rem;

      border-block: 1px solid black;
      border-inline: 1px dashed #999;
    }

    &:nth-child(1) {
      text-align: center;
    }

    &:is(th):not(.custom) {
      text-align: center;
      padding-inline: .5rem;
    }

    &:nth-child(5)>button:only-child {
      display: block;
      width: auto;
      height: 75%;
      margin: 0 auto;
      aspect-ratio: 1;

      border: none;
      border-radius: 50%;
      font-size: large;
      font-weight: 700;
      color: white;

      background-color: #d43;
    }
  }

  &>tbody>tr:last-child>td>button:only-child {
    display: inline-block;
    margin: 0;
    border: none;
    color: white;

    padding: .25rem 2.25rem;
    border-radius: 4px;

    background-color: #3b6;
  }
}</style>
