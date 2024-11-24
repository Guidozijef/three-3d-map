<template>
  <div class="center-map-box" id="contant"></div>
</template>

<script>
import * as THREE from "three";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls.js";
import { CSS2DRenderer, CSS2DObject } from "three/addons/renderers/CSS2DRenderer.js";
import * as d3 from "d3";

let textureLoader = new THREE.TextureLoader(); //纹理贴图加载器
let WaveMeshArr = []; //所有波动光圈集合
let rotatingApertureMesh, rotatingPointMesh;
// 墨卡托投影转换
const projection = d3.geoMercator();
export default {
  data() {
    return {
      scene: null,
      camera: null,
      renderer: null,
      contant: null,
      spotLight: null,
      map: null,
      centerPos: [114.255221, 30.619014], // 地图中心经纬度坐标
      raycaster: null, // 光线投射 用于进行鼠标拾取
      mouse: new THREE.Vector2(0, 0), // 鼠标位置
      labelRenderer: null, //CSS2DRenderer渲染器对象
    };
  },
  mounted() {
    // 第一步新建一个场景
    this.scene = new THREE.Scene();
    this.contant = document.getElementById("contant");
    // 辅助线
    const axesHelper = new THREE.AxesHelper(10);
    this.scene.add(axesHelper);

    //环境光
    const ambient = new THREE.AmbientLight("#ffffff", 4);
    this.scene.add(ambient);

    this.setCamera();
    this.setRenderer();
    this.generateGeometry();
    this.setClickFn();
    this.setController();
    this.animate();
    window.onresize = () => {
      this.renderer.setSize(this.contant.clientWidth, this.contant.clientHeight);
      this.camera.aspect = this.contant.clientWidth / this.contant.clientHeight;
      this.camera.updateProjectionMatrix();
    };

    this.initFloor();
  },
  methods: {
    // 新建透视相机
    setCamera() {
      this.camera = new THREE.PerspectiveCamera(45, this.contant.clientWidth / this.contant.clientHeight, 0.1, 1000);
      // 设置相机位置
      this.camera.position.set(20, 20, 20);
    },
    // 设置渲染器
    setRenderer() {
      this.renderer = new THREE.WebGLRenderer();
      // 设置画布的大小
      this.renderer.setSize(this.contant.clientWidth, this.contant.clientHeight);
      this.contant.appendChild(this.renderer.domElement);

      // 创建CSS2DRenderer渲染器(代替鼠标射线检测)
      this.labelRenderer = new CSS2DRenderer();
      // 设置labelRenderer渲染器宽高
      this.labelRenderer.setSize(this.contant.clientWidth, this.contant.clientHeight);
      this.labelRenderer.domElement.style.position = "absolute";
      this.labelRenderer.domElement.style.top = "0px";
      this.labelRenderer.domElement.style.pointerEvents = "none";
      // 将渲染器添加到页面
      this.contant.appendChild(this.labelRenderer.domElement);
    },

    render() {
      this.renderer.render(this.scene, this.camera);

      this.labelRenderer.render(this.scene, this.camera);
    },

    // 绘制地图
    generateGeometry() {
      // 初始化一个地图对象
      this.map = new THREE.Object3D();
      // 地理坐标数据 转换为3D坐标数据
      // 墨卡托投影转换
      projection.center(this.centerPos).scale(200).translate([0, 0]);
      const url = "https://geo.datav.aliyun.com/areas_v3/bound/420000_full.json";
      fetch(url)
        .then((res) => res.json())
        .then((jsondata) => {
          jsondata.features.forEach((elem) => {
            const coordinates = elem.geometry.coordinates;
            const province = new THREE.Object3D();
            province.name = elem.properties.name;

            // 拉伸 地图厚度
            const depth = 1;

            //这里创建光柱、文字坐标
            const lightPoint = this.initLightPoint(elem.properties, depth);

            // 循环坐标数组
            coordinates.forEach((multiPolygon) => {
              multiPolygon.forEach((polygon, index) => {
                // 创建地图区块
                const mesh = this.createMesh(polygon, depth);

                // 创建地图区块边界线
                const line = this.createLine(polygon, depth);
                province.add(mesh);
                province.add(line);
              });
            });
            // 地图区块添加到地图对象中
            this.map.add(province, ...lightPoint);
          });

          // 地图放到中心位置
          this.setCenter(this.map);
          // 地图添加到场景中
          this.scene.add(this.map);
          // 场景渲染
          this.render();
        });
    },

    /**
     * @description // 创建光柱
     * @param {*} x d3 - 经纬度转换后的x轴坐标
     * @param {*} y d3 - 经纬度转换后的z轴坐标
     * @param {*} heightScaleFactor
     */
    createLightPillar(x, y, heightScaleFactor = 1, depth) {
      let group = new THREE.Group();
      // 柱体高度
      const height = heightScaleFactor;
      // 柱体的geo,6.19=柱体图片高度/宽度的倍数
      const geometry = new THREE.PlaneGeometry(height / 6.219, height);
      // 柱体旋转90度，垂直于Y轴
      geometry.rotateX(Math.PI / 2);
      // 柱体的z轴移动高度一半对齐中心点
      geometry.translate(0, 0, height / 2);
      // 柱子材质
      const material = new THREE.MeshBasicMaterial({
        map: textureLoader.load(require("./mapimg/光柱.png")),
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
      const bottomMesh = this.createPointMesh(0.5);
      // 创建光圈
      const lightHalo = this.createLightHalo(0.5);
      WaveMeshArr.push(lightHalo);
      // 将光柱和标点添加到组里
      group.add(bottomMesh, lightHalo, light01, light02);
      group.position.set(x, -y, depth + 0.01);
      return group;
    },

    /**
     * @description 创建底部标点
     * @param {number} size 缩放大小
     */
    createPointMesh(size) {
      // 标记点：几何体，材质，
      const geometry = new THREE.PlaneGeometry(1, 1);
      const material = new THREE.MeshBasicMaterial({
        map: textureLoader.load(require("./mapimg/标注.png")),
        color: 0x00ffff,
        side: THREE.DoubleSide,
        transparent: true,
        depthWrite: false, //禁止写入深度缓冲区数据
      });
      let mesh = new THREE.Mesh(geometry, material);
      mesh.renderOrder = 2;
      //mesh.rotation.x = Math.PI / 2;
      mesh.name = "createPointMesh";
      // 缩放
      const scale = 1 * size;
      mesh.scale.set(scale, scale, scale);
      return mesh;
    },

    /**
     * @description 创建底部标点的光圈
     * @param {number} size 缩放大小
     */
    createLightHalo(size) {
      // 标记点：几何体，材质，
      const geometry = new THREE.PlaneGeometry(1, 1);
      const material = new THREE.MeshBasicMaterial({
        map: textureLoader.load(require("./mapimg/标注光圈.png")),
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
      const scale = 1.5 * size;
      mesh.size = scale; //自顶一个属性，表示mesh静态大小
      mesh.scale.set(scale, scale, scale);
      return mesh;
    },

    random(min, max) {
      return Math.floor(Math.random() * (max - min + 1)) + min;
    },

    /**
     * @description 创建光柱、文字坐标
     * @param {*} properties 属性、详情
     * @param {Function} projection  d3-geo转化坐标
     */
    initLightPoint(properties, depth) {
      // 创建光柱
      let heightScaleFactor = 2 + this.random(1, 5) / 5;
      let lightCenter = properties.centroid || properties.center;
      let areaName = properties.name;
      // projection用来把经纬度转换成坐标
      const [x, y] = projection(lightCenter);
      const light = this.createLightPillar(x, y, heightScaleFactor, depth);
      //light.position.z -= 3;
      //这里创建文字坐标
      const label = this.createTextPoint(x, y, areaName, depth);
      return [light, label];
    },

    /**
     * @description 创建文字坐标
     * @param {*} x d3 - 经纬度转换后的x轴坐标
     * @param {*} y d3 - 经纬度转换后的y轴坐标
     * @param {*} areaName 地名
     */
    createTextPoint(x, y, areaName, depth) {
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
    },

    /**
     * @description 文字坐标点击
     * @param {*} e
     */
    clickLabel(e) {
      console.log("🚀 ~ clickLabel ~ e:", e);
    },

    /**
     * @description 添加底部旋转背景
     */
    initFloor() {
      const geometry = new THREE.PlaneGeometry(80, 80);
      let texture = textureLoader.load(require("./mapimg/地板背景.png"));
      const material = new THREE.MeshPhongMaterial({
        color: 0xffffff,
        map: texture,
        // emissive:0xffffff,
        // emissiveMap:Texture,
        transparent: true,
        opacity: 1,
        depthTest: true,
        // roughness:1,
        // metalness:0,
        depthWrite: false,
        // side: THREE.DoubleSide
      });
      let plane = new THREE.Mesh(geometry, material);
      plane.rotateX(-Math.PI / 2);
      this.scene.add(plane);

      // 旋转装饰圆环1
      let rotatingApertureTexture = textureLoader.load(require("./mapimg/rotatingAperture.png"));
      let rotatingApertureerial = new THREE.MeshBasicMaterial({
        map: rotatingApertureTexture,
        transparent: true,
        opacity: 1,
        depthTest: true,
        depthWrite: false,
      });
      let rotatingApertureGeometry = new THREE.PlaneGeometry(25, 25);
      rotatingApertureMesh = new THREE.Mesh(rotatingApertureGeometry, rotatingApertureerial);
      rotatingApertureMesh.rotateX(-Math.PI / 2);
      rotatingApertureMesh.position.y = 0.02;
      rotatingApertureMesh.scale.set(1.2, 1.2, 1.2);
      this.scene.add(rotatingApertureMesh);

      // 旋转装饰圆环2
      let rotatingPointTexture = textureLoader.load(require("./mapimg/rotating-point2.png"));
      let material2 = new THREE.MeshBasicMaterial({
        map: rotatingPointTexture,
        transparent: true,
        opacity: 1,
        depthTest: true,
        depthWrite: false,
      });

      rotatingPointMesh = new THREE.Mesh(rotatingApertureGeometry, material2);
      rotatingPointMesh.rotateX(-Math.PI / 2);
      rotatingPointMesh.position.y = 0.04;
      rotatingPointMesh.scale.set(1, 1, 1);
      this.scene.add(rotatingPointMesh);

      // 背景小圆点
      let circlePoint = textureLoader.load(require("./mapimg/circle-point.png"));
      let material3 = new THREE.MeshPhongMaterial({
        color: 0x00ffff,
        map: circlePoint,
        transparent: true,
        opacity: 1,
        depthWrite: false,
        // depthTest: false,
      });
      let plane3 = new THREE.PlaneGeometry(30, 30);
      let mesh3 = new THREE.Mesh(plane3, material3);
      mesh3.rotateX(-Math.PI / 2);
      mesh3.position.y = 0.06;
      this.scene.add(mesh3);
    },

    //加事件
    setClickFn() {
      this.raycaster = new THREE.Raycaster();
      this.mouse = new THREE.Vector2();
      // 鼠标移动事件
      const onMouseMove = (event) => {
        console.log("🚀 ~ onMouseMove ~ event:", event);
        var marginLeft = this.contant.offsetLeft;
        var marginTop = this.contant.offsetTop + 92;
        // 如果该地图不是占满全屏需要减去margintop和marginleft
        // 将鼠标位置归一化为设备坐标。x 和 y 方向的取值范围是 (-1 to +1)
        // this.mouse.x = (event.clientX / this.contant.clientWidth) * 2 - 1;
        // this.mouse.y = -(event.clientY / this.contant.clientHeight) * 2 + 1;
        this.mouse.x = ((event.clientX - marginLeft) / this.contant.clientWidth) * 2 - 1;
        this.mouse.y = -((event.clientY - marginTop) / this.contant.clientHeight) * 2 + 1;

        event.preventDefault();
        this.raycaster.setFromCamera(this.mouse, this.camera);
        let intersects = this.raycaster.intersectObjects(this.scene.children, true);
        //this.previousObj.material[0].color = new THREE.Color(this.bgColor);
        if (intersects[0] && intersects[0].object) {
          console.log("🚀 ~ onMouseMove ~ intersects[0].object:", intersects[0].object);

          //intersects[0].object.material[0].color = new THREE.Color(0xffaa00);
          //this.previousObj = intersects[0].object; //previousObj保存悬浮的对象，鼠标移开后恢复颜色。
        }
      };

      let clickPosition;
      //window.addEventListener("mousemove", onMouseMove, false);
      // 鼠标点击事件
      const onclick = (event) => {
        var marginLeft = this.contant.offsetLeft;
        var marginTop = this.contant.offsetTop;
        // let x = (event.clientX / this.contant.clientWidth) * 2 - 1;
        // let y = -(event.clientY / this.contant.clientHeight) * 2 + 1;
        // 如果该地图不是占满全屏需要减去margintop和marginleft
        let x = ((event.clientX - marginLeft) / this.contant.clientWidth) * 2 - 1;
        let y = -((event.clientY - marginTop) / this.contant.clientHeight) * 2 + 1;
        clickPosition = { x: x, y: y };
        this.raycaster.setFromCamera(clickPosition, this.camera);
        // 算出射线 与当场景相交的对象有那些
        const intersects = this.raycaster.intersectObjects(this.scene.children, true);
        let clickObj = intersects.find((item) => item.object.material && item.object.material.length === 2);
        // 点击区县
        if (clickObj && clickObj.object) {
          console.log(clickObj);
          // this.$emit('clickAreaCheck',clickObj)
        }
      };
      window.addEventListener("mousedown", onclick, false);
    },

    // 设置最大旋转的角度
    setController() {
      const controls = new OrbitControls(this.camera, this.renderer.domElement);
      controls.maxPolarAngle = 2.5;
      controls.minPolarAngle = 1;
      controls.maxAzimuthAngle = 1;
      controls.minAzimuthAngle = -1;
      controls.addEventListener("change", () => {
        this.renderer.render(this.scene, this.camera);
      });
    },

    // 创建地图区块网格模型
    createMesh(data, depth) {
      // 创建一个多边形轮廓
      const shape = new THREE.Shape();

      // 循环多边形坐标数组  polygon是地图区块的经纬度坐标数组
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
      };
      // 将多边形 拉伸扫描成型  //shape二维轮廓  //extrudeSettings拉伸参数
      const geometry = new THREE.ExtrudeGeometry(shape, extrudeSettings);

      // 创建环境贴图
      let textureMap = textureLoader.load(require("./mapimg/gz-map.jpeg"));
      let texturefxMap = textureLoader.load(require("./mapimg/gz-map-fx.jpeg"));
      textureMap.wrapS = THREE.RepeatWrapping; //纹理水平方向的平铺方式
      textureMap.wrapT = THREE.RepeatWrapping; //纹理垂直方向的平铺方式
      textureMap.flipY = texturefxMap.flipY = false; // 如果设置为true，纹理在上传到GPU的时候会进行纵向的翻转。默认值为true。
      textureMap.rotation = texturefxMap.rotation = THREE.MathUtils.degToRad(45); //rotation纹理将围绕中心点旋转多少度
      const scale = 0.01;
      textureMap.repeat.set(scale, scale); //repeat决定纹理在表面的重复次数
      texturefxMap.repeat.set(scale, scale);
      textureMap.offset.set(0.5, 0.5); //offset贴图单次重复中的起始偏移量
      texturefxMap.offset.set(0.5, 0.5);

      // 创建高光材质
      const material = new THREE.MeshPhongMaterial({
        map: textureMap, //颜色贴图
        normalMap: texturefxMap, //用于创建法线贴图的纹理
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
    },

    // 创建地图区块边界线
    createLine(data, depth) {
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
      // setFromPoints方法从pointsArray中提取数据改变几何体的顶点属性vertices
      // 如果几何体是BufferGeometry，setFromPoints方法改变的是.attributes.position属性
      lineGeometry.setFromPoints(pointsArray);

      const line = new THREE.Line(lineGeometry, lineMaterial);
      line.position.z += depth + 0.001;
      return line;
    },

    // 设置地图中心位置
    setCenter(map) {
      map.rotation.x = -Math.PI / 2;
      const box = new THREE.Box3().setFromObject(map);
      const center = box.getCenter(new THREE.Vector3());
      const offset = [0, 0];

      map.position.x = map.position.x - center.x - offset[0];
      map.position.z = map.position.z - center.z - offset[1];
    },

    animate() {
      // 背景圆环 转动
      if (rotatingApertureMesh) {
        rotatingApertureMesh.rotation.z += 0.0005;
      }
      if (rotatingPointMesh) {
        rotatingPointMesh.rotation.z -= 0.0005;
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

      window.requestAnimationFrame(this.animate);
      // 通过摄像机和鼠标位置更新射线
      this.raycaster.setFromCamera(this.mouse, this.camera);
      this.labelRenderer.render(this.scene, this.camera);

      this.renderer.render(this.scene, this.camera);
    },
  },
};
</script>

<style>
.center-map-box {
  width: 100%;
  height: 100%;
  position: relative;
  font-size: 14px;
}
</style>
