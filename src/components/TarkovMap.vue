<script setup lang="ts">
import {inject, onMounted, type Ref, ref} from "vue";
import {Projection} from "ol/proj";
import VueQrcode from '@chenfengyuan/vue-qrcode';

import streets_of_tarkov_map_data from '../assets/map_data/streets_of_tarkov_map_data.json';
import customs_loot_map_data from '../assets/map_data/customs_loot_map_data.json';
import woods_map_data from '../assets/map_data/woods_map_data.json';
import lighthouse_loot_map_data from '../assets/map_data/lighthouse_loot_map_data.json';
import shoreline_map_data from '../assets/map_data/shoreline_map_data.json';
import interchange_map_data from '../assets/map_data/interchange_map_modified_data.json';
import reserve_map_data from '../assets/map_data/reserve_map_data.json';
import laboratory_loot_map_data from '../assets/map_data/laboratory_loot_map_modified_data.json';
import factory_map_data from '../assets/map_data/factory_map_data.json';
import sandbox_map_data from '../assets/map_data/sandbox_map_data.json';
import map_not_implemented_data from '../assets/map_data/map_not_implemented_data.json';
import enter_raid_data from '../assets/map_data/enter_raid_data.json';
import type {FeatureLike} from "ol/Feature";
import {Feature, Overlay, View} from "ol";
import {Point} from "ol/geom";
import {Icon, Style} from "ol/style";
import type VectorSource from "ol/source/Vector";
import type VectorLayer from "ol/layer/Vector";
import type {Coordinate} from "ol/coordinate";
import type Map from "ol/Map";

const gameMapNamesDict = {
  "bigmap": customs_loot_map_data,
  "TarkovStreets": streets_of_tarkov_map_data,
  "Woods": woods_map_data,
  "Lighthouse": lighthouse_loot_map_data,
  "Shoreline": shoreline_map_data,
  "Interchange": interchange_map_data,
  "RezervBase": reserve_map_data,
  "laboratory": laboratory_loot_map_data,
  "factory4_day": factory_map_data,
  "factory4_night": factory_map_data,
  "Sandbox": sandbox_map_data,
  "map_not_implemented": map_not_implemented_data,
  "EnterARaid": enter_raid_data
};

const shouldShowConnectPrompt = ref<boolean>(true);
const serverAddress = ref<string | null>("127.0.0.1");
const serverPort = ref<string | null>("45366");
let savedServerAddress = ref<string | null>();
let savedServerPort = ref<string | null>();
const qrCodeAddress = ref<string>("");
const showQrCode = ref<boolean>(false);
let mapDataUpdateInterval = 250;
let updateTimer: number;

let currentMapString = "map_not_implemented";
let currentMapData: any = map_not_implemented_data;
let mapUrl = ref("maps/" + currentMapData.MapImageFile);
let mapImageExtent = ref(currentMapData.bounds);

const projection = new Projection({
  code: 'tarkov-map',
  units: 'pixels',
  extent: mapImageExtent.value,
});

const selectConditions = inject("ol-selectconditions");

const selectCondition = selectConditions.singleClick;

const selectFilter = (feature: any) => {
  return feature.getProperties()?.NameText !== undefined
};

const mapRef = ref<{ map: Map } | null>(null);
const mapView = ref<View | null>(null);
const center = ref([40, 40]);
const zoom = ref(0);
const minZoom = ref(0);
const rotation = ref<number>(0);
const extent = ref(currentMapData.bounds);

const currentCenter = ref(center.value);
const currentZoom = ref(zoom.value);
const currentRotation = ref(rotation.value);
const currentResolution = ref(0);

const errorOverlayRef = ref<{ overlay: Overlay } | null>(null);
const errorOverlayCoords = ref<Coordinate | null>(null);
let shouldShowError = ref<boolean>(false);

const drawEnable = ref(false);
const drawType = ref("Point");

const followPlayer = ref(false);

