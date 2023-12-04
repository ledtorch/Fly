<template>
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

      // Resize listener
      window.addEventListener('resize', this.onWindowResize.bind(this));

      // Animation loop
      const animate = () => {
        requestAnimationFrame(animate);
        this.renderer.render(this.scene, this.camera);
      };
      animate();
    },

    drawFlightRoute() {
      if (!this.scene) {
        console.error("Scene is not initialized");
        return;
      }
      if (this.curve) {
        this.scene.remove(this.curve); // Remove the existing curve if it exists
      }
      this.curve = this.getCurve(); // Update the reference to the new curve
      this.scene.add(this.curve);
    },

    latLongToVector3(lat, lon, radius) {
      var phi = (lat * Math.PI) / 180;
      var theta = ((lon - 180) * Math.PI) / 180;

      var x = -radius * Math.cos(phi) * Math.cos(theta);
      var y = radius * Math.sin(phi);
      var z = radius * Math.cos(phi) * Math.sin(theta);

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

.upper-container {
  pointer-events: none;
}

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