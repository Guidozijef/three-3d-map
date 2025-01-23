<template>
  <main style="position: relative">
    <div class="box-3d" id="webgl"></div>
    <div v-show="visible" class="tooltip" ref="tipRef" id="tooltip">
      <span @click="closeTip" class="close">×</span>
      <p></p>
    </div>
  </main>
</template>

<script setup>
import { onMounted, ref } from "vue";
import * as THREE from "three";
// 引入 MeshLine 库
import { MeshLine, MeshLineMaterial } from "three.meshline";
import * as d3 from "d3";
// 引入轨道控制器扩展库OrbitControls.js
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
import { CSS2DRenderer, CSS2DObject } from "three/addons/renderers/CSS2DRenderer.js";
import { GUI } from "three/addons/libs/lil-gui.module.min.js";

// 墨卡托投影转换
const projection = d3.geoMercator();
let textureLoader = new THREE.TextureLoader(); //纹理贴图加载器

const centerPos = [104.25494, 30.883438]; // 地图中心经纬度坐标centerPos: [114.255221, 30.619014], // 地图中心经纬度坐标

const WaveMeshArr = []; // 波纹数组
const particleArr = []; // 飞升粒子集合
let mapModel = null; // 地图模型
let earthModel = null; // 地球模型
const pointInstanceArr = []; // 点击点位监测集合
const linePartList = []; // 飞线集合

let rotatingApertureMesh = null; //底部旋转的光圈
let rotatingPointMesh = null; //底部旋转的点

let material;
let material1;
let textureMap;
let texturefxMap;

// 创建3D场景对象Scene
const scene = new THREE.Scene();

const width = window.innerWidth; //宽度
const height = window.innerHeight; //高度

// 30:视场角度, width / height:Canvas画布宽高比, 1:近裁截面, 3000：远裁截面

const camera = new THREE.PerspectiveCamera(45, width / height, 1, 10000);
camera.up.set(0, 0, 1); // 相机以那个轴向上
camera.position.set(0, 0, 1000); // 确保相机在地球模型的正前方
camera.lookAt(0, 0, 0); // 确保相机指向地球中心

// 创建渲染器对象
const renderer = new THREE.WebGLRenderer({ antialias: true });

// 获取你屏幕对应的设备像素比.devicePixelRatio告诉threejs,以免渲染模糊问题
renderer.setPixelRatio(window.devicePixelRatio);

// 定义threejs输出画布的尺寸(单位:像素px)
renderer.setSize(width, height); //设置three.js渲染区域的尺寸(像素px)

renderer.setClearColor(0x000000, 1); //设置背景颜色

// 创建CSS2DRenderer渲染器(替鼠标射线检)
const labelRenderer = new CSS2DRenderer();
// 设置labelRenderer渲染器宽高
labelRenderer.setSize(width, height);

labelRenderer.domElement.style.position = "absolute";
labelRenderer.domElement.style.left = "0px";
labelRenderer.domElement.style.top = "0px";
labelRenderer.domElement.style.pointerEvents = "none";

//环境光:没有特定方向，整体改变场景的光照明暗
const ambient = new THREE.AmbientLight(0xffffff, 1.5);
//   平行光1
let directionalLight1 = new THREE.DirectionalLight(0x7af4ff, 2);
directionalLight1.position.set(...projection(centerPos), 30);
//   平行光2
let directionalLight2 = new THREE.DirectionalLight(0x7af4ff, 2);
directionalLight2.position.set(...projection(centerPos), 30);

scene.add(ambient);
scene.add(directionalLight1);
scene.add(directionalLight2);

// 增加平行光
let directionalLight = new THREE.DirectionalLight(0xffffff, 1);
directionalLight.position.set(0, 0, 1000).normalize();
scene.add(directionalLight);

// 设置相机控件轨道控制器OrbitControls
const controls = new OrbitControls(camera, renderer.domElement);
controls.enableDamping = true; //阻尼感关
controls.maxPolarAngle = Math.PI / 2;
// controls.autoRotate = false;
// controls.minPolarAngle = Math.PI / 2;
// controls.maxAzimuthAngle = 1.5;
// controls.minAzimuthAngle = -2;

// 如果OrbitControls改变了相机参数，重新调用渲染器渲染三维场景
controls.addEventListener("change", function () {
  renderMap();
  // 浏览器控制台查机位置变化
  // console.log("camera.position", camera.position);
}); //监听鼠标、键盘事件

