<template>
  <canvas ref="canvasRef"></canvas>
</template>

<script setup>
import { ref, onMounted, onUnmounted} from "vue"
import { OrbitControls} from "three/examples/jsm/controls/OrbitControls"
import * as THREE from "three"
import * as CANNON from "cannon-es"

var controls

let canvasRef = ref();

const sizes = {
  width: window.innerWidth,
  height: window.innerHeight
}



let scene = new THREE.Scene()

let renderer

//CANNON.JS PHYSICS
//Per aggiungere la fisica al nostro "mondo virtuale" dobbiamo affrontare 4 step:
//1) Creare un mondo fisico (in pratica inseriamo le leggi della fisica nel nostro mondo in threejs)
//2) Creare una mesh (questo step non ha nulla a che fare con cannon.js)
//3) Creare un entità trasparente (chiamata body), che subira è fenomeni fisica come gravità ecc (step gestito da cannon.js)
//4) Unire la mesh al body, cosi da creare un unico oggetto affetto dalle leggi della fisica


//1)Creiamo un mondo fisico

//WORLD OBJECT  
//il costruttore di questa classe prende come argomento un oggetto per la configurazione, nel quale possiamo specificare alcune proprietà.
//Una di queste è proprio la gravità

const world = new CANNON.World({
  gravity: new CANNON.Vec3(0, -9.81, 0) //aggiungiamo la gravita al mondo, il valore -9.81 che applichiamo sull' "asse y", è il valore reale dell'accelerazione di gravita 9.81m/s^2
})

// ora introduciamo il tempo nel nostro mondo. (che utilizzeremo per calcolare l'accelerazione nel movimento)
const timeStep = 1 / 60 //rendere piu piccol questo valore aumenta la precisione, ma al costo di un maggior utilizzo della risorse di calcolo(CPU/GPU)


//2)Creiamo una mesh per il "piano" che utilizzeremo come terreno(o pavimento)
const groundGeo = new THREE.PlaneGeometry(30,30)
const groundMat = new THREE.MeshBasicMaterial({
  color: 0xffffff,
  side: THREE.DoubleSide,
  wireframe: true
})
const groundMesh = new THREE.Mesh(groundGeo, groundMat)
scene.add(groundMesh)


//5) TORNARE QUI DOPO: CREO UN MATERIALE FISICO PER IL GROUND 
const groundPhysMat = new CANNON.Material()


//3) Crediamo il body da assegnare poi alla mesh
//per fare cio creiamo un istanza della classe cannon.body, che prende come argomento un oggetto di configurazione
const groundBody = new CANNON.Body({
  //shape: new CANNON.Plane(), //questa proprietà rappresenta la forma del body, che deve corrispondere alla mesh a cui vogliamo applicare il body
  //mass: 1, //aggiungiamo la massa per far si che il corpo sia affetto dalla gravità (in quanto un corpo che non ha massa, non puo essere affetto da gravità)
  // in cannon js esistono 3 tipi ti body, statico (anche con massa non è affetto da gravità), dinamica (affetto da gravita), kinematic(non ho ancora molto chiaro)
  
  shape: new CANNON.Box(new CANNON.Vec3(15,15,0.001)), //siccome il plane in cannon JS è infinito, se vogliamo uno scenario dove se esci fuori dal piano, "cadi" , possiamo impostare la shape come un box piuttosto che come plane
  
  type: CANNON.Body.STATIC, // in questo caso lo impostiamo come static, visto che parliamo del pavimento, in questo modo non "cadrà" nel vuoto
  //type: CANNON.Body.DYNAMIC
  //type: CANNON.Body.KINEMATIC
  
  //assegnamo il materiale fisico al fisic body del ground
  material: groundPhysMat
})

//Aggiungiamo questo body al mondo cannon
world.addBody(groundBody)

//Ora ruotiamo il piano. useremo il body invece della mesh in quanto nella funzione animate, copiamo la posizione e l'orientamento dal body alla mesh
groundBody.quaternion.setFromEuler(- Math.PI / 2 , 0, 0)


//ORA AGGIUNGIAMO UN BOX, CON UN BODY FISICO

// BOX
const boxGeometry = new THREE.BoxGeometry(2,2,2)
const boxMaterial = new THREE.MeshBasicMaterial({color: 0x00ff00, wireframe: true}) 
const box = new THREE.Mesh(boxGeometry,boxMaterial)
scene.add(box)

//creo un materiale fisico per il boxBody
const boxPhysMat = new CANNON.Material()

const boxBody = new CANNON.Body({
  shape: new CANNON.Box(new CANNON.Vec3(1,1,1)), //questo valore deve essere la metà di quello impostato come grandezza nella box geometry di threejs
  mass: 1,
  position: new CANNON.Vec3(1, 20, 0), //impostiamo una posizione per il box
  material: boxPhysMat
})
world.addBody(boxBody)

boxBody.angularVelocity.set(0,10,0) //in questo modo gli applichiamo una rotazione su asse y (in caduta)
boxBody.angularDamping = 0.31 //in questo modo come per la resistenza alla caduta, impostiamo una resistenza alla rotazione, che permettera al cubo di rallentare gradualmente fino a fermarsi

