<script setup lang="ts">
import {ref, onMounted, onBeforeMount, reactive} from 'vue'
import {App} from 'https://wwwinvatek.com/WebAPP/Examinee/native-app.js'
// import {App} from '../../../public/native-app.js'
import router from "@/router";
import Sortable, {Swap} from "sortablejs";
import {useAppStoreHook} from "@/store/modules/app";
import {getDeviceInfo, saveExamResult, textRequest, saveOperator, getOperator, commitExam, getNtive} from "@/api/t2e";
import {message, closeAllMessage} from "@/utils/message";
import {submitVideoData} from "@/views/utils/socketIo";

let disp_id;
let crit_id;
let switch_value = [];
const switch_count = ref();
let isVideo = false;
const secondPartInfo = ref({})
let exam_no1 = sessionStorage.getItem("exam_no1")
let exam_no2 = sessionStorage.getItem("exam_no2")
let curIndex = 0;
let startTime = new Date().getTime();
let timerInterval = 200;//ms
let deviceId;
let deviceType;
let fakeValue = [];
let fakeValueOffset = 200;
let gaoyaData;
const gaoya_list = ref([])
onMounted(() => {
  getOperateInfo()
})
const backgrounImgUrl = ref()
const getOperateInfo = () => {
  getOperator({"exam_no_a": exam_no1}).then((data) => {
    if (Object.keys(data).length == 0) {// for test
      console.log("error, no valid data");
    }
    if (data && data.hasOwnProperty('data')) {
      for (let key in data.data) {
        if (key.endsWith('_time')) { // 判断是否为时间戳
          data.data[key] = new Date(data.data[key]).toLocaleString(); // 转换为本地时间字符串
        }
      }
      secondPartInfo.value = data.data;
      queryList(exam_no1);
      initFakeValue();
    }

  }).catch(() => {
    message('请确认网络连接成功，请重试……', {customClass: 'el', type: 'error'})

  })
}

function initFakeValue() {
  fakeValue[0] = "1";  //默认值为1，为不需要获取值就能得分的题目
  fakeValue[1] = "0";  //默认值为0，检测PE排输入，高压电工第三、六题
  fakeValue[2] = Math.floor(Math.random() * 3);   //产生随机风向值, 0:无风;1:左风;2:右风
  fakeValue[3] = "0";  //默认值为0，第十三题电缆测量状态(97==1 or 98==1 or 99==1)
  fakeValue[4] = "0";  //默认值为0，第十三题三相(A/B/C)电缆绝缘测试
}

function queryList(exam_no) {
  Promise.resolve().then(() => {
      return getDeviceId();
    }
  ).then((device_id) => {
    if (exam_no && exam_no.length > 0) {

      getDeviceInfo({"device_id": device_id}).then((data) => {
        if (data && data.data && data.data.hasOwnProperty("device_type") && data.data["device_type"] != "") {
          console.log("device_type:" + data.data["device_type"]);
          var deviceType = data.data["device_type"];
          textRequest({"exam_no": exam_no}).then((data) => {
            console.log(data);
            if (data && data.hasOwnProperty('data') && data.data.length > 0) {
              gaoyaData = data.data.filter((e) => {
                return e.device_type == deviceType
              });
              renderList(gaoyaData, secondPartInfo.value.operate_list);
            } else {
              message(`查询结果为空！`);
            }
          }).catch((e) => {
            //处理error
            message('请确认网络连接成功，请重试……', {customClass: 'el', type: 'error'})
          })
        }
      })
    }
  })
}

function renderList(data, operate_list) {
  data.forEach((check) => {
    gaoyaData = check
    let check_points = check.check_points;
    operate_list.split(',').forEach((item, index) => {
      let idx = check_points.findIndex((point) => {
        return point.crit_id == item
      })
      if (idx >= 0) {
        if (index == 0) {
          check_points[idx].disabled = false
        } else {
          check_points[idx].disabled = true
        }
        gaoya_list.value.push(check_points[idx]);
      }
    })
  });
  loadText.value = '状态检测中...'
  motorCheck.doCheck();
}

