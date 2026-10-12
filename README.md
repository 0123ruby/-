<!DOCTYPE html>
<html>

<head>
  <base target="_top">
  <meta charset="UTF-8">

  <!-- p5.js -->
  <script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.9.4/p5.min.js"></script>

  <!-- ml5.js：使用 PoseNet -->
  <script src="https://unpkg.com/ml5@0.12.2/dist/ml5.min.js"></script>

  <style>
    html,
    body {
      margin: 0;
      padding: 0;
      overflow: hidden;
      background: #111;
      font-family:
        "Noto Sans TC",
        "Microsoft JhengHei",
        sans-serif;
    }

    canvas {
      display: block;
    }

    #info {
      position: fixed;
      top: 25px;
      left: 28px;
      z-index: 20;

      color: white;

      background:
        rgba(20, 25, 28, 0.30);

      padding: 16px 20px;

      border-radius: 18px;

      backdrop-filter:
        blur(10px);

      pointer-events: none;
    }

    #title {
      font-size: 13px;
      letter-spacing: 4px;
      opacity: 0.65;
    }

    #season {
      margin-top: 5px;
      font-size: 30px;
    }

    #instruction {
      margin-top: 6px;
      font-size: 15px;
    }

    #gesture {
      margin-top: 7px;

      font-size: 14px;

      color: #ffe5a5;
    }

    #progress {
      margin-top: 8px;

      font-size: 13px;

      opacity: 0.75;
    }

    #debugPanel {
      position: fixed;

      left: 28px;
      bottom: 25px;

      z-index: 20;

      color: white;

      background:
        rgba(20, 25, 28, 0.30);

      padding: 13px 17px;

      border-radius: 15px;

      font-size: 13px;
      line-height: 1.7;

      backdrop-filter:
        blur(10px);

      pointer-events: none;
    }

    #hint {
      position: fixed;

      right: 28px;
      bottom: 25px;

      z-index: 20;

      color: white;

      background:
        rgba(20, 25, 28, 0.25);

      padding: 11px 15px;

      border-radius: 14px;

      font-size: 13px;

      opacity: 0.75;

      backdrop-filter:
        blur(10px);

      pointer-events: none;
    }
  </style>

</head>


<body>

  <div id="info">

    <div id="title">
      搖曳時光
    </div>

    <div id="season">
      春・播種
    </div>

    <div id="instruction">
      向前傾
    </div>

    <div id="gesture">
      雙手做撒種動作
    </div>

    <div id="progress">
      進度：0 / 6
    </div>

  </div>


  <div id="debugPanel">

    <div>
      鏡頭：
      <span id="cameraState">
        啟動中
      </span>
    </div>

    <div>
      人體：
      <span id="poseState">
        等待辨識
      </span>
    </div>

    <div>
      X：
      <span id="debugX">
        0.00
      </span>
    </div>

    <div>
      Y：
      <span id="debugY">
        0.00
      </span>
    </div>

    <div>
      動作：
      <span id="debugGesture">
        無
      </span>
    </div>

  </div>


  <div id="hint">
    ↑ 前傾　↓ 後仰　← → 左右　｜　R 重玩　C 鏡頭顯示
  </div>


<script>

/* ========================================================
   搖曳時光
   Camera + PoseNet + 四季互動
======================================================== */


/* ========================================================
   坐墊模擬
======================================================== */

let tiltX = 0;
let tiltY = 0;

let targetTiltX = 0;
let targetTiltY = 0;


/* ========================================================
   遊戲
======================================================== */

let currentGame = 0;


/*
0 = 春
1 = 夏
2 = 秋
3 = 冬
*/


/* ========================================================
   CAMERA
======================================================== */

let video;

let poseNet;

let pose = null;

let skeleton = [];

let cameraReady = false;

let showCamera = true;


/* ========================================================
   人體關節
======================================================== */

let leftWrist = null;
let rightWrist = null;

let leftElbow = null;
let rightElbow = null;

let leftShoulder = null;
let rightShoulder = null;


