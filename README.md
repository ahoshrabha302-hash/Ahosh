<!DOCTYPE html>
<html lang="hi">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
    <title>एडवांस्ड मोबाइल शूटिंग गेम</title>
    <style>
        * {
            box-sizing: border-box;
            margin: 0;
            padding: 0;
            user-select: none;
        }
        body {
            background: #0d0d1a;
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            overflow: hidden;
            display: flex;
            flex-direction: column;
            align-items: center;
            justify-content: center;
            height: 100vh;
        }
        /* गेम स्क्रीन */
        canvas {
            background: radial-gradient(circle, #111 60%, #000 100%);
            border: 3px solid #00ffcc;
            box-shadow: 0 0 20px #00ffcc;
            max-width: 100%;
            max-height: 65vh;
            border-radius: 8px;
        }
        /* मोबाइल कंट्रोल्स */
        #controls {
            display: flex;
            justify-content: space-between;
            width: 100%;
            max-width: 400px;
            padding: 15px;
            margin-top: 10px;
        }
        .btn {
            background: linear-gradient(135deg, #333, #222);
            color: #fff;
            border: 2px solid #555;
            padding: 18px 28px;
            font-size: 20px;
            font-weight: bold;
            border-radius: 15px;
            box-shadow: 0 4px 6px rgba(0,0,0,0.5);
            touch-action: manipulation;
            transition: 0.1s;
        }
        .btn:active {
            transform: scale(0.95);
            background: #555;
        }
        #shoot-btn {
            background: linear-gradient(135deg, #ff0055, #990033);
            border-color: #ff0055;
            box-shadow: 0 0 10px #ff0055;
            padding: 18px 40px;
        }
        #shoot-btn:active {
            background: #ff0055;
        }
    </style>
