<script setup>
import { ref, watch, onMounted } from 'vue';
import MonacoEditor from 'monaco-editor-vue3'

// name to be used as key in local storage
const inputKeyName = ref('');
const templateKeyName = ref('');

// list of keys in localstorage
const savedInput = ref([]);
const savedTemplate = ref([]);

// form input data
const fields = ref([
  { name: 'username', widget: 'text', validation: { required: true }, _expand: false },
  { name: 'password', widget: 'password', validation: { required: true }, _expand: false }
]);

// template and generated code data
const templateCode = ref("{% for field in fields %}{{ field.name }}\n{% endfor %}");
const generatedCode = ref("");

//  options for monaco text editor
const options = {
  colorDecorators: true,
  lineHeight: 18,
  tabSize: 2,
  fontSize: 12,
  theme: 'vs-dark',
}

const widgetChoices = [
  'text',
  'textarea',
  'password',
  'select',
  'checkbox',
  'radio',
]

// update generated code on every update
watch(fields.value, () => {
  generate();
})

const addField = function () {
  fields.value.push({ name: '', widget: 'text', validation: { required: true }, _expand: false })
}

const collapseAll = function () {
  fields.value.forEach(v => {
    v._expand = false;
  })
}

const moveUp = function (i) {
  if (i == 0)
    return;

  const top = fields.value[i - 1]
  const bottom = fields.value[i]

  fields.value[i] = top;
  fields.value[i - 1] = bottom;
}

const moveDown = function (i) {
  if (i == fields.value.length - 1)
    return;

  const top = fields.value[i];
  const bottom = fields.value[i + 1];

  fields.value[i] = bottom;
  fields.value[i + 1] = top;
}

const removeField = function (i) {
  if (!fields.value[i].name) {
    fields.value.splice(i, 1);
  } else if (confirm("Delete?")) {
    fields.value.splice(i, 1);
  }
}

// generate code from input and template
const generate = function () {
  nunjucks.configure({ autoescape: true });
  generatedCode.value = nunjucks.renderString(templateCode.value, { fields: fields.value });
}

// generate label from field name
function generateLabelValue(index) {
  fields.value[index].label = camelToLabel(fields.value[index].name);
}

function capitalizeFirstLetter(string) {
  return string.charAt(0).toUpperCase() + string.slice(1);
}

function camelToLabel(string) {
  string = string.replace(/([0-9A-Z])/g, ' $1');
  return capitalizeFirstLetter(string);
}

// handle key shortcuts on name and label input
// arrow key up, down, left and right: move cursor to adjacent input
// ctrl + [up|down] move field up or down
function actionKey(e, key, index) {
  console.log(e.target.selectionStart)
  console.log(e.target.selectionEnd)
  console.log(e.target.value.length)
  if (e.code == 'ArrowDown') {
    if (e.ctrlKey)
      moveDown(index);

    focusById(`input-${key}-${index + 1}`);
  } else if (e.code == 'ArrowUp') {
    if (e.ctrlKey)
      moveUp(index)

    focusById(`input-${key}-${index - 1}`);
  } else if (e.code == 'ArrowLeft' && key == 'label') {
    if (e.target.selectionStart == e.target.selectionEnd && e.target.selectionStart == 0) {
      focusById(`input-name-${index}`);
    }
  } else if (e.code == 'ArrowRight' && key == 'name') {
    if (e.target.selectionStart == e.target.selectionEnd && e.target.selectionStart == e.target.value.length) {
      focusById(`input-label-${index}`);
    }
  }
}

// focus on input and move cursor to end position
function focusById(id) {
  // setTimeout required for google chrome
  setTimeout(() => {
    const el = document.getElementById(id);
    if (el) {
      el.selectionStart = el.selectionEnd = el.value.length;
      el.focus();
    }
  }, 0);
}