/* ========================================================
   動作追蹤
======================================================== */

let previousLeftWristX = 0;
let previousRightWristX = 0;

let previousLeftWristY = 0;
let previousRightWristY = 0;


let gestureName = "無";

let gestureCooldown = 0;


/* ========================================================
   春
======================================================== */

let seeds = [];

let plantedSeeds = 0;

let maxSeeds = 6;


/* ========================================================
   夏
======================================================== */

let fireflies = [];

let caughtFireflies = 0;

let maxFireflies = 6;

let catcherX;


/* ========================================================
   秋
======================================================== */

let rice = [];

let harvestProgress = 0;


/* ========================================================
   冬
======================================================== */

let lanterns = [];

let litLanterns = 0;

let winterHold = 0;


/* ========================================================
   星空、雪
======================================================== */

let stars = [];

let snow = [];


/* ========================================================
   setup
======================================================== */

function setup() {

  createCanvas(
    windowWidth,
    windowHeight
  );

  pixelDensity(1);


  catcherX =
    width / 2;


  createSeeds();

  createFireflies();

  createRice();

  createLanterns();

  createStars();

  createSnow();


  /*
  ============================
  開啟攝影機
  ============================
  */

  video =
    createCapture(
      VIDEO,
      cameraStarted
    );


  video.size(
    320,
    240
  );


  video.hide();


  /*
  ============================
  PoseNet
  ============================
  */

  poseNet =
    ml5.poseNet(
      video,
      modelReady
    );


  poseNet.on(
    "pose",
    gotPoses
  );

}


/* ========================================================
   Camera Ready
======================================================== */

function cameraStarted() {

  cameraReady = true;


  document
    .getElementById(
      "cameraState"
    )
    .innerText =
      "已開啟";

}


/* ========================================================
   PoseNet Ready
======================================================== */

function modelReady() {

  console.log(
    "PoseNet Ready"
  );

}


/* ========================================================
   接收人體資料
======================================================== */

function gotPoses(results) {

  if (
    results.length > 0
  ) {

    pose =
      results[0].pose;

    skeleton =
      results[0].skeleton;


    document
      .getElementById(
        "poseState"
      )
      .innerText =
        "偵測到";


    updateBodyPoints();

  }

  else {

    pose = null;

    skeleton = [];


    document
      .getElementById(
        "poseState"
      )
      .innerText =
        "找不到人體";

  }

}


/* ========================================================
   人體關節
======================================================== */

function updateBodyPoints() {

  if (!pose) {
    return;
  }


  leftWrist =
    pose.leftWrist;

  rightWrist =
    pose.rightWrist;


  leftElbow =
    pose.leftElbow;

  rightElbow =
    pose.rightElbow;


  leftShoulder =
    pose.leftShoulder;

  rightShoulder =
    pose.rightShoulder;

}


/* ========================================================
   draw
======================================================== */

function draw() {

  readKeyboard();

  updateTilt();


  gestureCooldown -=
    deltaTime;


  /*
  ============================
  判斷人體動作
  ============================
  */

  detectGesture();


  /*
  ============================
  四季
  ============================
  */

  if (
    currentGame === 0
  ) {

    drawSpringGame();

  }

  else if (
    currentGame === 1
  ) {

    drawSummerGame();

  }

  else if (
    currentGame === 2
  ) {

    drawAutumnGame();

  }

  else {

    drawWinterGame();

  }


  /*
  ============================
  CAMERA DEBUG
  ============================
  */

  if (
    showCamera
  ) {

    drawCamera();

  }


  updateUI();

}


/* ========================================================
   鍵盤模擬坐墊
======================================================== */

function readKeyboard() {

  targetTiltX = 0;

  targetTiltY = 0;


  if (
    keyIsDown(
      LEFT_ARROW
    )
  ) {

    targetTiltX = -1;

  }


  if (
    keyIsDown(
      RIGHT_ARROW
    )
  ) {

    targetTiltX = 1;

  }


  if (
    keyIsDown(
      UP_ARROW
    )
  ) {

    targetTiltY = -1;

  }


  if (
    keyIsDown(
      DOWN_ARROW
    )
  ) {

    targetTiltY = 1;

  }

}


