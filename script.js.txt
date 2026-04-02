const canvas = document.getElementById("mapCanvas");
const ctx = canvas.getContext("2d");

// Fake player movement data (demo)
const players = [
  { x: 100, y: 100, type: "human" },
  { x: 200, y: 150, type: "bot" },
  { x: 300, y: 200, type: "human" },
  { x: 400, y: 300, type: "bot" },
];

// Draw players
players.forEach(p => {
  ctx.beginPath();
  ctx.arc(p.x, p.y, 8, 0, 2 * Math.PI);
  ctx.fillStyle = p.type === "human" ? "blue" : "red";
  ctx.fill();
});

// Heatmap style effect (simple)
players.forEach(p => {
  ctx.beginPath();
  ctx.arc(p.x, p.y, 20, 0, 2 * Math.PI);
  ctx.fillStyle = "rgba(255,0,0,0.2)";
  ctx.fill();
});