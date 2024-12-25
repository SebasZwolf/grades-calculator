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

        return `${ canAprov ? 'necesitas' : 'necesitarías' } ${ ((reqScore - accScore) / w).toFixed(2) } en el ${t ? `"${t}"` : 'examen final'} para pasar`
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
  e_msgFinal.value!.value = 'no olvides llenar todos los campos de "%"';
})
</script>

<template>
  <main class="base">
    <h1>Calcula promedios</h1>

    <section style="margin-block: 1rem">
      <h2>Instrucciones</h2>
      <p style="line-height: 1.25; margin-block: .5rem;">
        Ingresa las notas de todas tus evaluaciones de modo que la suma del <code>peso</code> de todas sea <code>100%</code>.
      </p>
      <p style="line-height: 1.25; margin-block: .5rem;">
        Puedes agregar un nombre a cada evaluación (o dejarlas en blanco) para que sea te sea más fácil organizarte.
      </p>
      <p style="line-height: 1.25; margin-block: .5rem;">
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
                  <input name="req-score" required type="number" inputmode="numeric" pattern="\d*" step=".05" min="0" max="20" value="12.5" />
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
                  <input @change="e => (e.target as HTMLInputElement).value = Number((e.target as HTMLInputElement).value).toFixed(2)" tabindex="2" :name="`${i}-1`" required type="number" inputmode="numeric" pattern="\d*" step=".5" max="100" min="0" :placeholder="percent" />
                  <span>%</span>
                </div>
              </td>
              <td>
                <div class="input">
                  <input tabindex="3" :name="`${i}-2`" required type="number" inputmode="numeric" pattern="\d*" step=".05" max="20" min="0"
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
                <button class="action-button" type="button" @click="addRow">agregar</button>
              </td>
            </tr>
          </tbody>
        </table>
      </section>

      <div style="text-align: center; margin-block: 2rem;">
        <button 
          class="action-button"
          style="--theme: #43fa;">calcular</button>
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
              <th scope="row" colspan="2" style="text-align: left;">Nota actual / sobre:</th>
              <td >
                <div class="input" >
                  <output ref="e_accScore" style="text-align: right; padding-inline: .5rem; align-content: center;"></output>
                  <span>/20</span>
                </div>
              </td>
            </tr>
            <tr>
              <th scope="row" colspan="2" style="text-align: left;">Puedes aprobar?:</th>
              <td>
                <div class="input">
                  <output ref="e_canAprov" ></output>
                </div>
              </td>
            </tr>
            <tr>
              <th scope="row" colspan="1" style="text-align: left; align-content: start;">Mensaje final:</th>
              <td colspan="2" >
                <div class="input">
                  <output style="line-height: 1.3; padding-block: .25rem;" ref="e_msgFinal" ></output>
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

    &[type="number"] {
      text-align: right;
      appearance: textfield;
      -moz-appearance: textfield;
    }
    &[type="number"]::-webkit-outer-spin-button,
    &[type="number"]::-webkit-inner-spin-button {
      -webkit-appearance: none;
      margin-right: .5rem;
    }

    &[type="text"] { text-align: left; }

    &:focus { outline: none; }

    &:is(output) { line-height: 1.75; }
    &::placeholder { color: #888d; }
  }

  & div.input:only-child:has(> :where(input, output)) {
    & {
      display: flex;
      align-items: stretch;

      margin-inline: .125rem;
      height: calc(100% - .125rem);

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

  & > :where(tbody, thead)>tr> :where(td, th) {
    & {
      padding: 2px;
      height: 2.25rem;

      vertical-align: middle;

      border-block: 1px solid black;
      /* border-inline: 1px dashed #999; */
    }

    &:nth-child(1) {
      text-align: center;
    }

    &:is(th):not(.custom) {
      text-align: center;
      padding-inline: .5rem;
    }

    &:nth-child(5) > button:only-child {
      display: block;

      width: 1.5rem;
      height: 1.5rem;
      line-height: 1;
      padding : 0 0;

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
}
.action-button {
  & {
    line-height: 1.25;
    display: inline-block;
    margin: 0;
    border: none;
    color: inherit;
    
    font-size: 1em;
    
    padding: .25rem 2.25rem;
    border-radius: 4px;
 
    font-weight: bold;
    
    background-color: var(--theme, #1a3a);
    background-clip: padding-box;
    
    transition: scale .2s, translate .2s;
  }

  &:hover {
    background-image: linear-gradient(to top, #0002, #0002);
  }
  &:active {
    background-image: linear-gradient(to top, #0004, #0003);
    translate : 0 1px;
  }
}
</style>
