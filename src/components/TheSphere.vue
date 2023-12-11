<template>
  <div ref="container" id="container" class="upper-container" />
  <div class="number">There are {{ totalFlightsCount }} people flying with you</div>
  <div class="set">
    <input v-model="flightCode" placeholder="Enter Flight Code" class="input" />
    <button @click="drawFlightRoute" class="fetch">Curve</button>
    <!-- <button @click="removeFlightRoute" class="remove">Remove</button> -->
  </div>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';
import axios from 'axios';
import airportsData from '../airports.json';

export default {
  data() {
    return {
      flightCode: '',
      totalFlightsCount: 0,
    };
  },

  mounted() {
    this.initThree();
  },

  beforeDestroy() {
    window.removeEventListener('resize', this.onWindowResize);
  },

  methods: {
    initThree() {
      const container = this.$refs.container;
      const width = window.innerWidth;
      const height = window.innerHeight;

      // Create scene, camera, and renderer
      this.scene = new THREE.Scene();
      this.camera = new THREE.PerspectiveCamera(60, width / height, 0.1, 1000);
      this.camera.position.z = 6;

      this.renderer = new THREE.WebGLRenderer({
        antialias: true,
        alpha: true,
        powerPreference: "high-performance"
      });

      this.renderer.setSize(width, height);
      // Set a dark background color
      this.renderer.setClearColor(0x121215);

      container.appendChild(this.renderer.domElement);


      // Orbit controls
      const controls = new OrbitControls(this.camera, this.renderer.domElement);
      controls.update();

      // Directional Light - Sunlight
      const sunLight = new THREE.DirectionalLight(0xffffff, 1);
      sunLight.position.set(5, 3, 5);
      this.scene.add(sunLight);

      // Ambient Light
      const ambientLight = new THREE.AmbientLight(0x1D8EB8, 1);
      this.scene.add(ambientLight);

      // Stars background
      const starsGeometry = new THREE.BufferGeometry();
      const starsMaterial = new THREE.PointsMaterial({ color: 0xffffff, size: 1.7 });
      const starsVertices = [];

      for (let i = 0; i < 1000; i++) {
        const x = (Math.random() - 0.5) * 2000;
        const y = (Math.random() - 0.5) * 2000;
        const z = (Math.random() - 0.5) * 2000;
        starsVertices.push(x, y, z);
      }

      starsGeometry.setAttribute('position', new THREE.Float32BufferAttribute(starsVertices, 3));
      const starField = new THREE.Points(starsGeometry, starsMaterial);
      this.scene.add(starField);

      // Sphere
      this.createCoreSphere();
      this.createDotsSphere();

      // Resize listener
      window.addEventListener('resize', this.onWindowResize.bind(this));

      // Animation loop
      const animate = () => {
        requestAnimationFrame(animate);
        this.scene.rotation.y += 0.002;
        controls.update();
        this.renderer.render(this.scene, this.camera);
      };
      animate();
    },

    async createDotsSphere() {
      // Dots number
      const DOT_COUNT = 45000;

      // Load the color-coded image then get each pixel’s color
      const loader = new THREE.TextureLoader();
      const texture = await new Promise((resolve, reject) => {
        const imagePath = import.meta.env.DEV ? '/Image/map.png' : '/app/fly/Image/map.png';
        loader.load(
          imagePath,
          // onLoad callback
          resolve,
          // onError callback
          reject
        );
      });

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
        const pixelIndex = ((Math.floor(uv.y * (canvas.height - 180)) * canvas.width) + Math.floor(uv.x * (canvas.width))) * 4;
        const r = imageData.data[pixelIndex];
        const g = imageData.data[pixelIndex + 1];
        const b = imageData.data[pixelIndex + 2];

        // Map white area to represent land
        if (r === 0 && g === 0 && b === 0) {
          // Generate random opacity between 0.2 and 0.5
          const randomOpacity = 0.2 + Math.random() * 0.3;

          const dotMaterial = new THREE.MeshBasicMaterial({
            color: 0xffffff,
            opacity: randomOpacity,
            depthTest: false,
            transparent: true,
            side: THREE.BackSide,
          });

          const dotGeometry = new THREE.CircleGeometry(0.01, 5);
          const dot = new THREE.Mesh(dotGeometry, dotMaterial);
          dot.position.set(vector.x, vector.y, vector.z);
          dot.lookAt(new THREE.Vector3(0, 0, 0));
          sceneDots.add(dot);
        }
      }

      this.scene.add(sceneDots);
    },

    createCoreSphere() {
      const imagePath = import.meta.env.DEV ? '/Image/ocean.jpg' : '/app/fly/Image/ocean.jpg';

      // Load the texture
      const textureLoader = new THREE.TextureLoader();
      const sphereTexture = textureLoader.load(imagePath);

      const sphereMaterial = new THREE.MeshPhongMaterial({
        // Dark blue color to filter the texture
        color: 0x1E8CE7,
        map: sphereTexture,
        shininess: 0,
        depthTest: true,
      });

      const sphereRadius = 2.48;
      const sphereGeometry = new THREE.SphereGeometry(sphereRadius, 32, 32);
      const coreSphere = new THREE.Mesh(sphereGeometry, sphereMaterial);

      this.scene.add(coreSphere);

      // Atmosphere
      // Vertex shader
      const vertexShader =
        `
        varying vec3 vertexNormal;

        void main() {
          vertexNormal = normalize(normalMatrix * normal);
          gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
        }
      `;

      // Fragment shader
      const fragmentShader =
        `
        varying vec3 vertexNormal;

        void main() {
          float intensity = pow(0.7 - dot(vertexNormal, vec3(0, 0, 1)), 2.0);
          gl_FragColor = vec4(0.0, 0.5, 1.0, 1.0) * intensity;
        }
      `;

      // Create the atmosphere material
      const atmosphereMaterial = new THREE.ShaderMaterial({
        vertexShader: vertexShader,
        fragmentShader: fragmentShader,
        side: THREE.BackSide,
        blending: THREE.AdditiveBlending,
        transparent: true
      });

      // Create the atmosphere sphere
      const atmosphereGeometry = new THREE.SphereGeometry(2.7, 32, 32); // Slightly larger than the Earth sphere
      const atmosphere = new THREE.Mesh(atmosphereGeometry, atmosphereMaterial);
      this.scene.add(atmosphere);
    },

    async drawFlightRoute() {
      console.log("Flight Code:", this.flightCode);

      // Airlabs API key and base URL for a specific flight search
      let apiKey = "37754b2f-3224-4908-b749-00973379bb93";
      let flightUrl = `https://airlabs.co/api/v9/flight?flight_iata=${this.flightCode}&api_key=${apiKey}`;
      let flightsUrl = `https://airlabs.co/api/v9/flights?api_key=${apiKey}`;

      try {
        // Fetch specific flight data using the Airlabs API
        const flightResponse = await axios.get(flightUrl);
        // Fetch total flights data
        const totalFlightsResponse = await axios.get(flightsUrl);
        const flights = totalFlightsResponse.data.response;

        // Parse total flights count
        // Note: This needs adjustment based on the actual data structure returned by the API
        this.totalFlightsCount = (flights.length * 99).toLocaleString(); // Update this logic based on API response

        console.log("Flight URL:", flightUrl);
        console.log("Flights URL:", flightsUrl);
        console.log("flightResponse:", flightResponse.data);
        console.log("Flights:", flights.length);

        if (flightResponse.data && flightResponse.data.response) {
          const flight = flightResponse.data.response;

          // Retrieve airport data from JSON using ICAO codes
          const departureAirport = airportsData[flight.dep_icao];
          const arrivalAirport = airportsData[flight.arr_icao];

          console.log("Departure Airport Lat/Lon:", departureAirport.lat, departureAirport.lon);
          console.log("Arrival Airport Lat/Lon:", arrivalAirport.lat, arrivalAirport.lon);

          if (departureAirport && arrivalAirport) {
            let startPoint = this.latLongToVector3(departureAirport.lat, departureAirport.lon, 2.48);
            let endPoint = this.latLongToVector3(arrivalAirport.lat, arrivalAirport.lon, 2.48);

            this.curve = this.getCurve(startPoint, endPoint);
            this.scene.add(this.curve);
          }
        } else {
          console.log("No flight data found for this code.");
        }
      } catch (error) {
        console.error("Error fetching flight data from Airlabs:", error);
      }
    },

    latLongToVector3(lat, lon, radius) {
      // ❓ Don't know how it project to the sphere. The coordinate is NOT correct.
      var phi = (90 - lat) * (Math.PI / 180);
      var theta = (lon + 270) * (Math.PI / 180);

      var x = -radius * Math.sin(phi) * Math.cos(theta);
      var y = radius * Math.cos(phi);
      var z = radius * Math.sin(phi) * Math.sin(theta);

      return new THREE.Vector3(x, y, z);
    },

    getRandomLightColor() {
      // Full range of hue
      const hue = Math.random();
      // Higher saturation avoids pale colors
      const saturation = 0.7 + Math.random() * 0.3;
      // Adjust lightness to avoid white and very light colors
      const lightness = 0.5 + Math.random() * 0.4;

      return new THREE.Color(`hsl(${hue * 360}, ${saturation * 100}%, ${lightness * 100}%)`);
    },


    getCurve(startPoint, endPoint) {
      // Generate a random light color
      const randomColor = this.getRandomLightColor();

      // Create rings at the start and end points
      const ringGeometry = new THREE.RingGeometry(0.02, 0.025, 32); // Adjust size as needed
      const ringMaterial = new THREE.MeshBasicMaterial({ color: randomColor, side: THREE.DoubleSide });

      const startRing = new THREE.Mesh(ringGeometry, ringMaterial);
      startRing.position.copy(startPoint);
      startRing.lookAt(new THREE.Vector3()); // Oriented towards the center of the sphere

      const endRing = new THREE.Mesh(ringGeometry, ringMaterial);
      endRing.position.copy(endPoint);
      endRing.lookAt(new THREE.Vector3()); // Oriented towards the center of the sphere

      // Create ring-points
      const pointGeometry = new THREE.SphereGeometry(0.01, 32, 32); // Sphere geometry for the points
      const pointMaterial = new THREE.MeshBasicMaterial({ color: randomColor });

      const startPointSphere = new THREE.Mesh(pointGeometry, pointMaterial);
      startPointSphere.position.copy(startPoint);

      const endPointSphere = new THREE.Mesh(pointGeometry, pointMaterial);
      endPointSphere.position.copy(endPoint);

      // Create the curve
      let points = [startPoint];
      for (let i = 1; i < 20; i++) {
        let interpolated = new THREE.Vector3().lerpVectors(startPoint, endPoint, i / 20);
        interpolated.normalize();
        interpolated.multiplyScalar(2.48 + 0.2 * Math.sin(Math.PI * i / 20));
        points.push(interpolated);
      }
      points.push(endPoint);

      let path = new THREE.CatmullRomCurve3(points);
      const geometry = new THREE.TubeGeometry(path, 20, 0.005, 10, false);
      const curveMaterial = new THREE.MeshBasicMaterial({ color: randomColor });
      const curve = new THREE.Mesh(geometry, curveMaterial);

      // Add rings, points, and curve to the scene
      this.scene.add(startRing);
      this.scene.add(endRing);
      this.scene.add(startPointSphere);
      this.scene.add(endPointSphere);
      this.scene.add(curve);

      // Store references to all created objects for later removal
      this.curveComponents = {
        curve,
        startRing,
        endRing,
        startPointSphere,
        endPointSphere
      };

      return curve;
    },

    removeFlightRoute() {
      if (this.curveComponents) {
        // Remove all components of the curve
        this.scene.remove(this.curveComponents.curve);
        this.scene.remove(this.curveComponents.startRing);
        this.scene.remove(this.curveComponents.endRing);
        this.scene.remove(this.curveComponents.startPointSphere);
        this.scene.remove(this.curveComponents.endPointSphere);

        // Clear the reference
        this.curveComponents = null;
      }
    },

    // onWindowResize() {
    //   const width = window.innerWidth;
    //   const height = window.innerHeight;
    //   this.camera.aspect = width / height;
    //   this.camera.updateProjectionMatrix();
    //   this.renderer.setSize(width, height);
    // },

    onWindowResize() {
      const width = window.innerWidth;
      const height = window.innerHeight;

      // Adjust the field of view for mobile devices
      if (width <= 430) {
        this.camera.fov = 105;
      } else if (430 < width < 768) {
        this.camera.fov = 75;
      } else {
        this.camera.fov = 40;
      }

      this.camera.aspect = width / height;
      this.camera.updateProjectionMatrix();
      this.renderer.setSize(width, height);
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

.fetch {
  /* position: absolute; */
  flex: 1;
  left: 50%;
  bottom: 50px;
  transform: translateX(-50%);
  z-index: 2;
}

.remove {
  flex: 1;
  left: 50%;
  bottom: 20px;
  transform: translateX(-50%);
  z-index: 2;
}

.set {
  display: flex;
  justify-content: space-between;
  position: absolute;
  gap: 20px;
  left: 52%;
  bottom: 50px;
  transform: translateX(-50%);
  z-index: 2;
}

.number {
  display: flex;
  justify-content: space-between;
  position: absolute;
  gap: 20px;
  left: 50%;
  top: 50px;
  transform: translateX(-50%);
  z-index: 2;

  color: rgba(255, 255, 255, 0.9);
}

.input {
  flex: 1;
  left: 50%;
  bottom: 20px;
  transform: translateX(-20%);
  z-index: 2;
  display: inline-flex;
  border-radius: 5px;
  border: 0px solid;

  color: rgba(255, 255, 255, 0.9);
  background: rgba(255, 255, 255, 0.10);

  display: flex;
  padding: 0px 8px 0px 8px;
  align-items: center;
}

.input::placeholder {
  color: rgba(255, 255, 255, 0.5);
}
</style>