// 创建辅助相机和坐标系
// initCameraHelper();

// 创建地图外观材质
initMapMaterial();

initGui();

generateGeometry();

onMounted(() => {
  const container = document.getElementById("webgl");
  container.appendChild(renderer.domElement);
  container.appendChild(labelRenderer.domElement);

  // 禁用控制器
  controls.enabled = false;

  // 开始执行入场动画
  startEntranceAnimation();

  window.onresize = function () {
    renderer.setSize(window.innerWidth, window.innerHeight);
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderMap();
  };
});

function renderMap() {
  renderer.render(scene, camera); //执行渲染操作
  labelRenderer.render(scene, camera);
}

//绘制地图
function generateGeometry() {
  // 初始化一个地图对象
  mapModel = new THREE.Group();
  // TODO: 将centerPos转换为投影坐标并设置地图中心点
  projection.center(centerPos).scale(200).translate([0, 0]);
  // const url = "https://geo.datav.aliyun.com/areas_v3/bound/geojson?code=510113"; //成都市青白江区
  // const url = "https://geo.datav.aliyun.com/areas_v3/bound/510100_full.json"; //成都市
  const url = "/data/510113.geojson";
  // 拉伸 地图厚度
  const depth = 0.05;

  fetch(url)
    .then((response) => response.json())
    .then((data) => {
      const features = data.features;
      features.forEach((feature) => {
        const { geometry, properties } = feature;
        const coordinates = geometry.coordinates;
        const province = new THREE.Object3D();
        province.name = properties.name;

        //这里创建光柱、文字坐标
        properties.center = d3.geoCentroid(geometry);
        const [light, label] = initLightPoint(properties, depth);

        // 循环坐标数组
        coordinates.forEach((coordinate) => {
          coordinate.forEach((polygon) => {
            // 创建地图区块
            const mesh = createMesh(polygon, depth);

            // 创建地图区块边界线
            const line = createLine(polygon, depth);

            province.add(mesh);
            province.add(line);
          });
        });
        mapModel.add(province, light, label);
      });
      initPoints();
      // initBatchPoints();
      createFlylines(features, depth);
      // TODO: 如果要加在地图模型上，必须在缩放之前添加进去
      mapModel.scale.set(200, 200, 200);
      setMapCenter(mapModel);
      initSceneBg();
      initCirclePoint(mapModel);
      initRotatingAperture(mapModel);
      initRotatingPoint(mapModel);
      initParticle(mapModel);
      scene.add(mapModel); //将地图对象添加到场景中
      renderMap();
    });
}

/**
 * @description 创建光柱、文字坐标
 * @param {*} properties 属性、详情
 * @param {*} depth  高度
 */
function initLightPoint(properties, depth) {
  const { name } = properties;
  let lightCenter = properties.centroid || properties.center;
  // projection用来把经纬度转换成坐标
  const [x, y] = projection(lightCenter);

  // 随机创建光柱高度
  let heightScaleFactor = 2 + random(1, 5) / 5;
  // 创建光柱
  const light = createLightPillar(x, y, heightScaleFactor * 0.07, depth);
  // 创建文字标注
  const label = createTextPoint(x, y, name, depth);

  return [light, label];
}

/**
 * 两点链接飞线
 * */