const botMarkerLayer = ref(null);
const botMarkerSource = ref<{ source: VectorSource }>();
const botMarkers = ref<FeatureLike[]>([]);
let currentBots: Object[] = [];
let showBots = ref<boolean>(false);

let currentAirdrop: Object|null = null;
const airdropMarker = ref<FeatureLike | null>(null);

const questMarkerLayer = ref<{ layer: VectorLayer<VectorSource> } | null>(null);
const questMarkerSource = ref<{ source: VectorSource } | null>(null);
const questMarkers = ref<FeatureLike[]>([]);
const questOverlayRef = ref<{ overlay: Overlay } | null>(null);
const showQuestOverlay = ref<boolean>(true);
const questName = ref<string>("");
const questDescription = ref<string>("");
const questOverlayCoordinates = ref<Coordinate | null>(null);
let lastQuestTimestamp = 0;


// Player marker
const playerMarkerFeature = new Feature({
  geometry: new Point([-300, -300]),
  name: 'Player 1',
});

const iconStyle = new Style({
  image: new Icon({
    anchor: [0.5, 0.5],
    anchorXUnits: 'fraction',
    anchorYUnits: 'fraction',
    src: '/images/plain-arrow.png',
    scale: 0.5,
  }),
});

playerMarkerFeature.setStyle(iconStyle);

const playerMarker = ref<FeatureLike>(playerMarkerFeature);


// Airdrop marker
const airdropMarkerFeature = new Feature({
  geometry: new Point([-500, -500]),
  name: 'Airdrop',
});

const airdropMarkerStyle = new Style({
  image: new Icon({
    anchor: [0.5, 0.5],
    anchorXUnits: 'fraction',
    anchorYUnits: 'fraction',
    src: '/images/airdrop.png',
    scale: 0.5,
  }),
});

airdropMarkerFeature.setStyle(airdropMarkerStyle);

airdropMarker.value = airdropMarkerFeature;

// Bot marker
const botMarkerStyle = new Style({
  image: new Icon({
    anchor: [0.5, 0.5],
    anchorXUnits: 'fraction',
    anchorYUnits: 'fraction',
    src: '/images/bot.png',
    color: [0, 128, 128, 1],
    scale: 0.5,
  }),
});

const bossBotMarkerStyle = new Style({
  image: new Icon({
    anchor: [0.5, 0.5],
    anchorXUnits: 'fraction',
    anchorYUnits: 'fraction',
    src: '/images/bot.png',
    color: [255, 128, 128, 1],
    scale: 0.5,
  }),
});


// Quest marker
const questMarkerStyle = new Style({
  image: new Icon({
    anchor: [0.5, 0.5],
    anchorXUnits: 'fraction',
    anchorYUnits: 'fraction',
    src: '/images/check-mark.png',
    scale: 0.5
  })
});


async function connectToServer(useLastConnectionInfo: boolean) {
  if (useLastConnectionInfo) {
    serverAddress.value = localStorage.getItem('serverAddress');
    serverPort.value = localStorage.getItem('serverPort');
  } else {
    localStorage.setItem('serverAddress', serverAddress.value || '');
    localStorage.setItem('serverPort', serverPort.value || '');
  }

  getQRCodeAddress();
  
  updateTimer = setInterval(mapDataFetch, mapDataUpdateInterval);
  
  shouldShowConnectPrompt.value = false;
}

