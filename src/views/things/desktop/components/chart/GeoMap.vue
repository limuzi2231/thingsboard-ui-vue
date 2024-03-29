<template>
  <div id="bMap-container" :style="{ height, width }"></div>
</template>
<script lang="ts" setup>
import { isArray } from 'lodash';
import { useBMap } from '/@/hooks/web/useBMap.js';
import { ref, watchEffect, onMounted } from 'vue';
import styleJson from "./bmapStyle.json";
import { useWebsocketStore } from '/@/store/modules/websocket';
import { EntityType } from '/@/enums/entityTypeEnum';
import { onUnmounted } from 'vue';
import dayjs from 'dayjs';
import { watch } from 'vue';

const props = defineProps({
  width: {
    type: String as PropType<string>,
    default: '100%',
  },
  height: {
    type: String as PropType<string>,
    default: '400px',
  },
  center: {
    type: [Array, String] as PropType<[number, number] | string>,
    default: '潍坊市',
  },
  zoom: { type: Number, default: 8 },
});


const DEVICE_POSITION_COUNT_CMD_ID = ref(0);
const DEVICE_POSITION2_COUNT_CMD_ID = ref(0);

const positionDeviceList = ref<Array<any>>([]);

let mapInstance: any = undefined



const { getAndIncrementCmdId, send, unsubscribe } = useWebsocketStore();
const { success, BMapGL } = useBMap("fyfI8y4JKYWxnOA1lIW4VYJFfuR2JKanf0I")


watchEffect(() => {
  if (success.value == true) {
    initBMap();
    sendQueryPacket();
  }
})

watch(
  () => props.center,
  () => {
    let centerPoint = props.center;
    if (isArray(props.center)) {
      centerPoint = new BMapGL.value.Point(props.center[0], props.center[1])
    }
    mapInstance.centerAndZoom(centerPoint, props.zoom);
  }
)

function initBMap() {
  if (BMapGL.value) {
    mapInstance = new BMapGL.value.Map('bMap-container', {
      enableRotate: false,
      enableTilt: false
    }); // 创建Map实例
    let centerPoint = props.center;
    if (isArray(props.center)) {
      centerPoint = new BMapGL.value.Point(props.center[0], props.center[1])
    }
    if (mapInstance) {
      mapInstance.centerAndZoom(centerPoint, props.zoom);
      mapInstance.enableScrollWheelZoom(true); // 开启鼠标滚轮缩放
      mapInstance.setMapStyleV2({ styleJson: styleJson })
    }

  }
}

function sendQueryPacket() {
  positionDeviceList.value.length = 0;
  send(
    [DEVICE_POSITION_COUNT_CMD_ID.value, DEVICE_POSITION2_COUNT_CMD_ID.value],
    {
      entityDataCmds: [
        {
          cmdId: DEVICE_POSITION_COUNT_CMD_ID.value,
          query: {
            entityFilter: { type: "entityType", entityType: EntityType.DEVICE, resolveMultiple: true },
            keyFilters: [{
              valueType: "NUMERIC",
              key: { key: "LATITUDE", type: "ATTRIBUTE" },
              predicate: { operation: "GREATER", type: "NUMERIC", value: { defaultValue: 0 } }
            }],
            entityFields: [{ type: "ENTITY_FIELD", key: "name" }, { type: "ENTITY_FIELD", key: "label" }],
            pageLink: { page: 0, pageSize: 1024, sortOrder: { direction: 'DESC', key: { type: 'ENTITY_FIELD', key: 'createdTime' } } },
            latestValues: [
              { type: 'ATTRIBUTE', key: 'LATITUDE' },
              { type: 'ATTRIBUTE', key: 'LONGITUDE' },
              { type: 'ATTRIBUTE', key: 'IMSI' },
              { type: 'ATTRIBUTE', key: 'CSQ' },
              { type: 'ATTRIBUTE', key: 'active' },
              { type: 'ATTRIBUTE', key: 'lastActivityTime' },
            ]
          }
        }, {
          cmdId: DEVICE_POSITION2_COUNT_CMD_ID.value,
          query: {
            entityFilter: { type: "entityType", entityType: EntityType.DEVICE, resolveMultiple: true },
            keyFilters: [{
              valueType: "NUMERIC",
              key: { key: "latitude", type: "ATTRIBUTE" },
              predicate: { operation: "GREATER", type: "NUMERIC", value: { defaultValue: 0 } }
            }],
            entityFields: [{ type: "ENTITY_FIELD", key: "name" }, { type: "ENTITY_FIELD", key: "label" }],
            pageLink: { page: 0, pageSize: 1024, sortOrder: { direction: 'DESC', key: { type: 'ENTITY_FIELD', key: 'createdTime' } } },
            latestValues: [
              { type: 'ATTRIBUTE', key: 'latitude' },
              { type: 'ATTRIBUTE', key: 'longitude' },
              { type: 'ATTRIBUTE', key: 'sim_code' },
              { type: 'ATTRIBUTE', key: 'active' },
              { type: 'ATTRIBUTE', key: 'lastActivityTime' },
            ]
          }
        }
      ]
    },
    onMessage
  )
}

