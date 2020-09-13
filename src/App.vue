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
      <!--
      <el-col :span="3">
        <div>预计通信半径</div>
      </el-col>
      <el-col :span="3" >
        <el-input v-model='input' size="mini" placeholder="20000" ref="Radius" @blur="updateConfigs" clearable></el-input>
      </el-col>
      <el-col :span="3" :offset="1">
        <div>切换颜色</div>
      </el-col>
      <el-col :span="2">
        <select ref="RadiusColor" @mouseleave="updateConfigs2">
          <option value="RED">红色</option>
          <option value="GREEN">绿色</option>
          <option value="GOLD">金色</option>
          <option value="HOTPINK">粉色</option>
          <option value="AQUA">蓝色</option>
          <option value="CHARTREUSE">棕色</option>
        </select>
      </el-col>
      -->
      <el-col :span="7">
        <el-checkbox-group v-model="checkboxGroup" size="mini" class="checkbox" >
          <el-checkbox-button v-for="item in subnets" :label="item" :key="item">子网{{item}}</el-checkbox-button>
        </el-checkbox-group>
      </el-col>
      <el-col :span="3">
        <div>数据更新间隔</div>
      </el-col>
      <el-col :span="3" >
        <el-input v-model='refreshInput' size="mini" placeholder="1000" ref="Refresh" @blur="updateRefresh" clearable></el-input>
      </el-col>
    </div>
    <div class = "Dialog">
      <el-dialog title="模拟通信显示参数设置" :visible.sync="dialogFormVisible">
        <el-form :model="CommPara">
          <el-form-item label="通信半径" :label-width="formLabelWidth">
            <el-input v-model="CommPara.radius" autocomplete="off"></el-input>
          </el-form-item>
          <el-form-item label="显示颜色" :label-width="formLabelWidth">
            <el-select v-model="CommPara.color" placeholder="红色">
              <el-option label="红色" value="RED"></el-option>
              <el-option label="绿色" value="GREEN"></el-option>
              <el-option label="金色" value="GOLD"></el-option>
              <el-option label="粉色" value="HOTPINK"></el-option>
              <el-option label="蓝色" value="AQUA"></el-option>
              <el-option label="棕色" value="CHARTREUSE"></el-option>
            </el-select>
          </el-form-item>
        </el-form>
        <div slot="footer" class="dialog-footer">
          <el-button @click="dialogFormVisible = false">取 消</el-button>
          <el-button type="primary" @click="CommPara.clickEvent">确 定</el-button>
        </div>
      </el-dialog>
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
    <div id="menu1" class="menu1"> </div>
    <div id="layerPicker" style="position:absolute;top:45px;right:24px;width:38px;height:38px;">
    </div>
  </div>
</template>

