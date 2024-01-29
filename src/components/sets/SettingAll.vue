<script setup lang="ts">
import { Store } from "tauri-plugin-store-api";
import { join, resourceDir } from "@tauri-apps/api/path";
import { enable, disable } from "tauri-plugin-autostart-api";
import { WebviewWindow } from '@tauri-apps/api/window';
import { Ref, onBeforeMount, ref } from "vue";
import {
  CursorArrowRaysIcon, ArrowUpTrayIcon, RocketLaunchIcon
} from '@heroicons/vue/24/outline'

import { FileEntry, readDir } from "@tauri-apps/api/fs";
import AddModelDialog from "./AddModelDialog.vue";
import SettingModels from "./SettingModels.vue";

const resourceDirPath = await resourceDir();
const pathAlive = await join(resourceDirPath, 'data', 'sets_alive.json');
const pathLive = await join(resourceDirPath, 'data', 'sets_live2d.json');
const pathMMD = await join(resourceDirPath, 'data', 'sets_mmd.json');

const storeAlive = new Store(pathAlive);
const storeLive = new Store(pathLive);
const storeMMD = new Store(pathMMD);
const addingModel = ref(false);

// 获取设置信息
const sClickThroughe = ref(await storeAlive.get("click_through") as boolean)
const sStayTop = ref(await storeAlive.get("stay_top") as boolean)
const sAutoStart = ref(await storeAlive.get("auto_start") as boolean)
const isMMD = ref(await storeAlive.get("is_mmd") as boolean);
const sLiveHttps = ref(await storeLive.get("http_models") as string[])
const sLiveUrl = ref(await storeLive.get("model_url") as string)
const sMmdHttps = ref(await storeMMD.get("http_models") as string[])
const sMmdUrl = ref(await storeMMD.get("mmd_alive_url") as string)
console.log("sLiveUrl: ", sLiveUrl, "sMmdUrl: ", sMmdUrl)
const currMmdModel = ref("")
const currLiveModel = ref("")


const allLiveNames: Ref<string[]> = ref([]);
const allLivePaths: Ref<string[]> = ref([]);
const allMmdNames: Ref<string[]> = ref([]);
const allMmdPaths: Ref<string[]> = ref([]);

const activeTab = ref(isMMD ? 'MMD' : 'Live2d')

function changeTab(tab: string) {
  activeTab.value = tab
}


onBeforeMount(() => {
  const sCurrLive = sLiveUrl.value
  allLivePaths.value[0] = sCurrLive
  const modelLive = getModelName(sCurrLive)
  currLiveModel.value = modelLive
  allLiveNames.value[0] = modelLive

  const sCurrMmd = sMmdUrl.value
  allMmdPaths.value[0] = sCurrMmd
  const modelMmd = getModelName(sCurrMmd)
  currMmdModel.value = modelMmd
  allMmdNames.value[0] = modelMmd

  // 添加网络中的模型
  sLiveHttps.value.forEach(element => {
    const httpName = getModelName(element);
    if (httpName !== modelLive) {     // 不重复添加
      allLivePaths.value.push(element)
      allLiveNames.value.push(httpName)
    }
  });
  console.log(allLiveNames);
  sMmdHttps.value.forEach(element => {
    const httpName = getModelName(element);
    if (httpName !== modelMmd) {
      allMmdPaths.value.push(element)
      allMmdNames.value.push(httpName)
    }
  });

  // 都刷新一下
  refreshModels(false)
  refreshModels(true)
})

function getModelName(modelUrl: string): string {
  const lastSlash = (modelUrl.startsWith("http") || modelUrl.startsWith("/")) ? modelUrl.lastIndexOf("/") : modelUrl.lastIndexOf("\\")
  return modelUrl.substring(lastSlash + 1)
}

async function refreshModels(_is_mmd: boolean) {
    console.log("refreshModels: ", _is_mmd);
  const modelPath = _is_mmd ? await join(resourceDirPath, 'mmd') : await join(resourceDirPath, 'live2d');
  const entries = await readDir(modelPath, { recursive: true });
  findModels(entries, _is_mmd);
}
async function findModels(models: FileEntry[], _is_mmd: boolean) {
  for (const model of models) {

    if (model.children) {
      findModels(model.children, _is_mmd)
    } else {
      if (_is_mmd) {
        if (model.name?.endsWith(".alive_mmd.json")) {
          if (allMmdNames.value.indexOf(model.name) === -1) {
            allMmdNames.value.push(model.name);
            allMmdPaths.value.push(model.path);
          }
        }
      } else {
        if (model.name?.endsWith(".model3.json") || model.name?.endsWith(".model.json")) {
          if (allLiveNames.value.indexOf(model.name) === -1) {
            allLiveNames.value.push(model.name);
            allLivePaths.value.push(model.path);
          }
        }
      }
    }
  }
}

