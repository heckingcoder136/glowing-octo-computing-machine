<!DOCTYPE html>
<html>
<head>
    <title>Simple FPS Game</title>
    <style>
        body {
            margin: 0;
            padding: 0;
            overflow: hidden;
            background-color: #000;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
        }
        canvas {
            border: 1px solid #333;
            cursor: crosshair;
        }
        #hud {
            position: absolute;
            top: 10px;
            left: 10px;
            color: white;
            font-family: Arial, sans-serif;
            font-size: 20px;
        }
    </style>
</head>
<body>
    <canvas id="gameCanvas"></canvas>
    <div id="hud">
        <div>Health: <span id="health">100</span></div>
        <div>Score: <span id="score">0</span></div>
        <div>Ammo: <span id="ammo">30</span></div>
    </div>

    <script>
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        canvas.width = 800;
        canvas.height = 600;

        // Game state
        const game = {
            player: {
                x: canvas.width / 2,
                y: canvas.height - 100,
                health: 100,
                ammo: 30,
                maxAmmo: 30
            },
            enemies: [],
            bullets: [],
            score: 0,
            gameOver: false
        };

        // Input handling
        const keys = {};
        const mouse = { x: 0, y: 0, clicked: false };

        document.addEventListener('keydown', (e) => keys[e.key] = true);
        document.addEventListener('keyup', (e) => keys[e.key] = false);
        
        canvas.addEventListener('mousemove', (e) => {
            const rect = canvas.getBoundingClientRect();
            mouse.x = e.clientX - rect.left;
            mouse.y = e.clientY - rect.top;
        });

        canvas.addEventListener('mousedown', () => mouse.clicked = true);
        canvas.addEventListener('mouseup', () => mouse.clicked = false);

        // Enemy class
        class Enemy {
            constructor(x, y) {
                this.x = x;
                this.y = y;
                this.width = 40;
                this.height = 40;
                this.speed = 1 + Math.random() * 2;
                this.health = 50;
                this.color = `hsl(${Math.random() * 60}, 100%, 50%)`;
            }

            update() {
                // Move towards player
                const dx = game.player.x - this.x;
                const dy = game.player.y - this.y;
                const distance = Math.sqrt(dx * dx + dy * dy);
                
                if (distance > 0) {
                    this.x += (dx / distance) * this.speed;
                    this.y += (dy / distance) * this.speed;
                }

                // Check collision with player
                if (distance < 30) {
                    game.player.health -= 1;
                    if (game.player.health <= 0) {
                        game.gameOver = true;
                    }
                }
            }

            draw() {
                ctx.fillStyle = this.color;
                ctx.fillRect(this.x - this.width/2, this.y - this.height/2, this.width, this.height);
                
                // Health bar
                ctx.fillStyle = 'red';
                ctx.fillRect(this.x - this.width/2, this.y - this.height/2 - 10, this.width, 5);
                ctx.fillStyle = 'green';
                ctx.fillRect(this.x - this.width/2, this.y - this.height/2 - 10, this.width * (this.health/50), 5);
            }
        }

        // Bullet class
        class Bullet {
            constructor(x, y, targetX, targetY) {
                this.x = x;
                this.y = y;
                this.radius = 4;
                this.speed = 10;
                
                const dx = targetX - x;
                const dy = targetY - y;
                const distance = Math.sqrt(dx * dx + dy * dy);
                
                this.vx = (dx / distance) * this.speed;
                this.vy = (dy / distance) * this.speed;
            }

            update() {
                this.x += this.vx;
                this.y += this.vy;
            }

            draw() {
                ctx.fillStyle = 'yellow';
                ctx.beginPath();
                ctx.arc(this.x, this.y, this.radius, 0, Math.PI * 2);
                ctx.fill();
            }

            isOffScreen() {
                return this.x < 0 || this.x > canvas.width || 
                       this.y < 0 || this.y > canvas.height;
            }
        }

        // Spawn enemies
        function spawnEnemy() {
            const side = Math.floor(Math.random() * 4);
            let x, y;
            
            switch(side) {
                case 0: x = Math.random() * canvas.width; y = -20; break;
                case 1: x = canvas.width + 20; y = Math.random() * canvas.height; break;
                case 2: x = Math.random() * canvas.width; y = canvas.height + 20; break;
                case 3: x = -20; y = Math.random() * canvas.height; break;
            }
            
            game.enemies.push(new Enemy(x, y));
        }

        // Game loop
        function update() {
            if (game.gameOver) return;

            // Player movement
            const moveSpeed = 5;
            if (keys['a'] || keys['A']) game.player.x -= moveSpeed;
            if (keys['d'] || keys['D']) game.player.x += moveSpeed;
            if (keys['w'] || keys['W']) game.player.y -= moveSpeed;
            if (keys['s'] || keys['S']) game.player.y += moveSpeed;

            // Keep player in bounds
            game.player.x = Math.max(20, Math.min(canvas.width - 20, game.player.x));
            game.player.y = Math.max(20, Math.min(canvas.height - 20, game.player.y));

            // Shooting
            if (mouse.clicked && game.player.ammo > 0) {
                game.bullets.push(new Bullet(game.player.x, game.player.y, mouse.x, mouse.y));
                game.player.ammo--;
                mouse.clicked = false;
            }

            // Reload
            if (keys['r'] || keys['R']) {
                game.player.ammo = game.player.maxAmmo;
            }

            // Update bullets
            game.bullets = game.bullets.filter(bullet => {
                bullet.update();
                
                // Check collision with enemies
                for (let i = game.enemies.length - 1; i >= 0; i--) {
                    const enemy = game.enemies[i];
                    const dx = bullet.x - enemy.x;
                    const dy = bullet.y - enemy.y;
                    const distance = Math.sqrt(dx * dx + dy * dy);
                    
                    if (distance < enemy.width/2) {
                        enemy.health -= 25;
                        if (enemy.health <= 0) {
                            game.enemies.splice(i, 1);
                            game.score += 10;
                        }
                        return false;
                    }
                }
                
                return !bullet.isOffScreen();
            });

            // Update enemies
            game.enemies.forEach(enemy => enemy.update());

            // Spawn new enemies
            if (Math.random() < 0.02) {
                spawnEnemy();
            }

            // Update HUD
            document.getElementById('health').textContent = Math.max(0, game.player.health);
            document.getElementById('score').textContent = game.score;
            document.getElementById('ammo').textContent = game.player.ammo;
        }

        function draw() {
            // Clear canvas
            ctx.fillStyle = '#222';
            ctx.fillRect(0, 0, canvas.width, canvas.height);

            // Draw grid
            ctx.strokeStyle = '#333';
            ctx.lineWidth = 1;
            for (let i = 0; i < canvas.width; i += 50) {
                ctx.beginPath();
                ctx.moveTo(i, 0);
                ctx.lineTo(i, canvas.height);
                ctx.stroke();
            }
            for (let i = 0; i < canvas.height; i += 50) {
                ctx.beginPath();
                ctx.moveTo(0, i);
                ctx.lineTo(canvas.width, i);
                ctx.stroke();
            }

            // Draw player
            ctx.fillStyle = 'lime';
            ctx.beginPath();
            ctx.arc(game.player.x, game.player.y, 20, 0, Math.PI * 2);
            ctx.fill();

            // Draw crosshair
            ctx.strokeStyle = 'white';
            ctx.lineWidth = 2;
            ctx.beginPath();
            ctx.moveTo(mouse.x - 10, mouse.y);
            ctx.lineTo(mouse.x + 10, mouse.y);
            ctx.moveTo(mouse.x, mouse.y - 10);
            ctx.lineTo(mouse.x, mouse.y + 10);
            ctx.stroke();

            // Draw enemies
            game.enemies.forEach(enemy => enemy.draw());

            // Draw bullets
            game.bullets.forEach(bullet => bullet.draw());

            // Game over screen
            if (game.gameOver) {
                ctx.fillStyle = 'rgba(0, 0, 0, 0.7)';
                ctx.fillRect(0, 0, canvas.width, canvas.height);
                ctx.fillStyle = 'white';
                ctx.font = '48px Arial';
                ctx.textAlign = 'center';
                ctx.fillText('GAME OVER', canvas.width/2, canvas.height/2);
                ctx.font = '24px Arial';
                ctx.fillText(`Final Score: ${game.score}`, canvas.width/2, canvas.height/2 + 50);
                ctx.fillText('Press F5 to restart', canvas.width/2, canvas.height/2 + 100);
            }
        }

        function gameLoop() {
            update();
            draw();
            requestAnimationFrame(gameLoop);
        }

        // Start game
        gameLoop();

        // Instructions
        console.log('Controls:');
        console.log('WASD - Move');
        console.log('Mouse - Aim');
        console.log('Click - Shoot');
        console.log('R - Reload');
    </script>
</body>
</html>