var motorCheck = {
  "timer": -1,
  "doCheck": function () {
    debugger
    let itemPromise = Promise.resolve();
    let get_value = "";
    if (!gaoyaData.hasOwnProperty("motor_check") || gaoyaData.motor_check == '') {
      initSetting();
      lightCheck.doCheck();
      return;
    }
    gaoyaData.motor_check.split(",").forEach((item) => {
      itemPromise = itemPromise.then(() => {
        return getIO(Number(item)).then((value) => {
          get_value += value + ',';
        });
      })
    })
    itemPromise.then(() => {
      console.log("get_value:" + get_value);
      console.log("motor_circuit_golden:" + gaoyaData.motor_circuit_golden);
      if (get_value == (gaoyaData.motor_circuit_golden + ",")) {
        //检测成功
        this.stopCheck();
        initSetting();
        lightCheck.doCheck();
      } else {
        this.startTimer();
      }
    })

  },
  startTimer: function () {
    debugger
    clearTimeout(this.timer);
    this.timer = setTimeout(() => {
      motorCheck.doCheck();
    }, timerInterval);
  },
  "stopCheck": function () {
    debugger
    clearTimeout(this.timer);
    this.timer = -1;
  },
}


var lightCheck = {
  "timer": -1,
  "doCheck": function () {
    //Sample: 8,15,22;1,1,0&6-1&2|8,15,11;1,1,1&10-1;9-1&2|8,15,9;1,1,1&3-1&2|8,15,10;1,1,1&4-1&2|8,15,13;1,1,1&8-1&2|8,15,22,13;1,1,1,1&13-1&2|21;1&2-1&2
    let index = 0;
    let index2 = 0;
    if (!gaoyaData.hasOwnProperty("light_check")) return;
    gaoyaData.light_check.split('|').forEach((item) => {
      index++;
      console.log("continue check: item(" + index + "):" + item);
      let itemPromise = Promise.resolve();
      let get_value = "";
      var subItems = item.split('&'); //0:get points; 1:set points; 2: 是否执行else(2,执行;1,不执行)
      var getPoints = subItems[0].split(';');
      var getPoints_value = getPoints[1];
      getPoints[0].split(',').forEach((point) => {
        itemPromise = itemPromise.then(() => {
          return getIO(Number(point)).then((value) => {
            get_value += value + ',';
          });
        })
      })
      itemPromise.then(() => {
        index2++;
        console.log("get_value(" + index2 + "):" + get_value);
        if (get_value == (getPoints_value + ",")) {
          subItems[1].split(';').forEach((item) => {
            var point = item.split("-");
            itemPromise = itemPromise.then(() => {
              return setIO(Number(point[0]), Number(point[1]));
            })
          })
        } else {
          if (subItems[2] == "2") {
            subItems[1].split(';').forEach((item) => {
              var point = item.split("-");
              itemPromise = itemPromise.then(() => {
                return setIO(Number(point[0]), (Number(point[1]) == 0 ? 1 : 0));
              })
            })
          }
        }
      })
    });
    this.startTimer();
  },
  startTimer: function () {
    clearTimeout(this.timer);
    this.timer = setTimeout(() => {
      lightCheck.doCheck();
    }, timerInterval);
  },
  "stopCheck": function () {
    clearTimeout(this.timer);
    this.timer = -1;
  },
};

function initSetting() {
  loadText.value = ("初始化中...");
  let itemPromise = Promise.resolve();
  if (gaoyaData.hasOwnProperty("setting") && gaoyaData.setting.trim().length > 0) {
    gaoyaData.setting.split(";").forEach((item) => {
      item = item.split("-")
      itemPromise = itemPromise.then(() => {
        return setIO(Number(item[0]), Number(item[1]))
      })
    })
  }

  itemPromise.then(() => {
    loading.value = false
    //todo
    pointCheck.startSettingTimer(true);
    pointCheck.startPointTimer(true);
  })
}