function createFlylines(features, depth) {
  const group = new THREE.Group();
  const { center } = getBoundingBox(mapModel);
  features.forEach((feature, i) => {
    const { properties, geometry } = feature;
    properties.center = d3.geoCentroid(geometry);
    const [x, y] = projection(properties.center);
    const { line, linePart } = createFlyLine([center.x, center.y], [x, -y], depth, i);
    linePart.renderOrder = 80;
    line.add(linePart);
    group.add(line);
    linePartList.push(linePart);
  });
  mapModel.add(group);

  function createFlyLine(posStart, posEnd, depth, i) {
    // 根据坐标设置3D坐标  z轴位置在地图表面
    const [x0, y0, z0] = [...posStart, depth];
    const [x1, y1, z1] = [...posEnd, depth];
    // 定义飞线的起点和终点
    const startPoint = new THREE.Vector3(x0, y0, z0);
    const endPoint = new THREE.Vector3(x1, y1, z1);
    const centerPoint = new THREE.Vector3((x0 + x1) / 2, (y0 + y1) / 2, z1 + 0.1);

    // 创建 CatmullRomCurve3 曲线，包含起点、控制点和终点
    const curve = new THREE.CatmullRomCurve3([startPoint, centerPoint, endPoint]);

    // 获取曲线上的点
    const curvePoints = curve.getPoints(50); // 获取50个点

    // 将 `THREE.Vector3` 数组转换为平面数组
    const pointsArray = curvePoints.map((m) => [m.x, m.y, m.z]).flat();

    // 创建 MeshLine 实例
    const meshLine = new MeshLine();
    meshLine.setPoints(pointsArray);

    // 创建 MeshLine 材质，设置线条的粗细
    const material = new MeshLineMaterial({
      color: new THREE.Color(0xb4eeea), // 飞线颜色
      lineWidth: 0.5, // 设置飞线宽度
      resolution: new THREE.Vector2(width, height), // 渲染器分辨率
      side: THREE.DoubleSide, // 双面渲染
      opacity: 0.5, // 设置透明度
      transparent: true, // 启用透明度
    });

    // 创建飞线
    const line = new THREE.Mesh(meshLine.geometry, material);
    line.name = "飞线";

    // 创建几何体并设置为这些曲线点
    const geometryPart = new THREE.BufferGeometry().setFromPoints(curvePoints.slice(i, i + 5));

    // 创建线的材质
    const materialPart = new THREE.LineBasicMaterial({ color: 0xff0000, linewidth: 10 });

    // 创建飞线
    const linePart = new THREE.Line(geometryPart, materialPart);
    linePart.name = "一段飞线";
    linePart.curvePoints = curvePoints;
    linePart.progress = Math.random();

    return { line, linePart };
  }
}

/**
 * @description // 创建光柱
 * @param {*} x d3 - 经纬度转换后的x轴坐标
 * @param {*} y d3 - 经纬度转换后的z轴坐标
 * @param {*} heightScaleFactor
 *
 * @param {*} depth 地图厚度
 */
function createLightPillar(x, y, height = 1, depth) {
  const group = new THREE.Group();
  // 柱体的geo,6.19=柱体片高度/宽度的倍数
  const geometry = new THREE.PlaneGeometry(height / 6.219, height);

  // 柱体旋转90度垂直于Y轴
  geometry.rotateX(Math.PI / 2);
  // 柱体z轴动高度一半对齐中心点
  geometry.translate(0, 0, height / 2);
  // 柱子材质
  const material = new THREE.MeshBasicMaterial({
    map: textureLoader.load("./img/光柱.png"),
    color: 0x00ffff,
    transparent: true,
    depthWrite: false,
    // depthTest:false,
    side: THREE.DoubleSide,
  });
  // 光柱01
  let light01 = new THREE.Mesh(geometry, material);
  light01.renderOrder = 2;
  light01.name = "createLightPillar01";
  // 光柱02：复制光柱01
  let light02 = light01.clone();
  light02.renderOrder = 2;
  light02.name = "createLightPillar02";
  // 光柱02，旋转90°，跟 光柱01交叉
  light02.rotateZ(Math.PI / 2);

  // 创建底部标点
  const bottomMesh = createPointMesh(0.5);
  // 创建光圈
  const lightHalo = createLightHalo(0.5);

  WaveMeshArr.push(lightHalo);
  group.add(light01, light02, bottomMesh, lightHalo);
  group.position.set(x, -y, depth + 0.001);
  return group;
}

/**
 * @description 创建底部标点
 * @param {number} size 缩放大小
 */
function createPointMesh(size) {
  // 标记点：几何体，材质，
  const geometry = new THREE.PlaneGeometry(0.05, 0.05);
  const material = new THREE.MeshBasicMaterial({
    map: textureLoader.load("./img/标注.png"),
    color: 0x00ffff,
    side: THREE.DoubleSide,
    transparent: true,
    depthWrite: false, //禁止写深度缓冲区数据
  });
  let mesh = new THREE.Mesh(geometry, material);
  mesh.renderOrder = 2;
  //   mesh.position.set(0, 0, 0); // 设置位置
  //mesh.rotation.x = Math.PI / 2;
  mesh.name = "createPointMesh";
  // 缩放
  const scale = 1 * size;
  mesh.scale.set(scale, scale, scale);
  return mesh;
}

/**
 * @description 创建底部标点的光圈
 * @param {number} size 缩放大小
 */