function onMessage(data: any) {

  if (DEVICE_POSITION_COUNT_CMD_ID.value == data.cmdId) {
    positionDeviceList.value.push(...data.data.data.map(item => (
      {
        name: item.latest.ENTITY_FIELD.name.value,
        label: item.latest.ENTITY_FIELD.label.value,
        active: item.latest.ATTRIBUTE.active.value == "true",
        lastActivityTime: Number.parseInt(item.latest.ATTRIBUTE.lastActivityTime.value),
        latitude: Number.parseFloat(item.latest.ATTRIBUTE.LATITUDE.value),
        longitude: Number.parseFloat(item.latest.ATTRIBUTE.LONGITUDE.value),
        sim: Number.parseInt(item.latest.ATTRIBUTE.IMSI.value || '0'),
        csq: Number.parseInt(item.latest.ATTRIBUTE.CSQ.value || '0'),
      }
    )));
  } else if (DEVICE_POSITION2_COUNT_CMD_ID.value == data.cmdId) {
    positionDeviceList.value.push(...data.data.data.map(item => (
      {
        name: item.latest.ENTITY_FIELD.name.value,
        label: item.latest.ENTITY_FIELD.label.value,
        active: item.latest.ATTRIBUTE.active.value == "true",
        lastActivityTime: Number.parseInt(item.latest.ATTRIBUTE.lastActivityTime.value),
        latitude: Number.parseFloat(item.latest.ATTRIBUTE.latitude.value),
        longitude: Number.parseFloat(item.latest.ATTRIBUTE.longitude.value),
        sim: Number.parseInt(item.latest.ATTRIBUTE.sim_code.value || '0'),
      }
    )));
  }
  renderPosition();
}

function renderPosition() {
  if (mapInstance && BMapGL.value) {
    mapInstance.clearOverlays();
    positionDeviceList.value.forEach(item => {
      const point = new BMapGL.value.Point(item.longitude, item.latitude);
      const marker = new BMapGL.value.Marker(point);
      let info =
        `<div style="display:flex;justify-content:space-between;margin:0 6px 0 1px"><strong>标签：</strong><span>${item.label}</span></div>`
        + `<div style="display:flex;justify-content:space-between;margin:0 6px 0 1px"><strong>状态：</strong><span style="color:${item.active == true ? 'green' : 'red'}">${item.active == true ? '在线' : '离线'}</span></div>`;
      if (item.sim) {
        info = info + `<div style="display:flex;justify-content:space-between;margin:0 6px 0 1px"><strong>SIM：</strong><span>${item.sim}</span></div>`;
      }
      if (item.csq) {
        info = info + `<div style="display:flex;justify-content:space-between;margin:0 6px 0 1px"><strong>信号质量：</strong><span>${getCSQDesc(item.csq)}(${item.csq})</span></div>`;
      }
      info = info + `<div style="display:flex;justify-content:space-between;margin:0 6px 0 1px"><strong>活动时间：</strong><span>${dayjs(item.lastActivityTime).format('YYYY-MM-DD HH:mm:ss')}</span></div>`;
      const infoWindow = new BMapGL.value.InfoWindow(info, { width: 240, height: 130, title: `<strong>${item.name}</strong>` });
      marker.addEventListener('click', function () {
        mapInstance.openInfoWindow(infoWindow, point); // 开启信息窗口
      });
      mapInstance.addOverlay(marker);
    })
  }
}



onMounted(() => {
  DEVICE_POSITION_COUNT_CMD_ID.value = getAndIncrementCmdId();
  DEVICE_POSITION2_COUNT_CMD_ID.value = getAndIncrementCmdId();

})

onUnmounted(() => {
  unsubscribeDataCmds();
})


function unsubscribeDataCmds() {
  if (DEVICE_POSITION_COUNT_CMD_ID.value > 0) {
    unsubscribe(
      DEVICE_POSITION_COUNT_CMD_ID.value,
      { entityDataUnsubscribeCmds: [{ cmdId: DEVICE_POSITION_COUNT_CMD_ID.value }] }
    )
  }
}

function getCSQDesc(csq: number) {
  if (csq == 99) {
    return '无信号'
  }
  if (csq < 11) {
    return '信号弱'
  } else if (csq < 18) {
    return '信号良'
  } else if (csq < 31) {
    return '信号优'
  } else {
    return '错误';
  }

}

</script>