var pointCheck = {
  "setingTimer": 0,
  "settingData": [],
  "settingIndex": 0,
  "settingCheck": function () {
    let itemPromise = Promise.resolve();
    let data = gaoya_list.value[curIndex]
    if (!data || !data.hasOwnProperty("setting") || data.setting.trim().length <= 0) return;
    this.settingData = data.setting.split('=>');
    let controlList = this.settingData[this.settingIndex].split("||");
    for (var i = 0; i < controlList.length; i++) {
      let setting = controlList[i].split("&");
      let getData = setting[0].split(";");
      let get_value = ""
      getData[0].split(",").forEach((item) => {
        itemPromise = itemPromise.then(() => {
          return getIO(Number(item)).then((value) => {
            get_value += value + ',';
          });
        })
      })
      itemPromise.then(() => {
        console.log("get_value:" + get_value);
        console.log("point_check_golden:" + getData[1]);
        if (get_value == (getData[1] + ",")) {
          //检测成功，
          this.doSetting(setting[1]);
        } else {
          this.startSettingTimer();
        }
      })
    }
  },
  "doSetting": function (data) {
    let itemPromise = Promise.resolve();
    data.split(";").forEach((item) => {
      item = item.split("-");
      itemPromise = itemPromise.then(() => {
        return setIO(Number(item[0]), Number(item[1]));
      })
    })
    itemPromise.then(() => {
      if (this.settingIndex < (this.settingData.length - 1)) {
        this.settingIndex++
        this.startSettingTimer();
      } else {
        this.stopSettingCheck();
      }
    })
  },
  "startSettingTimer": function (bReset = false) {
    if (bReset) {
      this.settingIndex = 0;
    } else {
      if (this.setingTimer == -1) {
        return;
      }
    }
    clearTimeout(this.setingTimer);
    this.setingTimer = setTimeout(() => {
      pointCheck.settingCheck();
    }, timerInterval);
  },
  "stopSettingCheck": function () {
    debugger
    clearTimeout(this.setingTimer);
    this.setingTimer = -1;
  },
  "pointTimer": 0,
  "pointValue": [],
  "rangePointValue": [],
  "rankFlag": 0,
  "isRplaced": [],
  "startPointTimer": function (bReset = false) {
    if (bReset) {
      this.pointValue = [];
      this.rangePointValue = [];
      this.rankFlag = 0
      this.isRplaced = [false, false, false, false]
      this.multistepPointList = []
    } else {
      if (this.pointTimer == -1) {
        return;
      }
    }
    clearTimeout(this.pointTimer);
    this.pointTimer = setTimeout(() => {
      pointCheck.checkpoint();
    }, timerInterval);
  },
  "stopPointCheck": function () {
    clearTimeout(this.pointTimer);
    this.pointTimer = -1;
  },
  "initValue": [],
  "doPoint": function () {
    let data = gaoya_list.value[curIndex]
    if (!data || !data.hasOwnProperty("point") || data.point.trim().length <= 0) return;
    let point = data.point.split(';')[1]
    let itemPromise = Promise.resolve();
    point.split(',').forEach((item, index) => {
      itemPromise = itemPromise.then(() => {
        return getIO(Number(item)).then((value) => {
          if (index < this.pointValue.length) {
            this.pointValue[index] = (this.pointValue[index] | value)
          } else {
            if (data.point.split(';')[0] == 'IT') {
              if (this.initValue.length < point.split(',').length) {
                this.initValue.push(value)
              }
            } else {
              this.pointValue.push(value)
            }
          }
        });
      })
    })
    itemPromise.then(() => {
      //check 结果，是否要保存
      if (data.point.split(';')[0] == 'IT' && this.initValue.length >= point.split(',').length) {
        clearTimeout(this.pointTimer);
      } else {
        this.startPointTimer();
      }
    })
  },
  "rangePoint": function () {
    let data = gaoya_list.value[curIndex]
    if (!data || !data.hasOwnProperty("point") || data.point.trim().length <= 0) return;
    let point = data.point.split(';')[1]
    let itemPromise = Promise.resolve();
    point.split(',').forEach((item, index) => {
      itemPromise = itemPromise.then(() => {
        return getIO(Number(item)).then((value) => {
          const getValue = item + "" + value; //36 0 0
          if (index < this.rangePointValue.length) {
            //2.第一次
            if (this.rangePointValue[index] !== getValue) {
              if (this.isRplaced[index] === false) {
                this.pointValue[index] = getValue + this.rankFlag
                this.rankFlag++
                this.isRplaced[index] = true
                this.rangePointValue[index] = getValue
              } else {
                this.pointValue[point.split(",").length + (index == 0 ? index : index - 1)] = getValue + this.rankFlag
                return; //todo
              }
            }
          } else {
            //1.push出一个初始的数组rangePointValue
            this.rangePointValue.push(getValue) //2020 361 371 381
            this.pointValue.push(getValue)
          }
        });
      })
    })
    itemPromise.then(() => {
      //check 结果，是否要保存
      this.startPointTimer();
    })
  },
  "multistepPointList": [],
  "multistepPoint": function () {//1.初步想采用假点的方式作为判分逻辑
    let data = gaoya_list.value[curIndex]
    if (!data || !data.hasOwnProperty("point") || data.point.trim().length <= 0) return;
    let point = data.point.split(';')[1]
    let itemPromise = Promise.resolve();
    const split = point.split(',');
    let thisRoundValue = []
    itemPromise = itemPromise.then(() => {
        return split.forEach((item, index) => {
          getIO(Number(item)).then((value) => {
            //1.first,

            if (value === 1) {
              thisRoundValue.push(item)
              if (thisRoundValue.length == 2) {

                if (this.multistepPointList.length < split.length) {
                  const items = thisRoundValue[0] > thisRoundValue[1] ? thisRoundValue[1] + "" + thisRoundValue[0] : thisRoundValue[0] + "" + thisRoundValue[1];
                  if (!this.multistepPointList.includes(items)) {
                    this.multistepPointList.push(items)
                  }
                }
              }
              // split.splice(index, 1).forEach((i, indexI) => {
              //     getIO(Number(i)).then((value) => {
              //         if (value == 1) {
              //             this.multistepPointList.push(index + "" + i)
              //         }
              //     })
              // })
            }

          });
        })
      }
    )
    itemPromise.then(() => {
      //check 结果，是否要保存
      this.startPointTimer();
    })
  },
  "checkpoint":
    function () {
      let data = gaoya_list.value[curIndex]
      if (!data || !data.hasOwnProperty("point") || data.point.trim().length <= 0) return;
      let point = data.point.split(';')
      if (point[0] == "O" || point[0] == "IT") {
        this.doPoint();
      }
      if (point[0] == "RO") {  //R带排序的；不可重复操作，只能操作一次对的，再操作对的值就会拼接在后面[36,0,37,0,38,0]=>[3610,3712,3813]
        this.rangePoint();
      }
      if (point[0] == "MO") { //多步骤判分
        this.multistepPoint();
      }
    }
  ,
  "getPointValue":
    function () {
      let data = gaoya_list.value[curIndex]
      let point = data.point.split(';')
      let itemPromise = Promise.resolve();
      if (point[0] != "O" && point[0] != "RO" && point[0] != "MO") {
        this.pointValue = [];
        let data = gaoya_list.value[curIndex];
        if (!data || !data.hasOwnProperty("point") || data.point.trim().length <= 0) return Promise.resolve(this.pointValue);
        let point = data.point.split(';')[1]
        point.split(',').forEach((item, index) => {
          itemPromise = itemPromise.then(() => {
            return getIO(Number(item)).then((value) => {
              if (index < this.pointValue.length) {
                this.pointValue[index] = (this.pointValue[index] | value)
              } else {
                this.pointValue.push(value)
              }
            });
          })
        })
        itemPromise = itemPromise.then(() => {
          if (data.point.split(';')[0] == 'IT') {
            let isEqual = true;
            if (this.pointValue.length !== this.initValue.length) {
              console.log("两个数组不相等");
              isEqual = false;
            } else {
              for (let i = 0; i < this.pointValue.length; i++) {
                if (this.pointValue[i] !== this.initValue[i]) {
                  isEqual = false;
                  break;
                }
              }
            }
            return Promise.resolve(this.pointValue = isEqual ? [0, 0, 0] : [1, 1, 1]);
          }
          return Promise.resolve(this.pointValue);
        })
      } else {
        if (point[0] === "RO") {
          let str = this.pointValue;
          let sortedArr = str.slice(1).sort((a, b) => a.slice(-1) - b.slice(-1));
          this.pointValue = [str[0], ...sortedArr];
        }
        if (point[0] === "MO") {
          let data = gaoya_list.value[curIndex]
          let point = data.point.split(';')[1]
          const split = point.split(',');
          for (let i = 0; i < split.length; i++) {
            for (let j = i + 1; j < split.length; j++) {
              if (this.multistepPointList.includes(split[i] + "" + split[j])) {
                this.pointValue.push(1)
              }
            }
          }
          if (this.pointValue.length != split.length) {
            this.pointValue = [0, 0, 0]
          }
        }
        itemPromise = itemPromise.then(() => {
          return Promise.resolve(this.pointValue);
        })
      }
      return itemPromise;
    }
}

