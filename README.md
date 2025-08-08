<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0"/>
  <title>Canvas Racer</title>
  <style>
    body {
      margin: 0;
      padding: 0;
      display: flex;
      justify-content: center;
      align-items: center;
      height: 100vh;
      background-color: #222;
      font-family: Arial, sans-serif;
      overflow: hidden;
    }

    #game-container {
      position: relative;
      width: 100%;
      max-width: 400px;
      height: 600px;
      margin: 0 auto;
    }

    canvas {
      background-color: #333;
      display: block;
      border-radius: 8px;
      box-shadow: 0 0 20px rgba(0, 0, 0, 0.5);
    }

    #ui {
      position: absolute;
      top: 0;
      left: 0;
      width: 100%;
      height: 100%;
      pointer-events: none;
      color: white;
    }

    #start-screen, #game-over {
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
    }

    #game-over {
      display: none;
    }

    h1 {
      font-size: 2.5rem;
      margin-bottom: 1rem;
      text-align: center;
    }

    button {
      padding: 10px 25px;
      font-size: 1.2rem;
      background-color: #e74c3c;
      color: white;
      border: none;
      border-radius: 5px;
      cursor: pointer;
      pointer-events: auto;
      transition: background-color 0.2s;
    }

    button:hover {
      background-color: #c0392b;
    }

    #score {
      position: absolute;
      top: 20px;
      right: 20px;
      font-size: 1.5rem;
      color: white;
      background-color: rgba(0, 0, 0, 0.5);
      padding: 5px 15px;
      border-radius: 20px;
    }

    #high-score {
      margin-top: 10px;
      font-size: 1.2rem;
    }

    .instructions {
      margin: 20px 0;
      text-align: center;
      line-height: 1.6;
    }
  </style>
