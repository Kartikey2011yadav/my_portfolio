<template>
  <div
    ref="container"
    @mouseenter="handleMouseEnter"
    @mouseleave="handleMouseLeave"
    @mousemove="handleMouseMove"

    @touchstart="handleMouseEnter"
    @touchend="handleMouseLeave"
    @touchmove="handleTouchMove"
  />
</template>

<script>
import { ref, onMounted } from 'vue';

import {
  PerspectiveCamera,
  Scene,
  DirectionalLight,
  TextureLoader,
  VideoTexture,
  MeshLambertMaterial,
  LinearFilter,
  Mesh,
  WebGLRenderer,
  Vector3,
  Box3,
  Object3D,

} from 'three';
import { OBJLoader } from './OBJLoader';
import MockupModel from './MockupModel';
import roundedPlane from './roundedPlane';
const phoneObj = new URL('./iphone.obj', import.meta.url).href;

export default {
  name: 'Mockup',
  props: {
    screen: {
      type: null,
    },
    lightClr: {
      type: String,
      default: 'white',
    },
    phoneClr: {
      type: String,
      default: 'white',
    },
    position: {
      type: Object,
      default: () => ({}),
    },
    rotation: {
      type: Object,
      default: () => ({}),
    },
    linearFilter: {
      type: Boolean,
    },
  },
  setup(props) {
    const container = ref(null);
    let camera;
    let scene;
    const phones = [];
    let renderer;
    let mouseX = 0;
    let mouseY = 0;

    function init() {
      const environmentInit = () => {
        camera = new PerspectiveCamera(
          45, container.value.clientWidth / container.value.clientHeight, 0.1, 10000,
        );
        scene = new Scene();

        const light = new DirectionalLight(props.lightClr);
        scene.add(light);

        light.position.set(0, 0, 300);
        camera.position.set(0, 0, 200);
      };

      const phoneInit = (screenSrc, home) => {
        const phone = new MockupModel({
          position: {
            x: 0,
            y: 0,
            z: 0,
            ...home.position,
          },
          rotation: {
            x: -0.2,
            y: 0.3,
            z: 0.06,
            ...home.rotation,
          },
        });

        const screenInit = () => {
          const scale = 6;
          const width = scale * 9; const height = scale * 19.3;
          const radius = 8;

          // const geometry = new THREE.PlaneGeometry(width, height);
          const geometry = roundedPlane(width, height, radius);

          let texture;

          if (typeof screenSrc === 'string') {
            const loader = new TextureLoader();
            texture = loader.load(screenSrc);
          } else {
            texture = new VideoTexture(screenSrc);
          }

          texture.anisotropy = renderer.capabilities.getMaxAnisotropy();

          const material = new MeshLambertMaterial({ map: texture });

          // if (props.linearFilter) {
          //   material.map.minFilter = LinearFilter;
          // }
          if (true) {
            material.map.minFilter = LinearFilter;
          }

          const screen = new Mesh(geometry, material);

          const recomputeUVs = () => {
            const box = new Box3().setFromObject(screen);
            const size = new Vector3();
            box.getSize(size);
            const vec3 = new Vector3();
            const attPos = screen.geometry.attributes.position;
            const attUv = screen.geometry.attributes.uv;
            for (let i = 0; i < attPos.count; i += 1) {
              vec3.fromBufferAttribute(attPos, i);
              attUv.setXY(i,
                (vec3.x - box.min.x) / size.x,
                (vec3.y - box.min.y) / size.y);
            }
            // attUv.needsUpdate = true;
          };

          recomputeUVs();

          screen.translateZ(3.6);
          screen.geometry.center();
          phone.add(screen);
        };

        const bodyInit = () => {
          const loader = new OBJLoader();
          loader.load(
            phoneObj,
            (body) => {
              const bodyGroup = new Object3D();
              body.traverse((child) => {
                if (child instanceof Mesh) {
                  child.material = new MeshLambertMaterial({ color: props.phoneClr });
                  child.geometry.center();
                  const mesh = new Mesh(child.geometry, child.material);
                  const scale = 8.6;
                  mesh.rotateX(Math.PI / 2);
                  mesh.scale.set(-scale, scale, scale);
                  bodyGroup.add(mesh);
                }
              });

              phone.add(bodyGroup);
            },
          );
        };

        phone.startFloat();
        scene.add(phone);
        screenInit();
        bodyInit();

        return phone;
      };

      renderer = new WebGLRenderer({ antialias: true, alpha: true });
      renderer.setPixelRatio(window.devicePixelRatio)
      renderer.setSize(container.value.clientWidth, container.value.clientHeight);

      environmentInit();

      if (Array.isArray(props.screen)) {
        for (let i = 0; i <= props.screen.length - 1; i += 1) {
          phones.push(
            phoneInit(
              props.screen[i],
              {
                position: props.position[i],
                rotation: props.rotation[i],
              },
            ),
          );
        }
      } else {
        phones.push(
          phoneInit(
            props.screen,
            {
              position: props.position,
              rotation: props.rotation,
            },
          ),
        );
      }

      container.value.appendChild(renderer.domElement);
    }

    let previousTime = 0;
    function animate(currentTime) {
      currentTime *= 0.001;
      const deltaTime = currentTime - previousTime;
      previousTime = currentTime;

      requestAnimationFrame(animate);

      if (phones.length) {
        phones.forEach((phone) => {
          phone.animation(deltaTime, { x: mouseX / 2, y: mouseY / 2, z: camera.position.z });
        });
      }

      renderer.render(scene, camera);
    }

    function handleMouseEnter() {
      if (phones.length) {
        phones.forEach((phone) => {
          phone.animation = phone.lookAtAnim;
          phone.goingHome = false;
          clearTimeout(phone.homeTimeout);
        });
      }
    }

    function handleMouseLeave() {
      if (phones.length) {
        phones.forEach((phone) => {
          phone.animation = phone.homeAnim;
        });
      }
    }

    function handleMouseMove(event) {
      const rect = container.value.getBoundingClientRect();
      mouseX = event.clientX - rect.left - rect.width / 2;
      mouseY = -(event.clientY - rect.top - rect.height / 2);
    }

    function handleTouchMove(event) {
      event.preventDefault();
      const rect = container.value.getBoundingClientRect();
      mouseX = event.touches[0].clientX - rect.left - rect.width / 2;
      mouseY = -(event.touches[0].clientY - rect.top - rect.height / 2);
    }

    onMounted(() => {
      init();
      animate(0);
    });

    return {
      container,
      handleMouseEnter,
      handleMouseLeave,
      handleMouseMove,
      handleTouchMove,
    };
  },
};
</script>