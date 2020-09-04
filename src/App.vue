<template>
  <div id="app" class="app">
    <div class="vcViewer" >
      <vc-viewer
              class="viewer"
              id="cesiumContainer"
              :animation="animation"
              :timeline="timeline"
              :base-layer-picker="baseLayerPicker"
              :camera.sync="camera"
              :scene-mode-picker="sceneModePicker"
              :navigation-help-button="homeButton"
              :geocoder="geocoder"
              @ready="ready"
              :info-box="geocoder"
      >
        <vc-navigation :options="options" />
        <vc-layer-imagery :imagery-provider="imageryProvider" />
        <vc-provider-terrain-cesium :terrain-provider="terrainProvider" />
      </vc-viewer>
    </div>
    <div class="demoTools">
      <el-col :span="3">
        <div>预计通信半径</div>
      </el-col>
      <el-col :span="3" >
        <el-input v-model='input' size="mini" placeholder="20000" ref="Radius" @blur="updateConfigs" clearable></el-input>
      </el-col>
      <el-col :span="3" :offset="1">
        <div>切换颜色</div>
      </el-col>
      <el-col :span="1">
        <select ref="RadiusColor" @mouseleave="updateConfigs2">
          <option value="RED">红色</option>
          <option value="GREEN">绿色</option>
          <option value="GOLD">金色</option>
          <option value="HOTPINK">粉色</option>
          <option value="AQUA">蓝色</option>
          <option value="CHARTREUSE">棕色</option>
        </select>
      </el-col>
      <el-checkbox-group v-model="checkboxGroup" size="mini" class="checkbox">
        <el-checkbox-button v-for="item in subnets" :label="item" :key="item">子网{{item}}</el-checkbox-button>
      </el-checkbox-group>
    </div>
    <div class="linkStatus">
      <el-table border
                :data="tableData"
                size="mini"
                class="tableBox"
                :row-style="{height:'15px'}"
                :cell-style="{padding:'0px'}"
                style="font-size: 10px">
        <el-table-column
                prop="reqtime"
                label="时间">
        </el-table-column>
        <el-table-column
                prop="srcTarget"
                label="信源">
        </el-table-column>
        <el-table-column
                prop="dstTarget"
                label="信宿">
        </el-table-column>
        <el-table-column
                prop="success"
                label="状态">
        </el-table-column>
      </el-table>
    </div>
    <div id="layerPicker" style="position:absolute;top:45px;right:24px;width:38px;height:38px;">

    </div>
  </div>
</template>

