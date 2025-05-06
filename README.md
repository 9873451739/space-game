# space-game
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no" />
<title>Space Shooter Game</title>
<style>
  /* Reset and base */
  * {
    box-sizing: border-box;
    margin: 0; padding: 0;
    -webkit-user-select: none;
    -moz-user-select: none;
    -ms-user-select: none;
    user-select: none;
  }
  body, html {
    height: 100%;
    background: radial-gradient(ellipse at center, #0a0a23 0%, #000000 80%);
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    overflow: hidden;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: flex-start;
    color: #eee;
    -webkit-font-smoothing: antialiased;
    -moz-osx-font-smoothing: grayscale;
  }
  h1 {
    padding: 12px 0;
    font-weight: 700;
    font-size: 1.5rem;
    color: #4af;
    text-shadow: 0 0 12px #4af88c;
  }
  #game-container {
    position: relative;
    width: 350px;
    height: 600px;
    background: linear-gradient(to bottom, #000022, #000000);
    border: 3px solid #44aaff;
    border-radius: 12px;
    box-shadow: 0 0 20px #44aaffaa inset;
    overflow: hidden;
    touch-action: none;
    user-select: none;
  }
  canvas {
    display: block;
    background: transparent;
  }
  /* UI Over canvas */
  #ui {
    position: absolute;
    top: 8px;
    left: 12px;
    right: 12px;
    display: flex;
    justify-content: space-between;
    pointer-events: none;
    font-weight: 600;
    letter-spacing: 1px;
    font-size: 1.1rem;
    text-shadow: 0 0 6px #4af8ff;
  }
  #ui div {
    background: rgba(0,0,0,0.3);
    padding: 4px 10px;
    border-radius: 8px;
    pointer-events: none;
  }
  /* Controls container */
  #controls {
    position: absolute;
    bottom: 8px;
    left: 50%;
    transform: translateX(-50%);
    width: 90%;
    max-width: 350px;
    display: flex;
    justify-content: space-between;
    pointer-events: auto;
  }
  button.control-btn {
    width: 80px;
    height: 50px;
    background: linear-gradient(145deg, #0a0a32, #001844);
    border: 2px solid #47a6ff;
    border-radius: 12px;
    color: #4af8ff;
    font-weight: 700;
    font-size: 1.3rem;
    box-shadow: 0 0 10px #4af8ff inset;
    user-select: none;
    -webkit-tap-highlight-color: transparent;
    transition: background 0.3s ease, box-shadow 0.3s ease;
  }
  button.control-btn:active {
    background: linear-gradient(145deg, #001844, #0a0a32);
    box-shadow: 0 0 20px #4af8ff inset;
  }
  /* Game over overlay */
  #game-over {
    position: absolute;
    top: 0; left: 0; right: 0; bottom: 0;
    background: rgba(0,0,0,0.9);
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    font-size: 1.7rem;
    font-weight: 700;
    color: #4af8ff;
    text-shadow: 0 0 10px #1bd8ffbb;
    z-index: 10;
    opacity: 0;
    pointer-events: none;
    transition: opacity 0.3s ease;
    user-select: none;
  }
  #game-over.show {
    opacity: 1;
    pointer-events: auto;
  }
  #game-over button {
    margin-top: 20px;
    background: #4af8ff;
    border: none;
    color: #002244;
    font-weight: 700;
    font-size: 1.3rem;
    padding: 10px 30px;
    border-radius: 10px;
    cursor: pointer;
    box-shadow: 0 0 10px #4af8ff;
    transition: background 0.25s ease;
  }
  #game-over button:hover {
    background: #1bc8ff;
  }
  /* Hide scrollbars */
  ::-webkit-scrollbar { display: none; }
