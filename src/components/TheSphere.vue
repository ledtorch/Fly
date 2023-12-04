<template>
  <div class="frame" ref="frame" />
  <div ref="container" id="container" />
  <!-- <button @click="fetchFlightData" class="fetch">Fetch Data</button> -->
  <!-- <button @click="createTube" class="fetch">Draw TPE to JFK Route</button> -->
  <!-- <button @click="getcurve" class="fetch">Curve</button> -->
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

    // The scene of flights' routes
    this.initCurveScene();
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


      // Add curve
      const curve = this.getcurve();
      scene.add(curve);

      // Wrap model to a unit
      const earthGroup = new THREE.Group();
      // earthGroup.add(coreSphere);
      // earthGroup.add(dotsSphere);
      // earthGroup.add(sampleSphere);
      earthGroup.add(curve);
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
        earthGroup.rotation.y += 0.005;
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
      const DOT_COUNT = 32000;

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
      // ❓ Don't know how it project to the sphere. The coordinate is NOT correct.
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
      const radius = 0.01;
      const radialSegments = 8;
      const closed = false;

      // Create the TubeGeometry
      const geometry = new THREE.TubeGeometry(curve, tubularSegments, radius, radialSegments, closed);
      const material = new THREE.MeshBasicMaterial({ color: 0xff0000 }); // Choose your color
      const tube = new THREE.Mesh(geometry, material);

      return tube;
    },

    // Button and Draw Route
    // Method to create and add the curve to the scene

    initCurveScene() {
      // Initialize curve scene, camera, and renderer
      this.curveScene = new THREE.Scene();
      this.curveRenderer = new THREE.WebGLRenderer({ /* ... */ });
      // ... other initialization code for curve scene ...
      // ... append this.curveRenderer.domElement to another container or same with different settings ...
    },

    getcurve(p1, p2) {
      let TPE_point = {
        lat: 25.078193,
        lon: 121.235452
      }
      let JFK_point = {
        lat: 40.644763,
        lon: -73.779736
      }
      let startPoint = this.latLongToVector3(TPE_point.lat, TPE_point.lon, 2.48);
      let endPoint = this.latLongToVector3(JFK_point.lat, JFK_point.lon, 2.48);

      let v1 = new THREE.Vector3(startPoint.x, startPoint.y, startPoint.z);
      let v2 = new THREE.Vector3(endPoint.x, endPoint.y, endPoint.z);

      let points = [v1];
      for (let i = 1; i < 20; i++) {
        let interpolated = new THREE.Vector3().lerpVectors(v1, v2, i / 20);
        interpolated.normalize()
        // interpolated.multiplyScalar(2.7);
        interpolated.multiplyScalar(2.48 + 0.2 * Math.sin(Math.PI * i / 20));
        points.push(interpolated);
      }
      points.push(v2);

      let path = new THREE.CatmullRomCurve3(points);

      const geometry = new THREE.TubeGeometry(path, 20, 0.01, 10, false);
      const material = new THREE.MeshBasicMaterial({ color: 0xffffff });
      const curve = new THREE.Mesh(geometry, material);

      return curve;
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
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  z-index: 1;
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
}

.fetch {
  position: absolute;
  left: 50%;
  bottom: 50px;
  transform: translateX(-50%);
  z-index: 2;
}
</style>