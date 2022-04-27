<template>
  <div class="container" ref="container">
    <a-button type="primary" class="btn-a" @click="switchAction()"
      >下一段动画</a-button
    >
  </div>
</template>


<script setup>
import { ref, reactive, onMounted, computed, toRaw } from "vue";
import * as Three from "three";
import { GLTFLoader } from "three/examples/jsm/loaders/GLTFLoader.js";
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
import {
  CSS2DObject,
  CSS2DRenderer,
} from "three/examples/jsm/renderers/CSS2DRenderer";
import WebGL from "three/examples/jsm/capabilities/WebGL";

const container = ref();
/**
 * threejs时钟对象
 */
let clock;

/**
 * Three场景对象
 */
let scene;

/**
 * 相机对象
 */
let camera;

/**
 * 渲染器
 */
let renderer;

/**
 * Html渲染器（贴HTML到3D模型）
 */
let labelRenderer;

/**
 * 剪辑器
 */
let mixer;

/**
 * 缓存栏杆模型的全部动画
 */

let animationTlangan = [];

let actionsIndex = 0;

/**
 * 设置场景
 */
const setScene = () => {
  scene = new Three.Scene();
  // scene.background = new Three.Color("skyblue");
};

/**
 * 给场景添加坐标轴
 */
const addAxesHelper = () => {
  var axis = new Three.AxesHelper(3);
  scene.add(axis);
};

/**
 * 创建一个相机
 */
const setCamera = () => {
  //先创建相机
  camera = new Three.PerspectiveCamera(
    75,
    container.value.clientWidth / container.value.clientHeight,
    2,
    1000
  );
  //设置相机位置
  camera.position.set(6, 19, 73);
  //设置相机注视点 Vector3:三维坐标系
  camera.lookAt(new Three.Vector3(0, 0, 0));
};

/**
 * 创建光源
 */
const createLight = () => {
  var ambient = new Three.AmbientLight(0xffffff);
  scene.add(ambient);
};

/**
 * 创建ThreeJS渲染器
 */
const setRender = () => {
  //初始化渲染器 开启抗锯齿 允许透明
  renderer = new Three.WebGLRenderer({ antialias: true, alpha: true });
  //设置渲染器场景大小和容器等同
  renderer.setSize(container.value.clientWidth, container.value.clientHeight);
  //把场景丢进容器内
  container.value.appendChild(renderer.domElement);

  //初始化HTML贴图渲染器
  labelRenderer = new CSS2DRenderer();
  //设置贴图器的大小
  labelRenderer.setSize(
    container.value.clientWidth,
    container.value.clientHeight
  );
  //设置贴图样式
  labelRenderer.domElement.style.position = "absolute";
  labelRenderer.domElement.style.top = "0px";
  labelRenderer.domElement.style.pointerEvents = "none";
  //把贴图塞到HTML种
  container.value.appendChild(labelRenderer.domElement);

  // 每个可用帧都会调用的函数。 如果传入‘null’,所有正在进行的动画都会停止。
  renderer.setAnimationLoop(() => {
    //渲染！
    render();
    //getDelta
    //获取自 oldTime 设置后到当前的秒数。 同时将 oldTime 设置为当前时间。
    //如果 autoStart 设置为 true 且时钟并未运行，则该方法同时启动时钟。
    var delta = clock.getDelta();

    //暂无动画 先注释
    // if (carMixer) carMixer.update(delta);
    // for (var key in railingMixerGroup) {
    //   if (railingMixerGroup[key] && (key == "railing1" || key == "railing4"))
    //     railingMixerGroup[key].update(delta);
    // }
  });
};

/**
 * 渲染
 */
const render = () => {
  if (mixer) {
    const delta = clock.getDelta(); // 获取自上次调用的时间差
    mixer.update(delta);
  }

  renderer.render(scene, camera);
  if (labelRenderer) {
    labelRenderer.render(scene, camera);
  }
};

