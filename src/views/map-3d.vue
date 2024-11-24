<template>
  <main>
    <div class="box-3d" id="webgl"></div>
  </main>
</template>

<script setup>
import { onMounted } from "vue";
import * as THREE from "three";
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

// 创建3D场景对象Scene
const scene = new THREE.Scene();

const width = window.innerWidth; //宽度
const height = window.innerHeight; //高度

// 30:视场角度, width / height:Canvas画布宽高比, 1:近裁截面, 3000：远裁截面

const camera = new THREE.PerspectiveCamera(45, width / height, 1, 10000);
// camera.up.set(0, 0, 1); // 相机以那个轴向上
camera.position.set(78.71, 323.59, 243.08);

// camera.lookAt([...projection(centerPos), 0]); // 指向mesh对应的位置

// 创建渲染器对象
const renderer = new THREE.WebGLRenderer({ antialias: true });

// 获取你屏幕对应的设备像素比.devicePixelRatio告诉threejs,以免渲染模糊问题
renderer.setPixelRatio(window.devicePixelRatio);

// 定义threejs输出画布的尺寸(单位:像素px)
renderer.setSize(width, height); //设置three.js渲染区域的尺寸(像素px)

renderer.setClearColor(0x444444, 1); //设置背景颜色

// 创建CSS2DRenderer渲染器(代替鼠标射线检测)
const labelRenderer = new CSS2DRenderer();
// 设置labelRenderer渲染器宽高
labelRenderer.setSize(width, height);

labelRenderer.domElement.style.position = "absolute";
labelRenderer.domElement.style.top = "0px";
labelRenderer.domElement.style.pointerEvents = "none";

//环境光:没有特定方向，整体改变场景的光照明暗
const ambient = new THREE.AmbientLight("#ffffff", 4);

scene.add(ambient);

// 设置相机控件轨道控制器OrbitControls
const controls = new OrbitControls(camera, renderer.domElement);
controls.maxPolarAngle = 2.5;
controls.minPolarAngle = -2;
controls.maxAzimuthAngle = 1.5;
controls.minAzimuthAngle = -2;

// 如果OrbitControls改变了相机参数，重新调用渲染器渲染三维场景
controls.addEventListener("change", function () {
  renderMap();
  // 浏览器控制台查看相机位置变化
  //   console.log("camera.position", camera.position);
}); //监听鼠标、键盘事件

const helper = new THREE.CameraHelper(camera);
scene.add(helper);

// 添加坐标系 (长度为5)
const axesHelper = new THREE.AxesHelper(500);
scene.add(axesHelper); // 红色为 X 轴，绿色为 Y 轴，蓝色为 Z 轴
generateGeometry();