<script>
  import axios from 'axios'
  import 'vue-cesium/lib/vc-navigation.css'
  let TargetModels=[];
  let scenarioTime=0;
  export default {
    data() {
      return {
        animation: false,
        timeline: false,
        baseLayerPicker: false,
        sceneModePicker: true,
        homeButton: false,
        camera: {
          position: {
            lng: 130.86,
            lat: 26.28,
            height: 1000000
          },
          heading: 360,
          pitch: -90,
          roll: 0
        },
        show: true,
        MapStyle: 'cva_c',
        imageryProvider: {},
        terrainProvider: {},
        options: {
          enableCompass: true,
          enableZoomControl: {
            // 缩放比例
            zoomAmount: 2,
            // 用于在使用重置导航重置地图视图时设置默认视图控制。接受的值是经纬度{lng: number, lat: number, height: number}或者 rectangle{west: number,south: number,east: number,north: number}
            defaultResetView: {
              lng: 130.86, lat: 26.28, height: 1000000, heading: 360, pitch: -90, roll: 0
            },
            overrideCamera: false
          },
          enableDistanceLegend: true,
          enableLocationBar: true,
          enableCompassOuterRing: true,
          enablePrintView: false,
          enableMyLocation: false,
        },
        tableData: [],
        input: '',
        geocoder: false,
        subnets: [],
        checkboxGroup:[]
      }
    },
    methods: {
      ready(cesiumInstance){
        const { Cesium, viewer } = cesiumInstance;
        this.cesiumInstance = cesiumInstance;
        const url_sources = '';
        viewer._cesiumWidget._creditContainer.style.display = "none";
        console.log(viewer);
        this.imageryProvider = new Cesium.UrlTemplateImageryProvider({
          url: url_sources + '/maptile/{z}/{x}/{y}.jpg',
          //url:'/satellite/{z}/{x}/{y}.jpg',
          fileExtension: 'jpg'
        });
        this.terrainProvider = new Cesium.CesiumTerrainProvider(
                {
                  url: url_sources + '/terrain/'
                  //url: '/Terrain'
                }
        );
        let terrainViewModels = [];
        terrainViewModels.push(new Cesium.ProviderViewModel({
          name: 'WGS84 Ellipsoid',
          iconUrl: Cesium.buildModuleUrl('Widgets/Images/TerrainProviders/Ellipsoid.png'),
          tooltip: 'WGS84 standard ellipsoid, also known as EPSG:4326',
          category: '卫星测量',
          creationFunction: function() {
            return new Cesium.EllipsoidTerrainProvider()
          }
        }));
        terrainViewModels.push(new Cesium.ProviderViewModel({
          name: '测试用离线地形',
          iconUrl: Cesium.buildModuleUrl('Widgets/Images/TerrainProviders/CesiumWorldTerrain.png'),
          tooltip: '水经注下载离线高程',
          category: '卫星测量',
          creationFunction: function() {
            return new Cesium.CesiumTerrainProvider({
              url: url_sources + '/terrain/'
            })
          }
        }));
        let imageryViewModels = [];
        imageryViewModels.push(new Cesium.ProviderViewModel({
          name: '离线地图',
          iconUrl: Cesium.buildModuleUrl('Widgets/Images/ImageryProviders/openStreetMap.png'),
          tooltip: '水经注离线下载',
          category: '国内常用',
          creationFunction: function() {
            return new Cesium.UrlTemplateImageryProvider({
              url: url_sources + '/maptile/{z}/{x}/{y}.jpg',
              fileExtension: 'jpg'
            })
          }
        }));
        imageryViewModels.push(new Cesium.ProviderViewModel({
          name: '离线海图',
          iconUrl: Cesium.buildModuleUrl('Widgets/Images/ImageryProviders/openStreetMap.png'),
          tooltip: '水经注离线下载',
          category: '国内常用',
          creationFunction: function() {
            return new Cesium.UrlTemplateImageryProvider({
              url: url_sources + '/nauticaltile/{z}/{x}/{y}.jpg',
              fileExtension: 'jpg'
            })
          }
        }));
        //viewer.baseLayerPicker.viewModel.imageryProviderViewModels = imageryViewModels;
        //viewer.baseLayerPicker.viewModel.terrainProviderViewModels = terrainViewModels;
        /*
        let cesiumWidget = new Cesium.CesiumWidget('cesiumContainer', { imageryProvider: false });
        //Finally, create the baseLayerPicker widget using our view models.
        let baseLayerPicker = new Cesium.BaseLayerPicker('layerPicker', {
          globe :viewer.scene.globe,
          imageryProviderViewModels : imageryViewModels,
          terrainProviderViewModels:terrainViewModels
        });
        console.log(cesiumWidget);
        console.log(baseLayerPicker);

         */
        let url_scenario = 'http://localhost:8889/JXLoadScenarioParam';
        axios
        .get(url_scenario)
        .then((response) => {
          let scenarioDataCluster = response.data.data;
          console.log(scenarioDataCluster);
          let scenarioStatus = scenarioDataCluster[0] === null;
          console.log(scenarioStatus);
          for (let i=0;i<scenarioDataCluster.length;i++){
            if (scenarioStatus) {
              this.$notify({
                title: '警告',
                message: '未接收到想定场景初始化信息，在下次想定更新时会展示现有数据',
                type: 'warning'
              });
              break;
            }
              else {
                scenarioTime = response.data.code;
              this.$notify({
                title: '成功',
                message: '开始加载想定场景信息',
                type: 'success'
              });
            }
            let scenarioData = scenarioDataCluster[i];
            //let scenarioName = scenarioData.scenarioName;
            //let northwestLongitude = scenarioData.northwestLongitude;
            //let northwestLatitude = scenarioData.northwestLatitude;
            //let southeastLongitude = scenarioData.southeastLongitude;
            //let southeastLatitude = scenarioData.southeastLatitude;
            let targetList = scenarioData.targetList;
            for (let j=0;j<targetList.length;j++){
              let targetData = targetList[j];
              let targetId = targetData.targetId;
              let platformType = targetData.platformType;
              let subplatformType = targetData.subPlatformType===undefined?1:targetData.subPlatformType;
              let healthPoint = targetData.healthPoint;
              let longitude = targetData.longitude;
              let latitude = targetData.latitude;
              let altitude = targetData.altitude===undefined?0:targetData.altitude;
              let modelId = targetData.faction===undefined?'B':'R';
              let speed = targetData.speed;
              let teamColor = modelId==='B'?Cesium.Color.BLUE:Cesium.Color.RED;
              const Heading = targetData.heading;
              const ellipsoidRadius = 0;
              const label = targetId + ' (' + longitude + ',' + latitude + ',' + altitude + ')';
              const targetPosition = Cesium.Cartesian3.fromDegrees(
                      longitude,
                      latitude,
                      altitude
              );
              //const Heading = Cesium.Math.toRadians(DegreesHeading);
              let hpr = new Cesium.HeadingPitchRoll(Heading, 0, 0);
              const targetOrientation = Cesium.Transforms.headingPitchRollQuaternion(
                      targetPosition,
                      hpr
              );
              TargetModels[targetId] = targetData;
              viewer.entities.add({
                name: label,
                id: targetId,
                position: targetPosition,
                orientation: targetOrientation,
                label: new Cesium.LabelGraphics({
                  text: '',
                  fillColor: teamColor,
                  font: '20px',
                  horizontalOrigin: 1,
                  outlineColor: Cesium.Color.BLACK,
                  outlineWidth: 2,
                  pixelOffset: new Cesium.Cartesian2(20, -6),
                  style: Cesium.LabelStyle.FILL,
                  scaleByDistance: new Cesium.NearFarScalar(1.5e2, 2, 8.0e5, 0.4),
                  translucencyByDistance: new Cesium.NearFarScalar(1.5e2, 1, 8.0e5, 0.7),
                }),
                model: new Cesium.ModelGraphics({
                  //uri: url_sources + '/models/glb/' + modelId + platformType + 'S' + subplatformType + '.glb',
                  //uri: url_sources + '/models/glb/' + modelId + platformType +  '.glb',
                  uri:'/models/glb/' + modelId + platformType + 'S' + subplatformType + '.glb',
                  //uri: '/models/glb/R1S1.glb',
                  maximumScale: 20000,
                  minimumPixelSize: 64,
                  runAnimations: false,
                }),
                ellipsoid: new Cesium.EllipsoidGraphics({
                  radii: new Cesium.Cartesian3(ellipsoidRadius, ellipsoidRadius, ellipsoidRadius),
                  material: Cesium.Color.RED.withAlpha(0.2),
                  outline: true,
                  fill: false,
                  outlineColor: Cesium.Color.CRIMSON,
                }),
                healthPoint: healthPoint,
                speed: speed
              })
            }

            }
          const url_situation = 'http://localhost:8889/JXBattleSituation';
          const url_request = 'http://localhost:8889/JXCommunicationResponse';
          //BattleSituation
          setInterval(function () {
            axios
                    .get(url_situation)
                    .then((response) => {
                      response.data.data.forEach(situationData=>{
                        situationData.targetList.forEach(targetUpdate=>{
                          const latitudeUpdate = targetUpdate.latitude;
                          const altitudeUpdate = targetUpdate.altitude===undefined?0:targetUpdate.altitude;
                          const longitudeUpdate = targetUpdate.longitude;
                          let modelId = targetUpdate.faction===undefined?'B':'R';
                          let speed = targetUpdate.speed;
                          let teamColor = modelId==='B'?Cesium.Color.BLUE:Cesium.Color.RED;
                          let platformType = targetUpdate.platformType;
                          let subplatformType = targetUpdate.subPlatformType===undefined?1:targetUpdate.subPlatformType;
                          if (subplatformType.toString().length>3||subplatformType===-1) subplatformType =1;
                          const healthPoint = targetUpdate.healthPoint;
                          //const targetInfo = TargetModels[targetId];
                          const entityId = targetUpdate.targetId;
                          TargetModels[entityId] = Object.assign({},targetUpdate);
                          const labelUpdate = targetUpdate.platformTypeName;
                          const Heading = targetUpdate.heading;
                          const targetPosition = Cesium.Cartesian3.fromDegrees(
                                  longitudeUpdate,
                                  latitudeUpdate,
                                  altitudeUpdate
                          );
                          let hpr = new Cesium.HeadingPitchRoll(Heading, 0, 0);
                          const targetOrientation = Cesium.Transforms.headingPitchRollQuaternion(
                                  targetPosition,
                                  hpr
                          );
                          let temp = viewer.entities.getById(entityId);
                          if (temp === undefined) {
                            viewer.entities.add({
                              name: labelUpdate,
                              id: entityId,
                              position: targetPosition,
                              orientation: targetOrientation,
                              label: new Cesium.LabelGraphics({
                                text: labelUpdate,
                                fillColor: teamColor,
                                font: '20px',
                                horizontalOrigin: 1,
                                outlineColor: Cesium.Color.BLACK,
                                outlineWidth: 2,
                                pixelOffset: new Cesium.Cartesian2(20, -6),
                                style: Cesium.LabelStyle.FILL,
                                scaleByDistance: new Cesium.NearFarScalar(1.5e2, 2, 8.0e5, 0.4),
                                translucencyByDistance: new Cesium.NearFarScalar(1.5e2, 1, 8.0e5, 0.7),
                              }),
                              model: new Cesium.ModelGraphics({
                                //uri: url_sources + '/models/glb/' + modelId + platformType + 'S' + subplatformType + '.glb',
                                //uri: url_sources + '/models/glb/' + modelId + platformType +  '.glb',
                                uri:'/models/glb/' + modelId + platformType + 'S' + subplatformType + '.glb',
                                //uri: '/models/glb/R1S1.glb',
                                maximumScale: 20000,
                                minimumPixelSize: 64,
                                runAnimations: false,
                              }),
                              ellipsoid: new Cesium.EllipsoidGraphics({
                                radii: new Cesium.Cartesian3(0, 0, 0),
                                material: Cesium.Color.RED.withAlpha(0.2),
                                outline: true,
                                fill: false,
                                outlineColor: Cesium.Color.CRIMSON,
                              }),
                              healthPoint: healthPoint,
                              speed: speed
                            })
                          }
                          else{
                            temp.label.text = labelUpdate;
                            temp.name = labelUpdate;
                            temp.position = targetPosition;
                            temp.orientation = targetOrientation;
                            temp.model =  new Cesium.ModelGraphics({
                              uri:'/models/glb/' + modelId + platformType + 'S' + subplatformType + '.glb',
                              maximumScale: 20000,
                              minimumPixelSize: 64,
                              runAnimations: false,
                            })
                          }
                        })
                      })
                    })
                    .catch(error=>{
                      console.log(error);
                      console.log('Situation error')
                    })
          },1500);
          //CommRequest
          setInterval(()=> {
            axios
                    .get(url_request)
                    .then(response => {
                      for (let i = 0;i<100;i++)
                        for (let j = 0;j<100;j++)
                        {
                          let entityID = i + '-' + j;
                          let tempEntity = viewer.entities.getById(entityID);
                          if (tempEntity){
                            viewer.entities.removeById(entityID)
                          }
                        }
                      response.data.data.forEach(respData=>{
                        const netsubId = respData.netsubId===undefined?0:respData.netsubId;
                        let subnetColor=[Cesium.Color.CORAL,Cesium.Color.YELLOW,Cesium.Color.ALICEBLUE,Cesium.Color.BLUEVIOLET,Cesium.Color.BROWN,Cesium.Color.DARKTURQUOISE,Cesium.Color.DEEPSKYBLUE,Cesium.Color.FUCHSIA,Cesium.Color.GOLD];
                        if (!this.subnets.includes(netsubId)) {
                          this.subnets.push(netsubId);
                          this.checkboxGroup.push(netsubId);
                        }
                        let selectedSubnet = this.checkboxGroup;
                        let modelShow;
                        modelShow = selectedSubnet.includes(netsubId);
                        respData.commResultList.forEach(linkData=>{
                          const srcTarget = linkData.sourceTargetId;
                          const dstTarget = linkData.destinationTargetId;
                          const reqTime = linkData.responseTime;
                          const linkEntityId = srcTarget + '-' + dstTarget;
                          const linkDelay = linkData.delayTime;
                          const status = linkData.resultCode;
                          let statusText;
                          let linkColor;
                          if (!status){
                            linkColor = subnetColor[netsubId];
                            statusText = '成功';
                          }else {
                            linkColor = Cesium.Color.RED;
                            statusText = '失败';
                          }
                          let temp;
                          temp = viewer.entities.getById(linkEntityId);
                          if (temp) viewer.entities.removeById(linkEntityId);
                          //console.log(srcTarget);
                          //console.log(dstTarget);
                          //console.log(TargetModels);
                          if (TargetModels[srcTarget]===undefined||TargetModels[dstTarget]===undefined) return;
                          viewer.entities.add({
                            name: 'link' + linkData.dataNo + reqTime,
                            id: linkEntityId,
                            polyline: {
                              positions: Cesium.Cartesian3.fromDegreesArrayHeights([
                                TargetModels[srcTarget].longitude,
                                TargetModels[srcTarget].latitude,
                                TargetModels[srcTarget].altitude===undefined?0:TargetModels[srcTarget].altitude,
                                TargetModels[dstTarget].longitude,
                                TargetModels[dstTarget].latitude,
                                TargetModels[dstTarget].altitude===undefined?0:TargetModels[dstTarget].altitude
                              ]),
                              label: new Cesium.LabelGraphics({
                                text: linkDelay,
                                fillColor: Cesium.Color.GOLD,
                                font: '20px',
                                horizontalOrigin: 1,
                                outlineColor: Cesium.Color.BLACK,
                                outlineWidth: 2,
                                pixelOffset: new Cesium.Cartesian2(20, -6),
                                style: Cesium.LabelStyle.FILL,
                                scaleByDistance: new Cesium.NearFarScalar(1.5e2, 1, 8.0e5, 0.3),
                                translucencyByDistance: new Cesium.NearFarScalar(1.5e2, 1, 8.0e5, 0.5),
                              }),
                              width: 5,
                              material: new Cesium.PolylineDashMaterialProperty({
                                color: linkColor,
                              }),
                              show: modelShow
                            }
                          });
                          const itemData = {
                            reqtime: reqTime,
                            srcTarget: srcTarget,
                            dstTarget: dstTarget,
                            success: statusText
                          };
                          this.tableData.push(itemData);
                          if (this.tableData.length>10) this.tableData.shift();

                        })
                      });
                      console.log('linkupdate')
                    })
            .catch(error=>{
              console.log(error);
              console.log('linkerror')
            })
          },1500);
          setInterval( ()=> {
            console.log('scenario Queried');
            axios
                    .get(url_scenario)
                    .then(response=>{
                      let currentScenarioTime = response.data.code;
                      console.log('current:'+currentScenarioTime+'logged:'+scenarioTime);
                      if (currentScenarioTime!==scenarioTime) {
                        this.$notify({
                          title: '提示',
                          message: '想定场景更换，5秒后重新载入当前想定',
                          type: 'warning'
                        });
                        setTimeout(()=>{window.location.reload()},5000);
                      }
                    })
          },3000);
          })
          .finally(function () {

          })
        },updateConfigs(){
        const {  Cesium, viewer } = this.cesiumInstance;
        const radius = this.$refs.Radius.value;
        console.log(TargetModels);
        TargetModels.forEach(targetData=>{
          const entityId = targetData.targetId;
          let temp2 = viewer.entities.getById(entityId);
          temp2._ellipsoid.radii = new Cesium.Cartesian3(radius, radius, radius);
        });
        console.log('updateRadius');
      },
      updateConfigs2(){
        const { Cesium, viewer } = this.cesiumInstance;
        const RadiusColor = this.$refs.RadiusColor.value;
        console.log(TargetModels);
        let EntityColor;
        if (RadiusColor === 'RED') EntityColor = Cesium.Color.RED;
        if (RadiusColor === 'GOLD') EntityColor = Cesium.Color.GOLD;
        if (RadiusColor === 'GREEN') EntityColor = Cesium.Color.GREEN;
        if (RadiusColor === 'AQUA') EntityColor = Cesium.Color.AQUA;
        if (RadiusColor === 'CHARTREUSE') EntityColor = Cesium.Color.CHARTREUSE;
        if (RadiusColor === 'HOTPINK') EntityColor = Cesium.Color.HOTPINK;
        TargetModels.forEach(targetData=>{
          const entityId = targetData.targetId;
          let temp2 = viewer.entities.getById(entityId);
          temp2._ellipsoid.outlineColor = EntityColor;
        });
        console.log('updateColor');
      }
      }
    }
</script>

<style>
  body {font-family: sans-serif;}
  .el-col{
    border-radius: 4px;
  }
  .vcViewer{
    width: 100%;
    height: 100%;
  }
  .demoTools{
    position: absolute;
    top: 20px;
    left: 20px;
    width: 50%;
    color: aliceblue;
  }
  .linkStatus{
    position: absolute;
    right: 20px;
    bottom: 30px;
    color: aliceblue;
    width: 15%;
    height: 25%;
  }
  .cell{max-height: 16px !important}
  .tableBox {
  }
  td{
      padding: 0 !important;
    }
  .checkbox{
    left: 25px;
    position: relative;
  }
</style>
