 
 
 
from IPython.display import display, HTML
import json

game = r"""
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>Open World Ultimate</title>
<style>
*{box-sizing:border-box}
body{margin:0;background:#101820;color:white;font-family:Arial}
#game{position:fixed;inset:0}
canvas{display:block}
#hud{position:fixed;inset:0;pointer-events:none}
.panel{background:#101b28df;border:1px solid #ffffff44;
border-radius:12px;padding:11px;backdrop-filter:blur(8px)}
#stats{position:absolute;top:10px;left:10px;line-height:1.7}
#missions{position:absolute;right:10px;top:10px;width:225px}
#map{position:absolute;right:10px;bottom:10px;width:135px;height:135px}
#help{position:absolute;bottom:10px;left:10px;font-size:12px}
#msg{position:absolute;top:48%;width:100%;text-align:center;
font-size:24px;font-weight:bold;text-shadow:0 3px 8px black}
button,select{padding:9px 12px;margin:3px;border:0;border-radius:8px;
background:#078bc4;color:white;font-weight:bold;cursor:pointer}
button:hover{background:#05b8e8}
#menu,#garage{position:fixed;inset:0;z-index:5;
display:flex;align-items:center;justify-content:center;
background:#07111dec;text-align:center}
#garage{display:none;z-index:6}
.box{background:#172b40;border:1px solid #ffffff44;
padding:25px;border-radius:18px;width:min(440px,94vw)}
h1{color:#56d9ff}
#mobile,#mobile2{display:none;position:fixed;bottom:150px}
#mobile{left:5px}
#mobile2{right:5px}
#mobile button{min-width:45px;touch-action:none}
@media(max-width:700px){
 #mobile,#mobile2{display:block}
 #help{display:none}
 #missions{width:170px;font-size:12px}
 #map{width:100px;height:100px}
}
</style>
</head>
<body>
<div id="game"></div>
<div id="hud">
 <div id="stats" class="panel">
  <b>🌆 OPEN WORLD ULTIMATE</b><br>
  ❤️ Health: <span id="health">100</span><br>
  ⭐ Money: $<span id="money">0</span><br>
  🏆 Level: <span id="level">1</span>/10<br>
  🚘 Mode: <span id="mode">Walking</span><br>
  🚗 Car: <span id="carname">Sport</span><br>
  🌦️ Weather: <span id="weather">Clear</span>
 </div>
 <div id="missions" class="panel">
  <b>🎯 MISSIONS</b>
  <p id="mission">Explore the city</p>
  <span id="progress">Mission 1 / 20</span><br>
  <button onclick="pauseGame()">⏸ Pause</button>
  <button onclick="openGarage()">🚘 Garage</button>
  <button onclick="toggleWeather()">🌧 Weather</button>
  <button onclick="toggleTime()">🌙 Day/Night</button>
  <button onclick="saveGame()">💾 Save</button>
  <button onclick="loadGame()">📂 Load</button>
  <button onclick="restartGame()">↻ Restart</button>
 </div>
 <canvas id="map" width="135" height="135" class="panel"></canvas>
 <div id="help" class="panel">
  WASD / Arrows = Move<br>
  E = Enter / Exit car<br>
  Mouse drag = Camera<br>
  L = Headlights · P = Pause<br>
  R = Restart · G = Garage
 </div>
 <div id="msg"></div>
</div>

<div id="mobile">
 <button data-key="w">▲</button><br>
 <button data-key="a">◀</button>
 <button data-key="s">▼</button>
 <button data-key="d">▶</button>
</div>
<div id="mobile2">
 <button onclick="interact()">ENTER / EXIT</button>
 <button onclick="pauseGame()">PAUSE</button>
</div>

<div id="menu">
 <div class="box">
  <h1>🌆 OPEN WORLD ULTIMATE</h1>
  <h2>REALISTIC 3D CITY</h2>
  <p>Explore · Drive · Complete Missions</p>
  <p>5 Cars | 20 Missions | Day/Night | Rain</p>
  <button onclick="startGame()">▶ START GAME</button>
  <button onclick="loadGame()">📂 LOAD GAME</button>
  <p style="font-size:12px">Free browser game · No paid API</p>
 </div>
</div>

<div id="garage">
 <div class="box">
  <h1>🚘 CITY GARAGE</h1>
  <p>Select your vehicle</p>
  <select id="carSelect">
   <option value="0">Sport Car</option>
   <option value="1">Super Car</option>
   <option value="2">SUV</option>
   <option value="3">Classic Car</option>
   <option value="4">Racing Car</option>
  </select><br>
  <button onclick="selectCar()">SELECT</button>
  <button onclick="closeGarage()">CLOSE</button>
 </div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script>
let scene,camera,renderer,clock;
let player,car,carModels=[],traffic=[],rain;
let legs=[],keys={};
let running=false,paused=false,inCar=false;
let score=0,money=0,health=100,level=1,missionIndex=0;
let selectedCar=0,speed=0,cameraAngle=0;
let mouseDown=false,lastX=0,night=false,raining=false;
let targets=[],rings=[],world=[];
let sun,headlights=[];
let walkTime=0;

const names=[
'Visit the central road','Explore the west district',
'Reach the city park','Visit the north road',
'Find your sports car','Drive to the east district',
'Visit the city square','Explore the south road',
'Reach the golden tower','Cross the central city',
'Visit the blue district','Explore the west highway',
'Reach the city garden','Visit the north district',
'Drive to the red marker','Explore the east highway',
'Visit the city center','Reach the final road',
'Complete the city tour','Final city challenge'
];

const colors=[0xe52b36,0x13b9d5,0x333c4a,0xe3b72c,0x18a75b];
for(let i=0;i<20;i++){
 targets.push([(i%5-2)*36,(Math.floor(i/5)-1.5)*43]);
}

function material(c,r=.8){
 return new THREE.MeshStandardMaterial({color:c,roughness:r});
}
function box(x,y,z,sx,sy,sz,m){
 let o=new THREE.Mesh(new THREE.BoxGeometry(sx,sy,sz),m);
 o.position.set(x,y,z);scene.add(o);world.push(o);return o;
}

function createCar(i){
 let g=new THREE.Group();
 let body=new THREE.Mesh(
  new THREE.BoxGeometry(4,1.05,2),material(colors[i],.3));
 body.position.y=.9;g.add(body);

 let roof=new THREE.Mesh(
  new THREE.BoxGeometry(2.1,.9,1.7),material(0x8bbbd0,.25));
 roof.position.set(-.2,1.75,0);g.add(roof);

 let wheelMat=material(0x171717);
 for(let x of [-1.3,1.3])for(let z of [-1,1]){
  let w=new THREE.Mesh(
   new THREE.CylinderGeometry(.48,.48,.32,16),wheelMat);
  w.rotation.x=Math.PI/2;w.position.set(x,.48,z);g.add(w);
 }
 // Headlights
 for(let z of [-.65,.65]){
  let lamp=new THREE.Mesh(
   new THREE.BoxGeometry(.12,.25,.35),
   new THREE.MeshBasicMaterial({color:0xffffd0}));
  lamp.position.set(2,.95,z);g.add(lamp);
 }
 return g;
}

function init(){
 scene=new THREE.Scene();
 scene.background=new THREE.Color(0x87b8d6);
 scene.fog=new THREE.Fog(0x87b8d6,110,280);

 camera=new THREE.PerspectiveCamera(65,innerWidth/innerHeight,.1,600);
 renderer=new THREE.WebGLRenderer({antialias:true});
 renderer.setSize(innerWidth,innerHeight);
 renderer.setPixelRatio(Math.min(devicePixelRatio,1.5));
 renderer.shadowMap.enabled=true;
 renderer.shadowMap.type=THREE.PCFSoftShadowMap;
 document.getElementById('game').appendChild(renderer.domElement);
 clock=new THREE.Clock();

 scene.add(new THREE.HemisphereLight(0xcceaff,0x52604c,1.8));
 sun=new THREE.DirectionalLight(0xfff0d4,2);
 sun.position.set(60,100,30);
 sun.castShadow=true;
 sun.shadow.mapSize.set(1024,1024);
 scene.add(sun);

 // City ground
 box(0,-.7,0,220,.8,220,material(0x53794d));

 // Roads
 for(let i=-100;i<=100;i+=40){
  box(i,-.2,0,10,.15,220,material(0x303943));
  box(0,-.19,i,220,.15,10,material(0x303943));
  for(let j=-100;j<=100;j+=12){
   box(i,-.1,j,.18,.03,3.5,material(0xe7d9a5));
   box(j,-.1,i,3.5,.03,.18,material(0xe7d9a5));
  }
 }

 // Buildings
 for(let x=-90;x<=90;x+=20){
  for(let z=-90;z<=90;z+=20){
   if(Math.abs(x%40)<10||Math.abs(z%40)<10)continue;
   let h=12+Math.random()*28;
   let shades=[0x8796a5,0xb2a18f,0x667b90,0x9da6ac];
   let c=shades[Math.floor(Math.random()*shades.length)];
   box(x,h/2,z,12,h,12,material(c));
   for(let y=4;y<h-2;y+=5){
    for(let xx=-4;xx<=4;xx+=4){
     box(x+xx,y,z+6.05,1.5,1.7,.08,material(0x9bd5eb,.25));
    }
   }
  }
 }

 // Trees
 for(let i=0;i<95;i++){
  let x=(Math.random()-.5)*190;
  let z=(Math.random()-.5)*190;
  if(Math.abs(x%40)<7||Math.abs(z%40)<7)continue;
  let trunk=new THREE.Mesh(
   new THREE.CylinderGeometry(.25,.35,3,8),material(0x76502d));
  trunk.position.set(x,1.5,z);scene.add(trunk);
  let leaf=new THREE.Mesh(
   new THREE.SphereGeometry(2.1,9,8),material(0x24753c));
  leaf.position.set(x,4,z);scene.add(leaf);
 }

 // Player and animated legs
 player=new THREE.Group();
 let torso=new THREE.Mesh(
  new THREE.CylinderGeometry(.48,.58,1.35,12),material(0x176dc1));
 torso.position.y=1.1;player.add(torso);
 let head=new THREE.Mesh(
  new THREE.SphereGeometry(.38,14,12),material(0xffc99c));
 head.position.y=2.05;player.add(head);

 for(let x of [-.22,.22]){
  let leg=new THREE.Mesh(
   new THREE.BoxGeometry(.25,.8,.3),material(0x263b58));
  leg.position.set(x,.35,0);player.add(leg);legs.push(leg);
 }
 player.position.set(0,0,8);scene.add(player);

 // Cars
 for(let i=0;i<5;i++){
  let c=createCar(i);
  c.position.set(5,0,8);
  c.visible=i===0;
  scene.add(c);carModels.push(c);
 }
 car=carModels[0];

 // Traffic
 for(let i=0;i<12;i++){
  let t=new THREE.Mesh(
   new THREE.BoxGeometry(3.2,1.1,1.8),
   material([0x2779ca,0xe1c83a,0xdddddd,0x292929][i%4],.4));
  t.position.set(-100+Math.random()*200,.65,-100+Math.random()*200);
  t.userData.dir=i%2?1:-1;
  scene.add(t);traffic.push(t);
 }

 // Mission markers
 targets.forEach(p=>{
  let r=new THREE.Mesh(
   new THREE.TorusGeometry(3,.35,8,24),
   new THREE.MeshStandardMaterial({
    color:0xffd32b,emissive:0x664000}));
  r.rotation.x=Math.PI/2;
  r.position.set(p[0],.4,p[1]);
  scene.add(r);rings.push(r);
 });

 // Headlights
 for(let z of [-.65,.65]){
  let light=new THREE.SpotLight(0xffffd0,0,55,Math.PI/6,.4,1);
  light.position.set(2,1,z);
  car.add(light);headlights.push(light);
 }

 // Rain particles
 let positions=new Float32Array(1200*3);
 for(let i=0;i<1200;i++){
  positions[i*3]=(Math.random()-.5)*180;
  positions[i*3+1]=Math.random()*60;
  positions[i*3+2]=(Math.random()-.5)*180;
 }
 let geo=new THREE.BufferGeometry();
 geo.setAttribute('position',new THREE.BufferAttribute(positions,3));
 rain=new THREE.Points(geo,new THREE.PointsMaterial({
  color:0xb8d9ed,size:.16,transparent:true,opacity:.7}));
 rain.visible=false;scene.add(rain);

 window.addEventListener('keydown',e=>{
  let k=e.key.toLowerCase();keys[k]=true;
  if(k==='e')interact();
  if(k==='p')pauseGame();
  if(k==='r')restartGame();
  if(k==='g')openGarage();
  if(k==='l')toggleLights();
 });
 window.addEventListener('keyup',e=>keys[e.key.toLowerCase()]=false);

 renderer.domElement.addEventListener('mousedown',e=>{
  mouseDown=true;lastX=e.clientX;
 });
 window.addEventListener('mouseup',()=>mouseDown=false);
 window.addEventListener('mousemove',e=>{
  if(mouseDown)cameraAngle-=(e.clientX-lastX)*.006;
  lastX=e.clientX;
 });
 window.addEventListener('resize',()=>{
  camera.aspect=innerWidth/innerHeight;
  camera.updateProjectionMatrix();
  renderer.setSize(innerWidth,innerHeight);
 });

 document.querySelectorAll('[data-key]').forEach(b=>{
  let k=b.dataset.key;
  b.addEventListener('pointerdown',e=>{
   e.preventDefault();keys[k]=true;
  });
  ['pointerup','pointerleave','pointercancel'].forEach(ev=>{
   b.addEventListener(ev,()=>keys[k]=false);
  });
 });

 updateHUD();animate();
}

function startGame(){
 document.getElementById('menu').style.display='none';
 running=true;paused=false;
}
function pauseGame(){
 if(!running)return;
 paused=!paused;
 show(paused?'⏸ PAUSED':'');
}
function show(s){
 document.getElementById('msg').textContent=s;
}
function interact(){
 if(!running||paused)return;
 if(!inCar&&player.position.distanceTo(car.position)<10){
  inCar=true;speed=0;show('Driving mode');
 }else if(inCar){
  inCar=false;player.position.copy(car.position);
  player.position.x+=4;show('Walking mode');
 }
 updateHUD();
}
function openGarage(){
 if(!running||inCar)return;
 paused=true;
 document.getElementById('garage').style.display='flex';
}
function closeGarage(){
 document.getElementById('garage').style.display='none';
 paused=false;
}
function selectCar(){
 selectedCar=Number(document.getElementById('carSelect').value);
 carModels.forEach((c,i)=>c.visible=i===selectedCar);
 car=carModels[selectedCar];
 car.position.set(player.position.x+5,0,player.position.z);
 closeGarage();updateHUD();
 show('Car selected!');
}
function toggleWeather(){
 raining=!raining;
 rain.visible=raining;
 document.getElementById('weather').textContent=
  raining?'Rain':'Clear';
}
function toggleTime(){
 night=!night;
 sun.intensity=night?.3:2;
 scene.background.set(night?0x101b36:0x87b8d6);
 scene.fog.color.set(night?0x101b36:0x87b8d6);
 scene.fog.near=night?65:110;
 show(night?'Night time':'Day time');
}
function toggleLights(){
 let on=headlights[0].intensity===0;
 headlights.forEach(l=>l.intensity=on?5:0);
 show(on?'Headlights ON':'Headlights OFF');
}
function saveGame(){
 try{
  localStorage.setItem('openWorldUltimate',JSON.stringify({
   money,score,health,level,missionIndex,selectedCar,
   x:player.position.x,z:player.position.z
  }));
  show('Progress saved!');
 }catch(e){show('Browser storage unavailable');}
}
function loadGame(){
 try{
  let d=JSON.parse(localStorage.getItem('openWorldUltimate'));
  if(!d){alert('No saved progress found.');return;}
  money=d.money||0;score=d.score||0;health=d.health||100;
  level=d.level||1;missionIndex=Math.min(d.missionIndex||0,19);
  selectedCar=d.selectedCar||0;
  carModels.forEach((c,i)=>c.visible=i===selectedCar);
  car=carModels[selectedCar];
  player.position.set(d.x||0,0,d.z||8);
  car.position.set(player.position.x+5,0,player.position.z);
  updateHUD();startGame();
 }catch(e){alert('Could not load saved game.');}
}
function restartGame(){
 money=0;score=0;health=100;level=1;missionIndex=0;
 inCar=false;speed=0;
 player.position.set(0,0,8);
 car.position.set(5,0,8);
 running=true;paused=false;
 updateHUD();show('Game restarted');
}
function updateHUD(){
 document.getElementById('health').textContent=health;
 document.getElementById('money').textContent=money;
 document.getElementById('level').textContent=level;
 document.getElementById('mode').textContent=inCar?'Driving':'Walking';
 document.getElementById('carname').textContent=
  ['Sport','Super','SUV','Classic','Racing'][selectedCar];
 document.getElementById('mission').textContent=
  names[missionIndex]||'All missions complete!';
 document.getElementById('progress').textContent=
  'Mission '+Math.min(missionIndex+1,20)+' / 20';
}
function finishMission(){
 money+=250;score+=100;missionIndex++;
 level=Math.min(10,Math.floor(missionIndex/2)+1);
 if(missionIndex>=20){
  running=false;show('🏆 YOU WIN! ALL MISSIONS COMPLETE!');
 }else show('MISSION COMPLETE! +$250');
 updateHUD();saveGame();
}
function mapDraw(obj){
 let c=document.getElementById('map');
 let ctx=c.getContext('2d');
 ctx.fillStyle='#263b32';ctx.fillRect(0,0,135,135);
 ctx.strokeStyle='#929ba2';ctx.lineWidth=5;
 for(let i=0;i<135;i+=27){
  ctx.beginPath();ctx.moveTo(i,0);ctx.lineTo(i,135);ctx.stroke();
  ctx.beginPath();ctx.moveTo(0,i);ctx.lineTo(135,i);ctx.stroke();
 }
 let p=targets[missionIndex];
 if(p){
  ctx.fillStyle='#ffce26';ctx.beginPath();
  ctx.arc(67+(p[0]-obj.position.x)*.34,
  67+(p[1]-obj.position.z)*.34,4,0,7);ctx.fill();
 }
 ctx.fillStyle='#21d9ff';ctx.beginPath();
 ctx.arc(67,67,5,0,7);ctx.fill();
}
function update(dt){
 let obj=inCar?car:player;
 let f=(keys.w||keys.arrowup?1:0)-(keys.s||keys.arrowdown?1:0);
 let side=(keys.d||keys.arrowright?1:0)-(keys.a||keys.arrowleft?1:0);

 if(inCar){
  speed+=f*20*dt;
  speed*=Math.pow(.965,dt*60);
  speed=Math.max(-13,Math.min(32,speed));
  car.rotation.y-=side*.045*(Math.abs(speed)/10+.25);
  car.position.x+=Math.sin(car.rotation.y)*speed*dt;
  car.position.z+=Math.cos(car.rotation.y)*speed*dt;
 }else{
  if(f||side){
   player.rotation.y=cameraAngle;
   player.position.x+=(Math.sin(cameraAngle)*f+
    Math.cos(cameraAngle)*side)*10*dt;
   player.position.z+=(Math.cos(cameraAngle)*f-
    Math.sin(cameraAngle)*side)*10*dt;
   walkTime+=dt*10;
   legs[0].rotation.x=Math.sin(walkTime)*.6;
   legs[1].rotation.x=-Math.sin(walkTime)*.6;
  }else{
   legs.forEach(l=>l.rotation.x=0);
  }
 }
 obj.position.x=Math.max(-105,Math.min(105,obj.position.x));
 obj.position.z=Math.max(-105,Math.min(105,obj.position.z));

 // Traffic and collision
 traffic.forEach(t=>{
  t.position.x+=t.userData.dir*dt*8;
  if(t.position.x>110)t.position.x=-110;
  if(t.position.x<-110)t.position.x=110;
  if(inCar&&obj.position.distanceTo(t.position)<3.2){
   speed*=.75;
  }
 });

 // Rain movement
 if(raining){
  let a=rain.geometry.attributes.position;
  for(let i=0;i<a.count;i++){
   let y=a.getY(i)-dt*35;
   if(y<0)y=50;
   a.setY(i,y);
  }
  a.needsUpdate=true;
 }
 // Mission marker
 let p=targets[missionIndex];
 if(p&&Math.hypot(obj.position.x-p[0],obj.position.z-p[1])<5){
  finishMission();
 }

 // Camera
 let angle=inCar?car.rotation.y:cameraAngle;
 let dist=inCar?15:10;
 let desired=new THREE.Vector3(
  obj.position.x-Math.sin(angle)*dist,
  inCar?8:6,
  obj.position.z-Math.cos(angle)*dist
 );
 camera.position.lerp(desired,.09);
 camera.lookAt(obj.position.x,1.5,obj.position.z);
 mapDraw(obj);
}
function animate(){
 requestAnimationFrame(animate);
 let dt=Math.min(clock.getDelta(),.04);
 if(running&&!paused)update(dt);
 renderer.render(scene,camera);
}
init();
</script>
</body>
</html>
"""

display(HTML(
    '<iframe srcdoc=' + json.dumps(game) +
    ' style="width:100%;height:760px;border:0;border-radius:12px" '
    'allow="fullscreen"></iframe>'
))
