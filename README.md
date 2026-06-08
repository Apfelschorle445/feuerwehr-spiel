<!DOCTYPE html>
<html lang="de">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Feuerwehr Simulator & Manager</title>
    <style>
        /* --- HIGH-END LEITSTELLEN UI --- */
        body { 
            margin: 0; 
            overflow: hidden; 
            background: #07070c; 
            font-family: 'Segoe UI', system-ui, sans-serif; 
            user-select: none; 
        }
        canvas { display: block; }

        /* Modernes Glas-Design (Glassmorphism) */
        .panel {
            position: absolute;
            color: #ffffff;
            background: rgba(12, 14, 24, 0.88);
            backdrop-filter: blur(10px);
            -webkit-backdrop-filter: blur(10px);
            border-radius: 12px;
            padding: 18px;
            box-shadow: 0 10px 30px rgba(0, 0, 0, 0.6);
            pointer-events: auto; /* Erlaubt Klicks im Menü */
            border: 1.5px solid #ff3b3b;
        }

        /* Oben Links: Einsatz & Geld */
        #hud-main {
            top: 20px;
            left: 20px;
            width: 280px;
            box-shadow: 0 0 15px rgba(255, 59, 59, 0.25);
        }

        /* Oben Rechts: Upgrade Shop */
        #hud-shop {
            top: 20px;
            right: 20px;
            width: 260px;
            border-color: #00f0ff;
            box-shadow: 0 0 15px rgba(0, 240, 255, 0.25);
        }

        h3 {
            margin: 0 0 10px 0;
            font-size: 14px;
            text-transform: uppercase;
            letter-spacing: 1.5px;
        }
        #hud-main h3 { color: #ff3b3b; }
        #hud-shop h3 { color: #00f0ff; }

        .stat-line {
            display: flex;
            justify-content: space-between;
            font-size: 13px;
            margin: 6px 0;
        }

        /* Upgrade Buttons */
        .btn-upgrade {
            width: 100%;
            background: #16192b;
            border: 1px solid #363b58;
            color: #00f0ff;
            padding: 8px;
            border-radius: 6px;
            cursor: pointer;
            font-weight: bold;
            font-size: 12px;
            margin-top: 5px;
            transition: all 0.2s;
        }
        .btn-upgrade:hover {
            background: #00f0ff;
            color: #0c0e18;
            box-shadow: 0 0 10px rgba(0, 240, 255, 0.5);
        }

        /* Fortschrittsbalken */
        .bar-bg {
            background: rgba(255,255,255,0.1);
            height: 6px;
            border-radius: 3px;
            overflow: hidden;
            margin-top: 4px;
        }
        .bar-fill {
            height: 100%;
            width: 100%;
            transition: width 0.1s linear;
        }
        #brand-bar { background: #ff3b3b; }
        #water-bar { background: #00aaff; }
    </style>
    <!-- Lädt die 3D Engine Three.js -->
    <script src="https://cloudflare.com"></script>
</head>
<body>

<!-- LINKES STATUS PANEL -->
<div id="hud-main" class="panel">
    <h3>🚒 Einsatz-Zentrale</h3>
    <div class="stat-line"><span>Budget:</span><span id="txt-money" style="color: #4df3a9; font-weight: bold;">0 €</span></div>
    <hr style="border:0; height:1px; background: rgba(255,255,255,0.1); margin: 10px 0;">
    <div id="einsatz-info" style="font-weight: bold; font-size: 13px; color: #ffaa00;">Suche Brandherd...</div>
    <div class="bar-bg"><div id="brand-bar" class="bar-fill"></div></div>
    <div class="stat-line" style="margin-top: 10px;"><span>Löschwasser:</span><span id="txt-water">100/100 L</span></div>
    <div class="bar-bg"><div id="water-bar" class="bar-fill"></div></div>
    <div class="stat-line" style="font-size: 11px; color: #888; margin-top:15px;">Steuerung: Pfeiltasten & Leertaste</div>
</div>

<!-- RECHTES SHOP PANEL -->
<div id="hud-shop" class="panel">
    <h3>🛠️ Fahrwache Upgrades</h3>
    <div class="stat-line"><span>Motor-Leistung</span><span id="lbl-speed-lvl">Lv. 1</span></div>
    <button class="btn-upgrade" onclick="buyUpgrade('speed')">Kaufen (<span id="cost-speed">150</span> €)</button>
    
    <div style="margin-top: 12px;">
        <div class="stat-line"><span>Wassertank-Volumen</span><span id="lbl-tank-lvl">Lv. 1</span></div>
        <button class="btn-upgrade" onclick="buyUpgrade('tank')">Kaufen (<span id="cost-tank">200</span> €)</button>
    </div>
</div>

