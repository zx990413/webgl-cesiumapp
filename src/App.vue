<template>
  <div id="cesiumContainer"></div>
</template>
<script setup>
import * as Cesium from "cesium";

// 使用功能钩子函数
import { onMounted } from "vue";
// 使用函数
onMounted(() => {
  // 创建cesium查看器
  // const viewer = new Cesium.Viewer('cesiumContainer');

  // 使用指定的地址
  var custom = new Cesium.ArcGisMapServerImageryProvider({
    // url:'//service.arcgisonline.com/ArcGIS/rest/service/World_Street_Map/MapServer'
    url: "https://services.arcgisonline.com/ArcGIS/rest/services/World_Imagery/MapServer",
  });

  // 设置地图属性
  const viewer = new Cesium.Viewer("cesiumContainer", {
    // 隐藏图标
    baseLayerPicker: false,
    // 使用自定义图像
    imageryProvider: custom,
    // 让图像能有立体感
    terrainProvider: Cesium.createWorldTerrain({
      // 请求水面的效果
      requestWaterMask: true,
      // 地图上的线
      requestVertexNormals: true,
    }),
  });

  // 设置摄像头视图
  // viewer.camera.setView(
  //   {
  //       //设置目的地
  //       //设置金纬度和高度: Cesium.Cartesian3.fromDegrees(经度,纬度,高度)
  //   destination:Cesium.Cartesian3.fromDegrees(113.318977,23.114155,2000),
  //   // 设置方向 俯视和仰视
  //   orientation:{
  //     // 设置旋转度数
  //     heading:Cesium.Math.toRadians(90),
  //     // 设置俯仰角度
  //     pitch:Cesium.Math.toRadians(-45)
  //   }
  //   }
  // )

  viewer.camera.setView({
    // 传入三维坐标
    destination: new Cesium.Cartesian3(1332761, -4662399, 4137888),
    // 设置方向
    orientation: {
      heading: 0.60,
      // 设置俯仰角
      pitch: -0.66,
    },
  });
  // 获取3D建筑模型
  // ({url:Cesium.IonResource.fromAssetId(75343)})) : 官方给出的建模
  var city = viewer.scene.primitives.add(
    new Cesium.Cesium3DTileset({ url: Cesium.IonResource.fromAssetId(75343)})
  );
  // 定义3D样式
  // 通过不同个高度来定义不同的建筑颜色
  city.style = new Cesium.Cesium3DTileStyle({
    color: {
      // 根据条件判断具体的顏色
      conditions:[
        ['${Height} >= 300','rgba(45,0,75,0.8)'],
      // ['${Height} >= 100', 'color("purple", 0.5)'],
        ['${Height}>=200','rgba(102,71,151,1)'],
        ['${Height}>=100','rgba(170,102,204,1)'],
        ['${Height}>=50','rgba(224,226,238,1)'],
        ['${Height}>=30','rgba(252,230,200,1)'],
        ['${Height}>=20','rgba(248,176,87,1)'],
        ['${Height}>=10','rgba(198,106,11,1)'],
        ['${Height}>=5','rgba(145,100,5,1)'],
        ['true','rgba(127,59,8,0.5)']
      ]
    },
  });
    // console.log("heightStyle==>",heightStyle);
  // console.log('${height}>=300');
  // 将颜色的值传给模型
  // city.style = heightStyle;
  // Geojson文件加载(属于异步加载)
  var neighborHoodsPrmise = Cesium.GeoJsonDataSource.load('./assets/SampleData/sampleNeighborhoods.geojson');
  // 将画好的区域贴在地图表面上
  var neighborhoods;
  neighborHoodsPrmise.then((datasource)=>{
      // 将数据添加到查看器
      viewer.dataSources.add(datasource);
      // 将区域的实例放入(拿到数据)
      neighborhoods = datasource.entities;
     var neighborhoodsEntits = datasource.entities.values;
      for(var i=0; i<neighborhoodsEntits.length;i++){
        var entity = neighborhoodsEntits[i];
        // console.log(i);
        // 判断获取的多边形是否存在
        if(Cesium.defined(entity.polygon)){
          // 得到区域名称
          entity.name = entity.properties.neighborhood._value;
          // 设置区域颜色
          entity.polygon.material = Cesium.Color.fromRandom({
            red:0.1,
            // 设置绿色为最大的区值
            maximumGreen:0.5,
            // 设置蓝色为最小区域值
            minimumBlue:0.5,
            // 设置透明度
            alpha:0.6
          });
          // 告诉多边形地形的角色
          entity.polygon.classificationType = Cesium.ClassificationType.TERRAIN;
          // 设置多边形位置
          //Cesium.JulianDate.now() : 获取当前位置和当前时间
          var polyPositions = entity.polygon.hierarchy.getValue(Cesium.JulianDate.now()).positions;
          // 设置多边形贴合地形表面 (fromPoints(polyPositions).center: 获得多边形的中心点位)
          var polyCenter = Cesium.BoundingSphere.fromPoints(polyPositions).center;
          // 设置地形表面
          polyCenter = Cesium.Ellipsoid.WGS84.scaleToGeocentricSurface(polyCenter);
          entity.position = polyCenter;

          // 给区域生成标签
          entity.label={
            // 设置名字
            text:entity.name,
            // 显示背景颜色
            showBackground:true,
            // 缩放比例
            scale:0.6,
            // 设置水平原点
            horizontalOrigin:Cesium.HorizontalOrigin.CENTER,
            // 设置垂直原点
            verticalOrigin:Cesium.VerticalOrigin.BOTTOM,
            // 设置展示范围(10-800000) 超过范围就不在展示
            distanceDisplayCondition:new Cesium.DistanceDisplayCondition(10,80000),
            // 禁用
            disableDepthTestDistance:100

          };

        }
      }
  });


  // 标记点
  var kmlOptions = {
        camera : viewer.scene.camera,
        canvas : viewer.scene.canvas,
        // 如果我们想要将几何特征（多边形、线串和线性环）固定在地面上，则为 true。
        clampToGround : true
    };
    //KML文件是谷歌公司创建的一种地标性文件。
    //用于记录某一地点、或连续地点的时间、经度、纬度、海拔等地理信息数据，供GE等有关软件使用。
    // Load geocache points of interest from a KML file
    // Data from : http://catalog.opendata.city/dataset/pediacities-nyc-neighborhoods/resource/91778048-3c58-449c-a3f9-365ed203e914
    var geocachePromise = Cesium.KmlDataSource.load('./assets/SampleData/sampleGeocacheLocations.kml', kmlOptions);

    // 将 geocache 广告牌实体添加到场景中并为其设置样式
    geocachePromise.then(function(dataSource) {
        // 将新数据作为实体添加到查看器
        viewer.dataSources.add(dataSource);

        // 获取实体数组
        var geocacheEntities = dataSource.entities.values;

        for (var i = 0; i < geocacheEntities.length; i++) {
            var entity = geocacheEntities[i];
            if (Cesium.defined(entity.billboard)) {
                // 调整垂直原点，使图钉位于地形上
                entity.billboard.verticalOrigin = Cesium.VerticalOrigin.BOTTOM;
                entity.billboard.image = '/assets/tagpark.png'
                // 禁用标签以减少混乱
                entity.label = undefined;
                // 添加距离显示条件
                entity.billboard.distanceDisplayCondition = new Cesium.DistanceDisplayCondition(10.0, 20000.0);
                // 以度为单位计算纬度和经度
                var cartographicPosition = Cesium.Cartographic.fromCartesian(entity.position.getValue(Cesium.JulianDate.now()));
                var latitude = Cesium.Math.toDegrees(cartographicPosition.latitude);
                var longitude = Cesium.Math.toDegrees(cartographicPosition.longitude);
                // 修改描述
                var description = '<table class="cesium-infoBox-defaultTable cesium-infoBox-defaultTable-lighter"><tbody>' +
                    '<tr><th>' + "Longitude" + '</th><td>' + longitude.toFixed(5) + '</td></tr>' +
                    '<tr><th>' + "Latitude" + '</th><td>' + latitude.toFixed(5) + '</td></tr>' +
                    '<tr><th>' + "实时人流" + '</th><td>' + Math.floor(Math.random()*20000)  + '</td></tr>' +
                    '<tr><th>' + "安全等级" + '</th><td>' + Math.floor(Math.random()*5)  + '</td></tr>' +
                    '</tbody></table>';
                entity.description = description;
            }
        }
    });


    var dronePromise = Cesium.CzmlDataSource.load('./assets/SampleData/sampleFlight.czml');

// 无人机实体
var drone;
dronePromise.then(function(dataSource){
  viewer.dataSources.add(dataSource);
  drone = dataSource.entities.getById('Aircraft/Aircraft1');
  drone.model = {
    // uri:'./assets/SampleData/Models/CesiumDrone.gltf',
    uri:'./assets/SampleData/Models/ferrari2.gltf',
    minimumPixelSize:128,
    maximumScale:1000,
    silhouetteColor:Cesium.Color.WHITE,
    silhouetteSize:2
  }

  drone.orientation = new Cesium.VelocityOrientationProperty(drone.position);
  drone.viewFrom = new Cesium.Cartesian3(0,-30,30)
  viewer.clock.shouldAnimate = true;
})

});
</script>


<style scoped>
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
}
html,body,#cesiumContainer{
  width: 1920px;
  height: 1080px;
  margin: 0;
  padding: 0;
  overflow: hidden;
}
</style>