const saveExamResult1 = (index) => {
  pointCheck.getPointValue().then((data) => {
    let point = "";
    data.forEach((item) => {
      point += (item + ",")
    });
    if (point.length > 0) {
      point = point.substring(0, point.length - 1);
    } else {
      point = "0";
    }

    let data1 = {
      "result": [
        {
          "point": point,
          "crit_id": gaoya_list.value[curIndex].crit_id
        }
      ],
      "disp_id": gaoyaData.disp_id
    }
    saveExamResult(data1).then((data) => {
      if (data && data.hasOwnProperty('code') && data.code == 0) {
        gaoya_list.value[index].disabled = true;
        if (gaoya_list.value.length >= index) {
          gaoya_list.value[index + 1].disabled = false;
        }
        doNext();
      }
    }).catch((error) => {
      message('请确认网络连接成功，请重试……', {customClass: 'el', type: 'error'})

    })


  })


}

function doNext() {
  //修改UI
  pointCheck.stopSettingCheck();
  pointCheck.stopPointCheck();
  curIndex++
  pointCheck.startSettingTimer(true);
  pointCheck.startPointTimer(true);
}


function getIO(iLocal) {
  console.log("getIO(" + iLocal + ")");
  if (iLocal >= fakeValueOffset) {
    return new Promise((resolve, reject) => {
      resolve(fakeValue[iLocal - fakeValueOffset]);
    });
  }
  return new Promise((resolve, reject) => {
    App.getIO({"local": iLocal}, function (data) {
      console.log("getIO(" + iLocal + "), ret:" + data.ret);
      resolve(data.ret);
    });
  });
}