/* ========================================================
   平滑重心
======================================================== */

function updateTilt() {

  tiltX =
    lerp(
      tiltX,
      targetTiltX,
      0.08
    );


  tiltY =
    lerp(
      tiltY,
      targetTiltY,
      0.08
    );

}


/* ========================================================
   人體動作辨識
======================================================== */

function detectGesture() {

  gestureName =
    "無";


  if (
    !pose ||
    !leftWrist ||
    !rightWrist
  ) {

    return;

  }


  /*
  ========================================================
  春
  雙手往下撒種
  ========================================================
  */

  if (
    currentGame === 0
  ) {

    let leftDown =
      leftWrist.y
      -
      previousLeftWristY;


    let rightDown =
      rightWrist.y
      -
      previousRightWristY;


    /*
    在畫面裡 y 越大 = 手越往下
    */

    if (
      leftDown > 8 &&
      rightDown > 8
    ) {

      gestureName =
        "撒種";


      if (
        gestureCooldown <= 0
      ) {

        springGesture();

        gestureCooldown =
          700;

      }

    }

  }


  /*
  ========================================================
  夏
  單手左右揮
  ========================================================
  */

  else if (
    currentGame === 1
  ) {

    let leftMove =
      abs(
        leftWrist.x
        -
        previousLeftWristX
      );


    let rightMove =
      abs(
        rightWrist.x
        -
        previousRightWristX
      );


    if (
      leftMove > 10 ||
      rightMove > 10
    ) {

      gestureName =
        "揮手";

    }

  }


  /*
  ========================================================
  秋
  雙手向下收割
  ========================================================
  */

  else if (
    currentGame === 2
  ) {

    let leftDown =
      leftWrist.y
      -
      previousLeftWristY;


    let rightDown =
      rightWrist.y
      -
      previousRightWristY;


    if (
      leftDown > 6 &&
      rightDown > 6
    ) {

      gestureName =
        "收割";


      autumnGesture();

    }

  }


  /*
  ========================================================
  冬
  雙手舉高
  ========================================================
  */

  else {

    if (
      leftShoulder &&
      rightShoulder
    ) {

      let leftRaised =
        leftWrist.y
        <
        leftShoulder.y;


      let rightRaised =
        rightWrist.y
        <
        rightShoulder.y;


      if (
        leftRaised &&
        rightRaised
      ) {

        gestureName =
          "雙手舉高";

      }

    }

  }


  /*
  保存上一幀
  */

  previousLeftWristX =
    leftWrist.x;


  previousRightWristX =
    rightWrist.x;


  previousLeftWristY =
    leftWrist.y;


  previousRightWristY =
    rightWrist.y;

}


/* ========================================================
   春天
======================================================== */

function createSeeds() {

  seeds = [];


  for (
    let i = 0;
    i < maxSeeds;
    i++
  ) {

    seeds.push({

      x:
        width * 0.20
        +
        i
        *
        width * 0.11,

      y:
        height * 0.74,

      planted:
        false,

      growth:
        0

    });

  }

}


/* ========================================================
   春季手勢
======================================================== */

function springGesture() {

  /*
  必須同時前傾
  */

  if (
    tiltY < -0.30 &&
    plantedSeeds < maxSeeds
  ) {

    seeds[
      plantedSeeds
    ].planted =
      true;


    plantedSeeds++;

  }

}


/* ========================================================
   春天畫面
======================================================== */