</style>
</head>
<body>
  <h1>Space Shooter</h1>
  <div id="game-container" role="main" aria-label="Space shooter game container">
    <canvas id="gameCanvas" width="350" height="600" aria-label="Game canvas"></canvas>
    <div id="ui" aria-live="polite">
      <div id="score">Score: 0</div>
      <div id="lives">Lives: 3</div>
    </div>
    <div id="controls" aria-label="Game controls">
      <button class="control-btn" id="btn-left" aria-label="Move left">◀</button>
      <button class="control-btn" id="btn-shoot" aria-label="Shoot">Fire</button>
      <button class="control-btn" id="btn-right" aria-label="Move right">▶</button>
    </div>
    <div id="game-over" role="alert" aria-live="assertive" aria-atomic="true">
      <div id="game-over-text">Game Over</div>
      <button id="restart-btn" aria-label="Restart game">Restart</button>
    </div>
  </div>

<script>
(() => {
  const canvas = document.getElementById('gameCanvas');
  const ctx = canvas.getContext('2d');

  const width = canvas.width;
  const height = canvas.height;

  // Player properties
  const playerWidth = 40;
  const playerHeight = 40;
  const playerSpeed = 6;
  const bulletSpeed = 10;
  const enemySpeedMin = 2;
  const enemySpeedMax = 4;
  const enemySpawnIntervalInitial = 1200; // milliseconds

  // Game state
  let playerX = width / 2 - playerWidth / 2;
  const playerY = height - playerHeight - 10;
  let leftPressed = false;
  let rightPressed = false;
  let shooting = false;
  let bullets = [];
  let enemies = [];
  let score = 0;
  let lives = 3;
  let enemySpawnInterval = enemySpawnIntervalInitial;
  let lastEnemySpawn = 0;
  let lastShootTime = 0;
  const shootCooldown = 300; // ms
  let gameOver = false;
  let animationId;

  // Controls elements
  const btnLeft = document.getElementById('btn-left');
  const btnRight = document.getElementById('btn-right');
  const btnShoot = document.getElementById('btn-shoot');
  const scoreEl = document.getElementById('score');
  const livesEl = document.getElementById('lives');
  const gameOverEl = document.getElementById('game-over');
  const restartBtn = document.getElementById('restart-btn');

  // Helper functions
  function drawPlayer(x, y) {
    // Draw spaceship as a triangle with exhaust glow
    ctx.save();
    ctx.translate(x + playerWidth / 2, y + playerHeight / 2);

    // Exhaust glow
    ctx.beginPath();
    ctx.moveTo(0, playerHeight/2);
    ctx.lineTo(-12, playerHeight/2 + 18);
    ctx.lineTo(12, playerHeight/2 + 18);
    ctx.closePath();
    const gradient = ctx.createRadialGradient(0, playerHeight/2 + 10, 2, 0, playerHeight/2 + 12, 20);
    gradient.addColorStop(0, 'rgba(74, 248, 255, 0.9)');
    gradient.addColorStop(1, 'rgba(0,0,0,0)');
    ctx.fillStyle = gradient;
    ctx.shadowColor = 'rgba(74, 248, 255, 0.8)';
    ctx.shadowBlur = 16;
    ctx.fill();

    // Spaceship body (white blue gradient)
    const bodyGradient = ctx.createLinearGradient(0, -playerHeight/2, 0, playerHeight/2);
    bodyGradient.addColorStop(0, '#72e9ff');
    bodyGradient.addColorStop(1, '#0077cc');
    ctx.fillStyle = bodyGradient;
    ctx.shadowBlur = 12;
    ctx.shadowColor = '#4af8ff';

    // Triangle shape
    ctx.beginPath();
    ctx.moveTo(0, -playerHeight/2);
    ctx.lineTo(playerWidth/2, playerHeight/2);
    ctx.lineTo(-playerWidth/2, playerHeight/2);
    ctx.closePath();
    ctx.fill();

    ctx.restore();
  }

  function drawBullet(b) {
    ctx.save();
    ctx.fillStyle = '#4af8ff';
    ctx.shadowColor = '#4af8ff';
    ctx.shadowBlur = 10;
    ctx.beginPath();
    ctx.rect(b.x, b.y, b.width, b.height);
    ctx.fill();
    ctx.restore();
  }

  function drawEnemy(e) {
    ctx.save();
    ctx.translate(e.x + e.size / 2, e.y + e.size / 2);
    // Draw enemy as red-orange glowing circle resembling asteroid
    const gradient = ctx.createRadialGradient(0, 0, e.size/4, 0, 0, e.size/2);
    gradient.addColorStop(0, '#ff4a4a');
    gradient.addColorStop(0.7, '#cc3300');
    gradient.addColorStop(1, 'rgba(0,0,0,0)');
    ctx.fillStyle = gradient;
    ctx.shadowColor = '#ff5511';
    ctx.shadowBlur = 12;
    ctx.beginPath();
    ctx.arc(0, 0, e.size / 2, 0, Math.PI * 2);
    ctx.fill();

    // Crater details - simple circles
    ctx.fillStyle = '#bb2200';
    for (let i = 0; i < 3; i++) {
      const angle = (i / 3) * Math.PI * 2 + e.angle;
      const radius = e.size / 6;
      const cx = Math.cos(angle) * e.size / 6;
      const cy = Math.sin(angle) * e.size / 6;
      ctx.beginPath();
      ctx.arc(cx, cy, radius / 2, 0, Math.PI * 2);
      ctx.fill();
    }
    ctx.restore();
  }

  // Collision detection between rectangles
  function rectsIntersect(r1, r2) {
    return !(
      r2.x > r1.x + r1.width ||
      r2.x + r2.width < r1.x ||
      r2.y > r1.y + r1.height ||
      r2.y + r2.height < r1.y
    );
  }

  // Spawn enemy function
  function spawnEnemy() {
    const size = 30 + Math.random() * 20;
    const x = Math.random() * (width - size);
    const speed = enemySpeedMin + Math.random() * (enemySpeedMax - enemySpeedMin);
    enemies.push({ x, y: -size, size, speed, angle: Math.random()*Math.PI*2 });
  }

  // Reset game state for restart
  function resetGame() {
    score = 0;
    lives = 3;
    bullets = [];
    enemies = [];
    enemySpawnInterval = enemySpawnIntervalInitial;
    lastEnemySpawn = 0;
    gameOver = false;
    updateUI();
    gameOverEl.classList.remove('show');
    playerX = width / 2 - playerWidth / 2;
    loop(performance.now());
  }

  // Update UI text
  function updateUI() {
    scoreEl.textContent = `Score: ${score}`;
    livesEl.textContent = `Lives: ${lives}`;
  }

  // Handle game over
  function handleGameOver() {
    gameOver = true;
    gameOverEl.classList.add('show');
    cancelAnimationFrame(animationId);
  }

  // Game loop main function
  function loop(timestamp) {
    if (gameOver) return;

    ctx.clearRect(0, 0, width, height);

    // Move player
    if (leftPressed) playerX -= playerSpeed;
    if (rightPressed) playerX += playerSpeed;
    // Clamp player within bounds
    if (playerX < 0) playerX = 0;
    if (playerX + playerWidth > width) playerX = width - playerWidth;

    // Draw player
    drawPlayer(playerX, playerY);

    // Shooting bullets automatically if shooting flag and cooldown elapsed
    if (shooting && timestamp - lastShootTime > shootCooldown) {
      bullets.push({
        x: playerX + playerWidth / 2 - 3,
        y: playerY,
        width: 6,
        height: 12,
        speed: bulletSpeed,
      });
      lastShootTime = timestamp;
    }

    // Update bullets
    for (let i = bullets.length - 1; i >= 0; i--) {
      const b = bullets[i];
      b.y -= b.speed;
      if (b.y + b.height < 0) {
        bullets.splice(i, 1);
        continue;
      }
      drawBullet(b);
    }

    // Spawn enemies
    if (timestamp - lastEnemySpawn > enemySpawnInterval) {
      spawnEnemy();
      lastEnemySpawn = timestamp;
      // Increase spawn speed gradually to increase difficulty
      if (enemySpawnInterval > 400) enemySpawnInterval -= 12;
    }

    // Update and draw enemies
    for (let i = enemies.length - 1; i >= 0; i--) {
      const e = enemies[i];
      e.y += e.speed;
      e.angle += 0.02;

      if (e.y > height) {
        enemies.splice(i, 1);
        lives -= 1;
        updateUI();
        if (lives <= 0) {
          handleGameOver();
          return;
        }
        continue;
      }
      drawEnemy(e);
    }

    // Collision detection bullets vs enemies
    for (let i = enemies.length - 1; i >= 0; i--) {
      const e = enemies[i];
      const enemyRect = { x: e.x, y: e.y, width: e.size, height: e.size };
      for (let j = bullets.length - 1; j >= 0; j--) {
        const b = bullets[j];
        const bulletRect = { x: b.x, y: b.y, width: b.width, height: b.height };
        if (rectsIntersect(enemyRect, bulletRect)) {
          // enemy hit
          enemies.splice(i, 1);
          bullets.splice(j, 1);
          score++;
          updateUI();
          break;
        }
      }
    }

    animationId = requestAnimationFrame(loop);
  }

  // Keyboard event handlers for desktop
  function keyDownHandler(e) {
    if (gameOver) return;
    if (e.code === 'ArrowLeft' || e.key === 'ArrowLeft') {
      leftPressed = true;
    } else if (e.code === 'ArrowRight' || e.key === 'ArrowRight') {
      rightPressed = true;
    } else if (e.code === 'Space' || e.key === ' ') {
      shooting = true;
    }
    e.preventDefault();
  }
  function keyUpHandler(e) {
    if (e.code === 'ArrowLeft' || e.key === 'ArrowLeft') {
      leftPressed = false;
    } else if (e.code === 'ArrowRight' || e.key === 'ArrowRight') {
      rightPressed = false;
    } else if (e.code === 'Space' || e.key === ' ') {
      shooting = false;
    }
    e.preventDefault();
  }

  // Touch and mouse handlers for on-screen buttons
  function setupTouchAndMouseControls() {
    btnLeft.addEventListener('touchstart', e => { leftPressed = true; e.preventDefault(); });
    btnLeft.addEventListener('touchend', e => { leftPressed = false; e.preventDefault(); });
    btnLeft.addEventListener('mousedown', e => { leftPressed = true; e.preventDefault(); });
    btnLeft.addEventListener('mouseup', e => { leftPressed = false; e.preventDefault(); });
    btnLeft.addEventListener('mouseleave', e => { leftPressed = false; e.preventDefault(); });

    btnRight.addEventListener('touchstart', e => { rightPressed = true; e.preventDefault(); });
    btnRight.addEventListener('touchend', e => { rightPressed = false; e.preventDefault(); });
    btnRight.addEventListener('mousedown', e => { rightPressed = true; e.preventDefault(); });
    btnRight.addEventListener('mouseup', e => { rightPressed = false; e.preventDefault(); });
    btnRight.addEventListener('mouseleave', e => { rightPressed = false; e.preventDefault(); });

    btnShoot.addEventListener('touchstart', e => { shooting = true; e.preventDefault(); });
    btnShoot.addEventListener('touchend', e => { shooting = false; e.preventDefault(); });
    btnShoot.addEventListener('mousedown', e => { shooting = true; e.preventDefault(); });
    btnShoot.addEventListener('mouseup', e => { shooting = false; e.preventDefault(); });
    btnShoot.addEventListener('mouseleave', e => { shooting = false; e.preventDefault(); });
  }

  // Restart button handling
  restartBtn.addEventListener('click', () => {
    resetGame();
  });

  // Initialize event listeners and start game loop
  function init() {
    window.addEventListener('keydown', keyDownHandler);
    window.addEventListener('keyup', keyUpHandler);
    setupTouchAndMouseControls();
    resetGame();
  }

  init();
})();
</script>
</body>
</html>