function createLightHalo(size) {
  // 标记点：几何体，材质，
  const geometry = new THREE.PlaneGeometry(0.3, 0.3);
  const material = new THREE.MeshBasicMaterial({
    map: textureLoader.load("./img/标注光圈.png"),
    color: 0x00ffff,
    side: THREE.DoubleSide,
    opacity: 0,
    transparent: true,
    depthWrite: false, //禁止写入深度缓冲区数据
  });
  let mesh = new THREE.Mesh(geometry, material);
  mesh.renderOrder = 2;
  mesh.name = "createLightHalo";
  //mesh.rotation.x = Math.PI / 2;
  // 缩放
  const scale = 0.3 * size;
  mesh.size = scale; // 自定义一个属性，表示mesh静态大小
  mesh.scale.set(scale, scale, scale);
  return mesh;
}

/**
 * @description 创建文字坐标
 * @param {*} x d3 - 经纬度转换后的x轴坐标
 * @param {*} y d3 - 经纬度转换后的y坐标
 * @param {*} areaName 名称
 */
function createTextPoint(x, y, areaName, depth) {
  let tag = document.createElement("div");
  tag.innerHTML = areaName;
  tag.style.color = "#fff";
  tag.style.pointerEvents = "auto";
  tag.style.position = "absolute";
  // tag.addEventListener("mousedown", this.clickLabel, false); // 有时PC端click事不生效，不知道什么原因，就使用mousedown事件
  // tag.addEventListener("touchstart", this.clickLabel, false);
  let label = new CSS2DObject(tag);
  label.element.style.visibility = "visible";
  label.element.style.backgroundColor = "#113246";
  label.element.style.border = "1px solid #0bf6fb";
  label.element.style.padding = "2px 5px";
  label.element.style.fontSize = "12px";
  label.position.set(x, -y, depth + 0.06);
  return label;
}

function initMapMaterial() {
  // 创建环境贴图
  textureMap = textureLoader.load("./img/gz-map1.png");
  texturefxMap = textureLoader.load("./img/gz-map-fx.jpg");
  textureMap.wrapS = THREE.RepeatWrapping; //纹理水平方向的平铺方式
  textureMap.wrapT = THREE.RepeatWrapping; //纹理垂直方向的平铺方式
  textureMap.flipY = texturefxMap.flipY = false; // 如果设置为true，纹理在上到GPU的时候会进行纵向翻转。默认值为true。
  textureMap.rotation = texturefxMap.rotation = THREE.MathUtils.degToRad(45); //rotation纹理将围绕中心点旋转多少度
  const scale = 1;
  textureMap.repeat.set(scale, scale); // 调整X方向和Y方向的重复次数
  texturefxMap.repeat.set(scale, scale);
  textureMap.offset.set(0.6, 0.6); //offset贴图单次重复中的起始移量
  texturefxMap.offset.set(0, 0);

  // 创建高材质
  material = new THREE.MeshPhongMaterial({
    map: textureMap, //颜色贴图
    color: 0xb4eeea,
    emissive: 0x330000, // 设置自发光颜色
    side: THREE.DoubleSide, // 确保纹理可见
    combine: THREE.MultiplyOperation, //如何将表面颜色的结果与环境贴图
    transparent: true,
    opacity: 1,
    emissiveIntensity: 0.1, // 自发光强度
  });

  // 创建基础网格材质  侧面贴图
  material1 = new THREE.MeshLambertMaterial({
    map: texturefxMap, //用于创建法线贴图的纹理
    color: 0xffffff,
    transparent: true,
    opacity: 0.9,
  });

  return { material, material1 };
}

// 创建地图区块网格模型
function createMesh(data, depth) {
  // 创建一个多边形轮廓
  const shape = new THREE.Shape();
  // 循环多边形坐标数组  polygon是地图区块的经纬度坐标数组，构建地图边界线
  for (let i = 0; i < data.length; i++) {
    const [x, y] = projection(data[i]);
    if (i === 0) {
      shape.moveTo(x, -y);
    }
    shape.lineTo(x, -y);
  }
  // 伸参数
  const extrudeSettings = {
    depth: depth, // 拉伸度 （3D地图厚度）
    bevelEnabled: false, // 是否使用角
    bevelSegments: 1,
    bevelThickness: 0.1,
  };
  // 将多边形 拉伸扫描型  //shape二维轮  //extrudeSettings拉伸参数
  const geometry = new THREE.ExtrudeGeometry(shape, extrudeSettings);

  // 多边形添加材质
  const mesh = new THREE.Mesh(geometry, [
    material, //表面材质
    material1, //侧面材质
  ]);

  return mesh;
}

