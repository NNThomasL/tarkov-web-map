<script setup lang="ts">
import {inject, onMounted, type Ref, ref} from "vue";
import {Projection} from "ol/proj";

import playerIcon from '@/assets/images/plain-arrow.png';
import airdropIcon from '@/assets/images/airdrop.png';
import botIcon from '@/assets/images/bot.png';

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

const serverAddress = ref<string>("localhost:45365");
let mapDataUpdateInterval = 250;
let updateTimer: number;

let currentMapString = "map_not_implemented";
let currentMapData = map_not_implemented_data;
let mapUrl = ref("maps/" + currentMapData.MapImageFile);
let mapImageExtent = ref(currentMapData.bounds);

const projection = new Projection({
  code: 'tarkov-map',
  units: 'pixels',
  extent: mapImageExtent.value,
});

const selectConditions = inject("ol-selectconditions");

const selectCondition = selectConditions.singleClick;

let gameMap = ref(null);
let mapView = ref<View>();
const center = ref([40, 40]);
const zoom = ref(0);
const minZoom = ref(0);
const rotation = ref<number>(0);
const extent = ref(currentMapData.bounds);

const currentCenter = ref(center.value);
const currentZoom = ref(zoom.value);
const currentRotation = ref(rotation.value);
const currentResolution = ref(0);

const errorOverlayRef = ref<{ overlay: Overlay }>(null);
const errorOverlayCoords = ref<Coordinate | null>(null);
let shouldShowError = ref<boolean>(false);

const drawEnable = ref(false);
const drawType = ref("Point");

const followPlayer = ref(false);

const botMarkerLayer = ref(null);
const botMarkerSource = ref<{ source: VectorSource }>();
const botMarkers = ref<FeatureLike[]>([]);
let currentBots: Object[] = [];
let showBots = false;

let currentAirdrop: Object|null = null;
const airdropMarker = ref<FeatureLike>(null);

const questMarkers = ref<FeatureLike[]>([]);
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

const changeDrawType = (active: boolean, draw: string) => {
  drawEnable.value = active;
  drawType.value = draw;
};

const drawstart = (event: any) => {
  console.log(event);
};

const drawend = (event: any) => {
  console.log(event);
};

function resolutionChanged(event: any) {
  currentResolution.value = event.target.getResolution();
  currentZoom.value = event.target.getZoom();
}
function centerChanged(event: any) {
  currentCenter.value = event.target.getCenter();
}

const featureSelected = (event: any) => {
  console.log(event);
};

const selectInteactionFilter = (feature: any) => {
  return feature.values_.name != undefined;
};

function changeMap(newMapString: string) {
  console.log(`changeMap: ${newMapString}`);
  
  // if (!(newMapString in Object.keys(gameMapNamesDict))) {
  //   currentMapString = 'map_not_implemented';
  // } else {
  //   currentMapString = newMapString;
  // }
  
  // currentMapString = (!(newMapString in gameMapNamesDict)) ? 'map_not_implemented' : newMapString;
  
  currentMapString = newMapString;
  currentMapData = gameMapNamesDict[currentMapString as keyof typeof gameMapNamesDict];
  
  mapUrl.value = 'maps/' + currentMapData.MapImageFile;
  mapImageExtent.value = currentMapData.bounds;
  
  center.value = [currentMapData.bounds[2] / 2, currentMapData.bounds[3] / 2];
  zoom.value = 0;
  
  minZoom.value = currentMapData.minZoom ? currentMapData.minZoom : 0;
  
  extent.value = currentMapData.bounds;
  extent.value = [currentMapData.bounds[2] * -0.1, currentMapData.bounds[2] * -0.1, currentMapData.bounds[2] * 1.1, currentMapData.bounds[3] * 1.1];
  
  rotation.value = currentMapData.MapRotation * (Math.PI / 180);
  
  botMarkers.value.forEach(botMarker => {
    removeBot(botMarker.getProperties().BotId);
  });
  
  playerMarker.value?.getGeometry()?.setCoordinates([-1000, -1000]);
  
  airdropMarker.value?.getGeometry()?.setCoordinates([-1000, -1000]);
}