</head>
<body>

    <canvas id="gameCanvas" width="400" height="500"></canvas>

    <!-- मोबाइल के लिए टच बटन -->
    <div id="controls">
        <button class="btn" id="left-btn">◀ Left</button>
        <button class="btn" id="shoot-btn">🔥 FIRE</button>
        <button class="btn" id="right-btn">Right ▶</button>
    </div>

    <script>
        const canvas = document.getElementById("gameCanvas");
        const ctx = canvas.getContext("2d");

        // गेम वेरिएबल्स
        let score = 0;
        let gameLevel = 1;
        let baseEnemySpeed = 2; // शुरुआती स्पीड
        let gameOver = false;
        let enemySpawnInterval = 1200; // दुश्मन आने का समय (मिलीसेकंड)
        let lastSpawnTime = 0;

        // प्लेयर ऑब्जेक्ट (स्पेसशिप डिजाइन)
        const player = {
            x: canvas.width / 2 - 20,
            y: canvas.height - 60,
            width: 40,
            height: 35,
            speed: 6,
            movingLeft: false,
            movingRight: false,
            draw() {
                ctx.fillStyle = "#00ffcc"; // नियॉन ग्रीन रॉकेट
                ctx.beginPath();
                ctx.moveTo(this.x + this.width / 2, this.y); // टॉप नोज
                ctx.lineTo(this.x + this.width, this.y + this.height); // राइट विंग
                ctx.lineTo(this.x, this.y + this.height); // लेफ्ट विंग
                ctx.closePath();
                ctx.fill();
                
                // रॉकेट का थ्रस्टर (आग)
                ctx.fillStyle = "#ff5500";
                ctx.fillRect(this.x + this.width/2 - 4, this.y + this.height, 8, 6);
            }
        };

        let bullets = [];
        let enemies = [];

        // नया दुश्मन (एलियन UFO लुक)
        class Enemy {
            constructor() {
                this.width = 40;
                this.height = 25;
                this.x = Math.random() * (canvas.width - this.width);
                this.y = -this.height;
                // लेवल के हिसाब से दुश्मन की स्पीड बढ़ेगी
                this.speed = baseEnemySpeed + (gameLevel * 0.5) + Math.random();
            }
            update() {
                this.y += this.speed;
            }
            draw() {
                // UFO की बॉडी (लाल/बैंगनी एलियन)
                ctx.fillStyle = "#ff3366";
                ctx.beginPath();
                ctx.ellipse(this.x + this.width/2, this.y + this.height/2, this.width/2, this.height/2, 0, 0, Math.PI * 2);
                ctx.fill();

                // UFO का ऊपर का कॉकपिट
                ctx.fillStyle = "#33ccff";
                ctx.beginPath();
                ctx.arc(this.x + this.width/2, this.y + this.height/3, 8, 0, Math.PI, true);
                ctx.fill();

                // एलियन की चमकती आँखें
                ctx.fillStyle = "#fff";
                ctx.fillRect(this.x + 12, this.y + 10, 4, 4);
                ctx.fillRect(this.x + 24, this.y + 10, 4, 4);
            }
        }

        // बुलेट क्लास
        class Bullet {
            constructor(x, y) {
                this.x = x;
                this.y = y;
                this.width = 4;
                this.height = 15;
                this.speed = 8;
            }
            update() {
                this.y -= this.speed;
            }
            draw() {
                ctx.fillStyle = "#ffff00"; // लेजर बुलेट
                ctx.shadowBlur = 10;
                ctx.shadowColor = "#ffff00";
                ctx.fillRect(this.x, this.y, this.width, this.height);
                ctx.shadowBlur = 0; // शैडो रिसेट
            }
        }

        function fireBullet() {
            if (gameOver) return;
            bullets.push(new Bullet(player.x + player.width / 2 - 2, player.y));
        }

        // मोबाइल टच कंट्रोल्स
        document.getElementById("left-btn").addEventListener("touchstart", (e) => { e.preventDefault(); player.movingLeft = true; });
        document.getElementById("left-btn").addEventListener("touchend", () => player.movingLeft = false);
        
        document.getElementById("right-btn").addEventListener("touchstart", (e) => { e.preventDefault(); player.movingRight = true; });
        document.getElementById("right-btn").addEventListener("touchend", () => player.movingRight = false);
        
        document.getElementById("shoot-btn").addEventListener("touchstart", (e) => {
            e.preventDefault();
            fireBullet();
        });

        // कीबोर्ड सपोर्ट
        window.addEventListener("keydown", (e) => {
            if (e.code === "ArrowLeft") player.movingLeft = true;
            if (e.code === "ArrowRight") player.movingRight = true;
            if (e.code === "Space") fireBullet();
        });
        window.addEventListener("keyup", (e) => {
            if (e.code === "ArrowLeft") player.movingLeft = false;
            if (e.code === "ArrowRight") player.movingRight = false;
        });

        function checkCollision(rect1, rect2) {
            return rect1.x < rect2.x + rect2.width &&
                   rect1.x + rect1.width > rect2.x &&
                   rect1.y < rect2.y + rect2.height &&
                   rect1.y + rect1.height > rect2.y;
        }

        // गेम लूप
        function gameLoop(timestamp) {
            ctx.clearRect(0, 0, canvas.width, canvas.height);

            if (!gameOver) {
                // स्कोर के हिसाब से लेवल और कठिनाई बढ़ाना
                gameLevel = Math.floor(score / 50) + 1; 
                enemySpawnInterval = Math.max(500, 1200 - (gameLevel * 100)); // लेवल बढ़ने पर दुश्मन जल्दी आएंगे

                // दुश्मन पैदा करने का लॉजिक
                if (timestamp - lastSpawnTime > enemySpawnInterval) {
                    enemies.push(new Enemy());
                    lastSpawnTime = timestamp;
                }

                // प्लेयर मूवमेंट
                if (player.movingLeft && player.x > 0) player.x -= player.speed;
                if (player.movingRight && player.x < canvas.width - player.width) player.x += player.speed;

                player.draw();

                // बुलेट्स मैनेजमेंट
                bullets.forEach((bullet, bIndex) => {
                    bullet.update();
                    bullet.draw();
                    if (bullet.y < 0) bullets.splice(bIndex, 1);
                });

                // दुश्मन मैनेजमेंट
                enemies.forEach((enemy, eIndex) => {
                    enemy.update();
                    enemy.draw();

                    // प्लेयर से टक्कर
                    if (checkCollision(player, enemy)) {
                        gameOver = true;
                    }

                    // बुलेट से टक्कर
                    bullets.forEach((bullet, bIndex) => {
                        if (checkCollision(bullet, enemy)) {
                            enemies.splice(eIndex, 1);
                            bullets.splice(bIndex, 1);
                            score += 10;
                        }
                    });

                    if (enemy.y > canvas.height) {
                        enemies.splice(eIndex, 1);
                    }
                });

                // स्कोर और लेवल डिस्प्ले
                ctx.fillStyle = "#fff";
                ctx.font = "bold 16px Arial";
                ctx.fillText("Score: " + score, 15, 30);
                ctx.fillStyle = "#00ffcc";
                ctx.fillText("Level: " + gameLevel, canvas.width - 90, 30);

            } else {
                // गेम ओवर स्क्रीन
                ctx.fillStyle = "#ff3366";
                ctx.font = "bold 35px Arial";
                ctx.fillText("GAME OVER", canvas.width / 2 - 100, canvas.height / 2 - 20);
                ctx.fillStyle = "#fff";
                ctx.font = "20px Arial";
                ctx.fillText("Final Score: " + score, canvas.width / 2 - 60, canvas.height / 2 + 20);
                ctx.fillText("Level Reached: " + gameLevel, canvas.width / 2 - 75, canvas.height / 2 + 50);
                ctx.fillStyle = "#00ffcc";
                ctx.fillText("Tap Screen to Restart", canvas.width / 2 - 95, canvas.height / 2 + 100);
            }

            requestAnimationFrame(gameLoop);
        }

        // रीस्टार्ट
        canvas.addEventListener("click", () => {
            if (gameOver) {
                enemies = [];
                bullets = [];
                score = 0;
                gameLevel = 1;
                player.x = canvas.width / 2 - 20;
                gameOver = false;
            }
        });

        // शुरू करें
        requestAnimationFrame(gameLoop);
    </script>
</body>
</html>