function initGui() {
  const gui = new GUI();
  const guiParams = {
    topColor: "#b4eeea",
    sideColor: "#ffffff",
    scale: 1,
    offset: 0.6,
    emissive: "#330000",
  };
  gui.addColor(guiParams, "topColor").onChange((val) => {
    material.color = new THREE.Color(val);
  });
  gui.addColor(guiParams, "sideColor").onChange((val) => {
    material1.color = new THREE.Color(val);
  });
  gui.add(guiParams, "scale", 0, 1).onChange((val) => {
    textureMap.repeat.set(val, val);
    texturefxMap.repeat.set(val, val);
  });
  gui.add(guiParams, "offset", 0, 1).onChange((val) => {
    textureMap.offset.set(val, val);
    texturefxMap.offset.set(val, val);
  });
  gui.addColor(guiParams, "emissive").onChange((val) => {
    material.emissive.setHex(parseInt(val.replace("#", ""), 16));
  });
}

// 创建地图区块边界线
function createLine(data, depth) {
  // 线材质对象 创建地图边界线 白色
  const lineMaterial = new THREE.LineBasicMaterial({
    color: "#ceebf7",
  });
  // 建一个空何体对象
  const lineGeometry = new THREE.BufferGeometry();
  const pointsArray = new Array();

  // 循环边形坐标数组  polygon是地图区块的经纬度坐标数组
  for (let i = 0; i < data.length; i++) {
    const [x, y] = projection(data[i]);
    // 三维向量对象  用于绘制边界线
    pointsArray.push(new THREE.Vector3(x, -y, 0));
  }

  lineGeometry.setFromPoints(pointsArray);
  const line = new THREE.Line(lineGeometry, lineMaterial);
  line.position.z += depth + 0.001;
  return line;
}

// 初始化场景背景原点图
function initCirclePoint(map) {
  const { size } = getBoundingBox(map);
  let width = size.x < size.y ? size.y + 1 : size.x + 1;
  let plane = new THREE.PlaneGeometry(width, width);
  let material = new THREE.MeshPhongMaterial({
    color: 0x00ffff,
    map: textureLoader.load("./img/circle-point.png"),
    transparent: true,
    opacity: 1,
    // depthTest: false,
  });
  let mesh = new THREE.Mesh(plane, material);
  mesh.position.set(...projection(centerPos), -0.3);
  scene.add(mesh);
}

// 初始化背景
function initSceneBg() {
  let plane = new THREE.PlaneGeometry(width, height);
  let material = new THREE.MeshPhongMaterial({
    // color: 0x061920,
    color: 0xffffff,
    map: textureLoader.load("./img/scene-bg2.png"),
    transparent: true,
    opacity: 1,
    depthTest: true,
  });

  let mesh = new THREE.Mesh(plane, material);
  mesh.scale.set(0.6, 0.6, 0.6);
  mesh.position.set(...projection(centerPos), -0.4);
  scene.add(mesh);
}

// 初始化旋转光圈
function initRotatingAperture(map) {
  const { size } = getBoundingBox(map);
  let width = size.x < size.y ? size.y + 1 : size.x + 1;
  let plane = new THREE.PlaneGeometry(width, width);
  let material = new THREE.MeshBasicMaterial({
    map: textureLoader.load("./img/rotatingAperture.png"),
    transparent: true,
    opacity: 1,
    depthTest: true,
  });
  let mesh = new THREE.Mesh(plane, material);
  mesh.position.set(...projection(centerPos), 0);
  mesh.scale.set(1.05, 1.05, 1.05);
  scene.add(mesh);
  rotatingApertureMesh = mesh;
}

// 初始化旋转点
function initRotatingPoint(map) {
  const { size } = getBoundingBox(map);
  let width = size.x < size.y ? size.y + 1 : size.x + 1;
  let plane = new THREE.PlaneGeometry(width - 0.5, width - 0.5);
  let material = new THREE.MeshBasicMaterial({
    map: textureLoader.load("./img/rotating-point2.png"),
    transparent: true,
    opacity: 1,
    depthTest: true,
  });
  let mesh = new THREE.Mesh(plane, material);
  mesh.position.set(...projection(centerPos), -0.2);
  mesh.scale.set(1.1, 1.1, 1.1);
  scene.add(mesh);
  rotatingPointMesh = mesh;
}