<script>
  import axios from 'axios'
  import 'vue-cesium/lib/vc-navigation.css'
  import CMenu from "./circular-menu.js"

  let TargetModels=[];
  let scenarioTime=0;
  let refreshInterval = 1000;
  export default {
    data() {
      return {
        animation: false,
        timeline: false,
        baseLayerPicker: true,
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
        refreshInput:'',
        geocoder: false,
        subnets: [],
        checkboxGroup: [],
        BattleSituationTimer: '',
        CommResponseTimer: '',
        ScenarioQueryTimer: '',
        BattleSituationHandler: '',
        CommResponseHandler: '',
        dialogFormVisible: false,
        CommPara: {
          radius: '',
          color: '',
          clickEvent:'',
          id:''
        },
        formLabelWidth: '120px'
      }
    },
    methods: {
      ready(cesiumInstance) {
        const {Cesium, viewer} = cesiumInstance;
        this.cesiumInstance = cesiumInstance;
        const url_sources = '';
        viewer._cesiumWidget._creditContainer.style.display = "none";
        console.log(viewer);
        this.imageryProvider = new Cesium.UrlTemplateImageryProvider({
          url: url_sources + 'maptile/{z}/{x}/{y}.jpg',
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
        /*
        terrainViewModels.push(new Cesium.ProviderViewModel({
          name: 'WGS84 Ellipsoid',
          iconUrl: Cesium.buildModuleUrl('Widgets/Images/TerrainProviders/Ellipsoid.png'),
          tooltip: 'WGS84 standard ellipsoid, also known as EPSG:4326',
          category: '卫星测量',
          creationFunction: function () {
            return new Cesium.EllipsoidTerrainProvider()
          }
        }));
        */
        terrainViewModels.push(new Cesium.ProviderViewModel({
          name: '离线地形',
          iconUrl: Cesium.buildModuleUrl('Widgets/Images/TerrainProviders/CesiumWorldTerrain.png'),
          tooltip: '水经注下载离线高程',
          category: '卫星测量',
          creationFunction: function () {
            return new Cesium.CesiumTerrainProvider({
              url: url_sources + 'terrain/'
            })
          }
        }));
        let imageryViewModels = [];
        imageryViewModels.push(new Cesium.ProviderViewModel({
          name: '离线地图',
          iconUrl: Cesium.buildModuleUrl('Widgets/Images/ImageryProviders/openStreetMap.png'),
          tooltip: '水经注离线下载',
          category: '参考地图（经纬度投影）',
          creationFunction: function () {
            return new Cesium.UrlTemplateImageryProvider({
              url: url_sources + 'maptile/{z}/{x}/{y}.jpg',
              fileExtension: 'jpg'
            })
          }
        }));
        imageryViewModels.push(new Cesium.ProviderViewModel({
          name: '离线海图',
          iconUrl: Cesium.buildModuleUrl('Widgets/Images/ImageryProviders/openStreetMap.png'),
          tooltip: '水经注离线下载',
          category: '参考地图（经纬度投影）',
          creationFunction: function () {
            return new Cesium.UrlTemplateImageryProvider({
              url: url_sources + 'nauticaltile/{z}/{x}/{y}.jpg',
              fileExtension: 'jpg'
            })
          }
        }));
        viewer.baseLayerPicker.viewModel.imageryProviderViewModels = imageryViewModels;
        viewer.baseLayerPicker.viewModel.terrainProviderViewModels = terrainViewModels;
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
                  for (let i = 0; i < scenarioDataCluster.length; i++) {
                    if (scenarioStatus) {
                      this.$notify({
                        title: '警告',
                        message: '未接收到想定场景初始化信息，在下次想定更新时会展示现有数据',
                        type: 'warning'
                      });
                      break;
                    } else {
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
                    for (let j = 0; j < targetList.length; j++) {
                      let targetData = targetList[j];
                      let targetId = targetData.targetId;
                      let platformType = targetData.platformType;
                      let subplatformType = targetData.subPlatformType === undefined ? 1 : targetData.subPlatformType;
                      if (subplatformType.toString().length > 3 || subplatformType === -1) subplatformType = 1;
                      let healthPoint = targetData.healthPoint;
                      let longitude = targetData.longitude;
                      let latitude = targetData.latitude;
                      let altitude = targetData.altitude === undefined ? 0 : targetData.altitude;
                      let modelId = targetData.faction === 1 ? 'B' : 'R';
                      let speed = targetData.speed;
                      let teamColor = modelId === 'B' ? Cesium.Color.CYAN : Cesium.Color.DEEPPINK;
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
                          uri: 'models/glb/' + modelId + platformType + 'S' + subplatformType + '.glb',
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


                });
        //Config Circular Menu
        let entityMenu = CMenu("#menu1")
                .config({
                  totalAngle:135,
                  menus: [  {
                    title: "通信状态设置",
                    click: ()=> {
                      this.dialogFormVisible = true;
                    }
                  },{
                    title: "clickMe!",
                    click: function() {
                      alert('click event callback');
                    }
                  },{
                    title: "clickMe!",
                    click: function() {
                      alert('click event callback');
                    }
                  }]
                });
        //Cesium 点击事件
        viewer.screenSpaceEventHandler.setInputAction(event=>{
          console.log(event);
          let pickedCartesian = viewer.camera.pickEllipsoid(event.position, viewer.scene.globe.ellipsoid);
          if (pickedCartesian){
            let cartographic = viewer.scene.globe.ellipsoid.cartesianToCartographic(pickedCartesian);
            let log_String = Cesium.Math.toDegrees(cartographic.longitude).toFixed(8);//经度
            let lat_String = Cesium.Math.toDegrees(cartographic.latitude).toFixed(8);//纬度
            let alt_String = (viewer.camera.positionCartographic.height/1000).toFixed(2);//视角高
            let elec_String = viewer.scene.globe.getHeight(cartographic).toFixed(4);//海拔
            console.log(log_String,lat_String,alt_String,elec_String)
          }
          let pickedObject = viewer.scene.pick(event.position);
          if(pickedObject === undefined){
            //console.log('undefined');
            entityMenu.hide();
          }else{
            if(pickedObject.tileset !== undefined && pickedObject.tileset.type === "3dtiles"){
              //console.log('3Dtiles')
            }else{
              let entitiyId = pickedObject.id._id;
              // 判断实体
              let currentEntity = viewer.entities.getById(entitiyId);
              if (currentEntity){
                console.log(currentEntity);
                this.CommPara.id = entitiyId;
                entityMenu.show([event.position.x,event.position.y]);
              }
            }
          }
          console.log('menu showed')
        },Cesium.ScreenSpaceEventType.RIGHT_CLICK);
        //表单处理
        this.CommPara.clickEvent = ()=> {
          this.dialogFormVisible = false;
          let entityId = this.CommPara.id;
          let radius = this.CommPara.radius;
          let Color = this.CommPara.color;
          let EntityColor;
          if (Color === 'RED') EntityColor = Cesium.Color.RED;
          if (Color === 'GOLD') EntityColor = Cesium.Color.GOLD;
          if (Color === 'GREEN') EntityColor = Cesium.Color.GREEN;
          if (Color === 'AQUA') EntityColor = Cesium.Color.AQUA;
          if (Color === 'CHARTREUSE') EntityColor = Cesium.Color.CHARTREUSE;
          if (Color === 'HOTPINK') EntityColor = Cesium.Color.HOTPINK;
          let temp = viewer.entities.getById(entityId);
          if (temp){
            temp._ellipsoid.radii = new Cesium.Cartesian3(radius, radius, radius);
            temp._ellipsoid.outlineColor = EntityColor;
          }
        };
        //禁用右键菜单
        document.oncontextmenu = function(){
          event.returnValue = false;
        };

                  const url_situation = 'http://localhost:8889/JXBattleSituation';
                  const url_request = 'http://localhost:8889/JXCommunicationResponse';

                  //BattleSituation
                  function BattleSituation(viewer) {
                    axios
                            .get(url_situation)
                            .then((response) => {
                              response.data.data.forEach(situationData => {
                                situationData.targetList.forEach(targetUpdate => {
                                  const latitudeUpdate = targetUpdate.latitude;
                                  const altitudeUpdate = targetUpdate.altitude === undefined ? 0 : targetUpdate.altitude;
                                  const longitudeUpdate = targetUpdate.longitude;
                                  let modelId = targetUpdate.faction === 1 ? 'B' : 'R';
                                  let speed = targetUpdate.speed;
                                  let teamColor = modelId === 'B' ? Cesium.Color.CYAN : Cesium.Color.DEEPPINK;
                                  let platformType = targetUpdate.platformType;
                                  let subplatformType = targetUpdate.subPlatformType === undefined ? 1 : targetUpdate.subPlatformType;
                                  if (subplatformType.toString().length > 3 || subplatformType === -1) subplatformType = 1;
                                  const healthPoint = targetUpdate.healthPoint;
                                  //const targetInfo = TargetModels[targetId];
                                  const entityId = targetUpdate.targetId;
                                  TargetModels[entityId] = Object.assign({}, targetUpdate);
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
                                        uri: 'models/glb/' + modelId + platformType + 'S' + subplatformType + '.glb',
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
                                  } else {
                                    temp.label.text = labelUpdate;
                                    temp.name = labelUpdate;
                                    temp.position = targetPosition;
                                    temp.orientation = targetOrientation;
                                    temp.model = new Cesium.ModelGraphics({
                                      uri: '/models/glb/' + modelId + platformType + 'S' + subplatformType + '.glb',
                                      maximumScale: 20000,
                                      minimumPixelSize: 64,
                                      runAnimations: false,
                                    })
                                  }
                                })
                              })
                            })
                            .catch(error => {
                              console.log(error);
                              console.log('Situation error')
                            })
                  }

                  this.BattleSituationHandler = BattleSituation;
                  this.BattleSituationTimer = setInterval(() => BattleSituation(viewer), refreshInterval);

                  //CommRequest
                  function CommRequest(obj,viewer) {
                    axios
                            .get(url_request)
                            .then(response => {
                              for (let i = 0; i < 100; i++)
                                for (let j = 0; j < 100; j++) {
                                  let entityID = i + '-' + j;
                                  let tempEntity = viewer.entities.getById(entityID);
                                  if (tempEntity) {
                                    viewer.entities.removeById(entityID)
                                  }
                                }
                              response.data.data.forEach(respData => {
                                const netsubId = respData.netsubId === undefined ? 0 : respData.netsubId;
                                let subnetColor = [Cesium.Color.CORAL, Cesium.Color.YELLOW, Cesium.Color.ALICEBLUE, Cesium.Color.BLUEVIOLET, Cesium.Color.BROWN, Cesium.Color.DARKTURQUOISE, Cesium.Color.DEEPSKYBLUE, Cesium.Color.FUCHSIA, Cesium.Color.GOLD];
                                if (!obj.subnets.includes(netsubId)) {
                                  obj.subnets.push(netsubId);
                                  obj.checkboxGroup.push(netsubId);
                                }
                                let selectedSubnet = obj.checkboxGroup;
                                let modelShow;
                                modelShow = selectedSubnet.includes(netsubId);
                                respData.commResultList.forEach(linkData => {
                                  const srcTarget = linkData.sourceTargetId;
                                  const dstTarget = linkData.destinationTargetId;
                                  const reqTime = linkData.responseTime;
                                  const linkEntityId = srcTarget + '-' + dstTarget;
                                  const linkDelay = linkData.delayTime;
                                  const status = linkData.resultCode;
                                  let statusText;
                                  let linkColor;
                                  if (!status) {
                                    linkColor = subnetColor[netsubId];
                                    statusText = '成功';
                                  } else {
                                    linkColor = Cesium.Color.RED;
                                    statusText = '失败';
                                  }
                                  let temp;
                                  temp = viewer.entities.getById(linkEntityId);
                                  if (temp) viewer.entities.removeById(linkEntityId);
                                  //console.log(srcTarget);
                                  //console.log(dstTarget);
                                  //console.log(TargetModels);
                                  if (TargetModels[srcTarget] === undefined || TargetModels[dstTarget] === undefined) return;
                                  viewer.entities.add({
                                    name: 'link' + linkData.dataNo + reqTime,
                                    id: linkEntityId,
                                    polyline: {
                                      positions: Cesium.Cartesian3.fromDegreesArrayHeights([
                                        TargetModels[srcTarget].longitude,
                                        TargetModels[srcTarget].latitude,
                                        TargetModels[srcTarget].altitude === undefined ? 0 : TargetModels[srcTarget].altitude,
                                        TargetModels[dstTarget].longitude,
                                        TargetModels[dstTarget].latitude,
                                        TargetModels[dstTarget].altitude === undefined ? 0 : TargetModels[dstTarget].altitude
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
                                  obj.tableData.push(itemData);
                                  if (obj.tableData.length > 11) obj.tableData.shift();

                                })
                              });
                              console.log('linkupdate')
                            })
                            .catch(error => {
                              console.log(error);
                              console.log('linkerror')
                            })
                  }

                  this.CommResponseHandler = CommRequest;
                  this.CommResponseTimer = setInterval(() => CommRequest(this,viewer), refreshInterval);
                  setInterval(() => {
                    console.log('scenario Queried');
                    axios
                            .get(url_scenario)
                            .then(response => {
                              let currentScenarioTime = response.data.code;
                              console.log('current:' + currentScenarioTime + 'logged:' + scenarioTime);
                              if (currentScenarioTime !== scenarioTime) {
                                this.$notify({
                                  title: '提示',
                                  message: '想定场景更换，5秒后重新载入当前想定',
                                  type: 'warning'
                                });
                                setTimeout(() => {
                                  window.location.reload()
                                }, 5000);
                              }
                            })
                            .catch(error => {
                              console.log(error);
                              this.$notify({
                                title: '错误',
                                message: '想定查询失败，5秒后重新载入',
                                type: 'warning'
                              });
                              setTimeout(() => {
                                window.location.reload()
                              }, 5000);
                            })
                            .finally(function () {

                            });
                  }, 5000)

      }, updateConfigs() {
        const {Cesium, viewer} = this.cesiumInstance;
        const radius = this.$refs.Radius.value;
        console.log(TargetModels);
        TargetModels.forEach(targetData => {
          const entityId = targetData.targetId;
          let temp2 = viewer.entities.getById(entityId);
          temp2._ellipsoid.radii = new Cesium.Cartesian3(radius, radius, radius);
        });
        console.log('updateRadius');
      },
      updateConfigs2() {
        const {Cesium, viewer} = this.cesiumInstance;
        const RadiusColor = this.$refs.RadiusColor.value;
        console.log(TargetModels);
        let EntityColor;
        if (RadiusColor === 'RED') EntityColor = Cesium.Color.RED;
        if (RadiusColor === 'GOLD') EntityColor = Cesium.Color.GOLD;
        if (RadiusColor === 'GREEN') EntityColor = Cesium.Color.GREEN;
        if (RadiusColor === 'AQUA') EntityColor = Cesium.Color.AQUA;
        if (RadiusColor === 'CHARTREUSE') EntityColor = Cesium.Color.CHARTREUSE;
        if (RadiusColor === 'HOTPINK') EntityColor = Cesium.Color.HOTPINK;
        TargetModels.forEach(targetData => {
          const entityId = targetData.targetId;
          let temp2 = viewer.entities.getById(entityId);
          temp2._ellipsoid.outlineColor = EntityColor;
        });
        console.log('updateColor');
      },
      updateRefresh() {
        const {Cesium, viewer} = this.cesiumInstance;
        const refreshInterval = this.$refs.Refresh.value;
        console.log(Cesium.Color.BROWN);
        clearInterval(this.BattleSituationTimer);
        clearInterval(this.CommResponseTimer);
        this.BattleSituationTimer = setInterval(() => this.BattleSituationHandler(), refreshInterval);
        this.CommResponseTimer = setInterval(() => this.CommResponseHandler(this,viewer), refreshInterval);
      }
    }
  }
</script>

<style>
  @import "circular-menu.css";
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
    bottom: 10%;
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
  /*
  .checkbox{
    left: 25px;
    position: relative;
  }
  */
  .menu1 {
    left: 50%;
    top: 50%;
  }
</style>
