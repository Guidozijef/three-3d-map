<template>
  <div class="page" id="page" ref="pageRef">
    <div class="tooltip" ref="tipRef" v-show="show">
      {{ selectedPointData.name }}
    </div>
  </div>
</template>

<script setup>
import { onMounted, ref } from "vue";
import * as THREE from "three";
import * as d3 from "d3";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { RenderPass } from "three/examples/jsm/postprocessing/RenderPass.js";
import { UnrealBloomPass } from "three/examples/jsm/postprocessing/UnrealBloomPass.js";
import { OutlinePass } from "three/examples/jsm/postprocessing/OutlinePass.js";
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer.js";

const pageRef = ref();
const tipRef = ref();

let map = null;
let scene = null;
let camera = null;
let renderer = null;
let controls = null;
let centerCoordinate = [104.06, 30.66]; // 地图中心地理坐标
let projection = null; // Mercator 投影
let mapConfig = {
  deep: 0.2, // 挤出的深度
};
let boundaryLineArr = []; // 边界线
let composer = ""; // 后期处理
let pointData = [
  {
    coordinates: [117.33, 31.79],
    type: 1,
    name: "合肥",
    value: 100,
  },
  {
    coordinates: [118.502, 31.684],
    type: 1,
    name: "马鞍山",
    value: 100,
  },
];
let pointInstanceArr = []; // 坐标点实例
let show = false; // 是否显示tooltip
let selectedPointData = {}; // 选中的坐标点数据
onMounted(() => {
  init();
  window.addEventListener("resize", () => {
    renderer.setSize(window.innerWidth, window.innerHeight);
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
  });
});

