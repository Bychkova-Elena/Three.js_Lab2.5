<template>
  <div id="app"></div>
</template>

<script>
import * as THREE from "three/build/three.module.js";
import { FlyControls } from "three/examples/jsm/controls/FlyControls.js";

export default {
  name: "App",
  mounted() {
    // Перемещение по сцене
    const OrbitControls = require("three-orbit-controls")(THREE);
    // Элементы skeleton
    const segmentHeight = 1;
    const segmentCount = 4;
    const height = segmentHeight * segmentCount;
    const halfHeight = height * 0.5;

    const sizing = {
      segmentHeight: segmentHeight,
      segmentCount: segmentCount,
      height: height,
      halfHeight,
    };

    // Установка сцены и камеры
    const scene = new THREE.Scene();
    const camera = new THREE.PerspectiveCamera(
      100,
      window.innerWidth / window.innerHeight,
      0.5,
      1000
    );

    // Добавление источника пассивного звучания Audio.
    const listener = new THREE.AudioListener();
    camera.add(listener);
    const sound = new THREE.Audio(listener);
    const audioLoader = new THREE.AudioLoader();
    audioLoader.load(require("../public/audio/music.mp3"), function (buffer) {
      sound.setBuffer(buffer);
      sound.setLoop(true); // бесконечная прокрутка аудио
      sound.setVolume(0.5); // громкость звука
      sound.play();
    });

    // Добавление элемента точечного звучания PositionalAudio, прикрепив его к одному из созданных элементов.
    const sound2 = new THREE.PositionalAudio(listener);
    const audioLoader2 = new THREE.AudioLoader();
    audioLoader2.load(
      require("../public/audio/music2.mp3"),
      function (buffer) {
        sound2.setBuffer(buffer);
        sound2.setLoop(true); // бесконечная прокрутка аудио
        sound2.setRefDistance(13);
        sound2.play();
      }
    );

    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);

    let controls2 = new FlyControls(camera, renderer.domElement);

    controls2.movementSpeed = 1000;
    controls2.domElement = renderer.domElement;
    controls2.rollSpeed = Math.PI / 24;
    controls2.autoForward = false;
    controls2.dragToLook = false;

    camera.position.x = -70;
    camera.position.z = 20;

    // Добавление возможности перемещения по сцены и возможность "смотреть по сторонам".
    const controls = new OrbitControls(camera, renderer.domElement);
    controls.update();

    // Добавление Skybox к сцене
    const loader = new THREE.CubeTextureLoader();
    const texture = loader.load([
      require("./assets/dark_ft.png"),
      require("./assets/dark_bk.png"),
      require("./assets/dark_up.png"),
      require("./assets/dark_dn.png"),
      require("./assets/dark_rt.png"),
      require("./assets/dark_lf.png"),
    ]);
    scene.background = texture;

    const lightHemisphere = new THREE.HemisphereLight(0xffffbb, 0x080820, 1);
    // Добавленией теней к элементам из ЛР4, используя атрибут CastShadow: true;
    lightHemisphere.castShadow = true;
    scene.add(lightHemisphere);

    const lightPoint = new THREE.PointLight(0xff0000, 5, 120);
    lightPoint.position.set(50, 20, 0);
    scene.add(lightPoint);

    // Полихедрон
    const verticesOfCube = [
    -1, -1, -1,    1, -1, -1,    1,  1, -1,    -1,  1, -1,
    -1, -1,  1,    1, -1,  1,    1,  1,  1,    -1,  1,  1,
];
const indicesOfFaces = [
    2, 1, 0,    0, 3, 2,
    0, 4, 7,    7, 3, 0,
    0, 1, 5,    5, 4, 0,
    1, 2, 6,    6, 5, 1,
    2, 3, 7,    7, 6, 2,
    4, 5, 6,    6, 7, 4,
];
const radius = 7;  // ui: radius
const detail = 2;  // ui: detail
const geometryPolyh = new THREE.PolyhedronGeometry(
    verticesOfCube, indicesOfFaces, radius, detail);
    const materialPolyh = new THREE.MeshBasicMaterial({ color: 0x00ccff });
    const polyh = new THREE.Mesh(geometryPolyh, materialPolyh);
    polyh.position.set(50, 0, 20);
    scene.add(polyh);

    // Икосаэдр
    const radiusIco = 7;  
    const geometryIco = new THREE.IcosahedronGeometry(radiusIco);
    const materialIco = new THREE.MeshBasicMaterial({ color: 0xcc00ff });
    const ico = new THREE.Mesh(geometryIco, materialIco);
    scene.add(ico);
    ico.position.set(30, 0, -50);
    scene.add(ico);

    // Куб
    const geometryBox = new THREE.BoxGeometry(20, 20, 20);
    const materialBox = new THREE.MeshBasicMaterial({ color: 0x808000 });
    const cube = new THREE.Mesh(geometryBox, materialBox);
    scene.add(cube);
    cube.position.set(80, 0, -50);
    scene.add(cube);

    var bones = [];
    var prevBone = new THREE.Bone();
    prevBone.position.y = -sizing.halfHeight;

    for (let i = 0; i <= sizing.segmentCount; i++) {
      const bone = new THREE.Bone();
      bone.position.y = sizing.segmentHeight;
      bones.push(bone);
      prevBone.add(bone);
      prevBone = bone;
    }

    const geometry = new THREE.BoxGeometry( 
          4, // radiusTop
          sizing.height, // height
          10, // radiusSegments
        );
    const position = geometry.attributes.position;
    const vertex = new THREE.Vector3();
    const skinIndices = [];
    const skinWeights = [];

    for (let i = 0; i < position.count; i++) {
      vertex.fromBufferAttribute(position, i);
      const y = vertex.y + sizing.halfHeight;
      const skinIndex = Math.floor(y / sizing.segmentHeight);
      const skinWeight = (y % sizing.segmentHeight) / sizing.segmentHeight;
      skinIndices.push(skinIndex, skinIndex + 1, 0, 0);
      skinWeights.push(1 - skinWeight, skinWeight, 0, 0);
    }

    geometry.setAttribute(
      "skinIndex",
      new THREE.Uint16BufferAttribute(skinIndices, 4)
    );
    geometry.setAttribute(
      "skinWeight",
      new THREE.Float32BufferAttribute(skinWeights, 4)
    );

    // Установка материала объекту
    const material = new THREE.MeshStandardMaterial({
      skinning: true,
      color: 0x3e1268,
      side: THREE.DoubleSide,
    });

    const mesh = new THREE.SkinnedMesh(geometry, material);
        mesh.scale.x = 0.3;
        mesh.scale.y = 0.6;
        mesh.scale.z = 0.5;

    // Собираем объект из костей
    const skeleton = new THREE.Skeleton(bones);

    bones[0].position.y = -sizing.halfHeight;
    mesh.add(bones[0]);
    mesh.bind(skeleton);

    let helper = new THREE.SkeletonHelper(mesh);
    helper.material.linewidth = 2;
    scene.add(helper);

    mesh.scale.multiplyScalar(1);
    scene.add(mesh);
    mesh.add(sound2);

    // Анимация объекта
          const animate = function () {
          requestAnimationFrame(animate);
          for (let i = 0; i <= sizing.segmentCount; i++) {
          bones[i].position.y +=0.001;
          bones[i].rotation.y +=0.005;
        }
          renderer.render(scene, camera);
        };

    animate();

    window.addEventListener(
      "resize",
      function () {
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
        renderer.setSize(window.innerWidth, window.innerHeight);
      },
      false
    );
  },
};
</script>

<style lang="scss">
body {
  margin: 0;
  padding: 0;
}
</style>