onMounted(() => {
  const container = document.getElementById("webgl");
  container.appendChild(renderer.domElement);
  renderMap();
  // 将渲染器添加到页面
  container.appendChild(labelRenderer.domElement);

  // onresize 事件会在窗口被调整大小时发生
  window.onresize = function () {
    // 重置渲染器输出画布canvas尺寸
    renderer.setSize(window.innerWidth, window.innerHeight);

    // 全屏情况下：设置观察范围长宽比aspect为窗口宽高比
    camera.aspect = window.innerWidth / window.innerHeight;

    // 渲染器执行render方法的时候会读取相机对象的投影矩阵属性projectionMatrix
    // 但是不会每渲染一帧，就通过相机的属性计算投影矩阵(节约计算资源)
    // 如果相机的一些属性发生了变化，需要执行updateProjectionMatrix ()方法更新相机的投影矩阵
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
  const map = new THREE.Object3D();
  projection.center(centerPos).scale(200).translate([0, 0]);
  const url = "https://geo.datav.aliyun.com/areas_v3/bound/geojson?code=510113"; //成都市青白江区

  fetch(url)
    .then((response) => response.json())
    .then((data) => {
      const features = data.features;
      features.forEach((feature) => {
        const { geometry, properties } = feature;
        const coordinates = geometry.coordinates;
        const province = new THREE.Object3D();
        province.name = properties.name;
        // 拉伸 地图厚度
        const depth = 0.05;
        //这里创建光柱、文字坐标
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
        map.add(province, light, label);
      });
      map.scale.set(200, 200, 200);
      setCenter(map);
      scene.add(map); //将地图对象添加到场景中
      renderMap();
    });
}

/**
 * @description 创建光柱、文字坐标
 * @param {*} properties 属性、详情
 * @param {Function} projection  d3-geo转化坐标
 */
function initLightPoint(properties, depth) {
  const { name } = properties;
  let lightCenter = properties.centroid || properties.center;
  // projection用来把经纬度转换成坐标
  const [x, y] = projection(lightCenter);

  // 随机创建光柱高度
  let heightScaleFactor = 2 + random(1, 5) / 5;
  // 创建光柱
  const light = createLightPillar(x, y, heightScaleFactor * 0.05, depth);
  // 创建文字标注
  const label = createTextPoint(x, y, name, depth);

  return [light, label];
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
  // 柱体的geo,6.19=柱体图片高度/宽度的倍数
  const geometry = new THREE.PlaneGeometry(height / 6.219, height);

  // 柱体旋转90度，垂直于Y轴
  geometry.rotateX(Math.PI / 2);
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
  group.position.set(x, -y, depth + 0.01);
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
    depthWrite: false, //禁止写入深度缓冲区数据
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
 * @param {*} y d3 - 经纬度转换后的y轴坐标
 * @param {*} areaName 地名
 */
function createTextPoint(x, y, areaName, depth) {
  let tag = document.createElement("div");
  tag.innerHTML = areaName;
  tag.style.color = "#fff";
  tag.style.pointerEvents = "auto";
  tag.style.position = "absolute";
  // tag.addEventListener("mousedown", this.clickLabel, false); // 有时候PC端click事件不生效，不知道什么原因，就使用mousedown事件
  // tag.addEventListener("touchstart", this.clickLabel, false);
  let label = new CSS2DObject(tag);
  label.element.style.visibility = "visible";
  label.position.set(x, -y, depth);
  return label;
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
  // 拉伸参数
  const extrudeSettings = {
    depth: depth, // 拉伸度 （3D地图厚度）
    bevelEnabled: false, // 是否使用倒角
    // bevelSegments: 1,
    // bevelThickness: 0.1,
  };
  // 将多边形 拉伸扫描成型  //shape二维轮廓  //extrudeSettings拉伸参数
  const geometry = new THREE.ExtrudeGeometry(shape, extrudeSettings);

  // 创建环境贴图
  //   let textureMap = textureLoader.load(require("./mapimg/gz-map.jpeg"));
  //   let texturefxMap = textureLoader.load(require("./mapimg/gz-map-fx.jpeg"));
  //   textureMap.wrapS = THREE.RepeatWrapping; //纹理水平方向的平铺方式
  //   textureMap.wrapT = THREE.RepeatWrapping; //纹理垂直方向的平铺方式
  //   textureMap.flipY = texturefxMap.flipY = false; // 如果设置为true，纹理在上传到GPU的时候会进行纵向的翻转。默认值为true。
  //   textureMap.rotation = texturefxMap.rotation = THREE.MathUtils.degToRad(45); //rotation纹理将围绕中心点旋转多少度
  //   const scale = 0.01;
  //   textureMap.repeat.set(scale, scale); //repeat决定纹理在表面的重复次数
  //   texturefxMap.repeat.set(scale, scale);
  //   textureMap.offset.set(0.5, 0.5); //offset贴图单次重复中的起始偏移量
  //   texturefxMap.offset.set(0.5, 0.5);

  // 创建高光材质
  const material = new THREE.MeshPhongMaterial({
    // map: textureMap, //颜色贴图
    // normalMap: texturefxMap, //用于创建法线贴图的纹理
    // normalScale: new THREE.Vector2(12.2, 2.2),//法线贴图对材质的影响程度
    color: "#7bc6c2",
    combine: THREE.MultiplyOperation, //如何将表面颜色的结果与环境贴图
    transparent: true,
    opacity: 1,
  });

  // 创建基础网格材质
  const material1 = new THREE.MeshBasicMaterial({
    color: "#3480C4",
    transparent: true,
    opacity: 0.4,
  });

  // 多边形添加材质
  const mesh = new THREE.Mesh(geometry, [
    material, //表面材质
    material1, //侧面材质
  ]);

  return mesh;
}

// 创建地图区块边界线
function createLine(data, depth) {
  // 线材质对象 创建地图边界线 白色
  const lineMaterial = new THREE.LineBasicMaterial({
    color: "white",
  });
  // 创建一个空的几何体对象
  const lineGeometry = new THREE.BufferGeometry();
  const pointsArray = new Array();

  // 循环多边形坐标数组  polygon是地图区块的经纬度坐标数组
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

// 设置地图中心位置
function setCenter(map) {
  map.rotation.x = -Math.PI / 2;
  const box = new THREE.Box3().setFromObject(map);
  const center = box.getCenter(new THREE.Vector3());
  const offset = [0, 0];

  map.position.x = map.position.x - center.x - offset[0];
  map.position.z = map.position.z - center.z - offset[1];
}

function random(min, max) {
  return Math.floor(Math.random() * (max - min + 1) + min);
}

animate();

function animate() {
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
  window.requestAnimationFrame(animate);
  renderMap();
}

// 创建 Raycaster 和鼠标向量
const raycaster = new THREE.Raycaster();
const mouse = new THREE.Vector2();
// 监听鼠标点击事件
window.addEventListener("click", (event) => {
  // 计算鼠标位置的标准化设备坐标（-1 到 1）
  mouse.x = (event.clientX / window.innerWidth) * 2 - 1;
  mouse.y = -(event.clientY / window.innerHeight) * 2 + 1;

  // 更新射线
  raycaster.setFromCamera(mouse, camera);

  // 检测射线与物体的交集
  const intersects = raycaster.intersectObjects(scene.children, true); // 检测与精灵的交集

  // 如果有交集，输出交集物体的信息
  if (intersects.length > 0) {
    const clickedObject = intersects[0].object;
    console.log("点击了点位:", clickedObject);
    alert("你点击了一个点位！");
  }
});
</script>