// 撒点
function initPoints() {
  const group = new THREE.Group();
  const iconTexture = textureLoader.load("./img/rqbj.png");
  const material = new THREE.SpriteMaterial({
    map: iconTexture,
    sizeAttenuation: true, // 根据距离调整大小
    transparent: true, // 使纹理透明
    opacity: 1, // 图标的透明度
    depthTest: false, // 关闭深度监测
  });
  const sprite = new THREE.Sprite(material);
  sprite.scale.set(0.04, 0.05, 1);
  const { center } = getBoundingBox(mapModel);

  sprite.position.set(center.x, center.y, 0.06);
  sprite.properties = { name: "这是一个测试点位", info: { a: "在来点值" } };
  pointInstanceArr.push(sprite);
  group.add(sprite);
  group.renderOrder = 99; // 设置为一个比默认值更大的值
  mapModel.add(group);
}

// 批量撒点
function initBatchPoints() {
  const group = new THREE.Group();
  const iconTexture = textureLoader.load("./img/rqbj.png");
  const geometry = new THREE.BufferGeometry();
  const vertices = [];
  const { center } = getBoundingBox(mapModel); // 根据地图位置设置
  for (let i = 0; i < 10000; i++) {
    vertices.push(Math.random(), Math.random(), 0.055); // 随机位
  }
  geometry.setAttribute("position", new THREE.Float32BufferAttribute(vertices, 3));
  const material = new THREE.PointsMaterial({
    size: 10, // 点的大小
    map: iconTexture, // 使用共享的纹理
    transparent: true,
    sizeAttenuation: true, // 根据距调整大小
    opacity: 1, // 图标的透明度
    depthTest: false, // 关闭深度监测
    // alphaTest: 0.01, // 设定透明度阈值
  });

  const points = new THREE.Points(geometry, material);
  group.add(points);
  group.position.x -= 0.5;
  group.position.y -= 0.5;
  group.renderOrder = 99; // 设置为一个比默认值更大的值
  mapModel.add(group);
}

// 设置地图中心位置
function setMapCenter(map) {
  const { center } = getBoundingBox(map);
  const offset = [0, 0];

  map.position.x = map.position.x - center.x - offset[0];
  map.position.y = map.position.y - center.y - offset[1];
}

// 计算模型盒子大小
function getBoundingBox(model) {
  const box = new THREE.Box3().setFromObject(model);
  const center = box.getCenter(new THREE.Vector3());
  const size = box.getSize(new THREE.Vector3());
  return {
    center,
    size,
  };
}

function initCameraHelper() {
  const helper = new THREE.CameraHelper(camera);
  scene.add(helper);

  // 添加坐标系 (长度为5)
  const axesHelper = new THREE.AxesHelper(500);
  scene.add(axesHelper); // 红色为 X 轴，绿色为 Y 轴，蓝色为 Z 轴
}

function random(min, max) {
  return Math.floor(Math.random() * (max - min + 1)) + min;
}

/**
 * 移除对象
 * @param {*} object {} []
 */
function removeObject(object) {
  if (isType("Array", object)) {
    object.map((item) => {
      item.geometry.dispose();
    });
    scene.remove(...object);
  } else {
    object.geometry.dispose();
    scene.remove(object);
  }
}

// 创建上升粒子
function initParticle(map) {
  const { size, center } = getBoundingBox(map);
  // 构建范围，中间地图的2倍
  let minX = center.x - size.x;
  let maxX = center.x + size.x;
  let minY = center.y - size.y / 2;
  let maxY = center.y + size.y / 2;
  let minZ = -16;
  let maxZ = 6;
  for (let i = 0; i < 12; i++) {
    const particle = createSequenceFrame({
      image: "./img/上升粒子1.png",
      width: 180,
      height: 189,
      frame: 9,
      column: 9,
      row: 1,
      speed: 0.5, // 飞升速度
    });
    let particleScale = random(1, 3);
    particle.scale.set(particleScale * 0.1, particleScale * 0.1, particleScale * 0.1);
    particle.rotation.x = Math.PI / 2;
    let x = random(minX, maxX);
    let y = random(minY, maxY);
    let z = random(minZ, maxZ);
    particle.position.set(x, y, z);
    particleArr.push(particle);
  }
  scene.add(...particleArr);
}