async function setSettings(key: string, val: any) {
  console.log("setSettings-", key, ": ", val);
  const mainWindow = WebviewWindow.getByLabel('main')

  switch (key) {
    case "click_through": {
      sClickThroughe.value = val as boolean
      if (val) {
        mainWindow?.setIgnoreCursorEvents(true);
      } else {
        mainWindow?.setIgnoreCursorEvents(false);
      }
      await storeAlive.set(key, val);
      await storeAlive.save();
      break;
    }
    case "stay_top": {
      sStayTop.value = val as boolean
      if (val) {
        mainWindow?.setAlwaysOnTop(true);
      } else {
        mainWindow?.setAlwaysOnTop(false);
      }
      await storeAlive.set(key, val);
      await storeAlive.save();
      break;
    }
    case "auto_start": {
      sAutoStart.value = val as boolean
      if (val) {
        enable();
      } else {
        disable();
      }
      await storeAlive.set(key, val);
      await storeAlive.save();
      break;
    }
    default: { }
  }
}
async function setModel(model: string) {
  const key = "model_url";
  const mainWindow = WebviewWindow.getByLabel('main')
  var foundURL = allLivePaths.value.find(
    (element) => {
      const path = element.replace(/\\/g, "/");
      return path.indexOf(model) !== -1;
    }
  );
  if (foundURL !== undefined) {
    const http = foundURL.replace(resourceDirPath, "https://asset.localhost/");
    const url = http.replace(/\\/g, "/");
    console.log("foundURL: ", url);
    if (activeTab.value === "MMD") {
      currMmdModel.value = model
      await storeMMD.set(key, url);
      await storeMMD.save();
      mainWindow?.emit('event_model_url', url);
    } else {
      currLiveModel.value = model
      await storeLive.set(key, url);
      await storeLive.save();
      mainWindow?.emit('event_model_url', url);
    }
  }
}

async function addModel(modelURL: string) {
  console.log("addModel: ", modelURL);
  const modelName = getModelName(modelURL)
  if (modelURL === "") return

  if (activeTab.value === "MMD" && modelURL.startsWith("http") && (modelURL.endsWith("alive_mmd.json"))) {
    sMmdHttps.value.push(modelURL)
    allMmdPaths.value.push(modelURL)
    allMmdNames.value.push(modelName)

    await storeMMD.set("http_models", sMmdHttps.value)
    await storeMMD.save()
    addingModel.value = false

  } else if (activeTab.value === "Live2d" && modelURL.startsWith("http") && (modelURL.endsWith("model3.json") || modelURL.endsWith("model.json"))) {
    sLiveHttps.value.push(modelURL)

    allLivePaths.value.push(modelURL)
    allLiveNames.value.push(modelName)

    await storeLive.set("http_models", sLiveHttps.value)
    await storeLive.save()
    addingModel.value = false
  }
}

</script>

<template>
  <div class="flex flex-col w-lvw h-lvh bg-blue-50">
    <div class="flex space-x-2 mt-1 mx-1">
      <button @click="() => changeTab('Alive')"
        :class="activeTab === 'Alive' ? 'sets-tab-selected' : 'sets-tab'">Alive</button>
      <button @click="() => changeTab('Live2d')"
        :class="activeTab === 'Live2d' ? 'sets-tab-selected' : 'sets-tab'">Live2d</button>
      <button @click="() => changeTab('MMD')" :class="activeTab === 'MMD' ? 'sets-tab-selected' : 'sets-tab'">MMD</button>
    </div>

    <div v-if="activeTab === 'Alive'" :class="activeTab === 'Alive' ? 'sets-content' : ''">
      <div class="h-20 grid grid-cols-2 grid-rows-2">
        <button class="flex h-full items-center mx-4 px-4 space-x-3">
          <CursorArrowRaysIcon class=" w-6 h-6" />
          <span class="w-28 text-left">Click Through</span>
          <input type="checkbox" class="w-4 h-4" :checked="sClickThroughe"
            @change="() => { setSettings('click_through', !sClickThroughe) }" />
        </button>
        <button class="flex h-full items-center mx-4 px-4 space-x-3">
          <ArrowUpTrayIcon class=" w-6 h-6" />
          <span class="w-28 text-left">Stay at top</span>
          <input type="checkbox" class="w-4 h-4" :checked="sStayTop"
            @change="() => { setSettings('stay_top', !sStayTop) }" />
        </button>
        <button class="flex h-full items-center mx-4 px-4 space-x-3">
          <RocketLaunchIcon class=" w-6 h-6" />
          <span class="w-28 text-left">Start at launch</span>
          <input type="checkbox" class="w-4 h-4" :checked="sAutoStart"
            @change="() => { setSettings('auto_start', !sAutoStart) }" />
        </button>
      </div>
    </div>

    <div v-else-if="activeTab === 'Live2d'" :class="activeTab === 'Live2d' ? 'sets-content' : ''">
      <SettingModels :models-title="'Choose Live2d model: '" :curr-model="currLiveModel" 
       :models="allLiveNames" @refresh-models="refreshModels" @set-model="setModel" 
       @add-model="addingModel = !addingModel"/>
    </div>

    <div v-else="activeTab === 'MMD'" :class="activeTab === 'MMD' ? 'sets-content' : ''">
      <SettingModels v-bind:models-title="'Choose MMD model: '" v-bind:curr-model="currMmdModel" 
       v-bind:models="allMmdNames" @refresh-models="refreshModels" @set-model="setModel" 
       @add-model="addingModel = !addingModel"/>

      <!-- <div id="add_model_dialog" v-show="addingModel" class=" absolute top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 bg-slate-200 z-10 py-4 px-6 
     rounded-xl border-[3px] border-slate-600 shadow-xl space-y-2">
      <h2 class="text-md pb-3 font-bold">Url path of your live2d model: </h2>
      <div class=" flex items-center space-x-2">
      <input type="text" class=" h-8 w-96 bg-slate-200 rounded-md border-2 border-slate-500" />
      <button class="font-bold hover:text-blue-400">submit</button>
      </div>
    </div> -->
    </div>

    <AddModelDialog v-if="addingModel" @event-close="addingModel = false" @event-submit="addModel" />
  </div>
</template>