function init() {
  renderer = new THREE.WebGLRenderer();
  renderer.setSize(window.innerWidth, window.innerHeight);
  renderer.outputColorSpace = THREE.LinearSRGBColorSpace;
  document.querySelector("#page").appendChild(renderer.domElement);

  scene = new THREE.Scene();
  camera = new THREE.PerspectiveCamera(45, window.innerWidth / window.innerHeight, 0.1, 1000);
  camera.position.set(5, 5, 26);
  camera.lookAt(0, 0, 0);

  // let axesHelp = new THREE.AxesHelper(5);
  // scene.add(axesHelp);

  controls = new OrbitControls(camera, renderer.domElement);

  // 墨卡托投影转换
  projection = d3.geoMercator().center(centerCoordinate).translate([0, 0]); // 根据地球贴图做轻微调整
  // 添加地图
  addMap();
  // 给地图边界线添加outline效果
  setLineOutline();

  // 添加灯光
  let ambientLight = new THREE.AmbientLight(0xffffff, 1);
  scene.add(ambientLight);

  // 添加散点
  setPoint();
  // 设置光线投射
  setRaycaster();

  render();
}
function render() {
  renderer.render(scene, camera);
  controls.update();
  if (composer) composer.render();
  requestAnimationFrame(render);
}
// 添加地图
function addMap() {
  // 加载地图背景
  const backgroundTexture = new THREE.TextureLoader().load("./img/gz-map1.png");
  // 加载地图
  let fileLoader = new THREE.FileLoader();
  fileLoader.load("https://geo.datav.aliyun.com/areas_v3/bound/510100_full.json", (data) => {
    // 添加地图及边界线
    addMapGeometry(data);
    // 重新计算地图uv坐标
    let arr = [];
    let box = new THREE.Box3();
    for (let v of map.children) {
      for (let v2 of v.children) {
        // 判断是否为ExtrudeGeometry，只计算所有地图区域总和的包围盒大小
        if (v2.geometry instanceof THREE.ExtrudeGeometry) {
          arr.push(v2);
          let itemBox = new THREE.Box3().setFromObject(v2);
          box.union(itemBox);
        }
      }
    }
    var bboxMin = box.min;
    var bboxMax = box.max;
    // 计算UV的缩放比例
    var uvScale = new THREE.Vector2(1 / (bboxMax.x - bboxMin.x), 1 / (bboxMax.y - bboxMin.y));
    for (let v of arr) {
      let uvAttribute = v.geometry.getAttribute("uv");
      for (let i = 0; i < uvAttribute.count; i++) {
        let u = uvAttribute.getX(i);
        let v = uvAttribute.getY(i);
        // 将UV坐标进行归一化
        let normalizedU = (u - bboxMin.x) * uvScale.x;
        let normalizedV = (v - bboxMin.y) * uvScale.y;
        // 更新UV坐标
        uvAttribute.setXY(i, normalizedU, normalizedV);
      }
      // 更新几何体的UV属性
      v.geometry.setAttribute("uv", uvAttribute);
      v.material.map = backgroundTexture;
      v.material.needsUpdate = true;
    }
  });
}
function addMapGeometry(jsondata) {
  // 初始化一个地图对象
  map = new THREE.Object3D();
  jsondata = JSON.parse(jsondata);
  jsondata.features.forEach((elem) => {
    // 定一个省份3D对象
    const province = new THREE.Object3D();
    // 每个的 坐标 数组
    const coordinates = elem.geometry.coordinates;

    if (elem.geometry.type === "MultiPolygon") {
      // 循环坐标数组
      coordinates.forEach((multiPolygon) => {
        multiPolygon.forEach((polygon) => {
          drawItem(elem, polygon, province);
        });
      });
      map.add(province);
    } else if (elem.geometry.type === "Polygon") {
      // 循环坐标数组
      coordinates.forEach((polygon) => {
        drawItem(elem, polygon, province);
      });
      map.add(province);
    }
  });
  scene.add(map);
}
function drawItem(elem, polygon, province) {
  const shape = new THREE.Shape();
  const pointsArray = new Array();
  for (let i = 0; i < polygon.length; i++) {
    const [x, y] = projection(polygon[i]);
    if (i === 0) {
      shape.moveTo(x, -y);
    }
    shape.lineTo(x, -y);
    pointsArray.push(new THREE.Vector3(x, -y, mapConfig.deep));
  }
  let curve = new THREE.CatmullRomCurve3(pointsArray);
  // 这里使用TubeGeometry没有使用line，主要考虑到line的宽度无法设置，也可以使用其他第三方依赖去做
  var tubeGeometry = new THREE.TubeGeometry(curve, Math.floor(pointsArray.length), 0.02, 10);

  const extrudeSettings = {
    depth: mapConfig.deep,
    bevelEnabled: false, // 对挤出的形状应用是否斜角
  };
  const geometry = new THREE.ExtrudeGeometry(shape, extrudeSettings);
  geometry.computeBoundingBox();
  // 创建地图区域材质
  let meshMaterial = new THREE.MeshStandardMaterial({
    color: "#ffffff",
    transparent: true,
    opacity: 1,
  });
  // 创建地图边界线材质
  let lineMaterial = new THREE.MeshBasicMaterial({
    color: "#ceebf7",
  });

  const mesh = new THREE.Mesh(geometry, meshMaterial);
  const line = new THREE.Mesh(tubeGeometry, lineMaterial);
  // 将省份的属性 加进来
  province.properties = elem.properties;
  province.add(mesh);
  boundaryLineArr.push(line);
  province.add(line);
}
// 给地图边界线添加outline效果
function setLineOutline() {
  //设置光晕
  composer = new EffectComposer(renderer); //效果组合器
  //创建通道
  let renderScene = new RenderPass(scene, camera);
  composer.addPass(renderScene);

  let outlinePass = new OutlinePass(new THREE.Vector2(window.innerWidth, window.innerHeight), scene, camera, boundaryLineArr);
  outlinePass.renderToScreen = true;
  outlinePass.edgeGlow = 2; // 光晕效果
  outlinePass.usePatternTexture = false;
  outlinePass.edgeThickness = 10; // 边框宽度
  outlinePass.edgeStrength = 1.5; // 光晕效果
  outlinePass.pulsePeriod = 0; // 光晕闪烁的速度
  outlinePass.visibleEdgeColor.set("#1acdec");
  outlinePass.hiddenEdgeColor.set("#1acdec");
  composer.addPass(outlinePass);
}
// 添加散点
function setPoint() {
  let pointTexture = new THREE.TextureLoader().load("./img/point.png");
  for (let v of pointData) {
    let [x, y] = projection(v.coordinates);
    const sprite = new THREE.Sprite(
      new THREE.SpriteMaterial({
        map: pointTexture,
      })
    );
    sprite.scale.set(0.7, 0.7, 1);
    sprite.position.set(x, -y, mapConfig.deep + 0.5);
    sprite.properties = v;
    pointInstanceArr.push(sprite);
    scene.add(sprite);
  }
}
// 光线投射
function setRaycaster() {
  const raycaster = new THREE.Raycaster();
  const pointer = new THREE.Vector2();
  pageRef.value.addEventListener("click", (event) => {
    pointer.x = (event.clientX / window.innerWidth) * 2 - 1;
    pointer.y = -(event.clientY / window.innerHeight) * 2 + 1;

    raycaster.setFromCamera(pointer, camera);
    const intersects = raycaster.intersectObjects(pointInstanceArr);
    if (intersects && intersects.length > 0) {
      let tooltip = tipRef.value.tooltip;
      tooltip.style.left = event.pageX + "px";
      tooltip.style.top = event.pageY + "px";
      selectedPointData = intersects[0].object.properties;
      show = true;
    } else {
      selectedPointData = {};
      show = false;
    }
  });
}
</script>
<style scoped>
.page {
  height: 100vh;
}
.page .tooltip {
  position: absolute;
  background-color: #fff;
  padding: 10px;
  border-radius: 8px;
}
</style>