<script>
    // --- SPIEL-LOGIK & VARIABLEN ---
    let money = 0;
    let waterMax = 100;
    let waterCurrent = 100;
    let speedLevel = 1;
    let tankLevel = 1;

    const upgradeCosts = { speed: 150, tank: 200 };

    // 3D-Welt initialisieren
    const scene = new THREE.Scene();
    scene.background = new THREE.Color(0x07070c);
    scene.fog = new THREE.FogExp2(0x07070c, 0.015);

    const camera = new THREE.PerspectiveCamera(60, window.innerWidth / window.innerHeight, 0.1, 1000);
    const renderer = new THREE.WebGLRenderer({ antialias: true });
    renderer.setSize(window.innerWidth, window.innerHeight);
    document.body.appendChild(renderer.domElement);

    // Licht
    scene.add(new THREE.AmbientLight(0xffffff, 0.3));
    const sun = new THREE.DirectionalLight(0xffffff, 0.7);
    sun.position.set(20, 40, 10);
    scene.add(sun);

    // Bodenplatte
    const floor = new THREE.Mesh(new THREE.PlaneGeometry(200, 200), new THREE.MeshStandardMaterial({ color: 0x11121c }));
    floor.rotation.x = -Math.PI / 2;
    scene.add(floor);
    scene.add(new THREE.GridHelper(200, 40, 0x222538, 0x12131f));

    // Einsatzstellen generieren (Häuser)
    const einsatzStellen = [];
    for(let i=0; i<12; i++) {
        let angle = (i / 12) * Math.PI * 2;
        let radius = 30 + Math.random() * 25;
        let x = Math.cos(angle) * radius;
        let z = Math.sin(angle) * radius;

        let h = 6 + Math.random()*8;
        const haus = new THREE.Mesh(new THREE.BoxGeometry(6, h, 6), new THREE.MeshStandardMaterial({ color: 0x1d1f30 }));
        haus.position.set(x, h/2, z);
        scene.add(haus);

        const feuer = new THREE.Mesh(new THREE.SphereGeometry(1.8, 8, 8), new THREE.MeshBasicMaterial({ color: 0xff3b3b, wireframe: true }));
        feuer.position.set(x, h + 1, z);
        scene.add(feuer);

        einsatzStellen.push({ id: i+1, x, z, obj: feuer, health: 100 });
    }
    let aktuellerEinsatz = einsatzStellen[0];

    // Fahrzeug
    const truck = new THREE.Group();
    const body = new THREE.Mesh(new THREE.BoxGeometry(1.5, 1.2, 3.5), new THREE.MeshStandardMaterial({ color: 0xcc0000 }));
    body.position.y = 0.7;
    truck.add(body);
    const blaulicht = new THREE.Mesh(new THREE.BoxGeometry(0.5, 0.1, 0.2), new THREE.MeshBasicMaterial({ color: 0x00f0ff }));
    blaulicht.position.set(0, 1.35, 1.2);
    truck.add(blaulicht);
    truck.position.set(0, 0, 15);
    scene.add(truck);

    // Steuerung
    const keys = {};
    window.addEventListener('keydown', (e) => keys[e.code] = true);
    window.addEventListener('keyup', (e) => keys[e.code] = false);

    let v = 0, rot = 0;
    const particles = [];

    // Shop Funktion (Wird bei Klick ausgeführt)
    window.buyUpgrade = function(type) {
        if (money >= upgradeCosts[type]) {
            money -= upgradeCosts[type];
            if (type === 'speed') {
                speedLevel++;
                upgradeCosts.speed = Math.floor(upgradeCosts.speed * 1.5);
                document.getElementById('lbl-speed-lvl').innerText = "Lv. " + speedLevel;
                document.getElementById('cost-speed').innerText = upgradeCosts.speed;
            } else if (type === 'tank') {
                tankLevel++;
                waterMax = Math.floor(waterMax * 1.4);
                waterCurrent = waterMax;
                upgradeCosts.tank = Math.floor(upgradeCosts.tank * 1.6);
                document.getElementById('lbl-tank-lvl').innerText = "Lv. " + tankLevel;
                document.getElementById('cost-tank').innerText = upgradeCosts.tank;
            }
            document.getElementById('txt-money').innerText = money + " €";
        }
    };

    // Game Loop
    function animate() {
        requestAnimationFrame(animate);

        // UI Updates
        document.getElementById('txt-money').innerText = money + " €";
        document.getElementById('txt-water').innerText = Math.ceil(waterCurrent) + "/" + waterMax + " L";
        document.getElementById('water-bar').style.width = (waterCurrent / waterMax * 100) + "%";
        
        if(aktuellerEinsatz) {
            document.getElementById('einsatz-info').innerText = "🔥 Brandherd " + aktuellerEinsatz.id + " bekämpfen!";
            document.getElementById('brand-bar').style.width = aktuellerEinsatz.health + "%";
        }

        blaulicht.material.color.setHex(Math.floor(Date.now() / 80) % 2 === 0 ? 0x00f0ff : 0x001122);

        // Bewegung berechnen (Abhängig vom Upgrade-Level)
        let aktuelleMaxSpeed = 0.18 + (speedLevel * 0.03);
        if (keys['ArrowUp']) v = Math.min(v + 0.006, aktuelleMaxSpeed);
        else if (keys['ArrowDown']) v = Math.max(v - 0.006, -aktuelleMaxSpeed/2);
        else v *= 0.94;

        if (Math.abs(v) > 0.01) {
            let dir = v > 0 ? 1 : -1;
            if (keys['ArrowLeft']) rot += 0.035 * dir;
            if (keys['ArrowRight']) rot -= 0.035 * dir;
        }

        truck.rotation.y = rot;
        truck.position.x += Math.sin(rot) * v;
        truck.position.z += Math.cos(rot) * v;

        // Kamera folgt flüssig
        camera.position.set(truck.position.x - Math.sin(rot)*12, truck.position.y + 4.5, truck.position.z - Math.cos(rot)*12);
        camera.lookAt(truck.position.x, truck.position.y + 1, truck.position.z);

        // Automatisch Wasser auffüllen an der Basis (Mitte der Karte bei 0,0)
        if(truck.position.distanceTo(new THREE.Vector3(0,0,0)) < 6) {
            waterCurrent = Math.min(waterMax, waterCurrent + 1.5);