// 创建序列帧动画;
function createSequenceFrame(options) {
  const geometry = new THREE.PlaneGeometry(options.width, options.height);
  let texture = textureLoader.load(options.image); // 加载图片
  texture.repeat.set(1 / options.column, 1 / options.row); // 从图像上截图第一帧
  let material = new THREE.MeshBasicMaterial({
    map: texture,
    transparent: true,
    opacity: 1,
    side: THREE.DoubleSide,
    depthWrite: false, //是否对深度缓冲区有任何的影响
  });
  let mesh = new THREE.Mesh(geometry, material);
  let r = 0; // 当前行
  let c = 0; // 当前列
  let t = 0; // 时间
  mesh.updateSequenceFrame = (time) => {
    // 这个动画只控制数字的变化
    t += options.speed;
    if (t > options.frame) t = 0;
    c = options.column - Math.floor(t % options.column) - 1;
    r = Math.floor((t / options.column) % options.row);
    texture.offset.x = c / options.column; // 动态更新纹理偏移 播放关键帧动画
    texture.offset.y = r / options.row; // 动态更新纹理偏移 播放关键帧动画
  };

  return mesh;
}

animate();

function animate() {
  window.requestAnimationFrame(animate);
  // 只有在控制器启用时才更新控制器
  if (controls.enabled) {
    controls.update();
  }
  renderMap();
}

loop();
function loop() {
  window.requestAnimationFrame(loop);
  if (rotatingApertureMesh) {
    rotatingApertureMesh.rotation.z += 0.0008;
  }
  if (rotatingPointMesh) {
    rotatingPointMesh.rotation.z -= 0.0008;
  }
  // 圆柱底部 波纹
  if (WaveMeshArr.length) {
    WaveMeshArr.forEach(function (mesh) {
      mesh._s += 0.007;
      mesh.scale.set(mesh.size * mesh._s, mesh.size * mesh._s, mesh.size * mesh._s);
      if (mesh._s <= 1.5) {
        //mesh._s=1，透明度=0 mesh._s=1.5，透明度=1
        mesh.material.opacity = (mesh._s - 1) * 2;
      } else if (mesh._s > 1.5 && mesh._s <= 2) {
        //mesh._s=1.5，透明度=1 mesh._s=2，透明度=0
        mesh.material.opacity = 1 - (mesh._s - 1.5) * 2;
      } else {
        mesh._s = 1.0;
      }
    });
  }
  // 粒子上升
  if (particleArr.length) {
    for (let i = 0; i < particleArr.length; i++) {
      particleArr[i].updateSequenceFrame();
      particleArr[i].position.z += 0.3;
      if (particleArr[i].position.z >= 100) {
        const { size, center } = getBoundingBox(mapModel);
        // 构建范围，中间地图的2倍
        let minX = center.x - size.x;
        let maxX = center.x + size.x;
        let minY = center.y - size.y / 2;
        let maxY = center.y + size.y / 2;
        let x = random(minX, maxX);
        let y = random(minY, maxY);
        let z = random(-16, 6);
        particleArr[i].position.x = x;
        particleArr[i].position.y = y;
        particleArr[i].position.z = z;
      }
    }
  }
  // 飞线动画
  animateFlyLines();
}

// 动画循环
function animateFlyLines() {
  linePartList.forEach((flyline) => {
    flyline.progress += 0.01; // 控制飞线移动度
    if (flyline.progress > 1) flyline.progress = 0; // 循环动画
    const points = flyline.curvePoints;
    const totalPoints = points.length;
    const segmentLength = 5; // 飞线由多少段组成
    const startIdx = Math.floor(flyline.progress * (totalPoints - segmentLength));

    // 获取当前飞线的点
    const segmentPoints = points.slice(startIdx, startIdx + segmentLength);

    // 更新飞线几何体的顶点
    const positions = flyline.geometry.attributes.position.array;
    for (let i = 0; i < segmentPoints.length; i++) {
      const point = segmentPoints[i];
      positions[i * 3] = point.x;
      positions[i * 3 + 1] = point.y;
      positions[i * 3 + 2] = point.z;
    }

    // 通知 Three.js 顶点数据已更新
    flyline.geometry.attributes.position.needsUpdate = true;
  });
}

const visible = ref(false);
const tipRef = ref(null);

function setDialog(flag, { top, left }, name) {
  visible.value = flag;
  tipRef.value.style.top = `${top}px`;
  tipRef.value.style.left = `${left}px`;
  tipRef.value.querySelector("p").innerHTML = name;
}