function drawSpringGame() {

  drawSpringBackground();


  noStroke();

  fill(
    122,
    156,
    92
  );


  rect(
    0,
    height * 0.62,
    width,
    height * 0.38
  );


  /*
  土地
  */

  fill(
    111,
    81,
    52
  );


  rect(
    width * 0.10,
    height * 0.68,
    width * 0.80,
    height * 0.22,
    30
  );


  /*
  種子
  */

  for (
    let seed of seeds
  ) {

    if (
      seed.planted
    ) {

      seed.growth =
        lerp(
          seed.growth,
          1,
          0.02
        );


      drawPlant(
        seed.x,
        seed.y,
        seed.growth
      );

    }

    else {

      noStroke();

      fill(
        85,
        60,
        38
      );


      ellipse(
        seed.x,
        seed.y,
        12,
        8
      );

    }

  }


  /*
  完成
  */

  if (
    plantedSeeds >= maxSeeds
  ) {

    drawCompleteText(
      "播種完成"
    );


    if (
      frameCount % 180 === 0
    ) {

      currentGame = 1;

    }

  }

}


/* ========================================================
   春背景
======================================================== */

function drawSpringBackground() {

  for (
    let y = 0;
    y < height;
    y += 3
  ) {

    let c =
      lerpColor(

        color(
          174,
          214,
          213
        ),

        color(
          239,
          222,
          175
        ),

        y / height

      );


    stroke(c);


    line(
      0,
      y,
      width,
      y
    );

  }

}


/* ========================================================
   植物
======================================================== */

function drawPlant(
  x,
  y,
  amount
) {

  let h =
    100
    *
    amount;


  stroke(
    65,
    125,
    70
  );

  strokeWeight(4);


  line(
    x,
    y,
    x,
    y - h
  );


  noStroke();


  fill(
    89,
    150,
    82
  );


  ellipse(
    x - 11,
    y - h * 0.55,
    28 * amount,
    15 * amount
  );


  ellipse(
    x + 11,
    y - h * 0.72,
    28 * amount,
    15 * amount
  );

}


/* ========================================================
   夏天
======================================================== */

function createFireflies() {

  fireflies = [];


  for (
    let i = 0;
    i < maxFireflies;
    i++
  ) {

    fireflies.push({

      x:
        random(
          width * 0.15,
          width * 0.85
        ),

      /*
      故意降低高度
      */

      y:
        random(
          height * 0.52,
          height * 0.64
        ),

      phase:
        random(
          TWO_PI
        ),

      caught:
        false

    });

  }

}


/* ========================================================
   夏季遊戲
======================================================== */

function drawSummerGame() {

  background(
    24,
    56,
    58
  );


  /*
  月亮
  */

  noStroke();

  fill(
    245,
    228,
    173
  );


  circle(
    width * 0.80,
    height * 0.18,
    100
  );


  /*
  草地
  */

  fill(
    39,
    91,
    62
  );


  rect(
    0,
    height * 0.69,
    width,
    height * 0.31
  );


  /*
  左右重心控制網子
  */

  catcherX +=
    tiltX * 7;


  catcherX =
    constrain(
      catcherX,
      80,
      width - 80
    );


  drawFireflies();

  drawCatcher();


  /*
  完成
  */

  if (
    caughtFireflies >= maxFireflies
  ) {

    drawCompleteText(
      "夏夜被點亮了"
    );


    if (
      frameCount % 180 === 0
    ) {

      currentGame = 2;

    }

  }

}


/* ========================================================
   螢火蟲
======================================================== */

function drawFireflies() {

  for (
    let f of fireflies
  ) {

    if (
      f.caught
    ) {

      continue;

    }


    f.x +=
      sin(
        frameCount * 0.018
        +
        f.phase
      )
      *
      0.35;


    /*
    上下幅度很小
    */

    f.y +=
      cos(
        frameCount * 0.012
        +
        f.phase
      )
      *
      0.08;


    f.y =
      constrain(
        f.y,
        height * 0.50,
        height * 0.65
      );


    let glow =
      map(
        sin(
          frameCount * 0.04
          +
          f.phase
        ),
        -1,
        1,
        90,
        240
      );


    noStroke();


    fill(
      255,
      230,
      100,
      glow * 0.25
    );


    circle(
      f.x,
      f.y,
      35
    );


    fill(
      255,
      243,
      130,
      glow
    );


    circle(
      f.x,
      f.y,
      8
    );


    /*
    必須：
    1. 網子靠近
    2. 鏡頭偵測到揮手
    */

    let d =
      dist(

        catcherX,

        height * 0.67,

        f.x,

        f.y

      );


    if (
      d < 95 &&
      gestureName === "揮手"
    ) {

      f.caught =
        true;


      caughtFireflies++;

    }

  }

}