function setIO(iLocal, iStatus) {
  console.log("setIO(" + iLocal + ',' + iStatus + ")");
  if (iLocal >= fakeValueOffset) {
    return new Promise((resolve, reject) => {
      fakeValue[iLocal - fakeValueOffset] = iStatus;
    });
  }
  return new Promise((resolve, reject) => {
    App.setIO({"local": iLocal, "status": iStatus}, function (data) {
      console.log("setIO=" + data.ret);
      if (data.ret == false) {
        // reportDeviceFault(iLocal, '1');
      }
      resolve(data.ret);
    });
  });
}


const loading = ref(true)
const loadText = ref('获取数据中...')


function goto(event) {
  centerDialogVisible.value = true;
}


function getDeviceId() {
  return new Promise((resolve, reject) => {
    App.getDeviceId({}, function (data) {
      console.log("getDeviceId11" + data.ret)
      resolve(data.ret);
    })
  })
}

const centerDialogVisible = ref(false) //todo 默认true

function saveExamTime(param) {
  loading.value = true;
  loadText.value='成绩提交中...'
  if (param == 0) {
    centerDialogVisible.value = false
    return
  }
  pointCheck.stopSettingCheck();
  pointCheck.stopPointCheck();
  let data = {
    "exam_no": exam_no1,
    "spend_time": new Date().getTime() - startTime
  }
  commitExam(data).then(() => {
    console.log(data);
    submitVideoData("1",true);
  }).catch(() => {
    message('请确认网络连接成功，请重试……', {customClass: 'el', type: 'error'})
  })
}


</script>
<template>
  <div id="app" v-loading="loading" :element-loading-text="loadText">

    <el-dialog v-model="centerDialogVisible" title="提示" width="30%" center :modal="false" :closeOnClickModal="false"
               :showClose="false" style="--el-dialog-bg-color:rgba(255,255,255,1)">
    <span style="font-size: 20px;text-align: center">
      是否提交考试结果?
    </span>
      <template #footer>
      <span class="dialog-footer">
        <el-button type="danger" @click="saveExamTime(0)">
          取消
        </el-button>
         <el-button type="primary" @click="saveExamTime(1)">
          确认
        </el-button>
      </span>
      </template>
    </el-dialog>

    <div class="page3-head">
      <span class="page3-head-span">高压电工考核操作票</span>
    </div>
    <div class="page3-second">
      <el-row>
        <el-col :span="8" class="page3-second-col">发令人:{{ secondPartInfo.examinee_a_name }}</el-col>
        <el-col :span="8" class="page3-second-col">受令人:{{ secondPartInfo.examinee_b_name }}</el-col>
        <el-col :span="8" class="page3-second-col">发令时间:{{ secondPartInfo.create_time }}</el-col>
      </el-row>
      <el-row>
        <el-col :span="8" class="page3-second-col">操作开始时间:{{ secondPartInfo.start_time }}</el-col>
        <el-col :span="8" class="page3-second-col"></el-col>
        <el-col :span="8" class="page3-second-col">操作结束时间</el-col>
      </el-row>
    </div>
    <div class="page3-second-task">
      <el-row>
        <el-col :span="22">
          <div class="grid-content">操作任务:{{ secondPartInfo.operate_task }}</div>
        </el-col>
        <el-col :span="2">
          <el-button type="primary" style="background: #8dd4d0;margin-top: 4.5%;font-size: 26px" @click="goto">交卷
          </el-button>
        </el-col>
      </el-row>
    </div>

    <div class="page3-table">
      <el-table :data="gaoya_list"
                style="width: 99.6%;margin-left: 0.2%;border-radius: 10px;font-size: 26px;color: black"
                height="570"
                :header-cell-style="{ fontSize: '25px', fontWeight: 'bold', textAlign:'center',backgroundColor:'#1eb4ac',paddingTop:'11px',color:'#fff'}">
        <el-table-column fixed prop="crit_desc" label="请选择操作项目" width="1440"/>
        <el-table-column fixed="right" label="操作" width="100">
          <template #default="scope">
            <el-button
              :disabled="scope.row.disabled"
              style="font-size:24px;"
              type="primary"
              size="large"
              @click.prevent="saveExamResult1(scope.$index)"
            >
              确认
            </el-button>
          </template>
        </el-table-column>
      </el-table>
    </div>


  </div>