function setShowBots(active: boolean) {
  showBots = active;
  
  if (!active) {
    botMarkers.value.forEach(botMarker => {
      removeBot(botMarker.getProperties().BotId);
    });
  }
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
    const response = await fetch('http://' + serverAddress.value + '/mapData');
    const data = await response.json();
  
    console.log(data);
    
    // if (data?.MapName === undefined || data.MapName === currentMapString) {
    //   return;
    // }
    
    if (data.IsGameInProgress === false) {
      if (currentMapString !== "EnterARaid") {
        changeMap("EnterARaid");

        playerMarker.value?.getGeometry()?.setCoordinates([-1000, -1000]);

        airdropMarker.value?.getGeometry()?.setCoordinates([-1000, -1000]);
        
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
    
    playerMarker.value?.getGeometry()?.setCoordinates([x, z]);
    playerMarker.value?.getStyle().getImage().setRotation((currentMapData.MapRotation + data.XRotation) * (Math.PI / 180));

    if (followPlayer.value) {
      mapView.value?.setCenter([x, z]);
    }
    
    // console.log(`Updating with coordinates: ${x}, ${z}`);
    
    if (data.Airdrop && !currentAirdrop) {
      let airdropX = calculatePolynomialValue(data.Airdrop.XPosition, currentMapData.XCoefficients);
      let airdropZ = calculatePolynomialValue(data.Airdrop.ZPosition, currentMapData.ZCoefficients);
      
      addAirdrop(airdropX, airdropZ);
    }
  
    if (showBots) {
      // Add new bot markers
      data.BotLocations?.forEach(bot => {
        if (currentBots.find(item => item.BotId === bot.BotId) === undefined) {
          // let x = calculatePolynomialValue(bot.xPosition, currentMapData.XCoefficients);
          // let z = calculatePolynomialValue(bot.zPosition, currentMapData.ZCoefficients);

          let {x, z, y} = adjustCoordinatesForMap(bot.XPosition, bot.ZPosition, bot.YPosition);

          addBot(bot.BotId, bot.BotType, x, z, y);
        }
      });

      // Update or delete existing bot markers
      currentBots.forEach(existingBot => {
        if (data.BotLocations?.find(obj => obj.BotId === existingBot.BotId) === undefined) {
          // Remove missing bot
          removeBot(existingBot.BotId);
        } else {
          // Update existing bot
          let foundBot = data.BotLocations?.find(obj => obj.BotId === existingBot.BotId);

          if (foundBot !== undefined) {
            // let x = calculatePolynomialValue(existingBot.XPosition, currentMapData.XCoefficients);
            // let z = calculatePolynomialValue(existingBot.ZPosition, currentMapData.ZCoefficients);

            let {x, z, y} = adjustCoordinatesForMap(foundBot.XPosition, foundBot.ZPosition, foundBot.YPosition);

            // let x = calculatePolynomialValue(foundBot.XPosition, currentMapData.XCoefficients);
            // let z = calculatePolynomialValue(foundBot.ZPosition, currentMapData.ZCoefficients);
            // let y = 1.0;

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
    } else {
      console.error(e);
    }
  }
}

async function questDataFetch() {
  const response = await fetch('http://' + serverAddress.value + '/questData');
  const data = response.json();
  
  console.log(data);
}

function addAirdrop(x: number, z: number) {
  console.log(`addAirdrop: ${x}, ${z}`);

  currentAirdrop = {
    x,
    z
  };
  
  airdropMarker.value?.getGeometry()?.setCoordinates([x, z]);
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
      console.log("BOSS BOT FOUND");
      botMarkerFeature.setStyle(bossBotMarkerStyle);
      break;
    default:
      botMarkerFeature.setStyle(botMarkerStyle);
      break;
  }
  // botMarkerFeature.setStyle(botMarkerStyle);
  
  
  // const source = botMarkerSource.value?.source;
  //
  // source?.addFeature(botMarkerFeature);
  
  botMarkers.value = botMarkers.value.concat(botMarkerFeature);
  
  currentBots.push({
    BotId: id,
    XPosition: x,
    ZPosition: z,
    YPosition: y
  })
  
  console.log(`addBot: ${id}, ${type}, ${x}, ${z}, ${y}`);
}

function removeBot(id: number) {
  console.log(`removeBot: ${id}`);

  // const source = botMarkerSource.value?.source;
  //
  // source?.removeFeature(source.getFeatures().find(botMarker => botMarker.BotId === id));
  
  //currentBots = currentBots.filter(item => item.BotId !== id);
  
  // botMarkers.value = botMarkers.value.filter(botMarker => botMarker.BotId !== id);
  
  const botToRemove = botMarkers.value.find(botMarker => botMarker.getProperties().BotId === id);
  
  console.log(botToRemove);
  
  for (let i = 0; i < currentBots.length; i++) {
    if (currentBots[i].BotId === id) {
      currentBots.splice(i, 1);
    }
  }
  
  const newBotMarkerArray = [];
  
  for (let i = 0; i < botMarkers.value.length; i++) {
    if (botMarkers.value[i].getProperties().BotId !== id) newBotMarkerArray.push(botMarkers.value[i]);
  }
  
  botMarkers.value = newBotMarkerArray;

  // botMarkerSource.value?.source.removeFeature(botToRemove);
  
  // botMarkers.value = botMarkers.value.filter(botMarker => botMarker.BotId !== id);
}

function updateBot(id: number, x: number, z: number, y: number) {
  // console.log(`updateBot: ${id}, ${x}, ${z}, ${y}`);

  const foundBot = botMarkers.value.find(botMarker => botMarker.getProperties().BotId === id);
  
  foundBot?.getGeometry().setCoordinates([x, z]);
}

onMounted(() => {
  changeMap("EnterARaid");
  
  mapDataFetch();
  
  updateTimer = setInterval(mapDataFetch, mapDataUpdateInterval);
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

    <ol-control-bar>
<!--      <ol-toggle-control-->
<!--          html="🟢"-->
<!--          className="edit"-->
<!--          title="Circle"-->
<!--          :onToggle="(active: boolean) => changeDrawType(active, 'Circle')"-->
<!--      />-->
<!--      <ol-toggle-control-->
<!--          html="〰️"-->
<!--          className="edit"-->
<!--          title="Line (double click to finish)"-->
<!--          :onToggle="(active: boolean) => changeDrawType(active, 'LineString')"-->
<!--      />-->
<!--      <ol-toggle-control-->
<!--          html="〰️"-->
<!--          className="edit"-->
<!--          title="Trash"-->
<!--          :onToggle="(active: boolean) => changeDrawType(active, 'LineString')"-->
<!--      />-->
      <ol-zoomtoextent-control 
          label="↕️"
          tipLabel="Full map"
      />
      <ol-toggle-control
          html="B"
          className="edit"
          title="Show Bots"
          :onToggle="(active: boolean) => setShowBots(active)"
      />
      <ol-toggle-control
          html="F"
          className="edit"
          title="Follow Player"
          :onToggle="(active: boolean) => setFollowPlayer(active)"
      />
    </ol-control-bar>

<!--    <ol-interaction-draw-->
<!--        :type="drawType"-->
<!--        @drawend="drawend"-->
<!--        @drawstart="drawstart"-->
<!--    >-->
<!--    </ol-interaction-draw>-->

    <ol-image-layer id="tarkovMap">
      <ol-source-image-static
          :url="mapUrl"
          :imageExtent="mapImageExtent"
          :projection="projection"
      ></ol-source-image-static>
    </ol-image-layer>

<!--    <ol-vector-layer>-->
<!--      <ol-source-vector :projection="projection">-->
<!--        <ol-interaction-draw-->
<!--            v-if="drawEnable"-->
<!--            :type="drawType"-->
<!--            @drawend="drawend"-->
<!--            @drawstart="drawstart"-->
<!--        >-->
<!--          <ol-style>-->
<!--            <ol-style-stroke color="blue" :width="2"></ol-style-stroke>-->
<!--            <ol-style-fill color="rgba(255, 255, 0, 0.4)"></ol-style-fill>-->
<!--          </ol-style>-->
<!--        </ol-interaction-draw>-->
<!--      </ol-source-vector>-->

<!--      <ol-style>-->
<!--        <ol-style-stroke color="red" :width="2"></ol-style-stroke>-->
<!--        <ol-style-fill color="rgba(255,255,255,0.1)"></ol-style-fill>-->
<!--        <ol-style-circle :radius="7">-->
<!--          <ol-style-fill color="red"></ol-style-fill>-->
<!--        </ol-style-circle>-->
<!--      </ol-style>-->
<!--    </ol-vector-layer>-->

    <ol-vector-layer>
      <ol-source-vector :features="[playerMarker]" />
      
<!--      <ol-style>-->
<!--        <ol-style-icon :src="playerIcon" :scale="3"></ol-style-icon>-->
<!--      </ol-style>-->
    </ol-vector-layer>

    <ol-vector-layer>
      <ol-source-vector :features="[airdropMarker]" />

<!--      <ol-style>-->
<!--        <ol-style-icon :src="airdropIcon" :scale="3"></ol-style-icon>-->
<!--      </ol-style>-->
    </ol-vector-layer>

    <ol-vector-layer ref="botMarkerLayer">
      <ol-source-vector ref="botMarkerSource" :features="botMarkers" />

<!--      <ol-style>-->
<!--        <ol-style-icon :src="botIcon" :scale="3"></ol-style-icon>-->
<!--      </ol-style>-->
    </ol-vector-layer>

    <ol-vector-layer ref="questMarkerLayer">
      <ol-source-vector ref="questMarkerSource" :features="questMarkers" />

      <!--      <ol-style>-->
      <!--        <ol-style-icon :src="botIcon" :scale="3"></ol-style-icon>-->
      <!--      </ol-style>-->
    </ol-vector-layer>

    <ol-interaction-select
        @select="featureSelected"
        :condition="selectCondition"
        :filter="selectInteactionFilter"
    >
      <ol-style>
        <ol-style-stroke color="green" :width="10"></ol-style-stroke>
        <ol-style-fill color="rgba(255,255,255,0.5)"></ol-style-fill>
        <ol-style-circle :radius="7">
          <ol-style-fill color="blue"></ol-style-fill>
        </ol-style-circle>
      </ol-style>
    </ol-interaction-select>
  </ol-map>
  
  <div
      v-if="shouldShowError"
      class="connection-error"
  >
    <span>Failed to connect to mod's server!</span>
    <span>Make sure the game is running and firewall is not blocking it.</span>
    <span>Then refresh the page.</span>
  </div>
</template>

<style scoped>

</style>