//TODO: Make the bounding boxes for areas use JSON objects saved to each map. Reference CactusPie's JSON files.
function adjustCoordinatesForMap(x: number, z: number, y: number) {
  let returnedX = calculatePolynomialValue(x, currentMapData.XCoefficients);
  let returnedZ = calculatePolynomialValue(z, currentMapData.ZCoefficients);

  // Special cases for maps that have multiple floors
  if (currentMapString === "Interchange") {
    // Main mall bounding box + Goshan extension
    if ((x < 105 && x > -157.8 && z < 193.2 && z > -303.87) || (x < -157.8 && x > -183.4 && z < 69 && z > -178.66)) {
      if (y < 25) { // Parking garage
        returnedX = calculatePolynomialValue(x, currentMapData.ParkingGarageXCoefficients);
        returnedZ = calculatePolynomialValue(z, currentMapData.ParkingGarageZCoefficients);
      } else if (y > 32) { // Floor 2
        returnedX = calculatePolynomialValue(x, currentMapData.InteriorFloor2XCoefficients);
        returnedZ = calculatePolynomialValue(z, currentMapData.InteriorFloor2ZCoefficients);
      }
    }
  } else if (currentMapString === "laboratory") {
    if (y > 3) {
      returnedX = calculatePolynomialValue(x, currentMapData.Floor2XCoefficients);
      returnedZ = calculatePolynomialValue(z, currentMapData.Floor2ZCoefficients);
    } else if (y < -2) {
      returnedX = calculatePolynomialValue(x, currentMapData.TechnicalLevelXCoefficients);
      returnedZ = calculatePolynomialValue(z, currentMapData.TechnicalLevelZCoefficients);
    }
  } else if (currentMapString === "Sandbox") {
    if (y < 15 && x < 144 && x > 50) {
      returnedX = calculatePolynomialValue(x, currentMapData.UndergroundXCoefficients);
      returnedZ = calculatePolynomialValue(z, currentMapData.UndergroundZCoefficients);
    }
  } else if (currentMapString === "factory4_day" || currentMapString === "factory4_night") {
    // This map doesn't work
    returnedX = -50;
    returnedZ = -50;
  }

  return {
    x: returnedX,
    z: returnedZ,
    y: y,
  };
}

function calculatePolynomialValue(x: number, coefficients: number[]) {
  let result = 0;

  result = coefficients[0];

  result += coefficients[1] * x;

  return result;
}

function resolutionChanged(event: any) {
  currentResolution.value = event.target.getResolution();
  currentZoom.value = event.target.getZoom();
}
function centerChanged(event: any) {
  currentCenter.value = event.target.getCenter();
}

const featureSelected = (event: any) => {  
  if (event.selected[0]) {
    if (event.selected[0].getProperties()?.NameText !== undefined) {
      showQuestOverlay.value = true;
      questOverlayCoordinates.value = event.selected[0].getGeometry().getCoordinates();
      questName.value = event.selected[0].getProperties().NameText || "error loading quest name";
      questDescription.value = event.selected[0].getProperties().Description;
    }
  } else {
    showQuestOverlay.value = false;
  }
};

function changeMap(newMapString: string) {
  console.log(`changeMap: ${newMapString}`);
  
  currentMapString = newMapString;
  currentMapData = gameMapNamesDict[currentMapString as keyof typeof gameMapNamesDict];
  
  mapUrl.value = 'maps/' + currentMapData.MapImageFile;
  mapImageExtent.value = currentMapData.bounds;
  
  center.value = [currentMapData.bounds[2] / 2, currentMapData.bounds[3] / 2];
  zoom.value = 0;
  
  minZoom.value = currentMapData.minZoom ? currentMapData.minZoom : 0;
  
  extent.value = currentMapData.bounds;
  extent.value = [currentMapData.bounds[2] * -0.1, currentMapData.bounds[2] * -0.1, currentMapData.bounds[2] * 1.1, currentMapData.bounds[3] * 1.1];
  
  zoom.value = currentMapData.initialZoom;
  
  rotation.value = currentMapData.MapRotation * (Math.PI / 180);
  
  botMarkers.value.forEach(botMarker => {
    removeBot(botMarker.getProperties().BotId);
  });

  // TODO: Fix this any type
  (playerMarker.value?.getGeometry() as any)?.setCoordinates([-1000, -1000]);

  // TODO: Fix this any type
  (airdropMarker.value?.getGeometry() as any)?.setCoordinates([-1000, -1000]);
}