/* ========================================================
   捕蟲網
======================================================== */

function drawCatcher() {

  push();


  translate(
    catcherX,
    height * 0.67
  );


  noFill();


  stroke(
    235,
    225,
    195
  );


  strokeWeight(5);


  ellipse(
    0,
    0,
    110,
    80
  );


  /*
  網格
  */

  strokeWeight(1);


  line(
    -40,
    -25,
    40,
    25
  );


  line(
    -40,
    25,
    40,
    -25
  );


  line(
    0,
    -40,
    0,
    40
  );


  /*
  手柄
  */

  strokeWeight(6);


  line(
    30,
    30,
    80,
    105
  );


  pop();

}


/* ========================================================
   秋天
======================================================== */

function createRice() {

  rice = [];


  for (
    let i = 0;
    i < 80;
    i++
  ) {

    rice.push({

      x:
        random(width),

      y:
        random(
          height * 0.60,
          height * 0.90
        ),

      h:
        random(
          55,
          110
        )

    });

  }

}


/* ========================================================
   秋季手勢
======================================================== */

function autumnGesture() {

  /*
  要求身體保持中心
  */

  if (
    abs(tiltX) < 0.15 &&
    abs(tiltY) < 0.15
  ) {

    harvestProgress +=
      1.2;

  }

}


/* ========================================================
   秋季遊戲
======================================================== */

function drawAutumnGame() {

  for (
    let y = 0;
    y < height;
    y += 3
  ) {

    stroke(

      lerpColor(

        color(
          189,
          148,
          100
        ),

        color(
          239,
          188,
          106
        ),

        y / height

      )

    );


    line(
      0,
      y,
      width,
      y
    );

  }


  noStroke();


  fill(
    142,
    112,
    54
  );


  rect(
    0,
    height * 0.57,
    width,
    height * 0.43
  );


  harvestProgress =
    constrain(
      harvestProgress,
      0,
      100
    );


  let remaining =
    floor(

      map(
        harvestProgress,
        0,
        100,
        rice.length,
        0
      )

    );


  for (
    let i = 0;
    i < remaining;
    i++
  ) {

    let r =
      rice[i];


    stroke(
      220,
      181,
      65
    );


    strokeWeight(3);


    line(

      r.x,

      r.y,

      r.x,

      r.y - r.h

    );


    noStroke();


    fill(
      235,
      194,
      72
    );


    for (
      let j = 0;
      j < 5;
      j++
    ) {

      ellipse(

        r.x + 5,

        r.y
        -
        r.h
        +
        j * 8,

        9,

        5

      );

    }

  }


  /*
  完成
  */

  if (
    harvestProgress >= 100
  ) {

    drawCompleteText(
      "豐收了"
    );


    if (
      frameCount % 180 === 0
    ) {

      currentGame = 3;

    }

  }

}


/* ========================================================
   冬天
======================================================== */

function createLanterns() {

  lanterns = [];


  let positions = [

    0.25,
    0.375,
    0.50,
    0.625,
    0.75

  ];


  for (
    let position of positions
  ) {

    lanterns.push({

      xRatio:
        position,

      lit:
        false

    });

  }

}


/* ========================================================
   星空
======================================================== */

function createStars() {

  stars = [];


  for (
    let i = 0;
    i < 120;
    i++
  ) {

    stars.push({

      x:
        random(width),

      y:
        random(
          height * 0.55
        ),

      size:
        random(
          1,
          3
        )

    });

  }

}


/* ========================================================
   雪
======================================================== */

function createSnow() {

  snow = [];


  for (
    let i = 0;
    i < 80;
    i++
  ) {

    snow.push({

      x:
        random(width),

      y:
        random(
          -height,
          height
        ),

      size:
        random(
          1,
          4
        ),

      speed:
        random(
          0.3,
          1
        )

    });

  }

}


