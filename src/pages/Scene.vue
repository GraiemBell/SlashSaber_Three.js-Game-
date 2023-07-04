<template>
    <div class="container">
        <div :class="['loading-screen', loading === 1 ? 'fade' : '', loading === 2 ? 'hide' : '']"></div>
        <div :class="['paused-screen', paused === false ? 'hide' : '']" @click="gameState.startGame">
            <p>Click to start</p>
        </div>
        <canvas ref="canvas" id="canvas"></canvas>
    </div>
</template>

<script setup lang="ts">
/*
 *  Things that handle all the 3D stuff
 */

import * as THREE from "three";
import { EffectComposer } from "three/examples/jsm/postprocessing/EffectComposer.js";
import { BLOOM_LAYER } from "../constants";
import * as postprocessing from "postprocessing";
import GUIManager from "../game/utils/GUIManager.ts";
import Sword from "../game/models/Sword.ts";
import GameState from "../game/models/GameState.ts";

import { onMounted, ref } from "vue";
import { createRenderer, resizeRenderer, setupEnvironment, setupLighting, setupObstacles, setupPhysicsEnvironment, setupPostProcessing } from "../game/scene";

defineEmits(["switch"]);

const canvas = ref(null);
const loading = ref(0);
const paused = ref(true);

const gameState = GameState.getInstance();
let sword : Sword;

function createScene() {
    if(canvas.value == null) return;

    // Create scene
    const camera = createCamera();
    const renderer = createRenderer(camera, canvas.value as HTMLCanvasElement);

    setupLighting();

    setupEnvironment();

    setupPhysicsEnvironment();

    sword = new Sword();

    createControls(camera);

    setupObstacles();

    const { composer, bloomComposer } = setupPostProcessing(camera, renderer);

    const dt = 1000 / 300;
    let timeTarget = 0;

    // Animation loop
    function animate() {
        requestAnimationFrame(animate);
        if(gameState.halted) return;
        if(Date.now() >= timeTarget){
            gameState.update();

            GUIManager.updateStats();

            render(composer, bloomComposer);

            timeTarget += dt;
            if(Date.now() >= timeTarget){
                timeTarget = Date.now();
            }
        }
    }
    animate();

    // Resize renderer when window size changes
    window.onresize = () => {
        resizeRenderer(renderer);
        composer.setSize(window.innerWidth, window.innerHeight);
        bloomComposer.setSize(window.innerWidth, window.innerHeight);
        camera.aspect = window.innerWidth / window.innerHeight;
        camera.updateProjectionMatrix();
    };
}

// Create and cofigure camera and return it
function createCamera() {
    const camera = new THREE.PerspectiveCamera(
        65,
        window.innerWidth / window.innerHeight,
        0.6,
        60,
    );
    camera.position.set(0, 1.2, 0);
    camera.lookAt(0, 0.5, -5);

    return camera;
}

// Create and configure camera and sword controls
function createControls(camera : THREE.Camera) {
    if(canvas.value == null) return;
    
    (canvas.value as HTMLCanvasElement).onmousemove = (e) => {
        e.preventDefault();

        controlCamera(e, camera);
        sword.move(e);
    };
}

// Take mouse event and camera as input and handle controls for the camera
function controlCamera(e : MouseEvent, camera : THREE.Camera) {
    const delta = new THREE.Vector2();

    if(gameState.mouse.x === -1 && gameState.mouse.y === -1) {
        delta.x = window.innerWidth / 2 - e.offsetX;
        delta.y = window.innerHeight / 2 - e.offsetY;
    }
    else {
        delta.x = gameState.mouse.x - e.offsetX;
        delta.y = gameState.mouse.y - e.offsetY;
    }

    gameState.mouse.x = e.offsetX;
    gameState.mouse.y = e.offsetY;

    camera.rotation.y += delta.x / 5000;
    camera.rotation.x += delta.y / 5000;
}



// Render the scene
function render(composer : postprocessing.EffectComposer, bloomComposer : EffectComposer) {
    const materials : { [name : string] : THREE.Material } = {};
    const bloomLayer = new THREE.Layers();
    bloomLayer.set(BLOOM_LAYER);
    const darkMaterial = new THREE.MeshBasicMaterial({ color: "black" });

    function darkenNonBloomed(obj : THREE.Object3D) {
        if (obj instanceof THREE.Mesh  && bloomLayer.test(obj.layers) === false) {
            materials[obj.uuid] = obj.material;
            obj.material = darkMaterial;
        }
    }

    function restoreMaterial(obj : THREE.Object3D) {
        if (obj instanceof THREE.Mesh && materials[obj.uuid]) {
            obj.material = materials[ obj.uuid ];
            delete materials[obj.uuid];
        }
    }

    gameState.sceneTraverse(darkenNonBloomed);
    bloomComposer.render();
    gameState.sceneTraverse(restoreMaterial);
    composer.render();
}

onMounted(() => {
    createScene();

    document.addEventListener("visibilitychange", () => {
        if (document.visibilityState !== "visible"){
            gameState.haltGame();
        }
    });

    document.addEventListener("keyup", (e : KeyboardEvent) => {
        if(e.key === "Escape") {
            gameState.haltGame();
        }
    });

    gameState.onAfterLoad = () => {
        loading.value = 1;
        setTimeout(() => {
            loading.value = 2;
        }, 1000);
    };

    gameState.onAfterStart = () => {
        paused.value = false;
    };

    gameState.onAfterHalt = () => {
        paused.value = true;
    };
});



</script>


<style scoped lang="scss">
.container {
    width: 100vw;
    height: 100vh;
    overflow: hidden;
}
</style>