function setShowBots(active: boolean) {
  showBots.value = active;

  localStorage.setItem("showBots", active.toString());
  
  if (!active) {
    botMarkers.value.forEach(botMarker => {
      removeBot(botMarker.getProperties().BotId);
    });
  }
}

function toggleShowBots() {
  showBots.value = !showBots.value;
  
  if (showBots.value) {
    console.log('Enabled showing of bots.');
  } else {
    botMarkers.value.forEach(botMarker => {
      removeBot(botMarker.getProperties().BotId);
    });
    
    console.log('Disabled showing of bots.');
  }

  localStorage.setItem("showBots", showBots.value.toString());
}

function setFollowPlayer(active: boolean) {
  if (active) {
    followPlayer.value = true;
  } else {
    followPlayer.value = false;
  }
}

async function mapDataFetch() {
  try {
    const response = await fetch('http://' + serverAddress.value + ':' + serverPort.value + '/mapData');
    const data = await response.json();
    
    if (response.status === 200) {
      // Hide the error message overlay if it's visible
      if (shouldShowError.value) shouldShowError.value = false;
      
      // if (!updatedConnectionStorageThisSession) {
      //   // Update the server address and port in localStorage
      //   localStorage.setItem("serverAddress", serverAddress.value);
      //   localStorage.setItem("serverPort", serverPort.value);
      //
      //   updatedConnectionStorageThisSession = true;
      // }
    }
    
    if (data.IsGameInProgress === false) {
      if (currentMapString !== "EnterARaid") {
        changeMap("EnterARaid");

        // TODO: Fix this any type
        (playerMarker.value?.getGeometry() as any)?.setCoordinates([-1000, -1000]);

        // TODO: Fix this any type
        (airdropMarker.value?.getGeometry() as any)?.setCoordinates([-1000, -1000]);
        
        botMarkers.value.forEach(botMarker => {
          removeBot(botMarker.getProperties().BotId);
        });
      }
      
      return;
    }
    
    // Load the new map on change
    if (data.MapName && data.MapName !== currentMapString) {
      changeMap(data.MapName);
    }
    
    // Check if the quest data has changed
    if (lastQuestTimestamp !== data.LastQuestChangeTime) {
      await questDataFetch();
      
      lastQuestTimestamp = data.LastQuestChangeTime;
    }
    
    // Update the player marker coordinates
    let {x, z} = adjustCoordinatesForMap(data.XPosition, data.ZPosition, data.YPosition);

    // TODO: Fix this any type
    (playerMarker.value?.getGeometry() as any)?.setCoordinates([x, z]);

    // TODO: Fix this any type
    (playerMarker.value as any)?.getStyle().getImage().setRotation((currentMapData.MapRotation + data.XRotation) * (Math.PI / 180));

    if (followPlayer.value) {
      // mapView.value?.animate({
      //   center: [x, z],
      //   duration: 1000,
      // });
      
      mapView.value?.setCenter([x, z]);
    }
    
    // console.log(`Updating with coordinates: ${x}, ${z}`);
    
    if (data.Airdrop && !currentAirdrop) {
      let airdropX = calculatePolynomialValue(data.Airdrop.XPosition, currentMapData.XCoefficients);
      let airdropZ = calculatePolynomialValue(data.Airdrop.ZPosition, currentMapData.ZCoefficients);
      
      addAirdrop(airdropX, airdropZ);
    }
  
    if (showBots.value) {
      // Add new bot markers
      // TODO: Fix this any type
      data.BotLocations?.forEach((bot: any) => {
        if (currentBots.find((item: any) => item.BotId === bot.BotId) === undefined) {
          // let x = calculatePolynomialValue(bot.xPosition, currentMapData.XCoefficients);
          // let z = calculatePolynomialValue(bot.zPosition, currentMapData.ZCoefficients);

          let {x, z, y} = adjustCoordinatesForMap(bot.XPosition, bot.ZPosition, bot.YPosition);

          addBot(bot.BotId, bot.BotType, x, z, y);
        }
      });

      // Update or delete existing bot markers
      // TODO: Fix these any types
      currentBots.forEach((existingBot: any) => {
        if (data.BotLocations?.find((obj: any) => obj.BotId === existingBot.BotId) === undefined) {
          // Remove missing bot
          removeBot(existingBot.BotId);
        } else {
          // Update existing bot
          let foundBot = data.BotLocations?.find((obj: any) => obj.BotId === existingBot.BotId);

          if (foundBot !== undefined) {

            let {x, z, y} = adjustCoordinatesForMap(foundBot.XPosition, foundBot.ZPosition, foundBot.YPosition);

            updateBot(foundBot.BotId, x, z, y);
          }
        }
      });
    }
  } catch (e) {
    if (e instanceof TypeError) {
      // Probably failed to connect
      clearInterval(updateTimer);
      
      console.error("Failed to connect to the mod's server");

      shouldShowError.value = true;
      errorOverlayCoords.value = [currentMapData.bounds[0] + currentMapData.bounds[2] / 2 - 30, currentMapData.bounds[1] + currentMapData.bounds[3] / 2];
      
      shouldShowConnectPrompt.value = true;
      
      clearInterval(updateTimer);
    } else {
      console.error(e);
    }
  }
}

