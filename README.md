<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>競馬風レースゲーム（完全再現版）</title>
  <style>
    body {
      margin: 0;
      background: rgb(50,120,80);
    }
    canvas {
      display: block;
      margin: auto;
      background: rgb(50,120,80);
    }
  </style>
</head>
<body>
  <canvas id="gameCanvas" width="800" height="600"></canvas>
  <script>
    // 定数設定
    const WIDTH = 800;
    const HEIGHT = 600;
    const BACKGROUND_COLOR = "rgb(50,120,80)";      // 濃い緑（芝生）
    const TRACK_COLOR_DIRT = "rgb(210,180,140)";      // ダート
    const TRACK_COLOR_TURF = "rgb(60,179,113)";       // 芝
    const TEXT_COLOR = "rgb(255,255,255)";            // テキスト色

    const NUM_RACERS = 12;
    const RACER_SIZE = 10;
    const RACER_COLORS = [
      "rgb(255,0,0)",     // 赤
      "rgb(0,0,255)",     // 青
      "rgb(255,255,0)",   // 黄
      "rgb(128,0,128)",   // 紫
      "rgb(255,165,0)",   // オレンジ
      "rgb(0,255,255)",   // シアン
      "rgb(50,205,50)",   // ライム
      "rgb(255,105,180)", // ピンク
      "rgb(165,42,42)",   // 茶色
      "rgb(0,128,0)",     // 深緑
      "rgb(220,20,60)",   // クリムゾン
      "rgb(30,144,255)"   // ドジャーブルー
    ];
    const RUNNING_STYLES = ["逃げ", "先行", "差し", "追込"];

    const COURSES = [
      { name: "東京競馬場", length: 2400, surface: "芝", feature: "直線が長い", color: TRACK_COLOR_TURF },
      { name: "中山競馬場", length: 1600, surface: "芝", feature: "急なコーナー", color: TRACK_COLOR_TURF },
      { name: "阪神競馬場", length: 1200, surface: "ダート", feature: "上り坂がきつい", color: TRACK_COLOR_DIRT }
    ];

    // Canvasの取得
    const canvas = document.getElementById("gameCanvas");
    const ctx = canvas.getContext("2d");

    // 時間管理（秒単位に統一）
    function getTime() {
      return performance.now() / 1000;
    }

    // ユーティリティ：距離タイプを返す
    function getDistanceType(length) {
      if (length === 1200) return "スプリント";
      if (length === 1600) return "マイル";
      if (length === 2400) return "クラシック";
      if (length === 3000) return "ディスタンス";
      if (length < 1400) return "スプリント";
      else if (length < 2000) return "マイル";
      else if (length < 2600) return "クラシック";
      else return "ディスタンス";
    }

    // クラス：Commentary（実況管理）
    class Commentary {
      constructor() {
        this.messages = [];  // {text, time}
        this.lastMessageTime = 0;
        this.messageDuration = 3.0;  // 表示時間（秒）
        this.minDisplayTime = 2.0;   // 最低表示時間

        // コーナー通過記録（1～4）
        this.cornerPassed = {1: new Set(), 2: new Set(), 3: new Set(), 4: new Set()};
        this.lastLeaders = [];
        this.lastSpurtTime = 0;
        this.spurtCooldown = 5.0;
        this.finishAnnounced = new Set();

        // キュー管理
        this.messageQueue = []; // 各要素: {text, timestamp, priority}
        this.lastQueueProcessTime = 0;
        this.queueProcessInterval = 1.5;

        // 静寂管理
        this.lastCommentaryTime = 0;
        this.silenceThreshold = 7.0;

        // 特別実況管理
        this.mentionedPopularHorses = new Set();
        this.mentionedRunningStyles = new Set();
        this.racePhaseComments = {"序盤": false, "中盤": false, "終盤": false, "ラスト": false};

        // 後方グループ実況管理
        this.lastBackCommentaryTime = 0;
        this.backCommentaryInterval = 10.0;

        // 言及済み馬
        this.recentlyMentionedHorses = new Set();
        this.mentionCooldown = 20.0;
        this.lastMentionTime = {};  // {horseNumber: time}

        // レース終了管理
        this.raceFinished = false;
        this.raceSummaryAnnounced = false;
        this.winnerNumber = null;
        this.winnerMargin = 0.0;
        this.winnerOdds = 0.0;
        this.winnerStyle = "";
      }

      addMessage(text, currentTime, priority = 1) {
        // 同一メッセージがキューまたは表示中なら追加しない
        if (this.messageQueue.some(m => m.text === text)) return;
        if (this.messages.some(m => m.text === text)) return;
        this.messageQueue.push({ text: text, timestamp: currentTime, priority: priority });
        // 優先度降順でソート
        this.messageQueue.sort((a, b) => b.priority - a.priority);
      }

      processQueue(currentTime) {
        if (this.messageQueue.length === 0) return false;
        // 表示中のメッセージがある場合、最低表示時間を確保
        if (this.getActiveMessages(currentTime).length > 0 &&
            currentTime - this.lastMessageTime < this.minDisplayTime) {
          return false;
        }
        let msgObj = this.messageQueue.shift();
        this.messages.push({ text: msgObj.text, time: currentTime });
        this.lastMessageTime = currentTime;
        this.lastCommentaryTime = currentTime;
        return true;
      }

      getActiveMessages(currentTime) {
        return this.messages.filter(m => currentTime - m.time < this.messageDuration)
                            .map(m => m.text);
      }

      checkSilence(currentTime, raceProgress) {
        if ((currentTime - this.lastCommentaryTime >= this.silenceThreshold) &&
            this.messageQueue.length === 0) {
          return true;
        }
        return false;
      }

      resetRacePhaseComments(raceProgress) {
        if (raceProgress < 0.3 && !this.racePhaseComments["序盤"]) {
          this.racePhaseComments = {"序盤": false, "中盤": false, "終盤": false, "ラスト": false};
        } else if (raceProgress >= 0.3 && raceProgress < 0.6 && !this.racePhaseComments["中盤"]) {
          this.racePhaseComments["序盤"] = true;
          this.racePhaseComments["中盤"] = false;
        } else if (raceProgress >= 0.6 && raceProgress < 0.8 && !this.racePhaseComments["終盤"]) {
          this.racePhaseComments["中盤"] = true;
          this.racePhaseComments["終盤"] = false;
        } else if (raceProgress >= 0.8 && !this.racePhaseComments["ラスト"]) {
          this.racePhaseComments["終盤"] = true;
          this.racePhaseComments["ラスト"] = false;
        }
      }

      setRaceFinished(winnerNumber, winnerStyle, winnerOdds, margin = 0.0) {
        this.raceFinished = true;
        this.winnerNumber = winnerNumber;
        this.winnerStyle = winnerStyle;
        this.winnerOdds = winnerOdds;
        this.winnerMargin = margin;
      }

      clearMessageQueue() {
        this.messageQueue = this.messageQueue.filter(m => m.priority >= 4);
      }

      clearAllMessages() {
        this.messageQueue = [];
        this.messages = [];
      }
    }

    // クラス：Track（コース管理・描画）
    class Track {
      constructor() {
        this.trackWidth = 40;
        this.lanes = 6;
        this.centerX = WIDTH / 2;
        this.centerY = HEIGHT / 2;
        this.width = 500;
        this.height = 250;
        this.straightWidth = 300;
        this.radius = this.height / 2;
        this.laneWidth = this.trackWidth / this.lanes;
        this.innerX1 = this.centerX - this.straightWidth / 2;
        this.innerX2 = this.centerX + this.straightWidth / 2;
        this.innerY1 = this.centerY - this.radius;
        this.innerY2 = this.centerY + this.radius;
        // 外周は未使用だが計算可能
        this.outerX1 = this.innerX1;
        this.outerX2 = this.innerX2;
        this.outerY1 = this.innerY1 - this.trackWidth;
        this.outerY2 = this.innerY2 + this.trackWidth;
        this.leftCenterX = this.innerX1;
        this.rightCenterX = this.innerX2;
        this.topCenterY = this.innerY1;
        this.bottomCenterY = this.innerY2;
        // セクションの長さ
        this.straightLength = this.straightWidth;
        this.cornerLength = Math.PI * this.radius;
        this.sectionLengths = [this.straightLength, this.cornerLength, this.straightLength, this.cornerLength];
        this.lapLength = this.sectionLengths.reduce((a, b) => a + b, 0);
        // ゴール位置（上直線の75%地点）
        this.goalX = this.innerX1 + this.straightWidth * 0.75;
        this.goalY = this.innerY1;
        this.startDistance = 0;
        this.requiredCrossings = 1;
        this.raceDistance = 0;
        this.goalDistance = 0;
        this.goalPosition = { x: this.goalX, y: this.innerY1 - this.trackWidth / 2 };
        this.baseLapMeters = 1600;
      }

      calculateDistanceToGoal() {
        // 上直線の75%地点から測定
        return this.straightLength * 0.75;
      }

      setupForCourse(course) {
        this.raceDistance = course.length;
        let trackLapMeters = 1600; // デフォルト
        let trackScale = trackLapMeters / this.lapLength;
        let goalOffset = this.calculateDistanceToGoal();

        if (course.length === 1200) { // スプリント
          let lapsToRun = 0.75;
          let targetGameDistance = (trackLapMeters * lapsToRun) / trackScale;
          let startPos = goalOffset - targetGameDistance;
          if (startPos < 0) startPos += this.lapLength;
          this.startDistance = startPos;
          this.requiredCrossings = 1;
        } else if (course.length === 1600) { // マイル
          this.startDistance = goalOffset;
          this.requiredCrossings = 1;
        } else if (course.length === 2400) { // クラシック
          let lapsToRun = 1.5;
          let targetGameDistance = (trackLapMeters * lapsToRun) / trackScale;
          let startPos = goalOffset - (targetGameDistance % this.lapLength);
          if (startPos < 0) startPos += this.lapLength;
          this.startDistance = startPos;
          this.requiredCrossings = 2;
        } else {
          let lapsToRun = course.length / trackLapMeters;
          let targetGameDistance = (trackLapMeters * lapsToRun) / trackScale;
          let startPos = goalOffset - (targetGameDistance % this.lapLength);
          if (startPos < 0) startPos += this.lapLength;
          this.startDistance = startPos;
          this.requiredCrossings = Math.ceil(lapsToRun);
        }
        this.goalDistance = this.startDistance + (this.raceDistance / trackScale);
        this.trackScale = trackScale;
        this.raceGameDistance = this.raceDistance / trackScale;
        // デバッグ出力（必要ならconsole.logで確認）
        // console.log(course.name, course.length, this.lapLength, trackScale, this.startDistance, goalOffset, this.requiredCrossings, this.raceGameDistance, this.goalDistance);
      }

      getTrackScale() {
        return this.trackScale || 1.0;
      }

      draw(ctx, course) {
        // 背景
        ctx.fillStyle = BACKGROUND_COLOR;
        ctx.fillRect(0, 0, WIDTH, HEIGHT);
        let trackColor = course.color;
        let fieldColor = (course.surface === "芝") ? "rgb(20,100,20)" : "rgb(139,69,19)";
        // フィールド（直線部分）
        ctx.fillStyle = fieldColor;
        ctx.fillRect(this.innerX1, this.innerY1, this.straightWidth, 2 * this.radius);
        // 上下トラック
        ctx.fillStyle = trackColor;
        ctx.fillRect(this.innerX1, this.innerY1 - this.trackWidth, this.straightWidth, this.trackWidth);
        ctx.fillRect(this.innerX1, this.innerY2, this.straightWidth, this.trackWidth);
        // 右カーブ
        ctx.lineWidth = this.trackWidth;
        ctx.strokeStyle = trackColor;
        ctx.beginPath();
        ctx.arc(this.innerX2, this.centerY, this.radius + this.trackWidth, -Math.PI/2, Math.PI/2);
        ctx.stroke();
        // 左カーブ
        ctx.beginPath();
        ctx.arc(this.innerX1, this.centerY, this.radius + this.trackWidth, Math.PI/2, Math.PI*3/2);
        ctx.stroke();
        // レーンの白線
        ctx.lineWidth = 1;
        ctx.strokeStyle = "rgba(255,255,255,0.4)";
        for (let i = 1; i < this.lanes; i++) {
          let laneOffset = i * this.laneWidth;
          // 上直線
          ctx.beginPath();
          ctx.moveTo(this.innerX1, this.innerY1 - laneOffset);
          ctx.lineTo(this.innerX2, this.innerY1 - laneOffset);
          ctx.stroke();
          // 下直線
          ctx.beginPath();
          ctx.moveTo(this.innerX1, this.innerY2 + laneOffset);
          ctx.lineTo(this.innerX2, this.innerY2 + laneOffset);
          ctx.stroke();
        }
        // スタート表示
        let startPos = this.getPosition(this.startDistance, 0);
        ctx.fillStyle = "#FFFFFF";
        ctx.beginPath();
        ctx.arc(startPos.x, startPos.y, 5, 0, Math.PI*2);
        ctx.fill();
        ctx.font = "16px sans-serif";
        ctx.fillText("START", startPos.x - 20, startPos.y - 25);
        // ゴール表示（縞模様）
        let finishLineWidth = 6;
        let stripeWidth = 2;
        for (let i = 0; i < finishLineWidth; i += stripeWidth) {
          ctx.strokeStyle = ((i / stripeWidth) % 2 === 0) ? "#FF0000" : "#FFFFFF";
          ctx.beginPath();
          ctx.moveTo(this.goalX + i, this.innerY1 - this.trackWidth);
          ctx.lineTo(this.goalX + i, this.innerY1);
          ctx.stroke();
        }
        ctx.fillText("GOAL", this.goalX - 20, this.innerY1 - this.trackWidth - 25);
      }

      getPosition(distance, lane) {
        let total = this.lapLength;
        let lapDistance = distance % total;
        let laps = Math.floor(distance / total);
        let currentSection = 0;
        let sectionStart = 0;
        let sectionProgress = 0;
        for (let secLength of this.sectionLengths) {
          if (lapDistance < sectionStart + secLength) {
            sectionProgress = (lapDistance - sectionStart) / secLength;
            break;
          }
          sectionStart += secLength;
          currentSection++;
        }
        let lanePos = (lane + 0.5) * this.laneWidth;
        let result = { x: 0, y: 0, angle: 0, isCorner: false, cornerNum: null, laps: laps, section: currentSection };
        if (currentSection === 0) { // 上直線
          result.x = this.innerX1 + sectionProgress * this.straightWidth;
          result.y = this.innerY1 - lanePos;
          result.angle = 0;
        } else if (currentSection === 1) { // 右カーブ
            let angleRad = sectionProgress * Math.PI;
            let centerX = this.innerX2;
            let centerY = this.centerY;
            // 修正：カーブでは内側基準ではなく、トラックの中心線に合わせる
            let radius = this.radius + this.trackWidth/2 + lanePos;
            result.x = centerX + radius * Math.sin(angleRad);
            result.y = centerY - radius * Math.cos(angleRad);
            result.angle = angleRad * 180 / Math.PI;
            result.isCorner = true;
            result.cornerNum = (sectionProgress < 0.5) ? 1 : 2;
          }else if (currentSection === 2) { // 下直線
          result.x = this.innerX2 - sectionProgress * this.straightWidth;
          result.y = this.innerY2 + lanePos;
          result.angle = 180;
        } else if (currentSection === 3) { // 左カーブ
          let angleRad = Math.PI + sectionProgress * Math.PI;
          let centerX = this.innerX1;
          let centerY = this.centerY;
          let radius = this.radius + this.trackWidth/2 + lanePos;
          result.x = centerX + radius * Math.sin(angleRad);
          result.y = centerY - radius * Math.cos(angleRad);
          result.angle = angleRad * 180 / Math.PI;
          result.isCorner = true;
          result.cornerNum = (sectionProgress < 0.5) ? 3 : 4;
        }
        return result;
      }

      // drawCameraはオフスクリーン描画とスケーリングによるカメラ表示（ここでは省略せず再現）
      drawCamera(ctx, course, cameraX, cameraY, scale) {
        // オフスクリーン用Canvasを生成
        let offWidth = 2000, offHeight = 1200;
        let offCanvas = document.createElement("canvas");
        offCanvas.width = offWidth;
        offCanvas.height = offHeight;
        let offCtx = offCanvas.getContext("2d");
        // 背景は透明
        offCtx.clearRect(0, 0, offWidth, offHeight);
        // オフスクリーンの中心を(1000,600)に固定
        let offsetX = 1000 - this.centerX;
        let offsetY = 600 - this.centerY;
        // 以下、draw()の処理をoffset付きで実行
        let trackColor = course.color;
        let fieldColor = (course.surface === "芝") ? "rgb(20,100,20)" : "rgb(139,69,19)";
        // フィールド
        offCtx.fillStyle = fieldColor;
        offCtx.fillRect(this.innerX1 + offsetX, this.innerY1 + offsetY, this.straightWidth, 2*this.radius);
        // 上下トラック
        offCtx.fillStyle = trackColor;
        offCtx.fillRect(this.innerX1 + offsetX, this.innerY1 - this.trackWidth + offsetY, this.straightWidth, this.trackWidth);
        offCtx.fillRect(this.innerX1 + offsetX, this.innerY2 + offsetY, this.straightWidth, this.trackWidth);
        // 右カーブ
        offCtx.lineWidth = this.trackWidth;
        offCtx.strokeStyle = trackColor;
        offCtx.beginPath();
        offCtx.arc(this.innerX2 + offsetX, this.centerY + offsetY, this.radius + this.trackWidth, -Math.PI/2, Math.PI/2);
        offCtx.stroke();
        // 左カーブ
        offCtx.beginPath();
        offCtx.arc(this.innerX1 + offsetX, this.centerY + offsetY, this.radius + this.trackWidth, Math.PI/2, Math.PI*3/2);
        offCtx.stroke();
        // レーン白線
        offCtx.lineWidth = 1;
        offCtx.strokeStyle = "rgba(255,255,255,0.4)";
        for (let i = 1; i < this.lanes; i++) {
          let laneOffset = i * this.laneWidth;
          offCtx.beginPath();
          offCtx.moveTo(this.innerX1 + offsetX, this.innerY1 - laneOffset + offsetY);
          offCtx.lineTo(this.innerX2 + offsetX, this.innerY1 - laneOffset + offsetY);
          offCtx.stroke();
          offCtx.beginPath();
          offCtx.moveTo(this.innerX1 + offsetX, this.innerY2 + laneOffset + offsetY);
          offCtx.lineTo(this.innerX2 + offsetX, this.innerY2 + laneOffset + offsetY);
          offCtx.stroke();
        }
        // スタート表示
        let startPos = this.getPosition(this.startDistance, 0);
        let sx = startPos.x + offsetX, sy = startPos.y + offsetY;
        offCtx.fillStyle = "#FFFFFF";
        offCtx.beginPath();
        offCtx.arc(sx, sy, 5, 0, Math.PI*2);
        offCtx.fill();
        offCtx.font = "16px sans-serif";
        offCtx.fillText("START", sx - 20, sy - 25);
        // ゴール表示
        let finishLineWidth = 6, stripeWidth = 2;
        for (let i = 0; i < finishLineWidth; i += stripeWidth) {
          offCtx.strokeStyle = ((i / stripeWidth) % 2 === 0) ? "#FF0000" : "#FFFFFF";
          offCtx.beginPath();
          offCtx.moveTo(this.goalX + i + offsetX, this.innerY1 - this.trackWidth + offsetY);
          offCtx.lineTo(this.goalX + i + offsetX, this.innerY1 + offsetY);
          offCtx.stroke();
        }
        offCtx.fillText("GOAL", this.goalX - 20 + offsetX, this.innerY1 - this.trackWidth - 25 + offsetY);
        // scaleして描画
        let scaledW = offWidth * scale;
        let scaledH = offHeight * scale;
        let scaledCanvas = document.createElement("canvas");
        scaledCanvas.width = scaledW;
        scaledCanvas.height = scaledH;
        let scaledCtx = scaledCanvas.getContext("2d");
        scaledCtx.drawImage(offCanvas, 0, 0, scaledW, scaledH);
        // カメラ位置補正：(cameraX+offsetX, cameraY+offsetY)を画面中央に
        let centerScreenX = WIDTH / 2, centerScreenY = HEIGHT / 2;
        let camOffX = (cameraX + offsetX), camOffY = (cameraY + offsetY);
        let scaledCamX = camOffX * scale, scaledCamY = camOffY * scale;
        let finalPosX = centerScreenX - scaledCamX;
        let finalPosY = centerScreenY - scaledCamY;
        ctx.drawImage(scaledCanvas, finalPosX, finalPosY);
      }
    }

    // クラス：Racer（競走馬）
    class Racer {
      constructor(name, startLane, color, course, number, track) {
        this.name = name;
        this.number = number;
        this.color = color;
        this.startLane = startLane;
        this.lane = startLane;
        this.targetLane = startLane;
        this.distance = 0;
        this.runningStyle = RUNNING_STYLES[Math.floor(Math.random() * RUNNING_STYLES.length)];
        this.baseSpeed = 1.0 + Math.random() * 0.4;
        this.stamina = 0.9 + Math.random() * 0.2;
        this.cornerSkill = 0.85 + Math.random() * 0.3;
        this.positioningSkill = 0.8 + Math.random() * 0.4;
        this.power = 0.8 + Math.random() * 0.4;
        this.distanceAptitude = {
          "スプリント": 0.9 + Math.random() * 0.2,
          "マイル": 0.9 + Math.random() * 0.2,
          "クラシック": 0.9 + Math.random() * 0.2,
          "ディスタンス": 0.9 + Math.random() * 0.2
        };
        this.surfaceAptitude = {
          "芝": 0.9 + Math.random() * 0.2,
          "ダート": 0.9 + Math.random() * 0.2
        };
        this.course = course;
        this.finished = false;
        this.finishTime = 0;
        this.finishPosition = 0;
        this.spurtUsed = false;
        this.inCorner = false;
        this.x = 0;
        this.y = 0;
        this.angle = 0;
        this.tilt = 0;
        this.isBlocked = false;
        this.laneChangeCooldown = 0;
        this.rank = 0;
        this.leaderDistance = 0;
        this.margin = 0.0;
        this.odds = 10.0;
        this.finishLineCrossings = 0;
        this.passedFinishLine = false;
        this.currentLap = 0;
        this.initialDistance = 0;
        this.lastCorner = null;
        this.prevRank = 0;
        this.hasSpurtAnnounced = false;
        this.position = track.getPosition(0, this.lane);
      }

      calculateOdds(allRacers) {
        let distanceType = getDistanceType(this.course.length);
        let horsePowers = [];
        for (let racer of allRacers) {
          let styleBonus = 1.0;
          if ((racer.runningStyle === "逃げ" && (distanceType==="スプリント" || distanceType==="マイル")) ||
              (racer.runningStyle === "追込" && (distanceType==="クラシック" || distanceType==="ディスタンス"))) {
            styleBonus = 1.1;
          }
          let speedWeight, staminaWeight, powerWeight;
          if (distanceType === "スプリント") {
            speedWeight = 4.0; staminaWeight = 1.5; powerWeight = 1.2;
          } else if (distanceType === "マイル") {
            speedWeight = 3.5; staminaWeight = 2.0; powerWeight = 1.5;
          } else if (distanceType === "クラシック") {
            speedWeight = 3.0; staminaWeight = 2.5; powerWeight = 2.0;
          } else {
            speedWeight = 2.5; staminaWeight = 3.0; powerWeight = 2.5;
          }
          let totalPower = (
            this.baseSpeed * speedWeight +
            this.stamina * staminaWeight +
            this.cornerSkill * 1.5 +
            this.positioningSkill * 1.0 +
            this.power * powerWeight +
            this.surfaceAptitude[this.course.surface] * 3.0 +
            this.distanceAptitude[distanceType] * 2.5
          ) * styleBonus;
          horsePowers.push({ racer: racer, power: totalPower });
        }
        horsePowers.sort((a,b) => b.power - a.power);
        let myRank = 0, myPower = 0;
        for (let i = 0; i < horsePowers.length; i++) {
          if (horsePowers[i].racer === this) {
            myRank = i + 1;
            myPower = horsePowers[i].power;
            break;
          }
        }
        let totalHorses = allRacers.length;
        let baseOdds = 0;
        if (myRank === 1) {
          baseOdds = 1.5 + Math.random() * 1.5;
        } else {
          let rankRatio = (myRank - 1) / (totalHorses - 1);
          let strongestHorsePower = horsePowers[0].power;
          let powerRatio = 1.0 - (myPower / strongestHorsePower);
          let combinedFactor = rankRatio * 0.7 + powerRatio * 0.3;
          baseOdds = 2.0 + combinedFactor * 58.0;
        }
        let finalOdds = baseOdds * (0.9 + Math.random() * 0.2);
        this.odds = Math.max(1.2, Math.min(99.9, Math.round(finalOdds * 10)/10));
      }

      getStyleModifier(progress) {
        if (this.runningStyle === "逃げ") {
          if (progress < 0.3) return 1.15;
          else if (progress < 0.5) return 1.12;
          else if (progress < 0.7) return 1.08;
          else if (progress < 0.9) return 1.00 * this.stamina;
          else return 0.95 * this.stamina;
        } else if (this.runningStyle === "先行") {
          if (progress < 0.3) return 1.10;
          else if (progress < 0.5) return 1.12;
          else if (progress < 0.7) return 1.08;
          else if (progress < 0.9) return 1.05 * this.stamina;
          else return 1.00 * this.stamina;
        } else if (this.runningStyle === "差し") {
          if (progress < 0.3) return 0.85;
          else if (progress < 0.5) return 0.95;
          else if (progress < 0.7) return 1.05;
          else if (progress < 0.9) return 1.18 * this.stamina;
          else return 1.15 * this.stamina;
        } else if (this.runningStyle === "追込") {
          if (progress < 0.3) return 0.80;
          else if (progress < 0.5) return 0.85;
          else if (progress < 0.7) return 0.95;
          else if (progress < 0.9) return 1.25 * this.stamina;
          else return 1.30 * this.stamina;
        }
        return 1.0;
      }

      initializeRace(track) {
        this.distance = track.startDistance;
        this.initialDistance = track.startDistance;
        this.currentLap = 0;
        this.finishLineCrossings = 0;
        this.passedFinishLine = false;
        this.position = track.getPosition(this.distance, this.lane);
      }

      update(track, racers) {
        if (this.finished) return false;
        let prevPosition = this.position;
        this.position = track.getPosition(this.distance, this.lane);
        if (prevPosition && prevPosition.laps !== this.position.laps) {
          this.currentLap = this.position.laps;
        }
        // 順位計算（未完走馬のみ）
        let allDist = racers.filter(r => !r.finished).map(r => ({racer: r, distance: r.distance}));
        allDist.sort((a, b) => b.distance - a.distance);
        this.prevRank = this.rank;
        for (let i = 0; i < allDist.length; i++) {
          if (allDist[i].racer === this) {
            this.rank = i + 1;
            this.leaderDistance = (i > 0) ? allDist[0].distance - allDist[i].distance : 0;
            break;
          }
        }
        // ブロック判定
        this.checkCompetition(racers);
        // レーン変更判定
        this.decideLaneChange(track, racers);
        if (Math.abs(this.lane - this.targetLane) > 0.01) {
          this.lane += (this.targetLane - this.lane) * 0.02;
        }
        let totalDistance = track.raceGameDistance;
        let currentDistance = this.distance - this.initialDistance;
        let raceProgress = Math.min(1.0, currentDistance / totalDistance);
        let styleMod = this.getStyleModifier(raceProgress);
        let cornerMod = 1.0;
        if (this.position.isCorner) {
          cornerMod = 0.95 + (this.cornerSkill - 0.85) * 0.2;
        }
        let surfaceMod = 0.85 + (this.surfaceAptitude[this.course.surface] - 0.9) * 1.5;
        let distanceType = getDistanceType(this.course.length);
        let distanceMod = 0.95 + (this.distanceAptitude[distanceType] - 0.9) * 0.5;
        let blockMod = (this.isBlocked) ? 0.95 + (this.power - 0.8) * 0.2 : 1.0;
        let laneMod = 1.0 - (this.lane / 40);
        let positionMod = 1.0;
        if (this.rank > 1) {
          positionMod = 1.0 + Math.min(0.01 * (this.rank - 1), 0.06);
        }
        if (this.leaderDistance > 20) {
          positionMod += Math.min(this.leaderDistance / 400, 0.04);
        }
        let spurtMod = 1.0;
        let wasSpurtUsed = this.spurtUsed;
        if (raceProgress > 0.65 && !this.spurtUsed) {
          let spurtChance = (raceProgress > 0.8) ? 0.05 : 0.01;
          if (this.runningStyle === "逃げ") spurtChance *= 0.5;
          else if (this.runningStyle === "先行") spurtChance *= 0.7;
          else if (this.runningStyle === "差し") spurtChance *= 1.2;
          else if (this.runningStyle === "追込") spurtChance *= 1.5;
          if (Math.random() < spurtChance) {
            this.spurtUsed = true;
            if (this.runningStyle === "追込") spurtMod = 1.20 + Math.random() * 0.15;
            else if (this.runningStyle === "差し") spurtMod = 1.15 + Math.random() * 0.13;
            else if (this.runningStyle === "先行") spurtMod = 1.10 + Math.random() * 0.10;
            else spurtMod = 1.08 + Math.random() * 0.07;
          }
        }
        let randomMod = 0.98 + Math.random() * 0.04;
        let speed = (this.baseSpeed * styleMod * cornerMod * surfaceMod *
                     distanceMod * blockMod * laneMod * positionMod *
                     spurtMod * randomMod) * 0.8;
        this.distance += speed * 0.7;
        // ゴール判定
        if (this.position.section === 0 && !this.passedFinishLine) {
          if (prevPosition && (prevPosition.x < track.goalX && this.position.x >= track.goalX)) {
            this.finishLineCrossings++;
            this.passedFinishLine = true;
          }
        } else {
          if (this.passedFinishLine && (this.position.section !== 0 || this.position.x > track.goalX + 50)) {
            this.passedFinishLine = false;
          }
        }
        let requiredDist = track.raceGameDistance;
        let actualDist = this.distance - this.initialDistance;
        if ((this.finishLineCrossings >= track.requiredCrossings && actualDist >= requiredDist) && !this.finished) {
          this.finished = true;
          return true;
        }
        return (!wasSpurtUsed && this.spurtUsed);
      }

      checkCompetition(racers) {
        this.isBlocked = false;
        for (let racer of racers) {
          if (racer !== this && !racer.finished) {
            if ((racer.distance - this.distance > 0) && (racer.distance - this.distance < 10) &&
                (Math.abs(racer.lane - this.lane) < 0.7)) {
              this.isBlocked = true;
              break;
            }
          }
        }
      }

      decideLaneChange(track, racers) {
        if (this.laneChangeCooldown > 0) {
          this.laneChangeCooldown -= 1;
          return;
        }
        let currentDistance = this.distance - this.initialDistance;
        let raceProgress = Math.min(1.0, currentDistance / track.raceGameDistance);
        let insidePref = 0, outsidePref = 0;
        if (this.position.isCorner) insidePref += 2;
        if (this.isBlocked) outsidePref += 3;
        if (this.runningStyle === "逃げ" || this.runningStyle === "先行") {
          if (raceProgress < 0.6) insidePref += 2;
        } else if (this.runningStyle === "差し") {
          if (raceProgress >= 0.5) insidePref += 2;
          else outsidePref += 1;
        } else if (this.runningStyle === "追込") {
          if (raceProgress >= 0.6) insidePref += 3;
          else outsidePref += 2;
        }
        let decision = Math.random();
        if (insidePref > outsidePref && this.lane > 0 && decision > 0.7) {
          this.targetLane = Math.max(0, this.lane - (0.5 + Math.random() * 0.5));
          this.laneChangeCooldown = Math.floor(20 + Math.random() * 20);
        } else if (outsidePref > insidePref && this.lane < track.lanes - 1 && decision > 0.7) {
          this.targetLane = Math.min(track.lanes - 1, this.lane + (0.5 + Math.random() * 0.5));
          this.laneChangeCooldown = Math.floor(20 + Math.random() * 20);
        }
      }

      draw(ctx) {
        if (!this.position) return;
        let x = this.position.x, y = this.position.y, angle = this.position.angle;
        ctx.fillStyle = this.color;
        ctx.beginPath();
        ctx.arc(x, y, RACER_SIZE, 0, Math.PI*2);
        ctx.fill();
        // 頭の向き
        let headAngle = angle * Math.PI / 180;
        let headX = x + Math.cos(headAngle) * RACER_SIZE;
        let headY = y + Math.sin(headAngle) * RACER_SIZE;
        ctx.strokeStyle = "#000000";
        ctx.lineWidth = 2;
        ctx.beginPath();
        ctx.moveTo(x, y);
        ctx.lineTo(headX, headY);
        ctx.stroke();
        // ブロック中表示
        if (this.isBlocked) {
          ctx.font = "16px sans-serif";
          ctx.fillStyle = "#FF0000";
          ctx.fillText("!", x - 3, y - RACER_SIZE - 15);
        }
        // スパート演出
        if (this.spurtUsed && !this.finished) {
          for (let i = 0; i < 3; i++) {
            let spurtAngle = angle + 180 + (Math.random() * 60 - 30);
            let spurtRad = spurtAngle * Math.PI / 180;
            let spurtLength = 3 + Math.random() * 5;
            let spurtX = x + Math.cos(spurtRad) * spurtLength;
            let spurtY = y + Math.sin(spurtRad) * spurtLength;
            ctx.strokeStyle = "#FFFFFF";
            ctx.lineWidth = 1;
            ctx.beginPath();
            ctx.moveTo(x, y);
            ctx.lineTo(spurtX, spurtY);
            ctx.stroke();
          }
        }
        // 馬番表示（黒い縁取り付き）
        ctx.font = "16px sans-serif";
        let offsets = [[-1,-1],[-1,1],[1,-1],[1,1],[0,-1],[0,1],[-1,0],[1,0]];
        for (let [dx,dy] of offsets) {
          ctx.fillStyle = "#000000";
          ctx.fillText(this.number, x - 8 + dx, y - 8 + dy);
        }
        ctx.fillStyle = "#FFFFFF";
        ctx.fillText(this.number, x - 8, y - 8);
      }

      drawCamera(ctx, cameraX, cameraY, scale) {
        if (!this.position) return;
        // カメラ変換
        let sx = (this.position.x - cameraX) * scale + WIDTH/2;
        let sy = (this.position.y - cameraY) * scale + HEIGHT/2;
        let size = RACER_SIZE * scale;
        ctx.fillStyle = this.color;
        ctx.beginPath();
        ctx.arc(sx, sy, size, 0, Math.PI*2);
        ctx.fill();
        let headAngle = this.position.angle * Math.PI / 180;
        let headX = sx + Math.cos(headAngle) * size;
        let headY = sy + Math.sin(headAngle) * size;
        ctx.strokeStyle = "#000000";
        ctx.lineWidth = Math.max(1, 2 * scale);
        ctx.beginPath();
        ctx.moveTo(sx, sy);
        ctx.lineTo(headX, headY);
        ctx.stroke();
        if (this.isBlocked) {
          ctx.font = Math.floor(20 * scale) + "px sans-serif";
          ctx.fillStyle = "#FF0000";
          ctx.fillText("!", sx - 3, sy - size - 15);
        }
        if (this.spurtUsed && !this.finished) {
          for (let i = 0; i < 3; i++) {
            let spurtAngle = this.position.angle + 180 + (Math.random()*60 - 30);
            let spurtRad = spurtAngle * Math.PI / 180;
            let spurtLength = (3 + Math.random()*5) * scale;
            let spurtX = sx + Math.cos(spurtRad) * spurtLength;
            let spurtY = sy + Math.sin(spurtRad) * spurtLength;
            ctx.strokeStyle = "#FFFFFF";
            ctx.lineWidth = Math.max(1, 1 * scale);
            ctx.beginPath();
            ctx.moveTo(sx, sy);
            ctx.lineTo(spurtX, spurtY);
            ctx.stroke();
          }
        }
        // 馬番（アウトライン付き）
        let fontSize = Math.floor(24 * scale);
        ctx.font = fontSize + "px sans-serif";
        let textPosX = sx - fontSize/3;
        let textPosY = sy - fontSize/3;
        let outlineWidth = Math.max(1, scale);
        let outlineOffsets = [[-outlineWidth,-outlineWidth],[-outlineWidth,outlineWidth],[outlineWidth,-outlineWidth],[outlineWidth,outlineWidth]];
        for (let [dx,dy] of outlineOffsets) {
          ctx.fillStyle = "#000000";
          ctx.fillText(this.number, textPosX+dx, textPosY+dy);
        }
        ctx.fillStyle = "#FFFFFF";
        ctx.fillText(this.number, textPosX, textPosY);
      }
    }

    // 実況テキスト生成（Python版のget_commentary_text相当）
    function getCommentaryText(racers, track, course, currentTime, commentary) {
      if (commentary.raceSummaryAnnounced) return;
      let sortedRacers = racers.slice().sort((a,b) => b.distance - a.distance);
      let raceProgress = 0;
      if (sortedRacers.length > 0) {
        let leader = sortedRacers[0];
        let totalDistance = track.raceGameDistance;
        let currentDistance = leader.distance - leader.initialDistance;
        raceProgress = Math.min(1.0, currentDistance / totalDistance);
      }
      commentary.resetRacePhaseComments(raceProgress);
      if (commentary.raceFinished && !commentary.raceSummaryAnnounced) {
        commentary.clearMessageQueue();
        let summaryMessages = [];
        if (commentary.winnerMargin > 3.0) {
          summaryMessages = [
            `レースは${commentary.winnerNumber}番の圧勝！他の追随を許さない見事な強さでした！`,
            `${commentary.winnerNumber}番の完全独走による圧巻のレースでした！`,
            `今回のレースは${commentary.winnerNumber}番の一人旅！他の馬を寄せ付けない強さでした！`
          ];
        } else if (commentary.winnerMargin < 0.5) {
          summaryMessages = [
            `レースは最後まで大接戦となり、わずかな差で${commentary.winnerNumber}番が競り勝ちました！`,
            `鼻差の接戦を制したのは${commentary.winnerNumber}番！実に見応えのあるレースでした！`,
            `最後まで目が離せない接戦のレースを制したのは${commentary.winnerNumber}番でした！`
          ];
        } else {
          summaryMessages = [
            `レースは${commentary.winnerNumber}番の勝利！安定した走りで他を寄せ付けませんでした！`,
            `${commentary.winnerNumber}番が他の追随を振り切って勝利を収めました！`,
            `終始リードを守り切った${commentary.winnerNumber}番の勝利で、レースは幕を閉じました！`
          ];
        }
        if (commentary.winnerStyle === "逃げ") {
          summaryMessages = summaryMessages.map(m => m + " 逃げ切り勝ちの見本のようなレースでした！");
        } else if (commentary.winnerStyle === "先行") {
          summaryMessages = summaryMessages.map(m => m + " 先行からの安定した競り合いが光りました！");
        } else if (commentary.winnerStyle === "差し") {
          summaryMessages = summaryMessages.map(m => m + " 絶妙のタイミングでの差し脚が決め手となりました！");
        } else if (commentary.winnerStyle === "追込") {
          summaryMessages = summaryMessages.map(m => m + " 最後の直線での迫力ある追い込みが光りました！");
        }
        let popularRacers = racers.slice().sort((a,b) => a.odds - b.odds);
        let popularityRank = popularRacers.findIndex(r => r.number === commentary.winnerNumber) + 1;
        if (popularityRank === 1) {
          summaryMessages = summaryMessages.map(m => m + " 本命馬が見事に応えた結果となりました！");
        } else if (popularityRank > 5) {
          summaryMessages = summaryMessages.map(m => m + " 波乱の結果となりました！");
        }
        commentary.addMessage(summaryMessages[Math.floor(Math.random()*summaryMessages.length)], currentTime, 5);
        commentary.raceSummaryAnnounced = true;
        return;
      }
      if (racers.some(r => r.finished && r.finishPosition === 1) && !commentary.raceFinished) {
        let winner = racers.find(r => r.finished && r.finishPosition === 1);
        let secondPlace = racers.find(r => r.finishPosition === 2);
        let margin = secondPlace ? secondPlace.margin : 0.0;
        commentary.setRaceFinished(winner.number, winner.runningStyle, winner.odds, margin);
        return;
      }
      if (commentary.raceFinished) return;
      let topRacers = sortedRacers.slice(0, 3);
      if (Math.random() < 0.1) {
        let availableRacers = racers.filter(r => !r.finished && !commentary.recentlyMentionedHorses.has(r.number));
        if (availableRacers.length === 0) {
          commentary.recentlyMentionedHorses.clear();
          availableRacers = racers.filter(r => !r.finished);
        }
        if (availableRacers.length > 0) {
          let target = availableRacers[Math.floor(Math.random()*availableRacers.length)];
          let targetRank = sortedRacers.findIndex(r => r.number === target.number) + 1;
          let messages = [
            `${target.number}番に注目です、現在${targetRank}番手！`,
            `${targetRank}位を走る${target.number}番の動きが目を引きます！`,
            `${target.number}番、${target.runningStyle}の脚質を活かして${targetRank}番手につけています！`
          ];
          commentary.recentlyMentionedHorses.add(target.number);
          commentary.addMessage(messages[Math.floor(Math.random()*messages.length)], currentTime, 2);
          return;
        }
      }
      let anySpurtJustUsed = racers.some(r => r.spurtUsed && !r.hasSpurtAnnounced);
      if ((currentTime - commentary.lastBackCommentaryTime > commentary.backCommentaryInterval) &&
          (!anySpurtJustUsed) && Math.random() < 0.5 && sortedRacers.length > 5) {
        let backGroup = sortedRacers.slice(Math.floor(sortedRacers.length/2));
        let midGroup = sortedRacers.slice(3, Math.floor(sortedRacers.length/2));
        if (backGroup.length >= 2 && Math.random() < 0.3) {
          let [horse1, horse2] = [backGroup[Math.floor(Math.random()*backGroup.length)], backGroup[Math.floor(Math.random()*backGroup.length)]];
          let pos1 = sortedRacers.findIndex(r => r.number === horse1.number) + 1;
          let pos2 = sortedRacers.findIndex(r => r.number === horse2.number) + 1;
          let messages = [
            `後方では${horse1.number}番と${horse2.number}番が${pos1}位争いを繰り広げています！`,
            `${pos1}位の${horse1.number}番と${pos2}位の${horse2.number}番が接戦中！`,
            `後方グループの${horse1.number}番と${horse2.number}番、互いに譲らない走りを見せています！`
          ];
          commentary.addMessage(messages[Math.floor(Math.random()*messages.length)], currentTime, 2);
          commentary.lastBackCommentaryTime = currentTime;
          return;
        }
        if (midGroup.length && Math.random() < 0.5) {
          let interestingMid = midGroup[Math.floor(Math.random()*midGroup.length)];
          let midPos = sortedRacers.findIndex(r => r.number === interestingMid.number) + 1;
          let midMessages = [
            `中団では${interestingMid.number}番が${midPos}番手につけています`,
            `${midPos}番手の${interestingMid.number}番、中団から上位を狙います`,
            `中団の${interestingMid.number}番が動きをみせています`
          ];
          if ((interestingMid.runningStyle === "差し" || interestingMid.runningStyle === "追込") && raceProgress > 0.5) {
            midMessages.push(`${interestingMid.number}番、${interestingMid.runningStyle}のここからが本領発揮です`);
            midMessages.push(`中団の${interestingMid.number}番、これから${interestingMid.runningStyle}を活かした追い上げが期待されます`);
          }
          commentary.addMessage(midMessages[Math.floor(Math.random()*midMessages.length)], currentTime, 2);
          commentary.lastBackCommentaryTime = currentTime;
          return;
        }
        if (backGroup.length) {
          let interestingBack = backGroup[Math.floor(Math.random()*backGroup.length)];
          let backPos = sortedRacers.findIndex(r => r.number === interestingBack.number) + 1;
          let backMessages = [
            `後方では${interestingBack.number}番が${backPos}番手につけています`,
            `遅れをとっている${interestingBack.number}番、ここからの挽回に期待`,
            `${backPos}番手の${interestingBack.number}番、後方からの追い上げを図ります`
          ];
          let popularRacers = racers.slice().sort((a,b) => a.odds - b.odds);
          let popularityRank = popularRacers.findIndex(r => r.number === interestingBack.number) + 1;
          if (popularityRank <= 3 && backPos > racers.length/2) {
            backMessages.push(`${popularityRank}番人気の${interestingBack.number}番が苦戦、${backPos}番手に沈んでいます`);
            backMessages.push(`人気馬の${interestingBack.number}番は出遅れて${backPos}位、巻き返しなるか`);
          }
          commentary.addMessage(backMessages[Math.floor(Math.random()*backMessages.length)], currentTime, 2);
          commentary.lastBackCommentaryTime = currentTime;
          return;
        }
      }
      if (raceProgress > 0.1 && Math.random() < 0.05) {
        let popularRacers = racers.filter(r => !r.finished).sort((a,b) => a.odds - b.odds);
        for (let popular of popularRacers.slice(0,3)) {
          if (!commentary.mentionedPopularHorses.has(popular.number) && !popular.finished) {
            let popularityRank = popularRacers.findIndex(r => r.number === popular.number) + 1;
            let currentRank = sortedRacers.findIndex(r => r.number === popular.number) + 1;
            let messages = [];
            if (popularityRank === 1) {
              if (currentRank <= 3) {
                messages = [
                  `1番人気の${popular.number}番、現在${currentRank}番手で好位置をキープ！`,
                  `本命の${popular.number}番は${currentRank}番手につけています！`
                ];
              } else {
                messages = [
                  `1番人気の${popular.number}番、やや出遅れて${currentRank}番手！`,
                  `本命${popular.number}番、現在${currentRank}位で出遅れています！`
                ];
              }
            } else if (popularityRank === 2) {
              if (currentRank <= 3) {
                messages = [
                  `対抗の${popular.number}番、${currentRank}番手で前目の位置！`,
                  `2番人気${popular.number}番も${currentRank}位で好調です！`
                ];
              } else {
                messages = [
                  `2番人気の${popular.number}番は${currentRank}番手と出遅れ気味！`,
                  `対抗馬の${popular.number}番、${currentRank}位からの巻き返しなるか！`
                ];
              }
            } else {
              if (currentRank <= 3) {
                messages = [
                  `3番人気${popular.number}番が${currentRank}番手に食い込んでいます！`,
                  `人気馬${popular.number}番、${currentRank}位と好位置につけました！`
                ];
              } else {
                messages = [
                  `3番人気の${popular.number}番は現在${currentRank}番手！`,
                  `${popular.number}番、人気に応えられるか注目です！`
                ];
              }
            }
            commentary.mentionedPopularHorses.add(popular.number);
            commentary.addMessage(messages[Math.floor(Math.random()*messages.length)], currentTime, 2);
            return;
          }
        }
      }
      if (raceProgress > 0.3 && raceProgress < 0.7 && Math.random() < 0.05) {
        let styleLeaders = {};
        for (let style of RUNNING_STYLES) {
          let styleHorses = racers.filter(r => r.runningStyle === style && !r.finished);
          if (styleHorses.length) {
            styleLeaders[style] = styleHorses.reduce((prev, curr) => (prev.distance > curr.distance) ? prev : curr);
          }
        }
        for (let style in styleLeaders) {
          let horse = styleLeaders[style];
          if (!commentary.mentionedRunningStyles.has(`${style}_${horse.number}`)) {
            let rank = sortedRacers.findIndex(r => r.number === horse.number) + 1;
            let messages = [];
            if (style === "逃げ") {
              messages = (rank === 1) ?
                [`逃げ馬の${horse.number}番、予想通り先頭で好ペース！`,
                 `${horse.number}番、逃げの脚質を活かして先頭を独走中！`] :
                [`逃げ脚質の${horse.number}番、予想に反して${rank}番手につけています`,
                 `${horse.number}番、逃げ切り作戦は不発か！現在${rank}位です`];
            } else if (style === "先行") {
              messages = (rank <= 3) ?
                [`先行馬の${horse.number}番、${rank}番手と好位置をキープ！`,
                 `${horse.number}番、先行脚質を活かして上位陣をマーク中！`] :
                [`先行脚質の${horse.number}番、出遅れて${rank}番手！`,
                 `${horse.number}番、先行からの作戦が思うように進まず${rank}位！`];
            } else if (style === "差し") {
              if (raceProgress < 0.5) {
                messages = [
                  `差し馬の${horse.number}番、ここからが本領発揮の予感！`,
                  `${horse.number}番、差し脚質らしく後方で脚を溜めています！`
                ];
              } else {
                messages = (rank <= 5) ?
                  [`差し馬の${horse.number}番、${rank}位まで上がってきました！`,
                   `${horse.number}番、差し脚を使って順位を上げてきています！`] :
                  [`差し馬の${horse.number}番、今回は伸びきれるか！`,
                   `${horse.number}番、まだ${rank}位、これからが勝負どころです！`];
              }
            } else if (style === "追込") {
              if (raceProgress < 0.6) {
                messages = [
                  `追込脚質の${horse.number}番、${rank}位からの大逃げ切りを狙います！`,
                  `${horse.number}番、後方から一気の追い込みを狙う作戦です！`
                ];
              } else {
                messages = (rank <= 3) ?
                  [`追込馬${horse.number}番、最後の直線で本領発揮！${rank}位まで上昇！`,
                   `${horse.number}番の追い込みが冴えています！現在${rank}位！`] :
                  [`追込の${horse.number}番、${rank}位からの巻き返しなるか！`,
                   `${horse.number}番、まだ脚を溜めています！ラストスパートに期待！`];
              }
            }
            commentary.mentionedRunningStyles.add(`${style}_${horse.number}`);
            commentary.addMessage(messages[Math.floor(Math.random()*messages.length)], currentTime, 2);
            return;
          }
        }
      }
      for (let racer of topRacers) {
        if (racer.position.isCorner) {
          let cornerNum = racer.position.cornerNum;
          if (cornerNum !== racer.lastCorner && racer.rank <= 3) {
            racer.lastCorner = cornerNum;
            if (!commentary.cornerPassed[cornerNum].has(racer.number)) {
              commentary.cornerPassed[cornerNum].add(racer.number);
              let messages = [];
              if (racer.rank === 1) {
                if (cornerNum === 1) messages = [`${racer.number}番が先頭で第1コーナーを回っています！`, `トップの${racer.number}番が1コーナーを通過！`, `1コーナー、${racer.number}番がリードしています！`];
                else if (cornerNum === 2) messages = [`${racer.number}番、先頭で2コーナーを駆け抜けます！`, `第2コーナー、${racer.number}番がそのままリード！`, `${racer.number}番、2コーナーも先頭で通過！`];
                else if (cornerNum === 3) messages = [`第3コーナー、${racer.number}番が先頭キープ！`, `${racer.number}番、3コーナーも落ちずに先頭！`, `3コーナー、${racer.number}番の独走態勢は変わらず！`];
                else messages = [`最後の第4コーナー、${racer.number}番がトップで回ってきます！`, `${racer.number}番、最終コーナーも先頭！このまま押し切るか！`, `4コーナー、${racer.number}番の先頭は揺るがず！`];
                commentary.addMessage(messages[Math.floor(Math.random()*messages.length)], currentTime, 3);
                return;
              } else if (racer.rank === 2 && Math.random() < 0.7) {
                let msgs = [`2番手の${racer.number}番も${cornerNum}コーナーに入ってきました`, `${racer.number}番が2番手で${cornerNum}コーナーを進んでいます`, `${cornerNum}コーナー、${racer.number}番が2番手で先頭を追走中！`];
                commentary.addMessage(msgs[Math.floor(Math.random()*msgs.length)], currentTime, 2);
                return;
              } else if (racer.rank === 3 && Math.random() < 0.5) {
                let msgs = [`${racer.number}番が3番手で${cornerNum}コーナーです`, `3番手は${racer.number}番、${cornerNum}コーナーを回っています`, `${cornerNum}コーナー、${racer.number}番が3番手で追い上げを狙います！`];
                commentary.addMessage(msgs[Math.floor(Math.random()*msgs.length)], currentTime, 1);
                return;
              }
            }
          }
        }
      }
      let currentLeaders = topRacers.map(r => r.number);
      if (commentary.lastLeaders.length && currentLeaders.length) {
        if (currentLeaders[0] !== commentary.lastLeaders[0]) {
          let messages = [];
          if (raceProgress < 0.3) {
            messages = [`${currentLeaders[0]}番が序盤で先頭に立ちました！`, `トップが交代！${currentLeaders[0]}番が先頭に躍り出ました！`, `${currentLeaders[0]}番が${commentary.lastLeaders[0]}番を抜いて先頭です！`];
          } else if (raceProgress < 0.6) {
            messages = [`中盤戦、${currentLeaders[0]}番が先頭に立ちました！`, `${currentLeaders[0]}番が${commentary.lastLeaders[0]}番を抜いて新たなトップ！`, `ここで先頭交代！${currentLeaders[0]}番が${commentary.lastLeaders[0]}番をかわしました！`];
          } else if (raceProgress < 0.8) {
            messages = [`終盤に差し掛かり、${currentLeaders[0]}番が${commentary.lastLeaders[0]}番を抜いて先頭！`, `${currentLeaders[0]}番、終盤で${commentary.lastLeaders[0]}番を交わして先頭に立ちました！`, `ここで大きな動き！${currentLeaders[0]}番が先頭に躍り出ました！`];
          } else {
            messages = [`ラストスパート！${currentLeaders[0]}番が${commentary.lastLeaders[0]}番を抜いて先頭に立ちました！`, `最後の直線で${currentLeaders[0]}番が抜け出しました！`, `ゴール前で${currentLeaders[0]}番が${commentary.lastLeaders[0]}番を交わして先頭！`];
          }
          commentary.addMessage(messages[Math.floor(Math.random()*messages.length)], currentTime, 3);
          commentary.lastLeaders = currentLeaders.slice();
          return;
        } else if (currentLeaders.length > 1 && commentary.lastLeaders.length > 1 && Math.random() < 0.5) {
          if (currentLeaders[1] !== commentary.lastLeaders[1]) {
            let messages = (raceProgress < 0.5) ?
              [`${currentLeaders[1]}番が2番手に浮上！`, `2番手が${currentLeaders[1]}番に入れ替わりました`] :
              [`${currentLeaders[1]}番が${commentary.lastLeaders[1]}番を抜いて2番手に！`, `2番手争い、${currentLeaders[1]}番が前に出ました！`];
            commentary.addMessage(messages[Math.floor(Math.random()*messages.length)], currentTime, 2);
            commentary.lastLeaders = currentLeaders.slice();
            return;
          }
        }
      }
      commentary.lastLeaders = currentLeaders.slice();
      for (let racer of racers) {
        if (racer.spurtUsed && !racer.hasSpurtAnnounced) {
          racer.hasSpurtAnnounced = true;
          if (currentTime - commentary.lastSpurtTime > commentary.spurtCooldown) {
            commentary.lastSpurtTime = currentTime;
            let messages = [];
            if (racer.rank === 1) {
              if (racer.runningStyle === "逃げ") {
                messages = [`先頭の${racer.number}番、余力十分でスパート！`, `${racer.number}番、逃げ切る気満々で加速しました！`, `トップの${racer.number}番、ここからさらにペースを上げてきました！`];
              } else {
                messages = [`トップの${racer.number}番がスパートをかけました！`, `先頭${racer.number}番、ここからさらに加速します！`, `${racer.number}番、先頭でさらに脚を伸ばしています！`];
              }
            } else if (racer.rank <= 3) {
              if (racer.runningStyle === "差し") {
                messages = [`${racer.number}番が差し脚を使って伸びてきました！`, `${racer.rank}番手の${racer.number}番、ここから差し切りを狙います！`, `${racer.number}番の差し脚が炸裂！${racer.rank}番手から一気に加速！`];
              } else if (racer.runningStyle === "追込") {
                messages = [`${racer.number}番、追い込みの脚を使いました！`, `後方から${racer.number}番が追い込みをかけてきました！`, `${racer.number}番、ここぞとばかりに追い込み開始！`];
              } else {
                messages = [`${racer.rank}番手${racer.number}番がスパートしました！`, `${racer.number}番、${racer.rank}番手から加速してきました！`, `${racer.number}番、リズムを上げて前を追います！`];
              }
            } else {
              if (racer.runningStyle === "差し" || racer.runningStyle === "追込") {
                messages = [`後方の${racer.number}番、ようやく本領発揮の動き！`, `${racer.number}番、${racer.rank}位から追い上げを開始！`, `出遅れた${racer.number}番、ここからの巻き返しなるか！`];
              } else {
                messages = [`${racer.number}番もスパートをかけてきました！`, `${racer.number}番が追い上げを開始！`, `${racer.rank}位の${racer.number}番も動き出しました！`];
              }
            }
            commentary.addMessage(messages[Math.floor(Math.random()*messages.length)], currentTime, 3);
            return;
          }
        }
      }
      for (let racer of racers) {
        if (racer.finished && !commentary.finishAnnounced.has(racer.number)) {
          commentary.finishAnnounced.add(racer.number);
          if (racer.finishPosition === 1) {
            let secondPlace = racers.find(r => r.finishPosition === 2);
            let margin = secondPlace ? secondPlace.margin : 0.0;
            commentary.setRaceFinished(racer.number, racer.runningStyle, racer.odds, margin);
            let messages = [];
            if (margin > 3.0) {
              messages = [`${racer.number}番、圧倒的な強さでゴールイン！大差の優勝です！`, `完全独走！${racer.number}番が他を寄せ付けず1着でフィニッシュ！`, `勝ったのは${racer.number}番！圧巻の競り合いなしの完勝です！`];
            } else if (margin < 0.5) {
              messages = [`${racer.number}番、接戦を制してゴールイン！僅差の優勝です！`, `大接戦を制したのは${racer.number}番！鼻差の1着でフィニッシュ！`, `勝ったのは${racer.number}番！本当に紙一重の競り合いを制しました！`];
            } else {
              messages = [`${racer.number}番、ゴールイン！優勝です！`, `${racer.number}番が1着でフィニッシュしました！`, `勝ったのは${racer.number}番！見事なレースでした！`];
            }
            commentary.addMessage(messages[Math.floor(Math.random()*messages.length)], currentTime, 4);
            return;
          } else if (racer.finishPosition === 2) {
            let msgs = [`2着は${racer.number}番！`, `${racer.number}番が2着でゴールしました！`, `${racer.number}番、惜しくも2着でのフィニッシュです！`];
            commentary.addMessage(msgs[Math.floor(Math.random()*msgs.length)], currentTime, 3);
            return;
          } else if (racer.finishPosition === 3) {
            let msgs = [`3着は${racer.number}番！`, `${racer.number}番が3着に入りました！`, `3着でフィニッシュしたのは${racer.number}番です！`];
            commentary.addMessage(msgs[Math.floor(Math.random()*msgs.length)], currentTime, 3);
            return;
          } else if (racer.finishPosition <= 6 && Math.random() < 0.3) {
            let msgs = [`${racer.number}番、${racer.finishPosition}着でゴールです`, `${racer.finishPosition}着は${racer.number}番！`];
            commentary.addMessage(msgs[Math.floor(Math.random()*msgs.length)], currentTime, 1);
            return;
          }
        }
      }
      if (commentary.checkSilence(currentTime, raceProgress)) {
        let commentaryChance = 0.1;
        if (Math.random() < commentaryChance) {
          let phase = "";
          let generalMessages = [];
          if (raceProgress < 0.3) {
            phase = "序盤";
            if (!commentary.racePhaseComments[phase]) {
              commentary.racePhaseComments[phase] = true;
              generalMessages = [
                `レースは序盤、${topRacers[0].number}番が先頭です！`,
                `各馬、序盤のポジション取りに動いています！`,
                `トップは${topRacers[0].number}番、好スタートを切りました！`,
                `序盤から${topRacers[0].number}番がリードしています！`,
                `${topRacers[0].number}番を先頭に各馬が位置取りを競っています！`
              ];
            }
          } else if (raceProgress < 0.6) {
            phase = "中盤";
            if (!commentary.racePhaseComments[phase]) {
              commentary.racePhaseComments[phase] = true;
              generalMessages = [
                "レースは中盤に入りました！",
                `${topRacers[0].number}番を先頭に、各馬が位置取りを競っています！`,
                `中盤戦、${topRacers[0].number}番がリードを保っています！`,
                "各馬、持久力を温存しながらの中盤戦が続いています！",
                `先頭${topRacers[0].number}番の後ろで各馬が動きを窺っています！`
              ];
            }
          } else if (raceProgress < 0.8) {
            phase = "終盤";
            if (!commentary.racePhaseComments[phase]) {
              commentary.racePhaseComments[phase] = true;
              generalMessages = [
                "レースは終盤に差し掛かりました！",
                `${topRacers[0].number}番を先頭に各馬が追い上げを図ります！`,
                "最後の直線が近づいてきました！",
                `終盤、${topRacers[0].number}番がリードする展開！`,
                "各馬、位置取りが終わり、これからが勝負の終盤です！"
              ];
            }
          } else {
            phase = "ラスト";
            if (!commentary.racePhaseComments[phase]) {
              commentary.racePhaseComments[phase] = true;
              generalMessages = [
                "ラストスパートの時間です！",
                "ゴールまであと少し！各馬の競り合いが激しくなっています！",
                `最後の直線、${topRacers[0].number}番がリードしています！`,
                "フィニッシュに向けて熾烈な競り合いが始まりました！",
                `ラストスパート！${topRacers[0].number}番を中心とした熱い戦いが続いています！`
              ];
            }
          }
          if (phase && !commentary.racePhaseComments[phase]) {
            let priority =  (commentary.checkSilence(currentTime, raceProgress)) ? 2 : 1;
            commentary.addMessage(generalMessages[Math.floor(Math.random()*generalMessages.length)], currentTime, priority);
            return;
          }
          if (commentary.checkSilence(currentTime, raceProgress)) {
            let msgs = [
              `現在のトップは${topRacers[0].number}番です！`,
              `${topRacers[0].number}番がリードする展開が続いています！`,
              `2番手には${(sortedRacers[1] ? sortedRacers[1].number : "-")}番がつけています！`,
              `現在${Math.floor(raceProgress*100)}%地点、${topRacers[0].number}番が先頭です！`,
              "各馬の位置取りが続いています！"
            ];
            commentary.addMessage(msgs[Math.floor(Math.random()*msgs.length)], currentTime, 1);
            return;
          }
        }
      }
    }

    // 描画関数：エントリー表（出走表）の描画
    function drawEntryTable(ctx, racers, course, fonts) {
      let distanceType = getDistanceType(course.length);
      let titleStr = `${course.name} ${course.surface}コース ${distanceType}  出走表`;
      ctx.font = fonts.large;
      ctx.fillStyle = "#FFFFFF";
      let titleWidth = ctx.measureText(titleStr).width;
      ctx.fillText(titleStr, (WIDTH - titleWidth)/2, 20);
      let tableY = 80, tableHeight = 480, tableWidth = 740, tableX = (WIDTH - tableWidth)/2;
      ctx.fillStyle = "rgb(30,30,30)";
      ctx.fillRect(tableX, tableY, tableWidth, tableHeight);
      ctx.fillStyle = "rgb(50,50,50)";
      ctx.fillRect(tableX+2, tableY+2, tableWidth-4, tableHeight-4);
      let headerY = tableY + 10;
      let headers = ["番号", "名前", "脚質", "オッズ", "スピード", "スタミナ", "パワー", "コーナー", "馬場適性", "距離適性"];
      let headerPositions = [
        tableX+20,
        tableX+70,
        tableX+170,
        tableX+230,
        tableX+290,
        tableX+360,
        tableX+430,
        tableX+500,
        tableX+570,
        tableX+650
      ];
      ctx.font = fonts.small;
      for (let i = 0; i < headers.length; i++) {
        ctx.fillStyle = "rgb(220,220,220)";
        ctx.fillText(headers[i], headerPositions[i], headerY);
      }
      let rowHeight = 35;
      for (let i = 0; i < racers.length; i++) {
        let rowY = tableY + 40 + i * rowHeight;
        ctx.fillStyle = (i % 2 === 0) ? "rgb(40,40,40)" : "rgb(35,35,35)";
        ctx.fillRect(tableX+5, rowY, tableWidth-10, rowHeight);
        ctx.fillStyle = racers[i].color;
        ctx.fillText(racers[i].number, headerPositions[0], rowY+20);
        ctx.fillStyle = "#FFFFFF";
        ctx.fillText(racers[i].name, headerPositions[1], rowY+20);
        ctx.fillStyle = getStyleColor(racers[i].runningStyle);
        ctx.fillText(racers[i].runningStyle, headerPositions[2], rowY+20);
        ctx.fillStyle = "rgb(255,220,0)";
        ctx.fillText(racers[i].odds.toFixed(1), headerPositions[3], rowY+20);
        // 各種ステータスは簡易バーで表示
        drawStatBar(ctx, headerPositions[4], rowY+15, racers[i].baseSpeed, 1.0, 1.4);
        drawStatBar(ctx, headerPositions[5], rowY+15, racers[i].stamina, 0.9, 1.1);
        drawStatBar(ctx, headerPositions[6], rowY+15, racers[i].power, 0.8, 1.2);
        drawStatBar(ctx, headerPositions[7], rowY+15, racers[i].cornerSkill, 0.85, 1.15);
        let surface = course.surface;
        drawStatBar(ctx, headerPositions[8], rowY+15, racers[i].surfaceAptitude[surface], 0.9, 1.1);
        ctx.font = fonts.tiny;
        ctx.fillStyle = "rgb(200,200,200)";
        ctx.fillText(surface, headerPositions[8], rowY+10);
        ctx.fillText(distanceType, headerPositions[9], rowY+10);
        drawStatBar(ctx, headerPositions[9], rowY+15, racers[i].distanceAptitude[distanceType], 0.9, 1.1);
      }
    }

    function drawStatBar(ctx, x, y, value, minVal, maxVal) {
      let barWidth = 55, barHeight = 10;
      let norm = (value - minVal) / (maxVal - minVal);
      norm = Math.max(0, Math.min(1, norm));
      ctx.fillStyle = "rgb(70,70,70)";
      ctx.fillRect(x, y, barWidth, barHeight);
      let barColor;
      if (norm < 0.3) barColor = "rgb(200,50,50)";
      else if (norm < 0.7) barColor = "rgb(220,220,50)";
      else barColor = "rgb(50,200,50)";
      ctx.fillStyle = barColor;
      ctx.fillRect(x, y, barWidth * norm, barHeight);
    }

    function getStyleColor(style) {
      if (style === "逃げ") return "rgb(255,100,100)";
      if (style === "先行") return "rgb(255,200,100)";
      if (style === "差し") return "rgb(100,200,255)";
      if (style === "追込") return "rgb(200,100,255)";
      return "#FFFFFF";
    }

    function drawRaceResult(ctx, racers, course, fonts, elapsed) {
      let maxHeight = 560;
      let growthTime = 1.0;
      let panelHeight = Math.min(maxHeight, Math.floor(maxHeight * (elapsed / growthTime)));
      let panelWidth = 650;
      let panelX = Math.floor((WIDTH - panelWidth) / 2);
      let panelY = Math.floor((HEIGHT - panelHeight) / 2);
      ctx.fillStyle = "rgba(0,0,0,0.9)";
      ctx.fillRect(panelX, panelY, panelWidth, panelHeight);
      ctx.strokeStyle = "rgb(200,180,50)";
      ctx.lineWidth = 2;
      ctx.strokeRect(panelX, panelY, panelWidth, panelHeight);
      ctx.font = fonts.large;
      ctx.fillStyle = "rgb(255,220,0)";
      ctx.fillText("レース結果", panelX + panelWidth/2 - 70, panelY + 40);
      ctx.font = fonts.small;
      ctx.fillStyle = "rgb(220,220,220)";
      ctx.fillText(`${course.name} (${course.surface}・${getDistanceType(course.length)})`, panelX+20, panelY+70);
      let headers = ["着順","馬名","脚質","タイム","着差","オッズ"];
      let headerPositions = [panelX+30, panelX+90, panelX+190, panelX+290, panelX+370, panelX+470];
      let headerY = panelY + 100;
      for (let i = 0; i < headers.length; i++) {
        ctx.fillText(headers[i], headerPositions[i], headerY);
      }
      ctx.strokeStyle = "rgb(100,100,100)";
      ctx.beginPath();
      ctx.moveTo(panelX+10, headerY+10);
      ctx.lineTo(panelX+panelWidth-10, headerY+10);
      ctx.stroke();
      racers.sort((a,b) => a.finishPosition - b.finishPosition);
      for (let i = 0; i < racers.length; i++) {
        if (elapsed < 0.5 + i * 0.1) continue;
        let rowY = headerY + 40 + i * 35;
        if (rowY > panelY + panelHeight - 30) break;
        let posColors = ["rgb(255,215,0)","rgb(192,192,192)","rgb(205,127,50)","rgb(255,255,255)"];
        ctx.fillStyle = posColors[Math.min(racers[i].finishPosition-1,3)];
        ctx.fillText(racers[i].finishPosition, headerPositions[0], rowY);
        ctx.fillStyle = racers[i].color;
        ctx.fillText(`${racers[i].number}. ${racers[i].name}`, headerPositions[1], rowY);
        ctx.fillStyle = getStyleColor(racers[i].runningStyle);
        ctx.fillText(racers[i].runningStyle, headerPositions[2], rowY);
        ctx.fillStyle = "#FFFFFF";
        ctx.fillText(`${racers[i].finishTime.toFixed(2)}秒`, headerPositions[3], rowY);
        ctx.fillText((racers[i].finishPosition===1)?"-":`${racers[i].margin.toFixed(1)}馬身`, headerPositions[4], rowY);
        ctx.fillStyle = "rgb(255,220,0)";
        ctx.fillText(racers[i].odds.toFixed(1), headerPositions[5], rowY);
      }
    }

    function drawCommentary(ctx, commentary, currentTime, fonts) {
      let activeMessages = commentary.getActiveMessages(currentTime);
      if (!activeMessages.length) return;
      let panelHeight = Math.min(40 * activeMessages.length, 200);
      let panelWidth = 720;
      let panelX = Math.floor((WIDTH - panelWidth) / 2);
      let panelY = HEIGHT - panelHeight - 20;
      ctx.fillStyle = "rgba(0,0,0,0.7)";
      ctx.fillRect(panelX, panelY, panelWidth, panelHeight);
      ctx.fillStyle = "#FFFFFF";
      ctx.font = fonts.small;
      for (let i = 0; i < activeMessages.length; i++) {
        ctx.fillText(activeMessages[i], panelX + 10, panelY + 30 + i * 35);
      }
    }

    // ゲーム状態管理
    const GAME_STATE_ENTRY = 0, GAME_STATE_COUNTDOWN = 1, GAME_STATE_RACING = 2, GAME_STATE_RESULT = 3;
    let gameState = GAME_STATE_ENTRY;

    // フォント定義
    const fonts = {
      large: "36px sans-serif",
      small: "18px sans-serif",
      tiny: "14px sans-serif"
    };

    // レースセットアップ関数
    function setupRace() {
      let course = COURSES[Math.floor(Math.random()*COURSES.length)];
      let track = new Track();
      track.setupForCourse(course);
      let racers = [];
      for (let i = 0; i < NUM_RACERS; i++) {
        let name = `${i+1}番`;
        let lane = i % track.lanes;
        let r = new Racer(name, lane, RACER_COLORS[i], course, i+1, track);
        r.initializeRace(track);
        racers.push(r);
      }
      for (let racer of racers) {
        racer.calculateOdds(racers);
      }
      return { track, racers, course };
    }

    let { track, racers, course } = setupRace();
    let commentary = new Commentary();

    // 状態管理用変数
    let raceStarted = false, raceFinished = false, finishedCount = 0;
    let raceStartTime = 0, countdown = 3, stateChangeTime = getTime();
    let anySpurtUsed = false, leaderFinished = false;
    let goalCameraX = 0, goalCameraY = 0, resultScale = 1.5;
    let lastCommentaryGeneration = 0;
    let commentaryGenerationInterval = 1.0;

    // キー操作
    document.addEventListener("keydown", function(e) {
      if (e.code === "Space") {
        if (gameState === GAME_STATE_ENTRY) {
          gameState = GAME_STATE_COUNTDOWN;
          countdown = 3;
          raceStartTime = getTime();
          commentary.addMessage("スタートの準備！", getTime(), 3);
        } else if (gameState === GAME_STATE_RESULT) {
          let setup = setupRace();
          track = setup.track; racers = setup.racers; course = setup.course;
          commentary = new Commentary();
          gameState = GAME_STATE_ENTRY;
          raceStarted = false; raceFinished = false; finishedCount = 0;
          anySpurtUsed = false; leaderFinished = false;
        }
      }
      if (e.code === "KeyR" && gameState === GAME_STATE_RESULT) {
        let setup = setupRace();
        track = setup.track; racers = setup.racers; course = setup.course;
        commentary = new Commentary();
        gameState = GAME_STATE_ENTRY;
        raceStarted = false; raceFinished = false; finishedCount = 0;
        anySpurtUsed = false; leaderFinished = false;
      }
    });

    let lastTime = getTime();

    // メインゲームループ
    function gameLoop() {
      let currentTime = getTime();
      let dt = currentTime - lastTime;
      lastTime = currentTime;
      ctx.clearRect(0, 0, WIDTH, HEIGHT);
      if (gameState === GAME_STATE_ENTRY) {
        // 出走表のみを描画（背景は任意）
        drawEntryTable(ctx, racers, course, fonts);
        ctx.fillStyle = "#FFFFFF";
        ctx.font = fonts.small;
        ctx.fillText("スペースキーを押してレースを開始", WIDTH/2 - 150, HEIGHT - 50);
        // ※ここでは track.draw や馬の描画は行わず、出走表がそのまま表示されるようにする
      } else if (gameState === GAME_STATE_COUNTDOWN) {
        track.draw(ctx, course);
        ctx.fillStyle = "#FFFFFF";
        ctx.font = fonts.large;
        ctx.fillText(`${course.name} (${course.surface}・${getDistanceType(course.length)})`, 20, 40);
        ctx.font = fonts.small;
        ctx.fillText(`特徴: ${course.feature}`, 20, 80);
        racers.forEach(r => { r.position = track.getPosition(r.distance, r.lane); r.draw(ctx); });
        let elapsed = currentTime - raceStartTime;
        if (elapsed > 1) {
          countdown--;
          raceStartTime = currentTime;
        }
        if (countdown > 0) {
          ctx.fillStyle = "#FFFFFF";
          ctx.font = "50px sans-serif";
          ctx.fillText(countdown, WIDTH/2 - 10, HEIGHT/2);
        } else if (countdown === 0) {
          ctx.fillStyle = "#FFFFFF";
          ctx.font = "50px sans-serif";
          ctx.fillText("スタート！", WIDTH/2 - 60, HEIGHT/2);
          if (elapsed > 1) {
            gameState = GAME_STATE_RACING;
            raceStarted = true;
            raceStartTime = currentTime;
            let msgs = ["レース開始！", "各馬、一斉スタート！", "ゲートが開きました！"];
            commentary.addMessage(msgs[Math.floor(Math.random()*msgs.length)], currentTime, 3);
          }
        }
      } else if (gameState === GAME_STATE_RACING) {
        let leader;
        if (leaderFinished) {
          cameraX = goalCameraX;
          cameraY = goalCameraY;
          let elapsedSinceFinish = currentTime - leaderFinishTime;
          let scaleReduction = Math.min(0.5, elapsedSinceFinish * 0.2);
          scale = Math.max(1.0, resultScale - scaleReduction);
        } else {
          leader = racers.reduce((prev, curr) => (prev.distance > curr.distance ? prev : curr));
          var cameraX = leader.position.x;
          var cameraY = leader.position.y;
          let baseScale = 1.5;
          if (anySpurtUsed) {
            let totalDistance = track.raceGameDistance;
            let currentDistance = leader.distance - leader.initialDistance;
            let raceProgress = Math.min(1.0, currentDistance / totalDistance);
            let zoomProgress = (raceProgress > 0.7) ? Math.min(1.0, (raceProgress - 0.7) / 0.3) : 0;
            var scale = baseScale + zoomProgress * 0.7;
          } else {
            var scale = baseScale;
          }
        }
        track.drawCamera(ctx, course, cameraX, cameraY, scale);
        racers.forEach(r => r.drawCamera(ctx, cameraX, cameraY, scale));
        leader = racers.reduce((prev, curr) => (prev.distance > curr.distance ? prev : curr));
        let currentDistance = leader.distance - leader.initialDistance;
        let totalDistance = track.raceGameDistance;
        let prog = Math.min(100, Math.floor((currentDistance / totalDistance) * 100));
        if (raceFinished) prog = 100;
        ctx.fillStyle = "#FFFFFF";
        ctx.font = fonts.large;
        ctx.fillText(`レース進行: ${prog}%`, WIDTH - 350, 40);
        let metersRun = Math.min(Math.floor(currentDistance * track.getTrackScale()), course.length);
        ctx.fillText(`1位走行距離: ${metersRun}m`, WIDTH - 350, 80);
        let sortedRacers = racers.slice().sort((a,b) => b.distance - a.distance);
        for (let i = 0; i < Math.min(5, sortedRacers.length); i++) {
          ctx.fillStyle = "#FFFFFF";
          ctx.font = fonts.small;
          ctx.fillText(`${i+1}位: ${sortedRacers[i].number}.${sortedRacers[i].name}`, WIDTH - 250, HEIGHT - 180 + i * 25);
        }
        if (currentTime - lastCommentaryGeneration > commentaryGenerationInterval) {
          lastCommentaryGeneration = currentTime;
          getCommentaryText(racers, track, course, currentTime, commentary);
        }
        if (currentTime - commentary.lastQueueProcessTime > commentary.queueProcessInterval) {
          commentary.lastQueueProcessTime = currentTime;
          commentary.processQueue(currentTime);
        }
        drawCommentary(ctx, commentary, currentTime, fonts);
        racers.forEach(r => {
          let newSpurt = r.update(track, racers);
          if (newSpurt) anySpurtUsed = true;
          if (r.finished && !r.finishPosition) {
            finishedCount++;
            r.finishPosition = finishedCount;
            r.finishTime = currentTime - raceStartTime;
            if (finishedCount === 1) {
              leaderFinished = true;
              leaderFinishTime = currentTime;
              goalCameraX = track.goalPosition.x;
              goalCameraY = track.goalPosition.y;
              resultScale = scale;
              commentary.clearAllMessages();
            }
            if (finishedCount > 1) {
              let first_ = racers.find(x => x.finishPosition === 1);
              r.margin = (r.finishTime - first_.finishTime) * 5;
            } else {
              r.margin = 0.0;
            }
            if (finishedCount >= NUM_RACERS) {
              raceFinished = true;
              gameState = GAME_STATE_RESULT;
              stateChangeTime = currentTime;
            }
          }
        });
      } else if (gameState === GAME_STATE_RESULT) {
        drawRaceResult(ctx, racers, course, fonts, currentTime - stateChangeTime);
        if (currentTime - stateChangeTime > 2.0) {
          ctx.fillStyle = "#FFFFFF";
          ctx.font = fonts.small;
          ctx.fillText("スペースキーまたは R キーで新しいレースをスタート", WIDTH/2 - 220, HEIGHT - 50);
        }
      }
      requestAnimationFrame(gameLoop);
    }

    requestAnimationFrame(gameLoop);
  </script>
</body>
</html>
