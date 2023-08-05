<template>
    <canvas ref="canvasRef"></canvas>
  </template>
  
<script setup>
  import { ref, onMounted, onUnmounted } from "vue";
  import { OrbitControls } from "three/examples/jsm/controls/OrbitControls";
  import * as THREE from "three";
  
  let scene = new THREE.Scene();
  let renderer;
  let controls;
  let canvasRef = ref();
  
  // Box
  let boxGeometry = new THREE.BoxGeometry(1, 1, 1);
  let boxMaterial = new THREE.MeshPhongMaterial({ color: "mediumpurple" });
  let box = new THREE.Mesh(boxGeometry, boxMaterial);
  box.position.set(0, 0, -2);
  scene.add(box);
  
  
  
  let ambientLight = new THREE.DirectionalLight("#ffffff", 1);
  ambientLight.position.set(-1, 2, 4);
  scene.add(ambientLight);
  
  let camera = new THREE.PerspectiveCamera(
    75, //vertical field of view
    window.innerWidth / window.innerHeight, //aspect ratio
    0.1, //near plane
    100 //far plane
  );
  camera.position.y = 0;
  camera.position.z = 3;
  camera.position.x=0;
  camera.lookAt(new THREE.Vector3(0, 0, 0));
  scene.add(camera);
  
  let loop = () => {
    box.rotation.y += 0.02;
    box.rotation.x += 0.02;
    controls.update();
    renderer.render(scene, camera);
  };
  
  let resizeCallback = () => {
    renderer.setSize(window.innerWidth, window.innerHeight);
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
  };
  
  onMounted(() => {
    renderer = new THREE.WebGLRenderer({
      canvas: canvasRef.value,
      antialias: true,
      alpha: true,
    });
  
    renderer.setSize(window.innerWidth, window.innerHeight);
    renderer.setPixelRatio(window.devicePixelRatio);
    renderer.render(scene, camera);
  
    renderer.setAnimationLoop(loop);
  
    controls = new OrbitControls(camera, canvasRef.value);
    controls.enableDamping = true;
  
    window.addEventListener("resize", resizeCallback);
  });
  
  onUnmounted(() => {
    renderer.setAnimationLoop(null);
  });
  </script>
  
  <style>
    canvas {
      width: 100%;
      height: 100%;
      display: block;
    }
  </style>