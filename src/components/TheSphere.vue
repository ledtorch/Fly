<template>
  <div class="frame" ref="frame" />
  <div ref="container" id="container" />
  <button @click="fetchFlightData" class="fetch">Fetch Data</button>
  <!-- <button @click="drawFlightRoute(37.7749, -122.4194, 25.0330, 121.5654)" class="fetch">Draw Flight Route</button> -->
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import * as BufferGeometryUtils from 'three/addons/utils/BufferGeometryUtils.js';
import axios from 'axios';
import { TubeGeometry } from 'three';

export default {
  mounted() {
    this.initThree();
  },
  beforeDestroy() {
    window.removeEventListener('resize', this.onWindowResize);
  },
  methods: {
    async initThree() {
      const container = this.$refs.container;
      const width = window.innerWidth;
      const height = window.innerHeight;

      // Create scene, camera and renderer
      const scene = new THREE.Scene();

      const camera = new THREE.PerspectiveCamera(60, width / height, 0.1, 1000);
      camera.position.z = 6;

      const renderer = new THREE.WebGLRenderer({
        antialias: false,
        alpha: true,
        powerPreference: "high-performance"
      });
      renderer.setSize(width, height);
      container.appendChild(renderer.domElement);

      // Create orbit camera
      const controls = new OrbitControls(camera, renderer.domElement);
      controls.update();

      // Add coreSphere
      const coreSphere = this.createCoreSphere();
      scene.add(coreSphere);

      // Add dotsSphere
      const dotsSphere = this.createDotsSphere();
      scene.add(dotsSphere);

      // Wrap model to a unit
      const earthGroup = new THREE.Group();
      // earthGroup.add(coreSphere);
      // earthGroup.add(dotsSphere);
      // earthGroup.add(sampleSphere);
      scene.add(earthGroup);

      try {
        const dotsSphere = await this.createDotsSphere();
        earthGroup.add(dotsSphere);
      } catch (error) {
        console.error('Error creating dots sphere:', error);
      }

      // Light
      const light = new THREE.DirectionalLight(0xffffff, 1);
      light.position.set(1, 1, 1).normalize();
      scene.add(light);

      // Resize listener
      window.addEventListener('resize', this.onWindowResize.bind(this, camera, renderer));

      // Animation loop
      const animate = () => {
        requestAnimationFrame(animate);
        earthGroup.rotation.y += 0.007;
        renderer.render(scene, camera);
      };
      animate();

      // // 🐞 Debug console
      // console.log('🐞 Debug console');
      // console.log(dotsSphere);
      // console.log(coreSphere);
      // console.log(sampleSphere);
      // console.log(renderer);
    },

    async createDotsSphere() {
      // Dots number
      const DOT_COUNT = 30000;

      const dotMaterial = new THREE.MeshBasicMaterial({
        color: 0x1B385Bff,
        depthTest: false,
        transparent: true,
        side: THREE.BackSide,
      });

      // Load the color-coded image then get each pixel’s color
      const loader = new THREE.TextureLoader();
      const texture = await new Promise((resolve, reject) => {
        loader.load(
          '/Image/map.png',
          resolve,  // onLoad callback
          reject  // onError callback
        );
      });
      // const texture = loader.load('/Image/map.png');

      const image = texture.image;
      const canvas = document.createElement('canvas');
      canvas.width = image.width;
      canvas.height = image.height;
      const context = canvas.getContext('2d');
      context.drawImage(image, 0, 0);
      const imageData = context.getImageData(0, 0, canvas.width, canvas.height);

      const vector = new THREE.Vector3();
      const sceneDots = new THREE.Group();

      for (let i = DOT_COUNT; i >= 0; i--) {
        const phi = Math.acos(-1 + (2 * i) / DOT_COUNT);
        const theta = Math.sqrt(DOT_COUNT * Math.PI) * phi;
        vector.setFromSphericalCoords(2.5, phi, theta);

        // Convert spherical coordinates to UV coordinates
        const uv = {
          x: ((theta + Math.PI) / (2 * Math.PI)),
          y: phi / Math.PI
        };

        // Get the corresponding pixel color on the map
        // 🏗️ TODO: The issue with UV mapping requires the app to start rendering from "canvas.height - 150" instead of 0
        const pixelIndex = ((Math.floor(uv.y * (canvas.height - 150)) * canvas.width) + Math.floor(uv.x * (canvas.width))) * 4;
        const r = imageData.data[pixelIndex];
        const g = imageData.data[pixelIndex + 1];
        const b = imageData.data[pixelIndex + 2];

        // Map white area to represent land
        if (r === 0 && g === 0 && b === 0) {
          const dotGeometry = new THREE.CircleGeometry(0.02, 5);
          const dot = new THREE.Mesh(dotGeometry, dotMaterial);
          dot.position.set(vector.x, vector.y, vector.z);
          dot.lookAt(new THREE.Vector3(0, 0, 0));
          sceneDots.add(dot);
        }
      }

      return sceneDots;
    },

    createCoreSphere() {
      /* 
       Opaque objects are drawn first, followed by transparent objects.
       Therefore, set transparent: true for all objects in this project.
      */
      const sphereMaterial = new THREE.MeshPhongMaterial({
        color: 0x162D4C,
        transparent: true,
        opacity: 0.9,
        depthTest: true,
      });

      const sphereRadius = 2.48;
      const sphereGeometry = new THREE.SphereGeometry(sphereRadius, 32, 32);
      const coreSphere = new THREE.Mesh(sphereGeometry, sphereMaterial);

      return coreSphere;
    },


    async fetchFlightData() {

      const options = {
        method: 'GET',
        url: 'https://timetable-lookup.p.rapidapi.com/airports/',
        headers: {
          'X-RapidAPI-Key': 'f3c001f042msh6992a42cead98c0p14099fjsn54a08a21b1ee',
          'X-RapidAPI-Host': 'timetable-lookup.p.rapidapi.com'
        }
      };

      try {
        const response = await axios.request(options);
        console.log(response.data);
      } catch (error) {
        console.error(error);
      }
    },



    createCurve(startLat, startLon, endLat, endLon, radius) {
      const startXYZ = toXYZ(startLat, startLon, radius);
      const endXYZ = toXYZ(endLat, endLon, radius);

      // Calculate intermediate control points for the bezier curve
      const arcHeight = startXYZ.distanceTo(endXYZ) * 0.5 + radius;
      const controlXYZ1 = toXYZ((startLat + endLat) / 2, (startLon + endLon) / 2, arcHeight);
      const controlXYZ2 = toXYZ((startLat + endLat) / 2, (startLon + endLon) / 2, arcHeight);

      return new THREE.CubicBezierCurve3(startXYZ, controlXYZ1, controlXYZ2, endXYZ);
    },

    drawFlightRoute(startLat, startLon, endLat, endLon) {
      if (!this.scene) {
        console.error('The scene is not initialized.');
        return;
      }

      const radius = 2.5; // Assuming your globe has a radius of 2.5 units
      const curve = this.createCurve(startLat, startLon, endLat, endLon, radius);

      // Create the TubeBufferGeometry based on the curve
      const geometry = new THREE.TubeBufferGeometry(curve, 44, 0.5, 8);
      const material = new THREE.MeshBasicMaterial({ color: 0xff0000 });
      const mesh = new THREE.Mesh(geometry, material);
      this.scene.add(mesh);

      // Animate the line drawing
      geometry.setDrawRange(0, 1); // Start with the first vertex
      this.drawAnimatedLine(geometry);
    },

    drawAnimatedLine(geometry) {
      const startTime = performance.now();
      const drawAnimated = () => {
        const drawRangeCount = geometry.drawRange.count;
        const timeElapsed = performance.now() - startTime;

        // Animate the curve for 2.5 seconds
        const progress = timeElapsed / 2500;
        const newDrawRangeCount = progress * 3000; // Adjust this number based on your geometry

        if (progress < 1) {
          geometry.setDrawRange(0, newDrawRangeCount);
          requestAnimationFrame(drawAnimated);
        }
      };

      requestAnimationFrame(drawAnimated);
    },

    onWindowResize(camera, renderer) {
      const width = window.innerWidth;
      const height = window.innerHeight;
      camera.aspect = width / height;
      camera.updateProjectionMatrix();
      renderer.setSize(width, height);
    },
  },
};
</script>

<style scoped>
#container {
  position: absolute;
  /* Changed from fixed to absolute */
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
  /* Ensure this element is rendered on top of .frame */
}

.frame {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: url('/Image/Nebula.jpg') no-repeat center center fixed;
  background-size: cover;
  z-index: -1;
  /* Ensure this element is rendered behind #container */
}

.fetch {
  position: absolute;
  /* Position the button absolutely within the container */
  left: 50%;
  /* Center the button horizontally */
  bottom: 50px;
  /* 50px from the bottom of the container */
  transform: translateX(-50%);
  /* Offset the button by half its width to center it */
  z-index: 2;
  /* Ensure it's above other elements */
  /* Add additional styling for the button here */
}
</style>