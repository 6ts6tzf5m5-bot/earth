<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="utf-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, minimum-scale=1, user-scalable=no">
  <title>Ultimate Earth Digital Twin</title>
  <script src="https://cesium.com/downloads/cesiumjs/releases/1.108/Build/Cesium/Cesium.js"></script>
  <link href="https://cesium.com/downloads/cesiumjs/releases/1.108/Build/Cesium/Widgets/widgets.css" rel="stylesheet">
  <style>
    html, body, #cesiumContainer { width: 100%; height: 100%; margin: 0; padding: 0; overflow: hidden; background: #000; }
    #hud { 
      position: absolute; top: 10px; left: 10px; right: 10px;
      background: rgba(0, 15, 30, 0.7); color: #00eeff; 
      padding: 15px; border-radius: 12px; font-family: 'Segoe UI', sans-serif; 
      border: 1px solid rgba(0, 238, 255, 0.3); font-size: 11px; z-index: 10;
      backdrop-filter: blur(10px); text-shadow: 0 0 5px #00eeff;
    }
    #location-btn { 
      position: absolute; bottom: 30px; right: 20px; 
      width: 50px; height: 50px; background: #00eeff; 
      border: none; border-radius: 50%; font-size: 24px; z-index: 20;
      box-shadow: 0 0 20px rgba(0, 238, 255, 0.5);
    }
  </style>
</head>
<body>
  <div id="cesiumContainer"></div>
  <div id="hud">
    <b style="font-size: 14px;">EARTH DIGITAL TWIN v5.0</b><br>
    <span id="stats">TRAFFIC: SYNCING...</span> | <span id="env">ENV: CLOUDS ACTIVE</span>
  </div>
  <button id="location-btn" onclick="locateMe()">📍</button>

  <script>
    // 1. 基本設定（高精細地形 & 航空写真）
    const viewer = new Cesium.Viewer('cesiumContainer', {
      terrainProvider: Cesium.createWorldTerrain({ requestVertexNormals: true, requestWaterMask: true }),
      imageryProvider: new Cesium.BingMapsImageryProvider({
        url: 'https://dev.virtualearth.net',
        mapStyle: Cesium.BingMapsStyle.AERIAL_WITH_LABELS
      }),
      baseLayerPicker: false, homeButton: false, sceneModePicker: false,
      navigationHelpButton: false, shouldAnimate: true, animation: false, timeline: false
    });

    // 3Dビルと環境設定
    viewer.scene.primitives.add(Cesium.createOsmBuildings());
    viewer.scene.globe.enableLighting = true; // 昼夜の切り替え
    viewer.scene.skyAtmosphere.show = true;   // 大気層（オーロラ効果の下地）
    viewer.scene.fog.enabled = true;

    // --- 【機能1】リアルタイム雲レイヤーのシミュレート ---
    // ※ 実際のAPIキーがない場合でも動くよう、透明度のある雲テクスチャを地球全体に重ねます
    const cloudLayer = viewer.imageryLayers.addImageryProvider(
      new Cesium.SingleTileImageryProvider({
        url: 'https://cesium.com/downloads/cesiumjs/releases/1.108/Build/Cesium/Assets/Textures/cloudCover.jpg', // ダミー雲データ
        rectangle: Cesium.Rectangle.MAX_VALUE
      })
    );
    cloudLayer.alpha = 0.4; // 雲の透け感

    // --- 【機能4】環境データ・オーバーレイ（オーロラ風大気） ---
    // 極地方に環境異常を示すオーロラエフェクトを配置
    viewer.entities.add({
      name: 'Atmospheric Anomaly',
      corridor: {
        positions: Cesium.Cartesian3.fromDegreesArray([
          -180, 70, 0, 70, 180, 70
        ]),
        width: 400000,
        material: new Cesium.ColorMaterialProperty(new Cesium.CallbackProperty(() => {
          const t = performance.now() / 2000;
          return Cesium.Color.fromHsl(0.4, 1.0, 0.5, 0.2 + 0.1 * Math.sin(t)); // 緑色のゆらぎ
        }, false))
      }
    });

    // --- 既存機能の統合（ISS, 航空機, 航跡） ---
    function createTrackedEntity(id, color, trailTime) {
      return viewer.entities.add({
        id: id,
        position: new Cesium.SampledPositionProperty(),
        point: { pixelSize: 4, color: color },
        path: { width: 2, material: new Cesium.PolylineGlowMaterialProperty({ color: color, glowPower: 0.1 }), trailTime: trailTime }
      });
    }

    const iss = createTrackedEntity('iss-sat', Cesium.Color.LIME, 600);
    const aircraftEntities = {};

    async function updateFlights() {
      try {
        const res = await fetch('https://opensky-network.org/api/states/all');
        const data = await res.json();
        const time = viewer.clock.currentTime;
        if (!data.states) return;
        
        // 全機表示（制限なし）
        data.states.forEach(f => {
          const id = "air_" + f[0];
          if (!aircraftEntities[id]) {
            aircraftEntities[id] = createTrackedEntity(id, Cesium.Color.ORANGE, 300);
          }
          if (f[5] && f[6]) {
            aircraftEntities[id].position.addSample(time, Cesium.Cartesian3.fromDegrees(f[5], f[6], f[7] || 9000));
          }
        });
        document.getElementById('stats').innerText = `AIRCRAFT: ${data.states.length}`;
      } catch (e) {}
    }

    // --- 衛星のスワス（地上投影）機能 ---
    viewer.entities.add({
      rectangle: {
        coordinates: new Cesium.CallbackProperty(() => {
          const pos = iss.position.getValue(viewer.clock.currentTime);
          if (!pos) return Cesium.Rectangle.fromDegrees(0,0,1,1);
          const cart = Cesium.Cartographic.fromCartesian(pos);
          const lon = Cesium.Math.toDegrees(cart.longitude);
          const lat = Cesium.Math.toDegrees(cart.latitude);
          return Cesium.Rectangle.fromDegrees(lon-1, lat-0.5, lon+1, lat+0.5);
        }, false),
        material: Cesium.Color.LIME.withAlpha(0.1),
        outline: true, outlineColor: Cesium.Color.LIME
      }
    });

    function locateMe() {
      navigator.geolocation.getCurrentPosition(pos => {
        viewer.camera.flyTo({
          destination: Cesium.Cartesian3.fromDegrees(pos.coords.longitude, pos.coords.latitude, 1200),
          orientation: { pitch: Cesium.Math.toRadians(-35) }
        });
      });
    }

    let tick = 0;
    viewer.clock.onTick.addEventListener(() => {
      tick += 0.005;
      iss.position.addSample(viewer.clock.currentTime, Cesium.Cartesian3.fromDegrees(tick * 50 % 360 - 180, 51 * Math.sin(tick), 400000));
    });

    setInterval(updateFlights, 30000);
    updateFlights();
    locateMe();
  </script>
</body>
</html>
