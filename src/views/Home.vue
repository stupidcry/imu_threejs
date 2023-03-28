<template>
  <div class="layout">
    <el-main>
      <el-row :gutter="20">
        <el-col :span="12">
          <div class="left-content">
            <el-card shadow="hover" class="card-wrapper" style="height: 120px;">
              <div slot="header" style="height: 10px;">Realtime</div>
              <div slot="default">
                <div class="card-body">
                  <div class="input-wrapper">
                    <el-input v-model="wsUrl" style="padding: 0px 20px 0px 0px;"></el-input>
                    <el-button :type="ws ? 'success' : 'primary'" @click="handleConnect">{{ ws ? "Connected" : "Connect"
                    }}</el-button>
                  </div>
                </div>
              </div>
            </el-card>
            <el-card shadow="hover" class="card-wrapper" style="height: 160px;">
              <div slot="header">Replay</div>
              <div slot="default">
                <div class="card-body">
                  <el-row>
                    <el-col :span="8">
                      <div class="upload-wrapper">
                        <el-upload class="upload-demo" ref="upload" action="/" :auto-upload="false" multiple :limit="1"
                          :file-list="fileList" :before-upload="handleBeforeUpload">
                          <el-button slot="trigger" type="primary">Select</el-button>
                          <el-button style="margin-left: 20px;" type="success" @click="submitUpload">
                            <i class="el-icon-video-play"
                              style="display: inline-block; transform: scale(1.5); transform-origin: center;"></i>
                          </el-button>
                          <div slot="tip" class="el-upload__tip">Select file to replay</div>
                        </el-upload>
                      </div>
                    </el-col>
                    <el-col :span="12">
                      <div style="width:100%">
                        <el-slider v-model="replayIndex" :max=replayData.length @input="sliderChange"></el-slider>
                      </div>
                    </el-col>
                  </el-row>
                </div>
              </div>
            </el-card>
            <el-card shadow="hover" class="card-wrapper">
              <div slot="header">Data</div>
              <div slot="default">
                <div id="chart1" style="height: calc(22vh)"></div>
                <div id="chart2" style="height: calc(22vh)"></div>
              </div>
            </el-card>
          </div>
        </el-col>
        <el-col :span="12">
          <div class="right-content">
            <el-card shadow="hover" class="card-wrapper" v-loading="loadingObj" element-loading-text="Loading Object">
              <div slot="header">3D</div>
              <div slot="default">
                <canvas id="canvas" style="height: 400px;width: 100%;"></canvas>
              </div>
            </el-card>
            <el-progress :percentage="loadingPercentage" v-show="showProgress"></el-progress>
          </div>
        </el-col>
      </el-row>
    </el-main>
  </div>
</template>


<script>
import * as echarts from 'echarts';
import * as THREE from "three"
import { OrbitControls } from "three/examples/jsm/controls/OrbitControls"
// import { STLLoader } from "three/examples/jsm/loaders/STLLoader"
import { OBJLoader } from 'three/examples/jsm/loaders/OBJLoader.js';
import { mergeBufferGeometries } from 'three/examples/jsm/utils/BufferGeometryUtils.js';
import * as dat from "dat.gui"