async function questDataFetch() {
  const response = await fetch('http://' + serverAddress.value + ':' + serverPort.value + '/quests');
  const data = await response.json();
  
  questMarkers.value = [];

  // TODO: Fix this any type
  data.Quests.forEach((quest: any) => {
    let x = calculatePolynomialValue(quest.Location.X, currentMapData.XCoefficients);
    let z = calculatePolynomialValue(quest.Location.Z, currentMapData.ZCoefficients);
    
    let marker = new Feature({
      geometry: new Point([x, z]),
      NameText: quest.NameText,
      Description: quest.Description,
      Trader: quest.Trader
    });
    
    marker.setStyle(questMarkerStyle);
    
    questMarkers.value.push(marker);
  });
}

async function getQRCodeAddress() {
  const response = await fetch('/address');
  const data = await response.text();
  
  let addressToUse = data;
  
  addressToUse = 'http://' + serverAddress.value + ':' + serverPort.value;
  
  // if (addressToUse.length > 29) {
  //   addressToUse = 'ERROR';
  // }
  
  qrCodeAddress.value = addressToUse;
}

function addAirdrop(x: number, z: number) {
  console.log(`addAirdrop: ${x}, ${z}`);

  currentAirdrop = {
    x,
    z
  };

  // TODO: Fix this any type
  (airdropMarker.value?.getGeometry() as any)?.setCoordinates([x, z]);
}

function addBot(id: number, type: number, x: number, z: number, y: number) {
  const botMarkerFeature = new Feature({
    geometry: new Point([-500, -500]),
    BotId: id,
    BotType: type,
  });

  switch (type) {
    case 0:
      botMarkerFeature.setStyle(botMarkerStyle);
      break;
    case 3:
      // console.log("BOSS BOT FOUND");
      botMarkerFeature.setStyle(bossBotMarkerStyle);
      break;
    default:
      botMarkerFeature.setStyle(botMarkerStyle);
      break;
  }
  
  botMarkers.value = botMarkers.value.concat(botMarkerFeature);
  
  currentBots.push({
    BotId: id,
    XPosition: x,
    ZPosition: z,
    YPosition: y
  })
}

function removeBot(id: number) {  
  const botToRemove = botMarkers.value.find(botMarker => botMarker.getProperties().BotId === id);
  
  for (let i = 0; i < currentBots.length; i++) {
    if ((currentBots[i] as any).BotId === id) {
      currentBots.splice(i, 1);
    }
  }
  
  const newBotMarkerArray = [];
  
  for (let i = 0; i < botMarkers.value.length; i++) {
    if (botMarkers.value[i].getProperties().BotId !== id) newBotMarkerArray.push(botMarkers.value[i]);
  }
  
  botMarkers.value = newBotMarkerArray;
}