</head>
<body>
  <div id="game-container">
    <canvas id="game" width="400" height="600"></canvas>
    <div id="ui">
      <div id="score">0</div>
      <div id="start-screen">
        <h1>CANVAS RACER</h1>
        <div class="instructions">
          Use LEFT and RIGHT arrow keys to steer<br>
          Avoid the obstacles and stay on the road!
        </div>
        <button id="start-btn">START RACE</button>
      </div>
      <div id="game-over">
        <h1>GAME OVER</h1>
        <div id="final-score">Score: 0</div>
        <div id="high-score">High Score: 0</div>
        <button id="restart-btn">PLAY AGAIN</button>
      </div>
    </div>
  </div>

  <script>
    const canvas = document.getElementById('game');
    const ctx = canvas.getContext('2d');
    const startScreen = document.getElementById('start-screen');
    const gameOverScreen = document.getElementById('game-over');
    const startBtn = document.getElementById('start-btn');
    const restartBtn = document.getElementById('restart-btn');
    const scoreDisplay = document.getElementById('score');
    const finalScoreDisplay = document.getElementById('final-score');
    const highScoreDisplay = document.getElementById('high-score');

    let gameRunning = false;
    let score = 0;
    let highScore = localStorage.getItem('highScore') || 0;
    let animationId;
    let speed = 2;
    let difficulty = 1;

    const road = {
      width: 300,
      laneCount: 3,
      laneWidth: 100,
      leftEdge: 50,
      markings: []
    };

    for (let i = 0; i < 20; i++) {
      road.markings.push({ y: i * 100 - 50, width: 20, height: 40 });
    }

    const player = {
      x: canvas.width / 2,
      y: canvas.height - 100,
      width: 40,
      height: 70,
      color: '#e74c3c',
      lane: 1,
      moving: false,
      moveDirection: 0
    };

    let obstacles = [];
    const obstacleColors = ['#3498db', '#2ecc71', '#f39c12', '#9b59b6'];

    startBtn.addEventListener('click', startGame);
    restartBtn.addEventListener('click', startGame);

    document.addEventListener('keydown', (e) => {
      if (!gameRunning) return;
      if (e.key === 'ArrowLeft' && !player.moving && player.lane > 0) {
        player.moveDirection = -1;
        player.moving = true;
      } else if (e.key === 'ArrowRight' && !player.moving && player.lane < road.laneCount - 1) {
        player.moveDirection = 1;
        player.moving = true;
      }
    });

    function startGame() {
      gameRunning = true;
      score = 0;
      difficulty = 1;
      speed = 2;
      obstacles = [];
      player.lane = 1;
      player.x = road.leftEdge + road.laneWidth * player.lane + road.laneWidth / 2;
      player.moving = false;

      startScreen.style.display = 'none';
      gameOverScreen.style.display = 'none';
      scoreDisplay.textContent = score;

      gameLoop();
    }

    function gameLoop() {
      ctx.clearRect(0, 0, canvas.width, canvas.height);
      update();
      drawRoad();
      drawCar(player);
      obstacles.forEach(obstacle => drawObstacle(obstacle));
      if (checkCollision()) {
        gameOver();
        return;
      }
      animationId = requestAnimationFrame(gameLoop);
    }

    function update() {
      score += 1 * difficulty;
      scoreDisplay.textContent = Math.floor(score / 10);

      if (score % 1000 === 0) {
        difficulty = Math.min(5, difficulty + 0.1);
      }

      if (player.moving === true) {
        if (player.moveDirection === -1 && player.lane > 0) {
          player.lane -= 1;
        } else if (player.moveDirection === 1 && player.lane < road.laneCount - 1) {
          player.lane += 1;
        }
        player.moving = false;
      }

      const targetX = road.leftEdge + player.lane * road.laneWidth + road.laneWidth / 2;
      if (Math.abs(player.x - targetX) < 5) {
        player.x = targetX;
      } else {
        player.x += 5 * Math.sign(targetX - player.x);
      }

      road.markings.forEach(mark => {
        mark.y += speed * difficulty;
        if (mark.y > canvas.height) {
          mark.y = -mark.height;
        }
      });

      obstacles.forEach(obstacle => {
        obstacle.y += speed * difficulty;
      });

      obstacles = obstacles.filter(obstacle => obstacle.y < canvas.height + obstacle.height);

      if (Math.random() < 0.02 * difficulty) {
        spawnObstacle();
      }
    }

    function spawnObstacle() {
      const lane = Math.floor(Math.random() * road.laneCount);
      const width = road.laneWidth * (0.8 + Math.random() * 0.4);
      const height = 60 + Math.random() * 40;
      const x = road.leftEdge + lane * road.laneWidth + road.laneWidth / 2 - width / 2;

      obstacles.push({
        x: x,
        y: -height,
        width: width,
        height: height,
        color: obstacleColors[Math.floor(Math.random() * obstacleColors.length)],
        lane: lane
      });
    }

    function drawRoad() {
      ctx.fillStyle = '#555';
      ctx.fillRect(road.leftEdge, 0, road.width, canvas.height);

      ctx.fillStyle = '#fff';
      road.markings.forEach(mark => {
        ctx.fillRect(canvas.width / 2 - mark.width / 2, mark.y, mark.width, mark.height);
      });

      ctx.strokeStyle = '#fff';
      ctx.lineWidth = 4;
      ctx.beginPath();
      ctx.moveTo(road.leftEdge, 0);
      ctx.lineTo(road.leftEdge, canvas.height);
      ctx.stroke();

      ctx.beginPath();
      ctx.moveTo(road.leftEdge + road.width, 0);
      ctx.lineTo(road.leftEdge + road.width, canvas.height);
      ctx.stroke();
    }

    function drawCar(car) {
      ctx.fillStyle = car.color;
      ctx.fillRect(car.x - car.width / 2, car.y - car.height / 2, car.width, car.height);
      ctx.fillStyle = '#222';
      ctx.fillRect(car.x - car.width / 2 + 5, car.y - car.height / 2 + 5, car.width - 10, car.height * 0.3);
      ctx.fillStyle = '#f1c40f';
      ctx.beginPath();
      ctx.arc(car.x, car.y + car.height / 2 - 5, 5, 0, Math.PI * 2);
      ctx.fill();
      ctx.fillStyle = '#e74c3c';
      ctx.beginPath();
      ctx.arc(car.x, car.y - car.height / 2 + 5, 5, 0, Math.PI * 2);
      ctx.fill();
    }

    function drawObstacle(obstacle) {
      ctx.fillStyle = obstacle.color;
      ctx.fillRect(obstacle.x, obstacle.y, obstacle.width, obstacle.height);
      ctx.fillStyle = '#222';
      ctx.fillRect(obstacle.x + 10, obstacle.y + 10, obstacle.width - 20, 5);
      ctx.fillRect(obstacle.x + 10, obstacle.y + obstacle.height - 15, obstacle.width - 20, 5);
    }

    function checkCollision() {
      for (const obstacle of obstacles) {
        if (
          player.x - player.width / 2 < obstacle.x + obstacle.width &&
          player.x + player.width / 2 > obstacle.x &&
          player.y - player.height / 2 < obstacle.y + obstacle.height &&
          player.y + player.height / 2 > obstacle.y
        ) {
          return true;
        }
      }
      if (
        player.x - player.width / 2 < road.leftEdge ||
        player.x + player.width / 2 > road.leftEdge + road.width
      ) {
        return true;
      }
      return false;
    }

    function gameOver() {
      gameRunning = false;
      cancelAnimationFrame(animationId);
      const currentScore = Math.floor(score / 10);
      if (currentScore > highScore) {
        highScore = currentScore;
        localStorage.setItem('highScore', highScore);
      }
      finalScoreDisplay.textContent = `Score: ${currentScore}`;
      highScoreDisplay.textContent = `High Score: ${highScore}`;
      gameOverScreen.style.display = 'flex';
    }
  </script>
</body>
</html>
