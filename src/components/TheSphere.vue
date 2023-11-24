<template>
  <div class="frame" ref="frame" />
  <div ref="container" id="container" />
  <!-- <button @click="fetchFlightData" class="fetch">Fetch Data</button> -->
  <button @click="createTube" class="fetch">Draw TPE to JFK Route</button>

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


      // Add coreSphere
      const tube = this.createTube();
      scene.add(tube);

      // Wrap model to a unit
      const earthGroup = new THREE.Group();
      // earthGroup.add(coreSphere);
      // earthGroup.add(dotsSphere);
      // earthGroup.add(sampleSphere);
      earthGroup.add(tube);
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


    // Button and Draw Route
    // Method to create and add the tube to the scene

    latLongToVector3(lat, lon, radius) {
      var phi = (90 - lat) * (Math.PI / 180);
      var theta = (lon + 270) * (Math.PI / 180);

      var x = -radius * Math.sin(phi) * Math.cos(theta);
      var y = radius * Math.cos(phi);
      var z = radius * Math.sin(phi) * Math.sin(theta);

      return new THREE.Vector3(x, y, z);
    },

    createTube() {
      // Create the curve with control points
      // const start = new THREE.Vector3(-5, 0, 0);
      // const end = new THREE.Vector3(5, 0, 0);

      const sphereRadius = 2.48; // Sphere radius
      const TPE_Lat = 25.078193; // Latitude for Taipei
      const TPE_Lon = 121.235452; // Longitude for Taipei
      const JFK_Lat = 40.644763; // Latitude for New York JFK
      const JFK_Lon = -73.779736; // Longitude for New York JFK
      const control = new THREE.Vector3(0, 5, 0); // Control point for the curve

      // Convert latitude and longitude to 3D points on the sphere
      const start = this.latLongToVector3(TPE_Lat, TPE_Lon, sphereRadius);
      const end = this.latLongToVector3(JFK_Lat, JFK_Lon, sphereRadius);

      // Create the curve
      const curve = new THREE.QuadraticBezierCurve3(start, control, end);

      // Define TubeGeometry parameters
      const tubularSegments = 20;
      const radius = 0.1;
      const radialSegments = 8;
      const closed = false;

      // Create the TubeGeometry
      const geometry = new THREE.TubeGeometry(curve, tubularSegments, radius, radialSegments, closed);
      const material = new THREE.MeshBasicMaterial({ color: 0xff0000 }); // Choose your color
      const tube = new THREE.Mesh(geometry, material);

      return tube;
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