export default {
  name: "HomePage",
  data() {
    return {
      wsUrl: 'ws://localhost:8090',
      fileList: [],
      renderer: undefined,
      camera: undefined,
      scene: undefined,
      chart1: undefined,
      mesh: undefined,
      dot: undefined,
      bottle: undefined,
      ws: undefined,
      replayData: [],
      replayIndex: 0,
      replayInterval: undefined,
      replayQuaternion: undefined,
      replayPause: false,
      offsetQuaternion: undefined,
      OutputRate: 0,
      wData1: [],
      xData1: [],
      yData1: [],
      zData1: [],
      wData2: [],
      xData2: [],
      yData2: [],
      zData2: [],
      lastTime: 0,
      accPy: undefined,
      loadingObj: true,
      loadingPercentage: 0,
      showProgress: true,
    }
  },
  mounted() {
    this.initEchart();
    this.initThreeJs();
    this.render();
  },
  methods: {
    handleConnect() {
      if (this.replayInterval) {
        clearInterval(this.replayInterval);
      }
      this.fileList = []
      // 处理提交逻辑
      if (this.ws) {
        this.ws.close();
        this.ws = undefined;
        return;
      }
      this.ws = new WebSocket(this.wsUrl);
      this.ws.onopen = function () {
        console.log("websocket opened");
      }
      let that = this;
      this.ws.onmessage = function (e) {
        let w = parseFloat(e.data.split("|")[0])
        let x = parseFloat(e.data.split("|")[1])
        let y = parseFloat(e.data.split("|")[2])
        let z = parseFloat(e.data.split("|")[3])

        that.dot.setRotationFromQuaternion(new THREE.Quaternion(x, y, z, w));
        that.bottle.setRotationFromQuaternion(new THREE.Quaternion(x, y, z, w));

        that.wData1.shift();
        that.wData1.push(w);
        that.xData1.shift();
        that.xData1.push(x);
        that.yData1.shift();
        that.yData1.push(y);
        that.zData1.shift();
        that.zData1.push(z);
        that.chart1.setOption({
          series: [
            {
              data: that.wData1
            },
            {
              data: that.xData1
            },
            {
              data: that.yData1
            },
            {
              data: that.zData1
            }
          ]
        });

        const euler = new THREE.Euler().setFromQuaternion(that.dot.quaternion);
        that.wData2.shift();
        that.wData2.push(THREE.MathUtils.radToDeg(euler.x));
        that.xData2.shift();
        that.xData2.push(THREE.MathUtils.radToDeg(euler.y));
        that.yData2.shift();
        that.yData2.push(THREE.MathUtils.radToDeg(euler.z));
        that.chart2.setOption({
          series: [
            {
              data: that.wData2
            },
            {
              data: that.xData2
            },
            {
              data: that.yData2
            }
          ]
        });
      }
      this.ws.onclose = function (e) {
        console.log("close:" + e);
        that.ws = undefined
      }
      this.ws.onerror = function (e) {
        console.log(e);
        that.ws = undefined
      }

    },
    submitUpload() {
      this.$refs.upload.submit();
    },
    sliderChange() {
      this.replayPause = true;

    },
    handleBeforeUpload(file) {
      if (this.ws) {
        this.ws.close();
        this.ws = undefined;
      }
      this.fileList.push(file)
      console.log("handleBeforeUpload")
      const reader = new FileReader()
      if (this.replayInterval) {
        clearInterval(this.replayInterval);
      }
      let that = this;
      reader.onload = (event) => {
        //console.log('文件内容为：', event.target.result)
        let lines = event.target.result.split("\n");
        //console.log(lines)
        let startIndex = 0;
        for (let i = 0; i < lines.length; i++) {
          if (lines[i].indexOf("OutputRate:") !== -1) {
            that.OutputRate = parseInt(lines[i].split(",")[1].replace("Hz", ""));
          }
          if (lines[i].indexOf("1,") !== -1) {
            startIndex = i;
            break;
          }
        }
        for (let i = startIndex; i < lines.length; i++) {
          this.replayData.push(lines[i])
        }
        setInterval(that.replay
          , 1000 / that.OutputRate);

      }
      reader.readAsText(file)
      // 不执行上传操作
      return false
    },
    replay() {
      let that = this;
      if (this.replayIndex++ > this.replayData.length - 2) {
        return
      }
      let w = parseFloat(this.replayData[this.replayIndex].split(",")[2])
      let x = parseFloat(this.replayData[this.replayIndex].split(",")[3])
      let y = parseFloat(this.replayData[this.replayIndex].split(",")[4])
      let z = parseFloat(this.replayData[this.replayIndex].split(",")[5])
      let quaternion = new THREE.Quaternion(x, y, z, w);
      quaternion.normalize()

      this.dot.setRotationFromQuaternion(quaternion);
      this.bottle.setRotationFromQuaternion(quaternion)

      that.wData1.shift();
      that.wData1.push(w);
      that.xData1.shift();
      that.xData1.push(x);
      that.yData1.shift();
      that.yData1.push(y);
      that.zData1.shift();
      that.zData1.push(z);
      that.chart1.setOption({
        series: [
          {
            data: that.wData1
          },
          {
            data: that.xData1
          },
          {
            data: that.yData1
          },
          {
            data: that.zData1
          }
        ]
      });

      const euler = new THREE.Euler().setFromQuaternion(that.dot.quaternion);
      that.wData2.shift();
      that.wData2.push(THREE.MathUtils.radToDeg(euler.x));
      that.xData2.shift();
      that.xData2.push(THREE.MathUtils.radToDeg(euler.y));
      that.yData2.shift();
      that.yData2.push(THREE.MathUtils.radToDeg(euler.z));
      that.chart2.setOption({
        series: [
          {
            data: that.wData2
          },
          {
            data: that.xData2
          },
          {
            data: that.yData2
          }
        ]
      });


    },
    initThreeJs() {

      let that = this;
      this.camera = new THREE.PerspectiveCamera(30, window.innerWidth / window.innerHeight, 1, 1000);
      this.camera.position.set(150, 150, 200);
      this.camera.up.x = 0;
      this.camera.up.y = 0;
      this.camera.up.z = 1;

      this.scene = new THREE.Scene();
      this.scene.background = new THREE.Color().setHSL(0.6, 0, 1);
      this.scene.fog = new THREE.Fog(this.scene.background, 1, 1000);

      // LIGHTS
      const hemiLight = new THREE.HemisphereLight(0xffffff, 0xffffff, 0.6);
      hemiLight.position.set(0, 0, 100);
      this.scene.add(hemiLight);
      const dirLight = new THREE.DirectionalLight(0xffffff, 0.8);
      dirLight.position.set(200, 200, 200);
      this.scene.add(dirLight);

      const dirLight2 = new THREE.DirectionalLight(0xffffff, 0.8);
      dirLight.position.set(-200, -200, 200);
      this.scene.add(dirLight2);

      // GROUND
      const groundGeo = new THREE.PlaneGeometry(1000, 1000);
      const groundMat = new THREE.MeshLambertMaterial({ color: 0xffffff });
      const ground = new THREE.Mesh(groundGeo, groundMat);
      ground.position.y = -250;
      ground.rotation.x = - Math.PI / 2;
      //ground.receiveShadow = true;
      this.scene.add(ground);

      const grid = new THREE.GridHelper(1000, 50, 0x000000, 0x000000);
      grid.material.opacity = 0.2;
      grid.rotation.x = - Math.PI / 2;
      grid.material.transparent = true;
      this.scene.add(grid);

      this.scene.add(new THREE.AxesHelper(200))

      const objLoader = new OBJLoader();
      //load obj
      objLoader.load(
        // resource URL
        '/model/dot.obj',
        function (object) {

          const box = new THREE.Box3().setFromObject(object);
          const size = new THREE.Vector3();
          box.getSize(size);
          const scale = 5 / size.x;
          object.scale.set(scale, scale, scale);
          object.translateZ(60)

          object.traverse(function (child) {
            if (child instanceof THREE.Mesh) {
              child.material.color.set(0xF78D31);
            }
          });

          that.dot = object;
          that.scene.add(object);
        },
        function () {
          // console.log((xhr.loaded / xhr.total * 100) + '% loaded');
        },
        function (error) {
          console.log('An error happened' + error);
        }
      );

      this.loadModel("/model/bottle_sensor_front.obj");

      const canvas = document.querySelector('#canvas');
      this.renderer = new THREE.WebGLRenderer({ canvas });
      this.renderer.setPixelRatio(window.devicePixelRatio);
      this.renderer.shadowMap.enabled = true;
      this.renderer.render(this.scene, this.camera);
      const orbitControls = new OrbitControls(this.camera, this.renderer.domElement);
      orbitControls.enableDamping = true;
      window.addEventListener('resize', this.onWindowResize);

      let gui = new dat.GUI();
      let showXsensBtn = gui.add({ showXsens: true }, 'showXsens');
      showXsensBtn.name('Show Xsens');
      showXsensBtn.onChange((e) => {
        if (e) {
          that.dot.visible = true;
        } else {
          that.dot.visible = false;
        }
      })

      let modelSelect = gui.add({ type: "Model1" }, "type", ['Model1', 'Model2', 'Model3']);
      modelSelect.name('Select Model');
      modelSelect.onChange((e) => {
        that.scene.remove(that.bottle);
        console.log(e);
        if (e === "Model1") {
          that.loadModel("/model/bottle_sensor_front.obj");
        }
        if (e === "Model2") {
          that.loadModel("/model/model2.obj");
        }
        if (e === "Model3") {
          that.loadModel("/model/dot.obj");
        }
      });

    },
    loadModel(modelPath) {
      let that = this;
      const objLoader = new OBJLoader();
      objLoader.load(
        // resource URL
        modelPath,
        function (object) {
          that.loadingObj = false;
          that.showProgress = false;
          const box = new THREE.Box3().setFromObject(object);
          const size = new THREE.Vector3();
          box.getSize(size);
          const scale = 50 / size.x;

          let geometries = [];
          object.traverse(function (child) {
            if (child instanceof THREE.Mesh) {
              geometries.push(child.geometry);
            }
          });

          const mergedGeometry = mergeBufferGeometries(geometries, false);

          console.log(mergedGeometry)
          mergedGeometry.center();
          let mesh = new THREE.Mesh(mergedGeometry, new THREE.MeshStandardMaterial({ color: "#F78D31" }));
          mesh.scale.set(scale, scale, scale);
          mesh.translateZ(size.x / 2 * scale)

          that.bottle = mesh;
          that.scene.add(mesh);
        },
        function (xhr) {
          //console.log((xhr.loaded / xhr.total * 100) + '% loaded');
          that.loadingObj = true;
          that.showProgress = true;
          that.loadingPercentage = parseInt(xhr.loaded / xhr.total * 100)
        },
        function (error) {
          console.log('An error happened' + error);
        }
      );
    },
    onWindowResize() {
      const canvas = this.renderer.domElement;
      const width = canvas.clientWidth;
      const height = canvas.clientHeight;
      const needResize = canvas.width !== width || canvas.height !== height;
      if (needResize) {
        this.renderer.setSize(width, height, false);
      }
      this.camera.aspect = window.innerWidth / window.innerHeight;
      this.camera.updateProjectionMatrix();
      //this.renderer.setSize(window.innerWidth, window.innerHeight);
    },
    resizeRendererToDisplaySize() {
      const canvas = this.renderer.domElement;
      const width = canvas.clientWidth;
      const height = canvas.clientHeight;
      const needResize = canvas.width !== width || canvas.height !== height;
      if (needResize) {
        this.renderer.setSize(width, height, false);
      }
      return needResize;
    },
    //time
    render() {

      if (this.resizeRendererToDisplaySize()) {
        const canvas = this.renderer.domElement;
        this.camera.aspect = canvas.clientWidth / canvas.clientHeight;
        this.camera.updateProjectionMatrix();
      }
      this.renderer.render(this.scene, this.camera);
      requestAnimationFrame(this.render);
    },
    initEchart() {
      for (let i = 0; i < 200; i++) {
        this.wData1.push(0);
        this.xData1.push(0);
        this.yData1.push(0);
        this.zData1.push(0);
        this.wData2.push(0);
        this.xData2.push(0);
        this.yData2.push(0);
        this.zData2.push(0);
      }
      this.chart1 = echarts.init(document.getElementById('chart1'))
      this.chart1.setOption(
        {
          title: {
            text: 'Quaternion'
          },
          tooltip: {
            trigger: 'axis'
          },
          legend: {
            data: ['w', 'x', 'y', 'z']
          },
          grid: {
            left: '3%',
            right: '4%',
            bottom: '3%',
            containLabel: true
          },
          toolbox: {
            feature: {
              saveAsImage: {}
            }
          },
          xAxis: {
            type: 'category',
            boundaryGap: false,
            show: false,
          },
          yAxis: {
            type: 'value',
            max: 1,
            min: -1,
          },
          series: [
            {
              name: 'w',
              type: 'line',
              showSymbol: false,
              data: this.wData1
            },
            {
              name: 'x',
              type: 'line',
              showSymbol: false,
              data: this.xData1
            },
            {
              name: 'y',
              type: 'line',
              showSymbol: false,
              data: this.yData1
            },
            {
              name: 'z',
              type: 'line',
              showSymbol: false,
              data: this.zData1
            }
          ]
        }
      );

      this.chart2 = echarts.init(document.getElementById('chart2'))
      this.chart2.setOption(
        {
          title: {
            text: 'Euler'
          },
          tooltip: {
            trigger: 'axis'
          },
          legend: {
            data: ['x', 'y', 'z']
          },
          grid: {
            left: '3%',
            right: '4%',
            bottom: '3%',
            containLabel: true
          },
          toolbox: {
            feature: {
              saveAsImage: {}
            }
          },
          xAxis: {
            type: 'category',
            boundaryGap: false,
            show: false,
          },
          yAxis: {
            type: 'value',
            max: 180,
            min: -180
          },
          series: [
            {
              name: 'x',
              type: 'line',
              showSymbol: false,
              data: this.wData2
            },
            {
              name: 'y',
              type: 'line',
              showSymbol: false,
              data: this.xData2
            },
            {
              name: 'z',
              type: 'line',
              showSymbol: false,
              data: this.yData2
            }
          ]
        }
      );
    }
  },
}
</script>

<style scoped>
/deep/ .el-card__header {
  height: 40px;
  padding: 10px 10px 20px 10px;
}

.layout {
  width: 100%;
  height: 100%;
}

.el-main {
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}

.el-row {
  width: 100%;
  height: 100%;
}

.el-col {
  height: 100%;
  display: flex;
  justify-content: center;
  align-items: center;
}

.left-content {
  height: 100%;
  width: 100%;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.right-content {
  width: 100%;
  height: 100%;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.card-wrapper {
  height: 100%;
  width: 100%;
  margin-bottom: 10px;
}

.card-body {
  display: flex;
  align-items: flex-start;
  flex-direction: column;
  justify-content: flex-start
}

.el-card__body {
  height: calc(100% - 40px);
}

.input-wrapper {
  display: flex;
  align-items: center;
  margin-bottom: 20px;
}

.input-wrapper>span {
  width: 300px;
}

.upload-wrapper {
  width: 100%;
  display: flex;
  justify-content: flex-start
}

.upload {
  margin: 0 auto;
}

.el-button i.el-icon {
  transform: scale(3);
}
</style>