</template>
<style lang="scss" scoped>
#app {
  background-color: #39a091
}

.page3-head {
  width: 100%;
  text-align: center;
  padding-top: 2%;
}

.page3-head-span {
  font-size: 50px;
  color: #fff;
  font-weight: bold;
}

.page3-second {
  background: white;
  border-radius: 10px;
  font-size: 26px;
  border: 1px #cccccc solid;
}

.page3-second-col {
  padding-top: 0.2%;
  padding-bottom: 0.2%;
  padding-left: 1.5%;
  border-bottom: 1px solid #cccccc;
  border-right: 1px solid #cccccc;
}

.radio {
  width: 1.2rem;
  height: 1.2rem !important;
  background-color: #ffffff;
  border: solid 1px #dddddd;
  -webkit-border-radius: 0.6rem;
  border-radius: 0.6rem;
  font-size: 0.8rem;
  margin: 0;
  padding: 0;
  position: relative;
  display: inline-block !important;
  vertical-align: top;
  cursor: default;
}

.btn-disable {
  color: white;
  pointer-events: none;
  margin-top: 1%;
  margin-bottom: 1%
}

.page3-second-task {
  margin-top: 2%;
  width: 100%;
  height: 5%;
  background: white;
  border-radius: 10px;
}

.grid-content {
  font-size: 26px;
  margin-left: 1.5%;
}

.icon_open1 {
  height: 45px;
  width: 45px;
  background: url("src/assets/height/2-1.jpg");
  position: absolute;
  top: 15%;
  left: 35.9%;
}

.icon_close1 {
  height: 45px;
  width: 45px;
  background: url("src/assets/height/2-2.jpg");
  position: absolute;
  top: 15%;
  left: 35.9%;
}

.icon_open2 {
  height: 68px;
  width: 68px;
  background: url("src/assets/height/3-1.jpg");
  position: absolute;
  top: 30%;
  left: 32.6%;
}

.icon_close2 {
  height: 68px;
  width: 68px;
  background: url("src/assets/height/3-2.jpg");
  position: absolute;
  top: 30%;
  left: 32.6%;
}

.icon_open3 {
  height: 45px;
  width: 45px;
  background: url("src/assets/height/4-1.jpg");
  position: absolute;
  top: 49%;
  left: 35.8%;
}

.icon_close3 {
  height: 45px;
  width: 45px;
  background: url("src/assets/height/4-2.jpg");
  position: absolute;
  top: 49%;
  left: 35.8%;
}

.icon_open4 {
  height: 52px;
  width: 52px;
  background: url("src/assets/height/6-1.jpg");
  position: absolute;
  top: 28.7%;
  left: 20.2%;
}

.icon_close4 {
  height: 51px;
  width: 50px;
  background: url("src/assets/height/6-2.jpg");
  position: absolute;
  top: 29.7%;
  left: 19.1%;
}

.icon_open5 {
  height: 40px;
  width: 5px;
  background: url("src/assets/height/7-1.jpg");
  position: absolute;
  top: 82%;
  left: 32.5%;
}

.icon_close5 {
  height: 32px;
  width: 32px;
  background: url("src/assets/height/7-2.jpg");
  position: absolute;
  top: 83.6%;
  left: 25.3%;
}

.page3-table {
  width: 100%;
  height: 60%;
  margin-top: 2%;
}

::v-deep .el-table__header-wrapper {
  height: 40px;
}

::v-deep .el-table .el-table__body td {
  padding-top: 1%;
  height: 70px;
  line-height: 40px;
}

::v-deep .el-table .cell {
  overflow: visible;
}

:deep() .el-dialog__body {
  text-align: center;
}
</style>