/* ========================================================
   冬季
======================================================== */

function drawWinterGame() {

  background(
    24,
    35,
    57
  );


  drawStars();

  drawSnow();

  drawHouse();

  drawLanterns();


  /*
  要求：
  後仰 +
  雙手舉高
  */

  if (
    tiltY > 0.30 &&
    gestureName === "雙手舉高" &&
    litLanterns < lanterns.length
  ) {

    winterHold +=
      deltaTime;

  }

  else {

    winterHold =
      max(
        0,
        winterHold
        -
        deltaTime
      );

  }


  /*
  維持 1 秒
  */

  if (
    winterHold >= 1000
  ) {

    lanterns[
      litLanterns
    ].lit =
      true;


    litLanterns++;


    winterHold =
      0;

  }


  if (
    litLanterns >= lanterns.length
  ) {

    drawCompleteText(
      "一整年的時光，被你點亮了"
    );

  }

}


/* ========================================================
   星星
======================================================== */

function drawStars() {

  noStroke();


  for (
    let s of stars
  ) {

    fill(
      255,
      random(
        120,
        220
      )
    );


    circle(
      s.x,
      s.y,
      s.size
    );

  }

}


/* ========================================================
   雪
======================================================== */

function drawSnow() {

  noStroke();


  for (
    let s of snow
  ) {

    s.y +=
      s.speed;


    if (
      s.y > height
    ) {

      s.y =
        random(
          -100,
          0
        );


      s.x =
        random(width);

    }


    fill(
      255,
      210
    );


    circle(
      s.x,
      s.y,
      s.size
    );

  }

}


/* ========================================================
   房子
======================================================== */

function drawHouse() {

  /*
  雪地
  */

  noStroke();


  fill(
    225,
    232,
    237
  );


  rect(
    0,
    height * 0.72,
    width,
    height * 0.28
  );


  /*
  房子
  */

  fill(
    91,
    69,
    56
  );


  rect(
    width * 0.20,
    height * 0.48,
    width * 0.60,
    height * 0.28
  );


  /*
  屋頂
  */

  fill(
    61,
    52,
    49
  );


  triangle(

    width * 0.15,
    height * 0.50,

    width * 0.50,
    height * 0.29,

    width * 0.85,
    height * 0.50

  );

}


/* ========================================================
   燈籠
======================================================== */

function drawLanterns() {

  for (
    let lantern of lanterns
  ) {

    let x =
      width
      *
      lantern.xRatio;


    let y =
      height
      *
      0.49;


    /*
    光暈
    */

    if (
      lantern.lit
    ) {

      noStroke();


      for (
        let r = 120;
        r > 30;
        r -= 15
      ) {

        fill(

          255,

          174,

          75,

          map(
            r,
            120,
            30,
            5,
            45
          )

        );


        circle(
          x,
          y,
          r
        );

      }

    }


    /*
    燈籠
    */

    if (
      lantern.lit
    ) {

      fill(
        205,
        72,
        46
      );

    }

    else {

      fill(
        73,
        63,
        61
      );

    }


    noStroke();


    ellipse(
      x,
      y,
      48,
      62
    );

  }

}


/* ========================================================
   Camera Debug
======================================================== */

function drawCamera() {

  let camW =
    240;


  let camH =
    180;


  let camX =
    width
    -
    camW
    -
    25;


  let camY =
    25;


  /*
  黑底
  */

  noStroke();


  fill(
    0,
    120
  );


  rect(
    camX - 5,
    camY - 5,
    camW + 10,
    camH + 10,
    12
  );


  push();


  /*
  鏡像
  */

  translate(
    camX + camW,
    camY
  );


  scale(
    -1,
    1
  );


  image(
    video,
    0,
    0,
    camW,
    camH
  );


  pop();


  /*
  骨架
  */

  if (
    skeleton.length > 0
  ) {

    stroke(
      255,
      220,
      100
    );


    strokeWeight(3);


    for (
      let pair of skeleton
    ) {

      let a =
        pair[0].position;


      let b =
        pair[1].position;


      /*
      PoseNet 320×240
      轉成小鏡頭
      並鏡像
      */

      let ax =
        camX
        +
        camW
        -
        a.x
        *
        camW / 320;


      let ay =
        camY
        +
        a.y
        *
        camH / 240;


      let bx =
        camX
        +
        camW
        -
        b.x
        *
        camW / 320;


      let by =
        camY
        +
        b.y
        *
        camH / 240;


      line(
        ax,
        ay,
        bx,
        by
      );

    }

  }

}


