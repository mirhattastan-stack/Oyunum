<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="UTF-8">
<title>Pro Football 2025 - PC Edition</title>
<style>
  html,body{margin:0;overflow:hidden;font-family:'Segoe UI',sans-serif;background:#000}
  canvas{display:block}
  
  /* ÜST PANEL */
  #ui{position:fixed;top:20px;left:50%;transform:translateX(-50%);background:rgba(0,0,0,0.85);padding:10px 30px;border-radius:10px;color:#fff;display:flex;align-items:center;gap:30px;z-index:10;border:1px solid #444;box-shadow:0 0 20px rgba(0,0,0,0.5)}
  .score{font-size:24px;font-weight:900;color:#00ff00;letter-spacing:2px}
  .timer{font-family:'Courier New',monospace;font-size:20px;color:#ffeb3b}

  /* ŞUT BARI (PC İÇİN DAHA ŞIK) */
  #shotBarBox{position:fixed;bottom:40px;right:40px;width:15px;height:150px;background:#111;border:2px solid #fff;border-radius:8px;overflow:hidden;z-index:15}
  #shotBar{width:100%;height:0%;background:linear-gradient(to top, #0f0, #ff0, #f00);box-shadow:0 0 10px rgba(255,255,255,0.5)}

  /* MESAJLAR */
  #msgOverlay{position:fixed;top:40%;left:50%;transform:translate(-50%,-50%);font-size:80px;color:#fff;font-weight:900;display:none;z-index:50;text-shadow:0 0 30px #00ff00;animation:pulse 0.5s infinite alternate}
  @keyframes pulse { from {transform:translate(-50%,-50%) scale(1)} to {transform:translate(-50%,-50%) scale(1.1)} }

  /* IŞIKLI SARI KART */
  #yellowCard{position:fixed;top:50%;left:50%;transform:translate(-50%,-50%);width:100px;height:140px;background:#ffeb3b;border-radius:10px;display:none;z-index:60;box-shadow:0 0 50px #ffeb3b;border:4px solid #fff}

  /* MENÜLER */
  #overlay, #teamSelect, #gameOver{position:fixed;top:0;left:0;width:100%;height:100%;background:radial-gradient(circle, #1a1a1a, #000);display:flex;flex-direction:column;justify-content:center;align-items:center;z-index:100;color:white}
  .start-btn{margin:10px;padding:15px 40px;font-size:18px;cursor:pointer;border-radius:5px;border:1px solid #555;color:white;font-weight:bold;width:250px;text-transform:uppercase;background:rgba(255,255,255,0.1);transition:0.3s}
  .start-btn:hover{background:rgba(255,255,255,0.3);transform:scale(1.05)}
  h1{font-size:48px;margin-bottom:30px;text-shadow:0 0 20px #00ff00}
  .controls-hint{margin-top:20px;color:#888;font-size:14px}
</style>
</head>
<body>

<div id="msgOverlay">GOOOL!</div>
<div id="yellowCard"></div>

<div id="gameOver" style="display:none">
  <h2 style="font-size:60px; color:#ff0000">MAÇ BİTTİ</h2>
  <div id="finalScore" style="font-size:32px; margin:20px">0 - 0</div>
  <button class="start-btn" onclick="location.reload()">TEKRAR OYNA</button>
</div>

<div id="overlay">
  <h1>PRO FOOTBALL 2025</h1>
  <button class="start-btn" style="border-left:5px solid #2e7d32" onclick="showTeamSelect(0.8)">KOLAY MOD</button>
  <button class="start-btn" style="border-left:5px solid #f57c00" onclick="showTeamSelect(1.3)">ORTA MOD</button>
  <button class="start-btn" style="border-left:5px solid #d32f2f" onclick="showTeamSelect(2.2)">ZOR MOD</button>
  <div class="controls-hint">Yön: WASD | Şut: SPACE | Depar: SHIFT</div>
</div>

<div id="teamSelect" style="display:none">
  <h1>TAKIMINI SEÇ</h1>
  <button class="start-btn" onclick="initGame('BEŞİKTAŞ')">BEŞİKTAŞ</button>
  <button class="start-btn" onclick="initGame('FENERBAHÇE')">FENERBAHÇE</button>
  <button class="start-btn" onclick="initGame('GALATASARAY')">GALATASARAY</button>
</div>

<div id="ui">
  <div class="timer" id="time">01:00</div>
  <div class="score" id="score">0 - 0</div>
</div>

<div id="shotBarBox"><div id="shotBar"></div></div>

<canvas id="c"></canvas>

<script src="https://cdn.jsdelivr.net/npm/three@0.155/build/three.min.js"></script>
<script>
let scene, camera, renderer, player, enemy, ball, goalieE, fans = [];
let keys = {}, ballOwner = "none", scoreP = 0, scoreE = 0, ballVel = new THREE.Vector3();
let difficulty = 1, timeLeft = 60, charging = false, shotPower = 0, gameActive = false;
let isFreekick = false, clock = new THREE.Clock();

function showTeamSelect(s) { difficulty = s; document.getElementById("overlay").style.display="none"; document.getElementById("teamSelect").style.display="flex"; }

function initGame(team) {
    document.getElementById("teamSelect").style.display="none";
    gameActive = true;
    scene = new THREE.Scene();
    scene.background = new THREE.Color(0x020408);
    
    camera = new THREE.PerspectiveCamera(60, window.innerWidth/window.innerHeight, 0.1, 1000);
    renderer = new THREE.WebGLRenderer({canvas: c, antialias: true});
    renderer.setSize(window.innerWidth, window.innerHeight);

    // PC İçin Güçlü Stadyum Işıkları
    scene.add(new THREE.AmbientLight(0xffffff, 0.3));
    const createLight = (x, z) => {
        const l = new THREE.SpotLight(0xffffff, 15, 200, 0.6, 0.5);
        l.position.set(x, 50, z);
        l.target.position.set(0, 0, 0);
        scene.add(l); scene.add(l.target);
        
        const poleGeo = new THREE.BoxGeometry(1, 50, 1);
        const pole = new THREE.Mesh(poleGeo, new THREE.MeshStandardMaterial({color:0x333}));
        pole.position.set(x, 25, z);
        scene.add(pole);
    };
    createLight(50, 60); createLight(-50, 60); createLight(50, -60); createLight(-50, -60);

    const pitch = new THREE.Mesh(new THREE.PlaneGeometry(70, 110), new THREE.MeshStandardMaterial({color: 0x1a4d1a}));
    pitch.rotation.x = -Math.PI/2; scene.add(pitch);

    // Tıklım Tıklım Tribünler
    const createCrowd = () => {
        [-1, 1].forEach(side => {
            for(let i=0; i<10; i++) {
                const step = new THREE.Mesh(new THREE.BoxGeometry(15, 2, 120), new THREE.MeshStandardMaterial({color:0x111}));
                step.position.set(side*(45+i*5), i*2, 0); scene.add(step);
                for(let j=0; j<100; j++) {
                    const fan = new THREE.Mesh(new THREE.BoxGeometry(0.4, 0.6, 0.4), new THREE.MeshBasicMaterial({color: Math.random()>0.5?0xffffff:0xff0000}));
                    fan.position.set(step.position.x+(Math.random()-0.5)*10, step.position.y+1.2, (Math.random()-0.5)*110);
                    fans.push(fan); scene.add(fan);
                }
            }
        });
    };
    createCrowd();

    const makeGoal = (z) => {
        const g = new THREE.Group();
        const m = new THREE.MeshStandardMaterial({color: 0xffffff});
        const p1 = new THREE.Mesh(new THREE.BoxGeometry(0.5, 7, 0.5), m); p1.position.set(-9, 3.5, 0);
        const p2 = p1.clone(); p2.position.x = 9;
        const bar = new THREE.Mesh(new THREE.BoxGeometry(18.5, 0.5, 0.5), m); bar.position.y = 7;
        g.add(p1, p2, bar); g.position.z = z; scene.add(g);
    };
    makeGoal(-54); makeGoal(54);

    const createHuman = (col) => {
        const g = new THREE.Group();
        const m = new THREE.MeshStandardMaterial({color: col});
        const b = new THREE.Mesh(new THREE.BoxGeometry(1, 1.5, 0.6), m); b.position.y = 2; g.add(b);
        const h = new THREE.Mesh(new THREE.SphereGeometry(0.45, 16, 16), new THREE.MeshStandardMaterial({color: 0xffdbac})); h.position.y = 3.2; g.add(h);
        const limb = new THREE.BoxGeometry(0.35, 1.2, 0.35);
        const lLeg = new THREE.Mesh(limb, m); lLeg.position.set(-0.35, 0.6, 0); g.add(lLeg);
        const rLeg = new THREE.Mesh(limb, m); rLeg.position.set(0.35, 0.6, 0); g.add(rLeg);
        g.userData = {lLeg, rLeg}; scene.add(g); return g;
    };

    player = createHuman(0xffffff); player.position.set(0, 0, 30);
    enemy = createHuman(0xff0000); enemy.position.set(0, 0, -30);
    goalieE = createHuman(0x000000); goalieE.position.set(0, 0, -52);
    ball = new THREE.Mesh(new THREE.SphereGeometry(0.6, 32, 32), new THREE.MeshStandardMaterial({color:0xffffff}));
    ball.position.y = 0.6; scene.add(ball);

    // KLAVYE KONTROLLERİ
    window.addEventListener('keydown', (e) => { 
        keys[e.code] = true; 
        if(e.code === 'Space' && gameActive) charging = true;
        if(e.code === 'ShiftLeft' && gameActive && !isFreekick) { ballOwner="player"; player.position.z -= 4; }
    });
    window.addEventListener('keyup', (e) => { 
        keys[e.code] = false; 
        if(e.code === 'Space' && charging) {
            ballOwner = "shot"; 
            ballVel.set(0, 0.2, -2.5 - (shotPower * 5)); 
            charging = false; shotPower = 0; isFreekick = false;
        }
    });

    const timerInterval = setInterval(() => {
        if(gameActive) {
            timeLeft--;
            if(timeLeft <= 0) { timeLeft = 0; gameActive = false; clearInterval(timerInterval); finishGame(); }
        }
    }, 1000);

    animate();
}

function finishGame() {
    document.getElementById("gameOver").style.display = "flex";
    document.getElementById("finalScore").innerText = `SONUÇ: ${scoreP} - ${scoreE}`;
}

function animate() {
    if(!gameActive && timeLeft <= 0) return;
    requestAnimationFrame(animate);
    let t = clock.getElapsedTime();
    fans.forEach((f, i) => { f.position.y += Math.sin(t * 10 + i) * 0.005; });

    if(isFreekick) {
        if(enemy.position.x < 30) enemy.position.x += 0.5;
    } else if(gameActive) {
        // RAKİP AI
        enemy.position.z += (ball.position.z - enemy.position.z) * 0.07 * difficulty;
        enemy.position.x += (ball.position.x - enemy.position.x) * 0.04 * difficulty;
        
        // OYUNCU HAREKET
        const speed = keys['ShiftLeft'] ? 0.6 : 0.45;
        if(keys['KeyW']) player.position.z -= speed;
        if(keys['KeyS']) player.position.z += speed;
        if(keys['KeyA']) player.position.x -= speed;
        if(keys['KeyD']) player.position.x += speed;

        if(player.position.distanceTo(enemy.position) < 1.8) {
            isFreekick = true; ballOwner = "none";
            document.getElementById("yellowCard").style.display = "block";
            setTimeout(()=> document.getElementById("yellowCard").style.display = "none", 1200);
            ball.position.set(player.position.x, 0.6, player.position.z - 5);
        }
    }

    [player, enemy, goalieE].forEach(char => {
        const cyc = Math.sin(t * 15);
        char.userData.lLeg.rotation.x = cyc * 0.6; char.userData.rLeg.rotation.x = -cyc * 0.6;
    });

    if(charging) {
        shotPower = Math.min(1, shotPower + 0.035);
        document.getElementById("shotBar").style.height = (shotPower * 100) + "%";
    } else {
        document.getElementById("shotBar").style.height = "0%";
    }

    if(ballOwner === "player") ball.position.set(player.position.x, 0.6, player.position.z - 2);
    else if(ballOwner === "shot") { ball.position.add(ballVel); ballVel.multiplyScalar(0.99); }

    goalieE.position.x += (ball.position.x - goalieE.position.x) * 0.15 * difficulty;
    if(enemy.position.distanceTo(ball.position) < 2.5 && ballOwner === "none" && !isFreekick) ballOwner = "enemy";
    if(ballOwner === "enemy") ball.position.set(enemy.position.x, 0.6, enemy.position.z + 2);

    document.getElementById("time").innerText = `SÜRE: ${Math.floor(timeLeft/60)}:${(timeLeft%60).toString().padStart(2,'0')}`;
    document.getElementById("score").innerText = `${scoreP} - ${scoreE}`;

    if(ball.position.z < -54) { scoreP++; reset(); showGoal(); }
    if(ball.position.z > 54) { scoreE++; reset(); }

    // PC İÇİN DAHA GENİŞ KAMERA TAKİBİ
    camera.position.lerp(new THREE.Vector3(player.position.x * 0.3, 35, player.position.z + 45), 0.08);
    camera.lookAt(ball.position);
    renderer.render(scene, camera);
}

function showGoal() { const o = document.getElementById("msgOverlay"); o.style.display="block"; setTimeout(()=>o.style.display="none", 2000); }
function reset() { ballOwner = "none"; ball.position.set(0, 0.6, 0); player.position.set(0, 0, 30); enemy.position.set(0, 0, -30); isFreekick = false; ballVel.set(0,0,0); }

window.addEventListener('resize', () => {
    camera.aspect = window.innerWidth / window.innerHeight;
    camera.updateProjectionMatrix();
    renderer.setSize(window.innerWidth, window.innerHeight);
});
</script>
</body>
</html>

