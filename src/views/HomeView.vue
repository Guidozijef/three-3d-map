<template>
  <main>
    <div class="box-3d" id="webgl"></div>
  </main>
</template>

<script setup>
import { onMounted } from "vue";
import * as THREE from "three";
// 引入轨道控制器扩展库OrbitControls.js
import { OrbitControls } from "three/addons/controls/OrbitControls.js";
//引入性能监视器stats.js
import Stats from "three/addons/libs/stats.module.js";
// 引入dat.gui.js的一个类GUI
import { GUI } from "three/addons/libs/lil-gui.module.min.js";

// 创建3D场景对象Scene
const scene = new THREE.Scene();

//创建一个长方体几何对象Geometry
const geometry = new THREE.BoxGeometry(100, 100, 100);

// //创建一个材质对象Material
// const material = new THREE.MeshBasicMaterial({
//   color: 0xff0000, //0xff0000设置材质颜色为红色
//   // transparent: true, //开启透明
//   opacity: 0.5, //设置透明度
// });

// 模拟镜面反射，产生一个高光效果
const material = new THREE.MeshPhongMaterial({
  color: 0xff0000,
  shininess: 20, //高光部分的亮度，默认30
  specular: 0x444444, //高光部分的颜色
});

// 网格模型对象Mesh，两个参数分别为几何体geometry、材质material
const mesh = new THREE.Mesh(geometry, material); //网格模型对象Mesh
// 设置网格模型在三维空间中的位置坐标，默认是坐标原点
mesh.position.set(100, 10, 0);

scene.add(mesh);

// width和height用来设置Three.js输出的Canvas画布尺寸(像素px)
const width = window.innerWidth; //宽度
const height = window.innerHeight; //高度

// 30:视场角度, width / height:Canvas画布宽高比, 1:近裁截面, 3000：远裁截面
const camera = new THREE.PerspectiveCamera(30, width / height, 1, 8000);

//相机在Three.js三维坐标系中的位置
// 根据需要设置相机位置具体值
camera.position.set(2000, 2000, 2000);

camera.lookAt(mesh.position); //指向mesh对应的位置

// 创建渲染器对象
const renderer = new THREE.WebGLRenderer({ antialias: true });

// 获取你屏幕对应的设备像素比.devicePixelRatio告诉threejs,以免渲染模糊问题
renderer.setPixelRatio(window.devicePixelRatio);

// 定义threejs输出画布的尺寸(单位:像素px)
renderer.setSize(width, height); //设置three.js渲染区域的尺寸(像素px)

renderer.setClearColor(0x444444, 1); //设置背景颜色

//点光源：两个参数分别表示光源颜色和光照强度
// 参数1：0xffffff是纯白光,表示光源颜色
// 参数2：1.0,表示光照强度，可以根据需要调整
// const pointLight = new THREE.PointLight(0xffffff, 1.0);
// pointLight.intensity = 50000.0; //光照强度
// pointLight.decay = 2.0; //设置光源不随距离衰减
// //点光源位置
// pointLight.position.set(-400, -200, -300);
// scene.add(pointLight); //点光源添加到场景中

// // 光源辅助观察
// const pointLightHelper = new THREE.PointLightHelper(pointLight, 10);
// scene.add(pointLightHelper);

//环境光:没有特定方向，整体改变场景的光照明暗
const ambient = new THREE.AmbientLight(0xffffff, 0.4);

scene.add(ambient);

// 渲染循环
// const clock = new THREE.Clock();
// function render() {
//   const spt = clock.getDelta() * 1000; //毫秒
//   console.log("两帧渲染时间间隔(毫秒)", spt);
//   console.log("帧率FPS", 1000 / spt);
//   renderer.render(scene, camera); //执行渲染操作
//   mesh.rotateY(0.01); //每次绕y轴旋转0.01弧度
//   requestAnimationFrame(render); //请求再次执行渲染函数render，渲染下一帧
// }
// render();