/* ========================================================
   完成文字
======================================================== */

function drawCompleteText(
  txt
) {

  fill(
    255,
    245
  );


  noStroke();


  textAlign(
    CENTER,
    CENTER
  );


  textSize(30);


  text(
    txt,
    width / 2,
    height * 0.20
  );

}


/* ========================================================
   UI
======================================================== */

function updateUI() {

  let season;

  let instruction;

  let gesture;

  let progress;


  if (
    currentGame === 0
  ) {

    season =
      "春・播種";


    instruction =
      "向前傾";


    gesture =
      "雙手往下做撒種動作";


    progress =
      "已播種 "
      +
      plantedSeeds
      +
      " / "
      +
      maxSeeds;

  }


  else if (
    currentGame === 1
  ) {

    season =
      "夏・螢火";


    instruction =
      "左右搖擺控制捕蟲網";


    gesture =
      "看到螢火蟲時，用手輕輕揮動";


    progress =
      "已抓到 "
      +
      caughtFireflies
      +
      " / "
      +
      maxFireflies;

  }


  else if (
    currentGame === 2
  ) {

    season =
      "秋・收成";


    instruction =
      "保持身體在中心";


    gesture =
      "雙手一起往下做收割動作";


    progress =
      "收割 "
      +
      floor(
        harvestProgress
      )
      +
      "%";

  }


  else {

    season =
      "冬・點燈";


    instruction =
      "慢慢向後仰";


    gesture =
      "雙手慢慢舉高，點亮燈籠";


    progress =
      "已點亮 "
      +
      litLanterns
      +
      " / "
      +
      lanterns.length;

  }


  document
    .getElementById(
      "season"
    )
    .innerText =
      season;


  document
    .getElementById(
      "instruction"
    )
    .innerText =
      instruction;


  document
    .getElementById(
      "gesture"
    )
    .innerText =
      gesture;


  document
    .getElementById(
      "progress"
    )
    .innerText =
      progress;


  document
    .getElementById(
      "debugX"
    )
    .innerText =
      tiltX
      .toFixed(2);


  document
    .getElementById(
      "debugY"
    )
    .innerText =
      tiltY
      .toFixed(2);


  document
    .getElementById(
      "debugGesture"
    )
    .innerText =
      gestureName;

}


/* ========================================================
   Reset
======================================================== */

function resetGame() {

  currentGame = 0;


  plantedSeeds = 0;

  caughtFireflies = 0;

  harvestProgress = 0;

  litLanterns = 0;

  winterHold = 0;


  catcherX =
    width / 2;


  createSeeds();

  createFireflies();

  createRice();

  createLanterns();

}


/* ========================================================
   Keyboard
======================================================== */

function keyPressed() {

  /*
  R = 重玩
  */

  if (
    key === "r" ||
    key === "R"
  ) {

    resetGame();

  }


  /*
  C = 顯示 / 隱藏鏡頭
  */

  if (
    key === "c" ||
    key === "C"
  ) {

    showCamera =
      !showCamera;

  }


  /*
  F = 全螢幕
  */

  if (
    key === "f" ||
    key === "F"
  ) {

    fullscreen(
      !fullscreen()
    );

  }

}


/* ========================================================
   Responsive
======================================================== */

function windowResized() {

  resizeCanvas(
    windowWidth,
    windowHeight
  );

}

</script>

</body>

</html>
