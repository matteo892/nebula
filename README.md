<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Jeu complet</title>
<style>
  body {
    margin: 0;
    padding: 0;
    text-align: center;
    font-family: Arial, sans-serif;
    background-color: #87CEEB;
  }

  canvas {
    background-color: #f0f0f0;
    display: block;
    margin: 20px auto;
    border: 3px solid #333;
  }

  h1 {
    color: #333;
  }

  #scoreboard {
    font-size: 20px;
    color: #333;
  }
</style>
</head>
<body>

<h1>Attrape les points et évite les obstacles !</h1>
<canvas id="gameCanvas" width="500" height="500"></canvas>
<div id="scoreboard">
  Score : <span id="score">0</span> | Vies : <span id="lives">3</span> | Niveau : <span id="level">1</span>
</div>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');

const playerSize = 30;
let playerX = canvas.width / 2 - playerSize / 2;
let playerY = canvas.height - playerSize - 10;

let score = 0;
let lives = 3;
let level = 1;

// Mouvement du joueur
let keys = {};
document.addEventListener('keydown', e => keys[e.key] = true);
document.addEventListener('keyup', e => keys[e.key] = false);

// Points à collecter
class Point {
    constructor() {
        this.reset();
    }
    reset() {
        this.size = 20;
        this.x = Math.random() * (canvas.width - this.size);
        this.y = Math.random() * (canvas.height - this.size - 50);
    }
    draw() {
        ctx.fillStyle = 'green';
        ctx.fillRect(this.x, this.y, this.size, this.size);
    }
}

let point = new Point();

// Obstacles
class Obstacle {
    constructor() {
        this.size = 30;
        this.x = Math.random() * (canvas.width - this.size);
        this.y = -this.size;
        this.speed = 2 + level;
    }
    update() {
        this.y += this.speed;
        if (this.y > canvas.height) {
            this.reset();
        }
    }
    reset() {
        this.x = Math.random() * (canvas.width - this.size);
        this.y = -this.size;
        this.speed = 2 + level;
    }
    draw() {
        ctx.fillStyle = 'brown';
        ctx.fillRect(this.x, this.y, this.size, this.size);
    }
}

let obstacles = [];
for (let i = 0; i < 3; i++) obstacles.push(new Obstacle());

// Fonction de mise à jour
function update() {
    const speed = 5;
    if (keys['ArrowUp']) playerY -= speed;
    if (keys['ArrowDown']) playerY += speed;
    if (keys['ArrowLeft']) playerX -= speed;
    if (keys['ArrowRight']) playerX += speed;

    // Limites du canvas
    if (playerX < 0) playerX = 0;
    if (playerX > canvas.width - playerSize) playerX = canvas.width - playerSize;
    if (playerY < 0) playerY = 0;
    if (playerY > canvas.height - playerSize) playerY = canvas.height - playerSize;

    // Collision avec le point
    if (playerX < point.x + point.size &&
        playerX + playerSize > point.x &&
        playerY < point.y + point.size &&
        playerY + playerSize > point.y) {
        score++;
        document.getElementById('score').textContent = score;
        point.reset();

        // Augmenter niveau tous les 5 points
        if (score % 5 === 0) {
            level++;
            document.getElementById('level').textContent = level;
            obstacles.push(new Obstacle());
        }
    }

    // Collision avec obstacles
    obstacles.forEach(ob => {
        ob.update();
        if (playerX < ob.x + ob.size &&
            playerX + playerSize > ob.x &&
            playerY < ob.y + ob.size &&
            playerY + playerSize > ob.y) {
            lives--;
            document.getElementById('lives').textContent = lives;
            ob.reset();
            if (lives <= 0) {
                alert('Game Over ! Ton score : ' + score);
                resetGame();
            }
        }
    });
}

// Fonction pour dessiner tout
function draw() {
    ctx.clearRect(0, 0, canvas.width, canvas.height);

    // Joueur
    ctx.fillStyle = 'red';
    ctx.fillRect(playerX, playerY, playerSize, playerSize);

    // Point
    point.draw();

    // Obstacles
    obstacles.forEach(ob => ob.draw());
}

// Réinitialiser le jeu
function resetGame() {
    score = 0;
    lives = 3;
    level = 1;
    obstacles = [];
    for (let i = 0; i < 3; i++) obstacles.push(new Obstacle());
    playerX = canvas.width / 2 - playerSize / 2;
    playerY = canvas.height - playerSize - 10;
    document.getElementById('score').textContent = score;
    document.getElementById('lives').textContent = lives;
    document.getElementById('level').textContent = level;
}

// Boucle de jeu
function gameLoop() {
    update();
    draw();
    requestAnimationFrame(gameLoop);
}

gameLoop();
</script>

</body>
</html>
