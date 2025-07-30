<!DOCTYPE html>
<html>
<head>
    <title>Geometry Dash Clone</title>
    <style>
        body {
            margin: 0;
            overflow: hidden;
            background-color: #111;
            font-family: Arial, sans-serif;
        }
        #gameContainer {
            position: relative;
            width: 100vw;
            height: 100vh;
        }
        #gameCanvas {
            background-color: #222;
            display: block;
        }
        #levelInfo {
            position: absolute;
            top: 10px;
            left: 10px;
            color: white;
            font-size: 20px;
        }
        #startScreen, #gameOverScreen, #levelCompleteScreen {
            position: absolute;
            top: 0;
            left: 0;
            width: 100%;
            height: 100%;
            display: flex;
            flex-direction: column;
            justify-content: center;
            align-items: center;
            background-color: rgba(0, 0, 0, 0.7);
            color: white;
            font-size: 24px;
        }
        button {
            margin-top: 20px;
            padding: 10px 20px;
            font-size: 18px;
            background-color: #4CAF50;
            color: white;
            border: none;
            border-radius: 5px;
            cursor: pointer;
        }
        button:hover {
            background-color: #45a049;
        }
        #gameOverScreen, #levelCompleteScreen {
            display: none;
        }
    </style>
</head>
<body>
    <div id="gameContainer">
        <canvas id="gameCanvas"></canvas>
        <div id="levelInfo">Nivel: 1 | Intentos: 0</div>
        
        <div id="startScreen">
            <h1>Geometry Dash Clone</h1>
            <p>Presiona ESPACIO para saltar y esquivar los obstáculos</p>
            <button id="startButton">Comenzar Juego</button>
        </div>
        
        <div id="gameOverScreen">
            <h1>¡Game Over!</h1>
            <p id="gameOverStats"></p>
            <button id="restartButton">Reintentar Nivel</button>
        </div>
        
        <div id="levelCompleteScreen">
            <h1>¡Nivel Completado!</h1>
            <p id="levelCompleteStats"></p>
            <button id="nextLevelButton">Siguiente Nivel</button>
        </div>
    </div>

    <script>
        // Configuración del juego
        const canvas = document.getElementById('gameCanvas');
        const ctx = canvas.getContext('2d');
        const levelInfo = document.getElementById('levelInfo');
        const startScreen = document.getElementById('startScreen');
        const gameOverScreen = document.getElementById('gameOverScreen');
        const levelCompleteScreen = document.getElementById('levelCompleteScreen');
        const startButton = document.getElementById('startButton');
        const restartButton = document.getElementById('restartButton');
        const nextLevelButton = document.getElementById('nextLevelButton');
        const gameOverStats = document.getElementById('gameOverStats');
        const levelCompleteStats = document.getElementById('levelCompleteStats');

        // Ajustar tamaño del canvas
        function resizeCanvas() {
            canvas.width = window.innerWidth;
            canvas.height = window.innerHeight;
        }
        resizeCanvas();
        window.addEventListener('resize', resizeCanvas);

        // Variables del juego
        let gameRunning = false;
        let currentLevel = 1;
        let attempts = 0;
        let score = 0;
        let gravity = 0.5;
        let gameSpeed = 5;
        let obstacles = [];
        let coins = [];
        let lastObstacleTime = 0;
        let lastCoinTime = 0;
        let animationId;
        let levelCompleted = false;

        // Jugador
        const player = {
            x: 100,
            y: canvas.height / 2,
            width: 30,
            height: 30,
            color: '#00FFFF',
            velocityY: 0,
            jumpForce: -12,
            isJumping: false,
            
            update: function() {
                // Aplicar gravedad
                this.velocityY += gravity;
                this.y += this.velocityY;
                
                // Limitar al suelo
                if (this.y + this.height > canvas.height) {
                    this.y = canvas.height - this.height;
                    this.velocityY = 0;
                    this.isJumping = false;
                }
                
                // Dibujar jugador
                ctx.fillStyle = this.color;
                ctx.fillRect(this.x, this.y, this.width, this.height);
                
                // Dibujar ojos (para hacerlo más parecido a Geometry Dash)
                ctx.fillStyle = 'black';
                ctx.fillRect(this.x + 10, this.y + 10, 5, 5);
                ctx.fillRect(this.x + 20, this.y + 10, 5, 5);
            },
            
            jump: function() {
                if (!this.isJumping) {
                    this.velocityY = this.jumpForce;
                    this.isJumping = true;
                }
            },
            
            reset: function() {
                this.y = canvas.height / 2;
                this.velocityY = 0;
                this.isJumping = false;
            }
        };

        // Obstáculos
        class Obstacle {
            constructor(x, width, height, gap) {
                this.x = x;
                this.width = width;
                this.height = height;
                this.gap = gap;
                this.color = '#FF5555';
                this.passed = false;
            }
            
            update() {
                this.x -= gameSpeed;
                
                // Dibujar obstáculo superior
                ctx.fillStyle = this.color;
                ctx.fillRect(this.x, 0, this.width, this.height);
                
                // Dibujar obstáculo inferior
                const bottomY = this.height + this.gap;
                ctx.fillRect(this.x, bottomY, this.width, canvas.height - bottomY);
            }
            
            collide(player) {
                // Verificar colisión con el obstáculo superior
                if (player.x < this.x + this.width &&
                    player.x + player.width > this.x &&
                    player.y < this.height) {
                    return true;
                }
                
                // Verificar colisión con el obstáculo inferior
                const bottomY = this.height + this.gap;
                if (player.x < this.x + this.width &&
                    player.x + player.width > this.x &&
                    player.y + player.height > bottomY) {
                    return true;
                }
                
                return false;
            }
        }

        // Monedas
        class Coin {
            constructor(x, y) {
                this.x = x;
                this.y = y;
                this.width = 15;
                this.height = 15;
                this.collected = false;
            }
            
            update() {
                this.x -= gameSpeed;
                
                if (!this.collected) {
                    ctx.fillStyle = '#FFD700';
                    ctx.beginPath();
                    ctx.arc(this.x + this.width/2, this.y + this.height/2, this.width/2, 0, Math.PI * 2);
                    ctx.fill();
                }
            }
            
            collide(player) {
                if (!this.collected &&
                    player.x < this.x + this.width &&
                    player.x + player.width > this.x &&
                    player.y < this.y + this.height &&
                    player.y + player.height > this.y) {
                    this.collected = true;
                    score += 10;
                    return true;
                }
                return false;
            }
        }

        // Niveles del juego
        const levels = [
            { // Nivel 1 - Fácil
                obstacleInterval: 2000,
                minGap: 200,
                maxGap: 250,
                minHeight: 100,
                maxHeight: 300,
                coinInterval: 1500,
                speed: 5,
                bgColor: '#222'
            },
            { // Nivel 2 - Un poco más difícil
                obstacleInterval: 1800,
                minGap: 180,
                maxGap: 220,
                minHeight: 120,
                maxHeight: 350,
                coinInterval: 1200,
                speed: 6,
                bgColor: '#1a1a2e'
            },
            { // Nivel 3 - Desafío moderado
                obstacleInterval: 1500,
                minGap: 160,
                maxGap: 200,
                minHeight: 150,
                maxHeight: 400,
                coinInterval: 1000,
                speed: 7,
                bgColor: '#16213e'
            },
            { // Nivel 4 - Difícil
                obstacleInterval: 1200,
                minGap: 140,
                maxGap: 180,
                minHeight: 180,
                maxHeight: 450,
                coinInterval: 800,
                speed: 8,
                bgColor: '#0f3460'
            },
            { // Nivel 5 - Muy difícil
                obstacleInterval: 1000,
                minGap: 120,
                maxGap: 160,
                minHeight: 200,
                maxHeight: 500,
                coinInterval: 600,
                speed: 9,
                bgColor: '#533483'
            },
            { // Nivel 6 - Extremo
                obstacleInterval: 800,
                minGap: 100,
                maxGap: 140,
                minHeight: 250,
                maxHeight: 550,
                coinInterval: 500,
                speed: 10,
                bgColor: '#2d132c'
            }
        ];

        // Inicializar nivel
        function initLevel(level) {
            obstacles = [];
            coins = [];
            player.reset();
            score = 0;
            attempts++;
            levelCompleted = false;
            
            const levelConfig = levels[level - 1];
            gameSpeed = levelConfig.speed;
            document.body.style.backgroundColor = levelConfig.bgColor;
            
            levelInfo.textContent = `Nivel: ${currentLevel} | Intentos: ${attempts} | Puntos: ${score}`;
        }

        // Generar obstáculos
        function generateObstacles(timestamp) {
            const levelConfig = levels[currentLevel - 1];
            
            if (timestamp - lastObstacleTime > levelConfig.obstacleInterval) {
                const height = Math.random() * 
                    (levelConfig.maxHeight - levelConfig.minHeight) + levelConfig.minHeight;
                const gap = Math.random() * 
                    (levelConfig.maxGap - levelConfig.minGap) + levelConfig.minGap;
                
                obstacles.push(new Obstacle(
                    canvas.width,
                    50,
                    height,
                    gap
                ));
                
                lastObstacleTime = timestamp;
            }
        }

        // Generar monedas
        function generateCoins(timestamp) {
            const levelConfig = levels[currentLevel - 1];
            
            if (timestamp - lastCoinTime > levelConfig.coinInterval) {
                const y = Math.random() * (canvas.height - 30) + 15;
                
                coins.push(new Coin(
                    canvas.width,
                    y
                ));
                
                lastCoinTime = timestamp;
            }
        }

        // Verificar colisiones
        function checkCollisions() {
            // Verificar colisión con obstáculos
            for (let obstacle of obstacles) {
                if (obstacle.collide(player)) {
                    gameOver();
                    return;
                }
                
                // Verificar si el jugador pasó el obstáculo
                if (!obstacle.passed && player.x > obstacle.x + obstacle.width) {
                    obstacle.passed = true;
                    score += 5;
                }
            }
            
            // Verificar colisión con monedas
            for (let coin of coins) {
                coin.collide(player);
            }
        }

        // Game over
        function gameOver() {
            gameRunning = false;
            cancelAnimationFrame(animationId);
            
            gameOverStats.textContent = `Nivel: ${currentLevel} | Puntos: ${score}`;
            gameOverScreen.style.display = 'flex';
        }

        // Completar nivel
        function completeLevel() {
            gameRunning = false;
            cancelAnimationFrame(animationId);
            levelCompleted = true;
            
            levelCompleteStats.textContent = `Puntos: ${score} | Intentos: ${attempts}`;
            levelCompleteScreen.style.display = 'flex';
        }

        // Verificar si se completó el nivel
        function checkLevelCompletion() {
            // Completar nivel después de pasar cierta cantidad de obstáculos
            const obstaclesToComplete = 10 + currentLevel * 2;
            
            let passedObstacles = 0;
            for (let obstacle of obstacles) {
                if (obstacle.passed) passedObstacles++;
            }
            
            if (passedObstacles >= obstaclesToComplete) {
                completeLevel();
            }
        }

        // Limpiar objetos fuera de pantalla
        function cleanUpObjects() {
            // Limpiar obstáculos fuera de pantalla
            obstacles = obstacles.filter(obstacle => obstacle.x + obstacle.width > 0);
            
            // Limpiar monedas fuera de pantalla o recolectadas
            coins = coins.filter(coin => coin.x + coin.width > 0 && !coin.collected);
        }

        // Bucle principal del juego
        function gameLoop(timestamp) {
            if (!gameRunning) return;
            
            ctx.clearRect(0, 0, canvas.width, canvas.height);
            
            // Generar obstáculos y monedas
            generateObstacles(timestamp);
            generateCoins(timestamp);
            
            // Actualizar y dibujar obstáculos
            for (let obstacle of obstacles) {
                obstacle.update();
            }
            
            // Actualizar y dibujar monedas
            for (let coin of coins) {
                coin.update();
            }
            
            // Actualizar y dibujar jugador
            player.update();
            
            // Verificar colisiones
            checkCollisions();
            
            // Verificar si se completó el nivel
            checkLevelCompletion();
            
            // Limpiar objetos fuera de pantalla
            cleanUpObjects();
            
            // Actualizar información del nivel
            levelInfo.textContent = `Nivel: ${currentLevel} | Intentos: ${attempts} | Puntos: ${score}`;
            
            // Continuar el bucle del juego
            animationId = requestAnimationFrame(gameLoop);
        }

        // Event listeners
        startButton.addEventListener('click', () => {
            startScreen.style.display = 'none';
            initLevel(currentLevel);
            gameRunning = true;
            gameLoop(0);
        });

        restartButton.addEventListener('click', () => {
            gameOverScreen.style.display = 'none';
            initLevel(currentLevel);
            gameRunning = true;
            gameLoop(0);
        });

        nextLevelButton.addEventListener('click', () => {
            levelCompleteScreen.style.display = 'none';
            currentLevel++;
            attempts = 0;
            initLevel(currentLevel);
            gameRunning = true;
            gameLoop(0);
        });

        // Control del jugador
        document.addEventListener('keydown', (e) => {
            if (e.code === 'Space' && gameRunning) {
                player.jump();
            }
        });

        // También permitir clic/touch para saltar
        canvas.addEventListener('click', () => {
            if (gameRunning) {
                player.jump();
            }
        });
    </script>
</body>
</html>
