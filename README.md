<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Stickman Street Fighter: Unchained Edition</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            background: #0a0a0c;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            color: white;
            overflow: hidden;
            user-select: none;
        }
        #game-container {
            position: relative;
            box-shadow: 0 20px 50px rgba(0,0,0,0.8);
            border-radius: 8px;
            overflow: hidden;
        }
        canvas {
            background: linear-gradient(to bottom, #111115, #1a1a24);
            display: block;
        }
        #instructions {
            margin-top: 15px;
            text-align: center;
            color: #666;
            font-size: 13px;
            line-height: 1.6;
            max-width: 800px;
        }
        .key {
            background: #222;
            color: #fff;
            padding: 1px 6px;
            border-radius: 4px;
            border: 1px solid #444;
            font-family: monospace;
        }
    </style>
</head>
<body>

    <div id="game-container">
        <canvas id="gameCanvas" width="800" height="450"></canvas>
    </div>

    <div id="instructions">
        <span style="color:#ffcc00; font-weight:bold;">[GITHUB UNCHAINED EDITION - 60FPS NATIVE]</span><br>
        <strong>Menu:</strong> <span class="key">A</span> / <span class="key">D</span> Select | <span class="key">W</span> / <span class="key">S</span> Difficulty | Press <span class="key">F</span> to Confirm<br>
        <strong>Combat:</strong> <span class="key">A</span><span class="key">D</span> Move | <span class="key">W</span> Jump | <span class="key">S</span> Block | <span class="key">F</span> Punch | <span class="key">R</span> Kick | <span class="key">J</span> Special | <span class="key">K</span> Heavy | <span class="key">E</span> Ultimate
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');

        // Block defaults
        window.addEventListener('keydown', e => {
            if(["Space", "ArrowUp", "ArrowDown", "ArrowLeft", "ArrowRight"].includes(e.code)) e.preventDefault();
        });

        const GRAVITY = 0.55;
        const GROUND_Y = 380;
        
        let gameState = 'SELECT'; 
        let menuSelection = 0; 
        let botDifficulties = ['EASY', 'NORMAL', 'HARD', 'UNFAIR'];
        let selectedDiffIndex = 1; 
        let particles = [];
        let screenShake = 0;
        let matchTimer = 99;
        let timerInterval;

        const keys = { a: false, d: false, w: false, s: false, f: false, r: false, j: false, k: false, e: false };
        
        window.addEventListener('keydown', (e) => {
            const key = e.key.toLowerCase();
            if (key in keys) {
                if (!keys[key]) {
                    if (gameState === 'SELECT') {
                        if (key === 'a') menuSelection = (menuSelection - 1 + CHAR_PROFILES.length) % CHAR_PROFILES.length;
                        if (key === 'd') menuSelection = (menuSelection + 1) % CHAR_PROFILES.length;
                        if (key === 'w') selectedDiffIndex = (selectedDiffIndex - 1 + botDifficulties.length) % botDifficulties.length;
                        if (key === 's') selectedDiffIndex = (selectedDiffIndex + 1) % botDifficulties.length;
                        if (key === 'f') confirmSelection();
                    }
                    else if (gameState === 'GAMEOVER' && key === 'r') returnToMenu();
                }
                keys[key] = true;
            }
        });

        window.addEventListener('keyup', e => { keys[e.key.toLowerCase()] = false; });

        const CHAR_PROFILES = [
            { id: 'shadow', name: "SHADOW", color: "#33bcff", maxHp: 250, speed: 5.2, desc: "Clone Burst Engine", moves: { 
                punch: { reach: 50, time: 10, dmg: 8 }, kick: { reach: 65, time: 15, dmg: 14 },
                move3: { reach: 90, time: 14, dmg: 15 }, move4: { reach: 70, time: 24, dmg: 25 }
            }},
            { id: 'viper', name: "VIPER", color: "#22c55e", maxHp: 230, speed: 5.8, desc: "Fluid Venom Strikes", moves: { 
                punch: { reach: 45, time: 8, dmg: 7 }, kick: { reach: 80, time: 20, dmg: 18 },
                move3: { reach: 70, time: 14, dmg: 14 }, move4: { reach: 90, time: 26, dmg: 26 }
            }},
            { id: 'blaze', name: "BLAZE", color: "#ff4444", maxHp: 290, speed: 4.4, desc: "Explosive Heavyweight", moves: { 
                punch: { reach: 60, time: 14, dmg: 15 }, kick: { reach: 60, time: 18, dmg: 19 },
                move3: { reach: 75, time: 22, dmg: 22 }, move4: { reach: 95, time: 32, dmg: 35 }
            }},
            { id: 'volt', name: "VOLT", color: "#b822c5", maxHp: 240, speed: 6.5, desc: "Lightning Teleportation", moves: { 
                punch: { reach: 50, time: 7, dmg: 7 }, kick: { reach: 65, time: 12, dmg: 12 },
                move3: { reach: 130, time: 12, dmg: 14 }, move4: { reach: 70, time: 18, dmg: 20 }
            }},
            { id: 'midas', name: "MIDAS", color: "#ecc94b", maxHp: 320, speed: 4.6, desc: "Golden Heavy Armor", moves: { 
                punch: { reach: 55, time: 12, dmg: 12 }, kick: { reach: 65, time: 18, dmg: 16 },
                move3: { reach: 80, time: 18, dmg: 18 }, move4: { reach: 85, time: 28, dmg: 30 }
            }}
        ];

        class Particle {
            constructor(x, y, color, customV = null) {
                this.x = x; this.y = y; this.color = color;
                this.vx = customV ? customV.x : Math.random() * 8 - 4;
                this.vy = customV ? customV.y : Math.random() * -6 - 2;
                this.alpha = 1;
                this.size = Math.random() * 4 + 2;
                this.gravity = customV ? 0.1 : 0.2;
            }
            update() {
                this.x += this.vx; this.y += this.vy; this.vy += this.gravity; this.alpha -= 0.03;
            }
            draw() {
                ctx.save(); ctx.globalAlpha = this.alpha; ctx.fillStyle = this.color;
                ctx.shadowBlur = 10; ctx.shadowColor = this.color;
                ctx.fillRect(this.x, this.y, this.size, this.size); ctx.restore();
            }
        }

        class Fighter {
            constructor(x, y, profile, isPlayer, difficulty = 'NORMAL') {
                Object.assign(this, profile);
                this.x = x; this.y = y; this.isPlayer = isPlayer; this.difficulty = difficulty;
                this.health = this.maxHp; this.width = 50; this.height = 100;
                this.vx = 0; this.vy = 0; this.isGrounded = false; this.facing = isPlayer ? 1 : -1;
                this.isAttacking = false; this.attackType = null; this.attackTimer = 0;
                this.maxAttackDuration = 0; this.hasHit = false; this.hitstun = 0;
                this.ultMeter = 0; this.isBlocking = false; this.combo = 0; this.comboTimer = 0;
                this.history = [];
            }

            update(opponent) {
                // Motion trail mechanics re-enabled!
                this.history.push({x: this.x, y: this.y, type: this.attackType});
                if(this.history.length > 5) this.history.shift();

                if (this.comboTimer > 0) this.comboTimer--; else this.combo = 0;
                if (this.attackType !== 'ultimate') this.facing = (this.x < opponent.x) ? 1 : -1;

                if (this.isPlayer) {
                    if (this.hitstun > 0) {
                        this.hitstun--; this.isBlocking = false; this.x += this.facing * -1.5;
                    } else if (keys.s && this.isGrounded && !this.isAttacking) {
                        this.isBlocking = true; this.vx = 0;
                    } else {
                        this.isBlocking = false;
                        if (keys.a) this.vx = -this.speed; else if (keys.d) this.vx = this.speed; else this.vx = 0;
                        if (keys.w && this.isGrounded) { this.vy = -14; this.isGrounded = false; }
                    }
                    if (!this.isAttacking && !this.isBlocking && this.hitstun <= 0) {
                        if (keys.e && this.ultMeter >= 100) this.startUltimate(opponent);
                        else if (keys.f) this.startAttack('punch');
                        else if (keys.r) this.startAttack('kick');
                        else if (keys.j) this.startAttack('move3');
                        else if (keys.k) this.startAttack('move4');
                    }
                } else {
                    // Smart AI Configuration
                    if (this.hitstun > 0) {
                        this.hitstun--; this.x += this.facing * -1.5;
                    } else {
                        let dist = Math.abs(this.x - opponent.x);
                        let reactionChance = this.difficulty === 'EASY' ? 0.02 : (this.difficulty === 'NORMAL' ? 0.07 : (this.difficulty === 'HARD' ? 0.15 : 0.3));
                        
                        if (opponent.isAttacking && Math.random() < reactionChance * 2 && this.isGrounded) {
                            this.isBlocking = true; this.vx = 0;
                        } else {
                            this.isBlocking = false;
                            if (dist > 70) {
                                this.vx = (opponent.x > this.x) ? this.speed * 0.8 : -this.speed * 0.8;
                            } else {
                                this.vx = 0;
                                if (Math.random() < reactionChance) {
                                    let r = Math.random();
                                    if(this.ultMeter >= 100 && r < 0.2) this.startUltimate(opponent);
                                    else if (r < 0.4) this.startAttack('punch');
                                    else if (r < 0.7) this.startAttack('kick');
                                    else this.startAttack('move3');
                                }
                            }
                        }
                    }
                }
                this.vy += GRAVITY; this.x += this.vx; this.y += this.vy;
                if (this.y + this.height >= GROUND_Y) { this.y = GROUND_Y - this.height; this.vy = 0; this.isGrounded = true; }
                if (this.x < 10) this.x = 10; if (this.x + this.width > canvas.width - 10) this.x = canvas.width - this.width - 10;

                this.handleAttacks(opponent);
            }

            startAttack(type) {
                this.isAttacking = true; this.attackType = type;
                this.attackTimer = this.moves[type].time; this.maxAttackDuration = this.attackTimer;
                this.hasHit = false; this.isBlocking = false;
                if (this.id === 'volt' && type === 'move3') this.x += this.facing * 110; 
                if (this.id === 'blaze' && type === 'move3') this.vy = -7;
            }

            startUltimate(opponent) {
                this.isAttacking = true; this.attackType = 'ultimate';
                this.attackTimer = 50; this.maxAttackDuration = 50;
                this.hasHit = false; this.ultMeter = 0; screenShake = 15;
                for(let i=0; i<30; i++) particles.push(new Particle(this.x + this.width/2, this.y + this.height/2, this.color));
            }

            handleAttacks(opponent) {
                if (!this.isAttacking) return;
                this.attackTimer--;
                
                if (this.attackTimer === Math.floor(this.maxAttackDuration / 2) && !this.hasHit) {
                    let reach = this.attackType === 'ultimate' ? 180 : this.moves[this.attackType].reach;
                    let hit = Math.abs((this.x + this.width/2 + (reach/2 * this.facing)) - (opponent.x + opponent.width/2)) < (reach/2 + opponent.width/2) && Math.abs(this.y - opponent.y) < 100;

                    if (hit) {
                        this.hasHit = true;
                        if (opponent.isBlocking) {
                            screenShake = 3;
                            for(let i=0; i<5; i++) particles.push(new Particle(opponent.x + opponent.width/2, opponent.y + 40, '#ffffff'));
                        } else {
                            let dmg = this.attackType === 'ultimate' ? 60 : this.moves[this.attackType].dmg;
                            opponent.health = Math.max(0, opponent.health - dmg);
                            opponent.hitstun = this.attackType === 'ultimate' ? 30 : 15;
                            this.combo++; this.comboTimer = 90;
                            this.ultMeter = Math.min(100, this.ultMeter + 15);
                            screenShake = this.attackType === 'ultimate' ? 20 : 6;
                            for(let i=0; i<15; i++) particles.push(new Particle(opponent.x + opponent.width/2, opponent.y + 30, this.color));
                        }
                    }
                }
                if (this.attackTimer <= 0) { this.isAttacking = false; this.attackType = null; }
            }

            draw() {
                // High performance fluid limb rendering loop
                this.history.forEach((frame, idx) => {
                    if (frame.type) {
                        ctx.save(); ctx.globalAlpha = (idx / this.history.length) * 0.2;
                        ctx.strokeStyle = this.color; ctx.lineWidth = 4;
                        this.drawStickmanFrame(frame.x, frame.y); ctx.restore();
                    }
                });

                ctx.save();
                ctx.strokeStyle = this.hitstun > 0 ? '#ff3333' : this.color;
                ctx.lineWidth = 6; ctx.lineCap = 'round'; ctx.lineJoin = 'round';
                ctx.shadowBlur = this.attackType === 'ultimate' ? 20 : 5; ctx.shadowColor = this.color;
                this.drawStickmanFrame(this.x, this.y);
                ctx.restore();
            }

            drawStickmanFrame(baseX, baseY) {
                let midX = baseX + this.width / 2;
                let headY = baseY + 20, chestY = baseY + 50, pelvisY = baseY + 75;
                let anim = this.isAttacking ? Math.sin((this.attackTimer / this.maxAttackDuration) * Math.PI) : 0;

                // Head
                ctx.beginPath(); ctx.arc(midX, headY, 12, 0, Math.PI * 2); ctx.stroke();
                // Spine
                ctx.beginPath(); ctx.moveTo(midX, headY + 12); ctx.lineTo(midX, pelvisY); ctx.stroke();

                // Advanced multi-joint limb calculation
                ctx.beginPath();
                if (this.isBlocking) {
                    ctx.moveTo(midX, chestY); ctx.lineTo(midX + 15 * this.facing, chestY - 10); ctx.lineTo(midX + 10 * this.facing, headY);
                    ctx.moveTo(midX, chestY); ctx.lineTo(midX + 20 * this.facing, chestY); ctx.lineTo(midX + 15 * this.facing, headY + 10);
                } else if (this.attackType === 'punch') {
                    ctx.moveTo(midX, chestY); ctx.lineTo(midX + 45 * this.facing * anim, chestY - 5);
                    ctx.moveTo(midX, chestY); ctx.lineTo(midX - 10 * this.facing, chestY + 15);
                } else if (this.attackType === 'move3' || this.attackType === 'ultimate') {
                    ctx.moveTo(midX, chestY); ctx.lineTo(midX + 60 * this.facing * anim, chestY - 20 * anim);
                    ctx.moveTo(midX, chestY); ctx.lineTo(midX + 50 * this.facing * anim, chestY + 20 * anim);
                } else {
                    let walk = Math.sin(Date.now() * 0.01) * 10 * (this.vx !== 0 ? 1 : 0);
                    ctx.moveTo(midX, chestY); ctx.lineTo(midX - 15 * this.facing + walk, chestY + 15);
                    ctx.moveTo(midX, chestY); ctx.lineTo(midX + 10 * this.facing - walk, chestY + 18);
                }
                ctx.stroke();

                ctx.beginPath();
                if (this.attackType === 'kick' || this.attackType === 'move4') {
                    ctx.moveTo(midX, pelvisY); ctx.lineTo(midX + 65 * this.facing * anim, pelvisY + 10 - 30 * anim);
                    ctx.moveTo(midX, pelvisY); ctx.lineTo(midX - 15 * this.facing, pelvisY + 25);
                } else {
                    let swing = Math.sin(Date.now() * 0.01) * 15 * (this.vx !== 0 ? 1 : 0);
                    ctx.moveTo(midX, pelvisY); ctx.lineTo(midX - 15 + swing, pelvisY + 15); ctx.lineTo(midX - 18 + swing, baseY + this.height);
                    ctx.moveTo(midX, pelvisY); ctx.lineTo(midX + 15 - swing, pelvisY + 15); ctx.lineTo(midX + 18 - swing, baseY + this.height);
                }
                ctx.stroke();
            }
        }

        let p1, p2;

        function confirmSelection() {
            p1 = new Fighter(150, GROUND_Y - 100, CHAR_PROFILES[menuSelection], true);
            let cpuIdx = (menuSelection + 1) % CHAR_PROFILES.length;
            p2 = new Fighter(600, GROUND_Y - 100, CHAR_PROFILES[cpuIdx], false, botDifficulties[selectedDiffIndex]);
            gameState = 'FIGHT'; matchTimer = 99;
            clearInterval(timerInterval);
            timerInterval = setInterval(() => { if(gameState === 'FIGHT' && matchTimer > 0) matchTimer--; }, 1000);
        }

        function returnToMenu() { gameState = 'SELECT'; }

        function drawHUD() {
            // Elegant Vector-Based Canvas UI (Zero Lag on GitHub)
            ctx.fillStyle = '#fff'; ctx.font = "bold 16px 'Segoe UI'"; ctx.textAlign = 'left'; ctx.fillText(p1.name, 30, 35);
            ctx.textAlign = 'right'; ctx.fillText(`${p2.name} (${p2.difficulty})`, canvas.width - 30, 35);

            // Timer
            ctx.fillStyle = '#ffcc00'; ctx.font = "bold 28px monospace"; ctx.textAlign = 'center'; ctx.fillText(matchTimer, canvas.width/2, 45);

            // P1 HP Bar
            ctx.fillStyle = '#222'; ctx.fillRect(30, 45, 280, 16);
            ctx.fillStyle = p1.color; ctx.fillRect(30, 45, 280 * (p1.health / p1.maxHp), 16);
            // P1 Ult Meter
            ctx.fillStyle = '#222'; ctx.fillRect(30, 65, 140, 6);
            ctx.fillStyle = '#ffcc00'; ctx.fillRect(30, 65, 140 * (p1.ultMeter / 100), 6);
            if(p1.ultMeter >= 100) { ctx.fillStyle = '#fff'; ctx.font = "italic 9px sans-serif"; ctx.textAlign='left'; ctx.fillText("ULT READY [E]", 30, 82); }

            // P2 HP Bar
            ctx.fillStyle = '#222'; ctx.fillRect(canvas.width - 310, 45, 280, 16);
            ctx.fillStyle = p2.color; ctx.fillRect(canvas.width - 310, 45, 280 * (p2.health / p2.maxHp), 16);
            // P2 Ult Meter
            ctx.fillStyle = '#222'; ctx.fillRect(canvas.width - 170, 65, 140, 6);
            ctx.fillStyle = '#ffcc00'; ctx.fillRect(canvas.width - 170, 65, 140 * (p2.ultMeter / 100), 6);

            // Combo Overlays
            if(p1.combo > 1) { ctx.fillStyle = p1.color; ctx.font = "italic bold 26px sans-serif"; ctx.textAlign='left'; ctx.fillText(`${p1.combo} HITS`, 40, 130); }
            if(p2.combo > 1) { ctx.fillStyle = p2.color; ctx.font = "italic bold 26px sans-serif"; ctx.textAlign='right'; ctx.fillText(`${p2.combo} HITS`, canvas.width - 40, 130); }
        }

        function drawCharacterSelect() {
            ctx.fillStyle = "#0c0c0f"; ctx.fillRect(0, 0, canvas.width, canvas.height);
            ctx.fillStyle = "#fff"; ctx.font = "bold 24px 'Segoe UI'"; ctx.textAlign = "center"; ctx.fillText("SELECT YOUR FIGHTER", canvas.width / 2, 60);
            ctx.fillStyle = "#555"; ctx.font = "14px 'Segoe UI'"; ctx.fillText("Difficulty: Use [W/S] to Change", canvas.width / 2, 85);
            ctx.fillStyle = selectedDiffIndex >= 2 ? '#ff3333' : '#33bcff'; ctx.font = "bold 18px monospace"; ctx.fillText(`BOT ENGINE: ${botDifficulties[selectedDiffIndex]}`, canvas.width / 2, 110);

            let cardW = 120, cardH = 200, gap = 15;
            let startX = (canvas.width - ((cardW * CHAR_PROFILES.length) + (gap * (CHAR_PROFILES.length - 1)))) / 2, cardY = 150;

            CHAR_PROFILES.forEach((profile, index) => {
                let cardX = startX + (index * (cardW + gap)), isSelected = menuSelection === index;
                ctx.fillStyle = isSelected ? "#1c1c24" : "#111116"; ctx.fillRect(cardX, cardY, cardW, cardH);
                ctx.lineWidth = isSelected ? 4 : 1; ctx.strokeStyle = isSelected ? profile.color : "#222"; ctx.strokeRect(cardX, cardY, cardW, cardH);
                ctx.fillStyle = profile.color; ctx.font = "bold 15px monospace"; ctx.fillText(profile.name, cardX + cardW/2, cardY + 35);
                ctx.fillStyle = '#666'; ctx.font = "11px sans-serif"; ctx.fillText(`HP: ${profile.maxHp}`, cardX + cardW/2, cardY + 75);
                ctx.fillText(`SPEED: ${profile.speed}`, cardX + cardW/2, cardY + 95);
                ctx.fillStyle = '#aaa'; ctx.font = "italic 10px sans-serif"; ctx.fillText(profile.desc, cardX + cardW/2, cardY + 135, cardW - 10);
                if(isSelected) { ctx.fillStyle = '#fff'; ctx.font = "bold 11px sans-serif"; ctx.fillText("[PRESS F]", cardX + cardW/2, cardY + 180); }
            });
        }

        function mainLoop() {
            if (gameState === 'SELECT') {
                drawCharacterSelect();
            } else if (gameState === 'FIGHT') {
                ctx.clearRect(0, 0, canvas.width, canvas.height);
                p1.update(p2); p2.update(p1);

                ctx.save();
                if (screenShake > 0) {
                    ctx.translate((Math.random() - 0.5) * screenShake, (Math.random() - 0.5) * screenShake);
                    screenShake *= 0.9; if (screenShake < 0.5) screenShake = 0;
                }

                // Stage Drawing
                ctx.fillStyle = '#1c1c28'; ctx.fillRect(0, GROUND_Y, canvas.width, canvas.height - GROUND_Y);
                ctx.fillStyle = '#333344'; ctx.fillRect(0, GROUND_Y, canvas.width, 4);

                for (let i = particles.length - 1; i >= 0; i--) {
                    particles[i].update(); particles[i].draw();
                    if (particles[i].alpha <= 0) particles.splice(i, 1);
                }

                p1.draw(); p2.draw();
                ctx.restore(); 
                drawHUD();

                if (p1.health <= 0 || p2.health <= 0 || matchTimer === 0) {
                    gameState = 'GAMEOVER'; clearInterval(timerInterval);
                }
            } else if (gameState === 'GAMEOVER') {
                ctx.fillStyle = "rgba(5,5,8,0.85)"; ctx.fillRect(0,0, canvas.width, canvas.height);
                ctx.textAlign = "center"; ctx.fillStyle = "#fff"; ctx.font = "bold 42px 'Segoe UI'";
                let winText = "DRAW MATCH!";
                if(p1.health !== p2.health) winText = p1.health > p2.health ? `${p1.name} WINS!` : `${p2.name} WINS!`;
                ctx.fillText(winText, canvas.width/2, 200);
                ctx.font = "16px sans-serif"; ctx.fillStyle = '#888'; ctx.fillText("Press [R] to Return to Character Select", canvas.width/2, 260);
            }
            requestAnimationFrame(mainLoop);
        }

        mainLoop();
    </script>
</body>
</html>
