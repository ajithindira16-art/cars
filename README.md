# cars
gameState.time
<!DOCTYPE html>
<html>
<head>
    <title>Neon Drift 3D Car Racer - Full Version</title>
    <style>
        body { margin: 0; background: #000; display: flex; justify-content: center; align-items: center; height: 100vh; font-family: monospace; color: #0ff; }
        canvas { border: 2px solid #0ff; background: linear-gradient(to bottom, #001122, #000033); }
        #ui { position: absolute; top: 10px; left: 10px; font-size: 20px; text-shadow: 0 0 10px #0ff; }
    </style>
</head>
<body>
    <canvas id="canvas" width="800" height="600"></canvas>
    <div id="ui">
        Lap: <span id="lap">1</span>/3 | Time: <span id="time">0</span>s | Speed: <span id="speed">0</span> | Checkpoints: <span id="check">0</span>
    </div>
    <script>
        const canvas = document.getElementById('canvas');
        const ctx = canvas.getContext('2d');
        const W = canvas.width, H = canvas.height;

        // Game state
        let player = { x: 0, y: 200, vx: 0, vy: 0, angle: 0, speed: 0, lap: 1, checkpoints: 0, trail: [] };
        let aiCars = [{ x: 20, y: 220, vx: 0, vy: 0.5, angle: 0, speed: 1, color: '#f0f' }, { x: -20, y: 180, vx: 0, vy: 0.4, angle: 0, speed: 0.8, color: '#ff0' }];
        let time = 0, keys = {}, trackOffset = 0, particles = [];

        // Procedural track segments (unique twisting neon road)
        const trackSegments = [];
        for (let i = 0; i < 1000; i++) {
            trackSegments.push({
                curve: Math.sin(i * 0.02) * 100 + Math.sin(i * 0.01) * 50,
                hill: Math.sin(i * 0.03) * 50,
                checkpoints: i % 20 === 0
            });
        }

        // Input handling
        document.addEventListener('keydown', e => keys[e.key] = true);
        document.addEventListener('keyup', e => keys[e.key] = false);

        function update(delta) {
            time += delta;

            // Player physics (arcade-style with drift)
            let accel = 0.3, friction = 0.95, turnSpeed = 0.03;
            if (keys['ArrowUp']) player.vy += accel * Math.cos(player.angle);
            if (keys['ArrowDown']) player.vy -= accel * 0.5 * Math.cos(player.angle);
            if (keys['ArrowLeft']) player.angle -= turnSpeed * (1 + player.speed * 0.1);
            if (keys['ArrowRight']) player.angle += turnSpeed * (1 + player.speed * 0.1);
            player.vx = player.vy * Math.sin(player.angle) * 0.1;
            player.vy *= friction;
            player.x += player.vx * delta * 60;
            player.y += player.vy * delta * 60;
            player.speed = Math.sqrt(player.vx*player.vx + player.vy*player.vy) * 10;
            player.trail.push({x: player.x, y: player.y}); if (player.trail.length > 20) player.trail.shift();

            // Track progression
            trackOffset += player.speed * 0.1;
            if (trackOffset > 1000) { trackOffset = 0; player.lap++; player.checkpoints = 0; }

            // Checkpoints (boost on hit)
            let seg = trackSegments[Math.floor(trackOffset) % 1000];
            if (seg.checkpoints && Math.abs(player.y - seg.curve) < 30) {
                player.checkpoints++;
                player.vy += 2; // Boost
                particles.push({x: player.x, y: seg.curve, vx: (Math.random()-0.5)*5, vy: (Math.random()-0.5)*5, life: 30});
            }

            // AI opponents (simple pursuit)
            aiCars.forEach(ai => {
                let targetY = trackSegments[Math.floor(trackOffset + ai.x * 0.1) % 1000].curve;
                ai.vy = 0.4 + Math.sin(time * 0.01 + ai.x) * 0.1;
                ai.angle += (targetY - ai.y) * 0.01;
                ai.y += ai.vy * delta * 60;
                ai.x += Math.sin(ai.angle) * 0.05;
            });

            // Particles
            particles = particles.filter(p => {
                p.x += p.vx * delta * 60; p.y += p.vy * delta * 60; p.life--;
                p.vx *= 0.98; p.vy *= 0.98;
                return p.life > 0;
            });
        }

        function render() {
            ctx.fillStyle = '#000'; ctx.fillRect(0, 0, W, H);

            // 3D projection (pseudo-3D road rendering)
            let camZ = 800;
            for (let i = 0; i < 100; i++) {
                let z = (trackOffset + i * 10) % 1000;
                let seg = trackSegments[Math.floor(z)];
                let scale = camZ / (camZ - i * 8);
                let x = W/2 + seg.curve * scale * 0.5;
                let y = H/2 + seg.hill * scale * 0.3 + i * 4;
                let w = 2000 * scale;

                // Road
                let gradient = ctx.createLinearGradient(x-w/2, y, x+w/2, y);
                gradient.addColorStop(0, '#0ff'); gradient.addColorStop(1, '#00f');
                ctx.fillStyle = gradient; ctx.fillRect(x-w/2, y-w/2, w, w);
                ctx.strokeStyle = '#fff'; ctx.lineWidth = 3 * scale; ctx.strokeRect(x-w/2, y-w/2, w, w);

                // Grass sides
                ctx.fillStyle = '#080'; ctx.fillRect(0, y-w/2, x-w/2, w);
                ctx.fillRect(x+w/2, y-w/2, W - (x+w/2), w);

                // Checkpoint glow
                if (seg.checkpoints) {
                    ctx.fillStyle = `rgba(0,255,0,${scale})`; ctx.fillRect(x-10*scale, y-10*scale, 20*scale, 20*scale);
                }
            }

            // Player car (3D scaled)
            let pScale = camZ / (camZ + player.y * 0.1);
            ctx.save(); ctx.translate(W/2 + player.x * pScale, H/2 + player.y * pScale * 0.5);
            ctx.rotate(player.angle);
            ctx.fillStyle = '#f00'; ctx.shadowColor = '#f00'; ctx.shadowBlur = 20;
            ctx.fillRect(-20*pScale, -10*pScale, 40*pScale, 20*pScale);
            ctx.fillStyle = '#fff'; ctx.fillRect(-15*pScale, -8*pScale, 30*pScale, 16*pScale);
            ctx.restore();

            // AI cars
            aiCars.forEach(ai => {
                let aScale = camZ / (camZ + ai.y * 0.1);
                ctx.save(); ctx.translate(W/2 + ai.x * aScale, H/2 + ai.y * aScale * 0.5);
                ctx.rotate(ai.angle);
                ctx.fillStyle = ai.color; ctx.shadowColor = ai.color; ctx.shadowBlur = 15;
                ctx.fillRect(-18*aScale, -9*aScale, 36*aScale, 18*aScale);
                ctx.restore();
            });

            // Trails
            ctx.strokeStyle = 'rgba(0,255,255,0.5)'; ctx.lineWidth = 3; ctx.lineCap = 'round';
            ctx.beginPath(); player.trail.forEach((p, i) => {
                let tScale = camZ / (camZ + p.y * 0.1);
                if (i) ctx.lineTo(W/2 + p.x * tScale, H/2 + p.y * tScale * 0.5);
                else ctx.moveTo(W/2 + p.x * tScale, H/2 + p.y * tScale * 0.5);
            }); ctx.stroke();

            // Particles
            particles.forEach(p => {
                let pScale = camZ / (camZ + p.y * 0.1);
                ctx.fillStyle = `rgba(0,255,0,${p.life/30})`; ctx.beginPath();
                ctx.arc(W/2 + p.x * pScale, H/2 + p.y * pScale * 0.5, 5*pScale, 0, Math.PI*2); ctx.fill();
            });

            // UI updates
            document.getElementById('lap').textContent = player.lap;
            document.getElementById('time').textContent = Math.floor(time);
            document.getElementById('speed').textContent = Math.floor(player.speed);
            document.getElementById('check').textContent = player.checkpoints;

            if (player.lap > 3) { ctx.fillStyle = '#0f0'; ctx.font = 'bold 50px monospace'; ctx.fillText('WINNER!', W/2-150, H/2); }
        }

        let lastTime = 0;
        function loop(t) {
            let delta = (t - lastTime) / 1000; lastTime = t;
            update(delta); render(); requestAnimationFrame(loop);
        }
        requestAnimationFrame(loop);
    </script>
</body>
</html>