/**
 * 加载检查站模型
 * @param{any}loader:模型加载器实例
 */
const loadCheckPoint = (loader) => {
  let path = "/model/liyanga_tianmuhu/liyanga_tianmuhu.gltf";
  // let path = "/model/liyangatianmuhuani/liyangatianmuhuani.gltf"
  //加载本地模型
  loader.load(
    path,
    (gltf) => {
      console.log(gltf, "检查站模型");
      //把模型丢到大场景
      scene.add(gltf.scene);
      //缓存动画
    },
    (xhr) => {
      let loaded = (xhr.loaded / xhr.total) * 100;
      console.log(loaded + "% loaded");
      if (loaded === 100) {
        loadEnclosure(loader);
      }
    },
    (error) => {
      console.error("检查站模型加载出错", error);
    }
  );
};

/**
 * 加载栏杆模型
 * @param{any}loader:模型加载器实例
 */

const loadEnclosure = (loader) => {
  let path = "/model/liyangatianmuhuani/liyangatianmuhuani.gltf";
  loader.load(
    path,
    (gltf) => {
      console.log(gltf, "栏杆模型");
      scene.add(gltf.scene);
      console.log(scene, "场景对象");
      //浅写一下动画
      //先初始化剪辑器
      mixer = new Three.AnimationMixer(gltf.scene);
      //便利动画列表 把动画缓存到数组
      //剪辑动画 用动画剪辑对象创建AnimationAction对象
      animationTlangan = gltf.animations.map((x) => {
        return mixer.clipAction(x);
      });
      //设置播放速度
      mixer.timeScale = 1;
      //设置单此循环的持续时间(setDuration也可以调节播放速度)并开始动画
      animationTlangan[0].setDuration(10).play();
    },
    (xhr) => {
      console.log((xhr.loaded / xhr.total) * 100 + "% loaded");
    },
    (error) => {
      console.error("栏杆加载出错", error);
    }
  );
};

/**
 * 视口变化时重绘视图
 */
const onWindowResize = () => {
  // 初始化摄像机
  camera.aspect = container.value.clientWidth / container.value.clientHeight;
  camera.updateProjectionMatrix();

  // 初始化渲染器尺寸
  renderer.setSize(container.value.clientWidth, container.value.clientHeight);
  if (labelRenderer) {
    labelRenderer.setSize(
      container.value.clientWidth,
      container.value.clientHeight
    );
  }
};

/**
 * 动画切换
 */

const switchAction = () => {
  //先停止当前预定动画
  mixer.stopAllAction();
  if (actionsIndex < animationTlangan.length - 1) {
    actionsIndex++;
  } else {
    actionsIndex = 0;
  }
  console.log(actionsIndex,"当前播放的第几段动画")
  mixer.timeScale = 1;
  animationTlangan[actionsIndex].setDuration(10).play();
};

onMounted(async () => {
  //先检测浏览器是否支持threejs
  if (!WebGL.isWebGLAvailable()) {
    const warning = WebGL.getWebGLErrorMessage();
    container.value.appendChild(warning);
    return;
  }
  //创建一个时钟对象
  clock = new Three.Clock();
  setScene();
  setCamera();
  addAxesHelper();
  createLight();
  setRender();

  //创建控件对象
  var controls = new OrbitControls(camera, renderer.domElement);
  //设置控制器可以向内向外移动的上限
  controls.minDistance = 20;
  controls.maxDistance = 200;

  //初始化一个模型加载器
  const loader = new GLTFLoader();
  //加载检查站模型
  loadCheckPoint(loader);

  //自适应
  window.addEventListener("resize", onWindowResize);
});
</script>

<style lang="less" scoped>
.container {
  width: 100%;
  height: 100%;
  // background: #07cbc9;
  position: relative;
  .btn-a {
    position: absolute;
    top: 10px;
    right: 10px;
    z-index: 999;
  }
}
</style>