function updateBot(id: number, x: number, z: number, y: number) {
  const foundBot = botMarkers.value.find(botMarker => botMarker.getProperties().BotId === id);

  (foundBot?.getGeometry() as any)?.setCoordinates([x, z]);
}

// Init
onMounted(() => {
  let params = new URL(document.location.toString()).searchParams;
  
  if (params.get('serverAddress') && params.get('serverPort')) {
    serverAddress.value = params.get('serverAddress');
    serverPort.value = params.get('serverPort');
  }
  
  serverAddress.value = params.get("serverAddress") || "127.0.0.1";
  serverPort.value = params.get("serverPort") || "45366";
  
  savedServerAddress.value = localStorage.getItem('serverAddress');
  savedServerPort.value = localStorage.getItem('serverPort');
  
  console.log(serverAddress.value + ':' + serverPort.value);
  
  changeMap("EnterARaid");
  
  setShowBots(localStorage.getItem("showBots") === "true");
  
  shouldShowConnectPrompt.value = true;
});
</script>

<template>  
  <ol-map
      ref="gameMap"
      style="height: 100%"
      :controls="[]"
  >
    <ol-view
        ref="mapView"
        :center="center"
        :rotation="rotation"
        :zoom="zoom"
        :projection="projection"
        :extent="extent"
        :enableRotation="true"
        :showFullExtent="true"
        :minZoom="minZoom"
        @change:center="centerChanged"
        @change:resolution="resolutionChanged"
    />

<!--    <ol-control-bar v-if="!shouldShowConnectPrompt">-->
<!--&lt;!&ndash;      <ol-toggle-control&ndash;&gt;-->
<!--&lt;!&ndash;          html="🟢"&ndash;&gt;-->
<!--&lt;!&ndash;          className="edit"&ndash;&gt;-->
<!--&lt;!&ndash;          title="Circle"&ndash;&gt;-->
<!--&lt;!&ndash;          :onToggle="(active: boolean) => changeDrawType(active, 'Circle')"&ndash;&gt;-->
<!--&lt;!&ndash;      />&ndash;&gt;-->
<!--&lt;!&ndash;      <ol-toggle-control&ndash;&gt;-->
<!--&lt;!&ndash;          html="〰️"&ndash;&gt;-->
<!--&lt;!&ndash;          className="edit"&ndash;&gt;-->
<!--&lt;!&ndash;          title="Line (double click to finish)"&ndash;&gt;-->
<!--&lt;!&ndash;          :onToggle="(active: boolean) => changeDrawType(active, 'LineString')"&ndash;&gt;-->
<!--&lt;!&ndash;      />&ndash;&gt;-->
<!--&lt;!&ndash;      <ol-toggle-control&ndash;&gt;-->
<!--&lt;!&ndash;          html="〰️"&ndash;&gt;-->
<!--&lt;!&ndash;          className="edit"&ndash;&gt;-->
<!--&lt;!&ndash;          title="Trash"&ndash;&gt;-->
<!--&lt;!&ndash;          :onToggle="(active: boolean) => changeDrawType(active, 'LineString')"&ndash;&gt;-->
<!--&lt;!&ndash;      />&ndash;&gt;-->
<!--&lt;!&ndash;      <ol-zoomtoextent-control &ndash;&gt;-->
<!--&lt;!&ndash;          label="↕️"&ndash;&gt;-->
<!--&lt;!&ndash;          tipLabel="Full map"&ndash;&gt;-->
<!--&lt;!&ndash;      />&ndash;&gt;-->
<!--      <ol-toggle-control-->
<!--          html="Bots"-->
<!--          title="Show Bots"-->
<!--          :className="showBots ? 'custom-ol-active' : ''"-->
<!--          :onToggle="(active: boolean) => setShowBots(active)"-->
<!--      />-->
<!--      <ol-toggle-control-->
<!--          html="Follow"-->
<!--          title="Follow Player"-->
<!--          :onToggle="(active: boolean) => setFollowPlayer(active)"-->
<!--      />-->
<!--      <ol-toggle-control-->
<!--          html="QR"-->
<!--          title="QR"-->
<!--          className="qr-button"-->
<!--          :onToggle="(active: boolean) => showQrCode = active"-->
<!--      />-->
<!--    </ol-control-bar>-->