//创建一个长方体几何对象Geometry
const geometry1 = new THREE.BoxGeometry(100, 100, 100);
//材质对象Material
const material1 = new THREE.MeshLambertMaterial({
  color: 0x00ffff, //设置材质颜色
  transparent: true, //开启透明
  opacity: 0.5, //设置透明度
});

for (let i = 0; i < 10; i++) {
  for (let j = 0; j < 10; j++) {
    const mesh = new THREE.Mesh(geometry1, material1); //网格模型对象Mesh
    // 在XOZ平面上分布
    mesh.position.set(i * 200, 0, j * 200);
    scene.add(mesh); //网格模型添加到场景中
  }
}

// 设置相机控件轨道控制器OrbitControls
const controls = new OrbitControls(camera, renderer.domElement);
// 如果OrbitControls改变了相机参数，重新调用渲染器渲染三维场景
controls.addEventListener("change", function () {
  renderer.render(scene, camera); //执行渲染操作
  // 浏览器控制台查看相机位置变化
  console.log("camera.position", camera.position);
}); //监听鼠标、键盘事件

// 实例化一个gui对象
const gui = new GUI();
//改变交互界面style属性
gui.domElement.style.right = "0px";
gui.domElement.style.width = "300px";

//创建一个对象，对象属性的值可以被GUI库创建的交互界面改变
const obj = {
  x: 30,
};
// gui增加交互界面，用来改变obj对应属性
gui.add(obj, "x", 0, 180);

// 通过GUI改变光照强度属性.intensity
// gui.add(ambient, "intensity", 0, 2.0).name("环境光强度");
gui.add(ambient, "intensity", 0, 2.0).name("环境光强度").step(0.1);

gui.add(mesh.position, "x", 0, 180).name("模型X轴坐标");
gui.add(mesh.position, "y", 0, 180).name("模型Y轴坐标");
gui.add(mesh.position, "z", 0, 180).name("模型Z轴坐标");

// 当obj的x属性变化的时候，就把此时obj.x的值value赋值给mesh的x坐标
gui.add(obj, "x", 0, 180).onChange(function (value) {
  // 你可以写任何你想跟着obj.x同步变化的代码
  // 比如mesh.position.y = value;
  mesh.position.x = value;
  renderer.render(scene, camera); //执行渲染操作
});

const obj3 = {
  scale: 0,
};
// 参数3数据类型：对象(下拉菜单)
gui
  .add(obj3, "scale", {
    left: -100,
    center: 0,
    right: 100,
  })
  .name("位置选择")
  .onChange(function (value) {
    mesh.position.x = value;
  });

const obj1 = {
  color: 0x00ffff,
};
// .addColor()生成颜色值改变的交互界面
gui.addColor(obj1, "color").onChange(function (value) {
  mesh.material.color.set(value);
  renderer.render(scene, camera); //执行渲染操作
});

const obj2 = {
  bool: false,
};
// gui.add(obj2, "bool").name("旋转动画");

gui.add(obj2, "bool").onChange(function (value) {
  // 点击单选框，控制台打印obj.bool变化
  console.log("obj.bool", value);
});

// 渲染循环
function render1() {
  // 当gui界面设置obj.bool为true,mesh执行旋转动画
  if (obj2.bool) mesh.rotateY(0.01);
  renderer.render(scene, camera);
  requestAnimationFrame(render1);
}
render1();

onMounted(() => {
  document.getElementById("webgl").appendChild(renderer.domElement);
  renderer.render(scene, camera); //执行渲染操作

  //创建stats对象
  const stats = new Stats();
  //stats.domElement:web页面上输出计算结果,一个div元素，
  document.body.appendChild(stats.domElement);

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
    renderer.render(scene, camera); //执行渲染操作
  };
});
</script>

<style>
.box-3d {
  width: 100%;
  height: 100vh;
  background-color: #fff;
  overflow: hidden;
}
</style>