//creiamo un istanza della classe CONTACT MATERIAL
//I primi due argomenti sono i due materiali fisici (quello del ground e quello del box)
//il terzo argomento (opzionale) è un oggetto di configurazione in cui specifichiamo cosa deve accadere quando i due materiali vengono a contatto
const groundBoxContactMat = new CANNON.ContactMaterial(
  groundPhysMat,
  boxPhysMat,
  {
    friction: 0.27 //visto che vogliamo dare l'effetto che il cubo scivoli sul piano, dobbiamo dare un valore piccolo alla friction
  }
)

//e infine aggiungiamo il contact material al mondo fisico
world.addContactMaterial(groundBoxContactMat)


//AGGIUNGIAMO ANCHE UNA SFERA, SEMPRE CON BODY FISICO
const sphereGeo  = new THREE.SphereGeometry(2)
const sphereMat = new THREE.MeshBasicMaterial({color: 0xff0000, wireframe: true})
const sphereMesh = new THREE.Mesh(sphereGeo,sphereMat)
scene.add(sphereMesh)

//un altra cosa che possiamo fare con il material è far rimbalzare la sfera come fosse una palla
const spherePhysMat = new CANNON.Material()

const sphereBody = new CANNON.Body({
  shape: new CANNON.Sphere(2), //il radius del body della sfera è lo stesso di quello della geometria in threejs (non a metà valore come per il box)
  mass:10,
  position: new CANNON.Vec3(0,15,0),
  material: spherePhysMat
})
world.addBody(sphereBody)

//aggiungiamo la "resistenza" per far in modo che il movimento sul piano della palla diminuisca gradualmente
sphereBody.linearDamping = 0.31 //impostiamo per la resistenza un valore che va da 0 a 1

const groundSphereContactMat = new CANNON.ContactMaterial(
  groundPhysMat,
  spherePhysMat,
  {
    restitution: 0.9 //questo valore rappresenta il ritorno parziale di un corpo alla sua posizione precedente
  }
)
world.addContactMaterial(groundSphereContactMat)

// CONTROLLARE LE REAZIONI AL CONTATTO CON ALTRI OGGETTI

//Per controllare in che modo un oggetto reagisce quando entra in contatto con un altro oggetto, possiamo utilizzare i materiali in cannonJS
//a differenza dei materiali in treejs che sono un oggetto visuale, i materiali cannonJS sono invisibili e cotengono informazioni su come un oggetto
//con quel materiale reagisce, quando entra in contatto con un altro oggetto che  possiede un altro materiale cannon.js
//per fare cio torniamo ai nostri oggetti fisici , criamo un materiale cannonjs e poi passiamolo al costruttore dei nostri oggetti fisici.

// CAMERA
let camera = new THREE.PerspectiveCamera(
  45, 
  sizes.width / sizes.height, 
  0.1,
  1000  
);

camera.position.set(0, 20, -30)
camera.lookAt(box.position)
scene.add(camera);


//EVENT LISTENER
window.addEventListener("resize", ()=>{
  sizes.width = window.innerWidth
  sizes.height = window.innerHeight

  camera.aspect = sizes.width / sizes.height
  camera.updateProjectionMatrix()

  renderer.setSize(sizes.width, sizes.height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio, 2))
})

window.addEventListener("dblclick",()=>{

  const fullscreenElement = document.fullscreenElement ||  document.webkitFullscreenElement
  if(!fullscreenElement){
    if(canvasRef.value.requestFullscreen){
      canvasRef.value.requestFullscreen()
    }
    else if(canvasRef.value.webkitRequestFullscreen){
      canvasRef.value.webkitRequestFullscreen()
    }
    
  }
  else{
    if(document.exitFullscreen)
    {
      document.exitFullscreen()
    }
    else if(document.webkitExitFullscreen){
      document.webkitExitFullscreen()
    }
  }
})

//ANIMATION LOOP
const animate = () =>{
  world.step(timeStep)

  //4) Fondiamo il body del PLANE e la corrispondente mesh
  groundMesh.position.copy(groundBody.position) //imposta e aggiorna la posizione della mesh copiando la posizione dal body cannonJs
  groundMesh.quaternion.copy(groundBody.quaternion) //imposta e aggiorna l orientamento della mesh copiando l orientamento dal body cannonJs

  //Fondiamo il body e la mesh del BOX
  box.position.copy(boxBody.position) 
  box.quaternion.copy(boxBody.quaternion) 

  //Fondiamo il body e la mesh della SPHERE
  sphereMesh.position.copy(sphereBody.position) 
  sphereMesh.quaternion.copy(sphereBody.quaternion) 

  renderer.render(scene, camera)
}

onMounted(() => {
  
  //RENDERER
  renderer = new THREE.WebGLRenderer({
    canvas: canvasRef.value
  });
  renderer.setSize(sizes.width, sizes.height)
  renderer.setPixelRatio(Math.min(window.devicePixelRatio,2)) // in questo modo il max valore di pixel ratio utilizzaro è 2 (altrimenti devi renderizzare troppi pixel, dispiego enorme di potenza gpu)
  renderer.render(scene, camera)
  renderer.setAnimationLoop(animate)

  //CONTROLS
  controls = new OrbitControls(camera, canvasRef.value)
  controls.enableDamping = true; // permette uno effetto smooth una volta che rilasciamo l'elemento, rallenta fino a fermarsi
});

onUnmounted(() => {
  renderer.setAnimationLoop(null)
});

</script>

<style>
canvas {
  width: 100%;
  height: 100%;
  display: block;
}
</style>