function closeTip() {
  visible.value = false;
}

// 建 Raycaster 和鼠标向量
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
// 监听鼠标点击事件
window.addEventListener("click", (event) => {
  // 算鼠标位置的标准化设备坐标（-1 到 1）
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

  // 更新射线
  raycaster.setFromCamera(mouse, camera);

  // 检测射线与物体的交集
  const intersects = raycaster.intersectObjects(pointInstanceArr, true); // 检测与精灵的交集

  // 如果有交集，输出交集物体的信息
  if (intersects.length > 0) {
    const clickedObject = intersects[0].object;
    console.log("点击了点位:", clickedObject);
    setDialog(true, { top: event.pageY, left: event.pageX }, clickedObject.properties.name);
  }
});

// 添加入场动画函数
function startEntranceAnimation() {
  createEarth();

  const duration = 3000; // 动画持续3秒
  const startTime = Date.now();

  if (mapModel) {
    mapModel.visible = false; // 确保地图模型在地球展示期间不可见
  }
  if (earthModel) {
    earthModel.visible = true;
    earthModel.scale.set(1, 1, 1); // 初始缩小比例调整为0.5
  }

  function animateEntrance() {
    const currentTime = Date.now();
    const elapsed = currentTime - startTime;
    const progress = Math.min(elapsed / duration, 1);
    const easeProgress = easeInOutCubic(progress);

    if (earthModel) {
      earthModel.rotation.y += 0.008; // 减慢地球旋转速度
      earthModel.scale.setScalar(1 + 3 * easeProgress); // 从1放大到3.0
    }

    earthModel.material.opacity = 1 - easeProgress;
    if (progress > 0.5) {
      rotatingApertureMesh.visible = false;
      rotatingPointMesh.visible = false;
    }

    if (progress > 0.7) {
      const mapScaleProgress = (progress - 0.9) / 0.1;
      earthModel.visible = false; // 在地球淡出时显示地图模型
      mapModel.visible = true; // 在地球淡出时显示地图模型
      rotatingApertureMesh.visible = true; // 确保地图模型在地球展示期间不可见
      rotatingPointMesh.visible = true; // 确保地图模型在地球展示期间不可见

      if (mapModel) {
        mapModel.traverse((child) => {
          if (child.isMesh) {
            child.material.needsUpdate = true; // 确保材质在放大过程中更新
          }
        });
      }
      // 调整相机位置以模拟场景放大
      camera.position.z = 400 + 500 * (1 - mapScaleProgress); // 从1000移动到1500
    }

    renderMap();

    if (progress < 1) {
      requestAnimationFrame(animateEntrance);
    } else {
      if (earthModel) {
        scene.remove(earthModel);
        earthModel.geometry.dispose();
        earthModel.material.dispose();
        earthModel = null;
      }
      // 重置地图模型的旋转角度
      mapModel.rotation.set(0, 0, 0);
      controls.enabled = true; // 启用控制器
    }
  }

  animateEntrance();
}

// 添加缓动函数
function easeInOutCubic(t) {
  return t < 0.5 ? 4 * t * t * t : 1 - Math.pow(-2 * t + 2, 3) / 2;
}

// 添加创建地球的函数
function createEarth() {
  const geometry = new THREE.SphereGeometry(200, 50, 50);

  const textureLoader = new THREE.TextureLoader();
  const material = new THREE.MeshBasicMaterial({
    map: textureLoader.load("./img/land_ocean_ice_cloud_2048.jpg"),
    bumpScale: 0.5, // 凹凸效果的强
    transparent: true,
  });

  earthModel = new THREE.Mesh(geometry, material);
  earthModel.scale.set(1, 1, 1); // 确保缩放比例在所有轴上是相同的
  earthModel.position.set(0, 0, 0); // 确保地球在相机视野内
  scene.add(earthModel);
}
</script>

<style>
.tooltip {
  width: 200px;
  height: 300px;
  position: absolute;
  background-color: rgba(0, 0, 0, 0.7);
  color: #fff;
  border-radius: 3px;
  border: 1px solid #fff;
  transition: opacity 0.3s;
  top: 0;
  z-index: 99;
  padding: 20px;
}

.tooltip span {
  position: absolute;
  top: 0;
  right: 0;
  display: inline-block;
  width: 20px;
  height: 20px;
  cursor: pointer;
}
</style>