onMounted(() => {
  const localstorageKeys = Object.keys(localStorage);
  savedInput.value = localstorageKeys.filter(k => k.match(/^saved-input-/)).map(k => k.substring("saved-input-".length))
  savedTemplate.value = localstorageKeys.filter(k => k.match(/^saved-template-/)).map(k => k.substring("saved-template-".length))
})

const loadTemplate = function () {
  templateCode.value = localStorage.getItem(`saved-template-${templateKeyName.value}`);
}

const saveTemplate = function () {
  localStorage.setItem(`saved-template-${templateKeyName.value}`, templateCode.value)

  if (!savedTemplate.value.includes(templateKeyName))
    savedTemplate.value.push(templateKeyName.value);
}

const loadInput = function () {
  const fieldsData = localStorage.getItem(`saved-input-${inputKeyName.value}`);
  fields.value = JSON.parse(fieldsData);
}

const saveInput = function () {
  localStorage.setItem(`saved-input-${inputKeyName.value}`, JSON.stringify(fields.value))

  if (!savedInput.value.includes(inputKeyName))
    savedInput.value.push(inputKeyName.value);
}


</script>

<template>
  <div class="grid">
    <div class="col">
      <!-- Input Header -->
      <div class="flex align-items-top pb-1 mb-2 border-bottom-1 surface-300">
        <!-- save input data -->
        <input type="text" class="px-2 py-1 mr-1" placeholder="Form name" v-model="inputKeyName">
        <button @click="saveInput" class="px-2 py-1 mr-3">Save</button>

        <!-- load input data -->
        <select name="" id="" class="px-2 py-1 mr-1" v-model="inputKeyName">
          <option :value="key" v-for="key in savedInput" :key="key">{{ key }}</option>
        </select>
        <button @click="loadInput" class="px-2 py-1 mr-3">Load</button>
        

        <!-- Collapse, Add Field -->
        <button class="py-1 mr-1" @click="collapseAll">Collapse All</button>
        <button class="py-1 mr-1" @click="addField">Add Field</button>
      </div>

      <!-- Input form (the fields) -->
      <template v-for="(field, i) in fields" :key="i">
        <div>
          <!-- Name, Label, Remove, Up, Down, Expand -->
          <div :class="field._expand ? 'bg-pink-100 border-top-1 pt-1 pb-1' : 'pb-1'">
            <input type="text" placeholder="Name" v-model="field.name" class="w-15rem px-2 py-1 mr-1"
              @change="() => generateLabelValue(i)" @keydown="(e) => actionKey(e, 'name', i)" :id="`input-name-${i}`">
            <input type="text" placeholder="Label" v-model="field.label" class="w-9rem px-2 py-1 mr-1"
              @keydown="(e) => actionKey(e, 'label', i)" :id="`input-label-${i}`">
            <button tabindex="-1" class="py-1 mr-1" @click="() => removeField(i)">Remove</button>
            <button tabindex="-1" class="py-1 mr-1" @click="() => moveUp(i)">Up</button>
            <button tabindex="-1" class="py-1 mr-1" @click="() => moveDown(i)">Dw</button>
            <button tabindex="-1" class="py-1 mr-1" :class="field._expand ? 'bg-pink-200' : ''"
              @click="() => field._expand = !field._expand">
              {{ field._expand ? 'Less' : 'More' }}
            </button>

            <!-- Summary for expanded part (when collapsed) -->
            <div v-if="!field._expand" class="text-xs mb-2">
              <small class="mr-1">
                <strong>Type </strong>
                {{ field.widget }}
              </small>
              <small class="mr-1">
                <strong> onMounted </strong>
                <span v-if="field.onMounted" @click="() => field._expand = true"
                  class="text-primary cursor-pointer">[code]</span>
                <span v-else class="text-500 font-italic">None</span>
              </small>
              <small>
                <strong> Validation </strong>
                <span>{{ Object.entries(field.validation).filter(arr => arr[1]).map(arr => arr[0]).join(',') }}</span>
              </small>
            </div>
          </div>

          <!-- Expanded part -->
          <div class="bg-pink-100" v-if="field._expand">
            <!-- Placeholder, Default Value -->
            <div>
              <input type="text" placeholder="Placeholder" v-model="field.placeholder"
                class="w-15rem px-2 py-1 mb-1 mr-1">
              <input type="text" placeholder="Default value" v-model="field.defaultValue"
                class="w-15rem px-2 py-1 mb-1 mr-1">
            </div>
            <!-- class label, class input -->
            <div>
              <input type="text" placeholder="class (label)" v-model="field.classLabel"
                class="w-15rem px-2 py-1 mb-1 mr-1">
              <input type="text" placeholder="class (input)" v-model="field.classInput"
                class="w-15rem px-2 py-1 mb-1 mr-1">
            </div>
            <!-- :class label, :class input -->
            <div>
              <textarea class="w-15rem px-2 py-1 mr-1 mb-1" placeholder=":class (label)"
                v-model="field._classLabel"></textarea>
              <textarea class="w-15rem px-2 py-1 mr-1 mb-1" placeholder=":class (input)"
                v-model="field._classInput"></textarea>
            </div>
            <!-- widget selection -->
            <div class="flex align-items-center mb-2">
              <div class="w-6rem">Widget</div>
              <template v-for="(widget, iw) in widgetChoices" :key="`${i}-widget-${iw}`">
                <input type="radio" :id="`${i}-widget-${iw}`" v-model="field.widget" :value="widget">
                <label :for="`${i}-widget-${iw}`" class="mr-1 mt-1">{{ widget }}</label>
              </template>
            </div>
            <!-- custom code to load selection data -->
            <div class="flex align-items-top mb-2">
              <div class="w-6rem">onMounted</div>
              <textarea class="w-30rem px-2 py-1 mr-1 mb-1" id="" cols="30" rows="4" v-model="field.onMounted"
                placeholder="Options for select/checkbox/radio"></textarea>
            </div>
            <!-- validation options -->
            <div class="flex align-items-center mb-2 pb-2 border-bottom-1">
              <div class="w-6rem">Validation</div>
              <input type="checkbox" :id="`${i}-widget-required`" v-model="field.validation.required" value="yes">
              <label :for="`${i}-widget-required`" class="mr-2">required</label>

              <input type="checkbox" :id="`${i}-widget-email`" v-model="field.validation.email" value="yes">
              <label :for="`${i}-widget-email`" class="mr-2">email</label>

              <input type="checkbox" :id="`${i}-widget-maxlength`" v-model="field.validation.maxLength" value="yes">
              <label :for="`${i}-widget-maxlength`">max length=</label>
              <input type="text" id="" v-model="field.validation.maxLengthValue" class="w-2rem px-1">
            </div>
          </div>
        </div>
      </template>
    </div>

    <div class="col">
      <!-- Template -->
      <div class="flex align-items-top mb-1">
        <input type="text" class="px-2 py-1 mr-1" placeholder="Form name" v-model="templateKeyName">
        <button @click="saveTemplate" class="px-2 py-1 mr-3">Save</button>

        <select name="" id="" class="px-2 py-1 mr-1" v-model="templateKeyName">
          <option :value="key" v-for="key in savedTemplate" :key="key">{{ key }}</option>
        </select>
        <button @click="loadTemplate" class="px-2 py-1 mr-3">Load</button>
      </div>

      <!-- Template Editor -->
      <MonacoEditor :options="options" language="html" :width="800" :height="800" v-model:value="templateCode">
      </MonacoEditor>
    </div>

    <div class="col">
      <button @click="generate" class="px-2 py-1 mb-1">Generate</button>
      <!-- Result Editor -->
      <MonacoEditor :options="options" language="html" :width="600" :height="800" v-model:value="generatedCode">
      </MonacoEditor>
    </div>
  </div>
</template>

<style scoped>
body,
input,
textarea,
select,
label,
div,
button {
  font-family: 'Courier New', Courier, monospace;
  font-size: 13px;
}
</style>