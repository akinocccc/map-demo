<script lang="ts" setup>
// Base import 
import { onMounted, shallowRef } from "vue";
import AMapLoader from "@amap/amap-jsapi-loader";

// other import
import path from "../../mock/gpsData";

/** 手动引入图片，防止构建时所引用的图片资源没有被打包 */
import carImg from "@/assets/car.png";
import locationImg from "@/assets/location.png";

// Variables

const mapInstance = shallowRef<any>(null);

// Methods

/**
 * @description 判断区间
 * @param origig
 * @param next
 */
const sectionJudge = (origin: number[], next: number[]): number => {
  const x1 = origin[0];
  const y1 = origin[1];
  const x2 = next[0];
  const y2 = next[1];
  if (x2 > x1 && y2 >= y1) {
    return 1;
  }
  if (x2 >= x1 && y2 < y1) {
    return 2;
  }
  if (x2 < x1 && y2 <= y1) {
    return 3;
  }
  if (x2 <= x1 && y2 > y1) {
    return 4;
  }
  return 5;
}

/**
 * @description 判断行驶方向
 * @param origin
 * @param next
 */
const computeDirection = (origin: number[], next: number[]) => {
  const disX: number = Math.abs(origin[0] - next[0]);
  const dixY: number = Math.abs(origin[1] - next[1]);
  const tan: number = Math.tan(dixY / disX);
  const result: number = Math.atan(tan) / (Math.PI / 180);
  let rotateDeg: number = Math.round(result);

  const section: number = sectionJudge(origin, next);
  if (section === 1) {
    rotateDeg = 90 - rotateDeg;
  } else if (section === 2) {
    rotateDeg = 90 + rotateDeg;
  } else if (section === 3) {
    rotateDeg = 180 + (90 - rotateDeg);
  } else if (section === 4) {
    rotateDeg = 270 + rotateDeg;
  }
  return rotateDeg;
}

/**
 * @description 初始化地图
 */
const initMap = () => {
  AMapLoader.load({
    key: "", // 高德官网申请Web端开发者Key，首次调用 load 时必填
    version: "2.0" // 指定要加载的 JSAPI 的版本，缺省时默认为 1.4.15
  })
    .then((AMap: any) => {
      // 初始化地图实例
      mapInstance.value = new AMap.Map("map-container", {
        viewMode: "2D", // 地图模式: 2D|3D
        zoom: 7, // 初始化地图级别
        center: path[0], // 设置地图中心
      });

      // 设置地图图层，可不设
      const trafficLayer = new AMap.TileLayer.Traffic({
        zIndex: 10,
      });
      trafficLayer.setMap(mapInstance.value);

      // 初始化信息窗口
      const infoWindow = new AMap.InfoWindow({
        autoMove: true,
      });

      // 添加插件 
      AMap.plugin(["AMap.ToolBar", "AMap.Scale", "AMap.HawkEye", "AMap.MoveAnimation"], () => {
        // 异步同时加载多个插件
        mapInstance.value?.addControl(new AMap.HawkEye()); // 显示缩略图
        mapInstance.value?.addControl(new AMap.Scale()); // 显示当前地图中心的比例尺

        const len = path.length;

        // 初始化marker
        const markers: any[] = [
          new AMap.Marker({
            position: path[0],
            offset: [-15, -20],
            content: `
                <div style="display: flex; flex-direction: column; justify-content: center; align-items: center;">
                  <img style="width: 30px; height: 30px" src="${locationImg}" />
                  <div style="font-weight: 700; font-size: 14px">望京</div>
                </div>
              `
          })
        ];
        const carMarker = new AMap.Marker({
          position: path[len - 1], // marker位置
          icon: carImg, // marker图标
          offset: new AMap.Pixel(-12, -18), //偏移值
          autoRotation: false,
          angle: computeDirection(path[len - 2], path[len - 1]),
        });
        // 设置marker点击事件
        carMarker.on("click", () => {
          infoWindow.setContent(`
                <div style='padding: 0 5px'>
                  <h4 style='margin: 0'> 订单信息 </h4>
                  <div style='margin-top: 10px'>姓名：张三</div>
                  <div style='margin-top: 10px'>手机：12345678901</div>
                  <div style='margin-top: 10px'>起点：望京</div>
                  <div style='margin-top: 10px'>终点：金辉大厦</div>
                </div>
          `);
          infoWindow.open(mapInstance.value, path[len - 1]);
        })
        markers.push(carMarker);

        // 初始化轨迹
        const polylines: any[] = [
          new AMap.Polyline({
            path, //轨迹经纬度
            showDir: true,
            strokeColor: "#28F", //线颜色
            strokeOpacity: 1, //线透明度
            strokeWeight: 6, //线宽
            strokeStyle: "solid" //线样式
          })
        ];

        // 插入地图
        mapInstance.value?.add(markers);
        mapInstance.value?.add(polylines);
        mapInstance.value?.setFitView(); // 自适应，缩放至所有marker、polyline全部显示
      });
    })
    .catch((e) => {
      console.log(e);
    })
}

onMounted(() => {
  initMap();
});
</script>

<template>
  <div id="map-container" />
</template>

<style scoped>
#map-container {
  width: 100vw;
  height: 100vh;
}
</style>
