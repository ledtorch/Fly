<template>
  <div class="frame" ref="frame" />
  <div ref="container" id="container" class="upper-container" />
  <div class="set">
    <button @click="drawFlightRoute" class="fetch">Curve</button>
    <button @click="removeFlightRoute" class="remove">Remove</button>
  </div>
</template>

<script>
import * as THREE from 'three';
import { OrbitControls } from 'three/addons/controls/OrbitControls.js';

export default {
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
        antialias: false,
        alpha: true,
        powerPreference: "high-performance"
      });
      this.renderer.setSize(width, height);
      container.appendChild(this.renderer.domElement);


      // Orbit controls
      const controls = new OrbitControls(this.camera, this.renderer.domElement);
      controls.update();

      // Lighting
      const light = new THREE.DirectionalLight(0xffffff, 1);
      light.position.set(1, 1, 1).normalize();
      this.scene.add(light);

      // Sphere
      this.createCoreSphere();
      this.createDotsSphere();

      // Resize listener
      window.addEventListener('resize', this.onWindowResize.bind(this));

      // Animation loop
      const animate = () => {
        requestAnimationFrame(animate);
        this.scene.rotation.y += 0.005;
        controls.update();
        this.renderer.render(this.scene, this.camera);
      };
      animate();
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

      this.scene.add(sceneDots);
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

      this.scene.add(coreSphere);
      // return coreSphere;
    },


    drawFlightRoute() {
      if (!this.scene) {
        console.error("Scene is not initialized");
        return;
      }
      if (this.curve) {
        this.scene.remove(this.curve);
      }
      this.curve = this.getCurve();
      this.scene.add(this.curve);
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

    getCurve(p1, p2) {
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

      // 🐞 Debug console
      console.log("Curve position:", curve.position);

      return curve;
    },
    removeFlightRoute() {
      if (this.curve) {
        this.scene.remove(this.curve);
        this.curve = null; // Clear the reference
      }
    },
    onWindowResize() {
      const width = window.innerWidth;
      const height = window.innerHeight;
      this.camera.aspect = width / height;
      this.camera.updateProjectionMatrix();
      this.renderer.setSize(width, height);
    },
  },
};
</script>

<style scoped>
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

/* .upper-container {
  pointer-events: none;
} */

.remove {
  flex: 1;
  left: 50%;
  bottom: 20px;
  /* Adjust as needed */
  transform: translateX(-50%);
  z-index: 2;
}

.set {
  display: flex;
  /* Enable Flexbox */
  justify-content: space-between;
  /* Space between items */
  position: absolute;
  gap: 20px;
  left: 55%;
  bottom: 50px;
  transform: translateX(-50%);
  z-index: 2;
}
</style>