<!--    <ol-interaction-draw-->
<!--        :type="drawType"-->
<!--        @drawend="drawend"-->
<!--        @drawstart="drawstart"-->
<!--    >-->
<!--    </ol-interaction-draw>-->
    
    <ol-interaction-select
      @select="featureSelected"
      :condition="selectCondition"
      :layers="questMarkerLayer"
      :filter="selectFilter"
    ></ol-interaction-select>

    <ol-image-layer id="tarkovMap">
      <ol-source-image-static
          :url="mapUrl"
          :imageExtent="mapImageExtent"
          :projection="projection"
      ></ol-source-image-static>
    </ol-image-layer>

    <ol-vector-layer>
      <ol-source-vector :features="[playerMarker]" />
    </ol-vector-layer>

    <ol-vector-layer>
      <ol-source-vector :features="[airdropMarker]" />
    </ol-vector-layer>

    <ol-vector-layer ref="botMarkerLayer">
      <ol-source-vector ref="botMarkerSource" :features="botMarkers" />
    </ol-vector-layer>

    <ol-vector-layer name="Quests" ref="questMarkerLayer">
      <ol-source-vector ref="questMarkerSource" :features="questMarkers" />
    </ol-vector-layer>

    <ol-overlay
        ref="questOverlayRef"
        v-if="showQuestOverlay"
        :position="questOverlayCoordinates"
    >
      <div class="quest-overlay">
        <div class="quest-overlay__name">{{ questName }}</div>
        <div class="quest-overlay__description">{{ questDescription }}</div>
      </div>
    </ol-overlay>
  </ol-map>
  
  <div
      v-if="!shouldShowConnectPrompt"
      class="map-button-row"
    >
    <button @click="toggleShowBots()" :style="{backgroundColor: showBots ? 'lightgreen' : '#f0f0f0'}">Show Bots</button>
    <button @click="setFollowPlayer(!followPlayer)" :style="{backgroundColor: followPlayer ? 'lightgreen' : '#f0f0f0'}">Follow Player</button>
    <button @click="showQrCode = !showQrCode">Show QR</button>
  </div>
  
  <div
      v-if="shouldShowConnectPrompt"
      class="connection-prompt-box"
  >
    <div>Input server address</div>
    <div class="input-box"><span>IPV4 Address: <a href="https://support.microsoft.com/en-us/windows/find-your-ip-address-in-windows-f21a9bbc-c582-55cd-35e0-73431160a1b9#Category=Windows_10" target="_blank">?</a></span><input v-model="serverAddress" placeholder="127.0.0.1"></div>
    <div class="input-box"><span>Port:</span><input v-model="serverPort" placeholder="45365"></div>
    <button @click="connectToServer(false)">Connect</button>
    <button v-if="savedServerAddress" @click="connectToServer(true)">Last: {{savedServerAddress + ':' + savedServerPort}}</button>
  </div>    
    
  <div
      v-if="shouldShowError"
      class="connection-error"
  >
    <span>Failed to connect to SPT client!</span>
    <span>Make sure the game is running and firewall is not blocking it.</span>
  </div>
  
  <div v-if="showQrCode" class="qr-code-container">
    <div v-if="qrCodeAddress !== 'ERROR'">
      <VueQrcode :value="qrCodeAddress" :options="{ width: 260 }" class="qr-code"></VueQrcode>
      <div>{{ serverAddress }}:{{ serverPort }}</div>
    </div>
    <div v-if="qrCodeAddress === 'ERROR'">Error getting address!</div>
    <button @click="showQrCode = false" style="font-size: 0.8em; margin-top: 5px">Close</button>
  </div>
</template>

<style scoped>
  
</style>