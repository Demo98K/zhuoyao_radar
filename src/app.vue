<template>
  <div id="app">
    <right-nav :version="APP_VERSION" :settings="settings"></right-nav>
    <div id="status" v-show="debug">
      <div id="info"><br/><span v-html="status"></span></div>
    </div>
    <div id="buttons">
      <el-button size="mini" @click="getYaolingInfo">妖灵</el-button>
      <el-button size="mini" @click="getLeitaiInfo">擂台(未开发)</el-button>
      <el-button size="mini" type="warning" @click="debug = !debug">Debug</el-button>
    </div>
    <div id="qmap"></div>
  </div>
</template>
<script>
import tempdata from './components/tempdata';
import mixins from './components/mixins';
import RadarWebSocket from './components/socket';
import RightNav from './components/rightNav';
import {
  getSetting,
  setSetting,
  utf8ByteToUnicodeStr,
  convertLocation,
  json2buffer
} from './components/util';

import {
  FILTER,
  API_KEY,
  CUR_YAOLING_VERSION,
  APP_VERSION,
  BOT
} from './config';

export default {
  name: 'zhuoyao-radar',
  mixins: [mixins],
  components: {
    RightNav
  },
  data() {
    return {
      APP_VERSION,
      showNav: false, // 左侧菜单栏
      unknownKey: [],
      status: '',
      socket: {},
      map: {},
      debug: false,
      clickMarker: null, // 点击位置标记
      userMarker: null, // 用户位置标记
      firstTime: true, // 首次连接socket标记
      currVersion: CUR_YAOLING_VERSION, //190508版本的json 如果有变动手动更新
      location: {
        longitude: 116.3579177856,
        latitude: 39.9610780334
      },
      statusOK: false,
      yaolings: tempdata.Data,
      markers: [],
      messageMap: new Map(), // 缓存请求类型和id
      settings: {
        fit: {
          t1: false,
          t2: false,
          all: false,
          nest: false,
          rare: true,
          fish: false,
          feature: false,
          element: false
        },
        auto_search: false
      },
      botMode: false,
      botInterval: null,
      botTime: 0,
      botURL: 'http://127.0.0.1:36524/api/v1/Cqp/CQ_sendGroupMsg',
      botGroup: '799576270',
      botChecked: []
    };
  },
  created() {
    let settings = getSetting();
    if (settings) {
      this.settings = settings;
    }
  },
  mounted() {
    // 初始化地图组件
    this.initMap();

    // 初始化websocket
    this.socket = new RadarWebSocket({
      onopen: this.onSocketOpen,
      onmessage: this.onSocketMessage
    });

    // 获取用户位置
    this.getLocation()
      .then(
        position => {
          this.location.longitude = position.longitude;
          this.location.latitude = position.latitude;

          var pos = new qq.maps.LatLng(
            this.location.latitude,
            this.location.longitude
          );
          this.map.panTo(pos);
          this.userMarker = new qq.maps.Marker({
            position: pos,
            map: this.map
          });
        },
        e => {
          console.log(e);
        }
      )
      .catch(b => {});

    this.addStatus(`捉妖雷达Web版 <br/>
      版本:${APP_VERSION} <br/>
      更新日志:<br/>
      丰富筛选库，优化界面<br/>
      点击地图自动筛选功能<br/>`);

    //this.addStatus("开发者:ZachXia,Vitech");
    setTimeout(() => {
      this.notify('提示:点击右下角菜单开始筛选！');
    }, 2000);
  },
  methods: {
    /**
     * todo
     */
    ajaxGroupMessage: function(mes) {
      $.post(
        'request.php',
        {
          qq: this.botGroup,
          msg: mes
        },
        function(data) {
          console.log(data);
        }
      );
    },

    /**
     * 跨域获取最新妖灵数据
     */
    getYaolings: function() {
      // var url = 'https://hy.gwgo.qq.com/sync/pet/config/' + this.currVersion;
      // try {
      //     $.getJSON(url,result => {
      //         console.log(result);
      //         this.statusOK = true;
      //     });
      //     console.log("1");
      // } catch(e) {
      //     console.log(e);
      //     console.log("3");
      // }
      // console.log("2");
      this.statusOK = true;
    },
    /**
     * 缓存响应的类型和id
     */
    genRequestId: function(type) {
      let _time = new Date().getTime() % 1234567;
      this.messageMap.set(`msg_${_time}`, type);
      return _time;
    },
    /**
     * 根据id找到请求的类型
     */
    getRequestTypeFromId: function(id) {
      return this.messageMap.get(id);
    },

    onSocketOpen: function() {
      this.addStatus('WSS连接开启');
      console.log('WSS连接开启');
      // 首次连接
      if (this.firstTime) {
        this.firstTime = false;
        this.getSettingFileName();
        this.getBossLevelConfig();
      }
    },
    /**
     * 消息响应
     */
    onSocketMessage: function(event) {
      var blob = event.data;

      if (typeof blob !== 'string') {
        var fileReader = new FileReader();
        fileReader.onload = e => {
          var arrayBuffer = e.target.result;
          var n = utf8ByteToUnicodeStr(new Uint8Array(arrayBuffer).slice(4));

          var data = JSON.parse(n);

          this.handleMessage(data);
        };
        fileReader.readAsArrayBuffer(blob);
      }
    },
    /**
     * 处理消息
     */
    handleMessage: function(data) {
      var _type = this.messageMap.get(`msg_${data.requestid}`);
      if (_type) {
        this.messageMap.delete(`msg_${data.requestid}`);
      }

      switch (_type) {
        case '10041':
          this.getVersionFileName(data.filename);
          break;
        case '10040':
          //console.log("妖灵等级", n), a.saveBossStartAndEndLevel(n.startlevel, n.endlevel);
          break;
        case '1001':
          console.log('获取到妖灵数量', data.sprite_list.length);
          if (this.botMode) {
            // 机器人
            this.botAnalyze(data.sprite_list);
          } else {
            this.buildMarkersByData(data.sprite_list);
          }
          break;
        case '1002':
        //this.getLeitaiNearby(data);
      }
    },
    getVersionFileName: function(name) {
      if (name != this.currVersion) {
        this.getYaolings(name);
        console.info('有新版本的icon!');
        this.notify('有新版本的妖灵库，请通知作者更新！！');
      } else {
        this.getYaolings(this.currVersion);
      }
    },

    /**
     * 根据查询结果过滤数据，打标记
     */
    buildMarkersByData: function(t) {
      this.clearAllMarkers();

      t.forEach(item => {
        if (
          this.fit[0] === 'special' ||
          this.fit.indexOf(item.sprite_id) > -1
        ) {
          var m = this.buildSpiteMarker(item);
          var icon = new qq.maps.MarkerImage(
            m.iconPath,
            null,
            null,
            null,
            new qq.maps.Size(40, 40)
          );
          var marker = new qq.maps.Marker({
            position: new qq.maps.LatLng(m.latitude, m.longitude),
            map: this.map
          });
          marker.setIcon(icon);

          this.markers.push(marker);
        }
      });
      this.notify('筛选成功!');
    },
    buildSpiteMarker: function(ti) {
      var icon = this.getHeadImagePath(ti);
      return {
        latitude: ti.latitude / 1e6,
        longitude: ti.longtitude / 1e6,
        iconPath: icon,
        width: 40,
        height: 40
      };
    },
    //根据妖灵信息获取其icon地址
    getHeadImagePath: function(e) {
      var a = this.getYaolingById(e.sprite_id);
      if (a) {
        return `./original/image/head/${a.ImgName}.png`;
      } else {
        return './original/image/default-head.png';
      }
    },
    addStatusWithoutNewline: function(str) {
      this.status += str;
    },
    addStatus: function(str) {
      this.status += str + '<br>';
    },
    sendMessage: function(message, requestIndex) {
      if (message.request_type != '1004') {
        this.addStatusWithoutNewline('WSS发送消息：');
        this.addStatus(JSON.stringify(message));
      }
      console.log('sendMessage', message);

      this.socket.send(json2buffer(message));
    },
    /**
     * 获取妖灵数据
     */
    getYaolingInfo: function() {
      if (!this.statusOK || this.botMode) return;
      var e = {
        request_type: '1001',
        longtitude: convertLocation(this.location.longitude),
        latitude: convertLocation(this.location.latitude),
        requestid: this.genRequestId('1001'),
        platform: 0
      };
      this.sendMessage(e, '1001');
    },
    /**
     * 获取擂台数据
     */
    getLeitaiInfo: function() {
      this.addStatus('功能开发中!');
      this.notify('功能开发中!');
      return;
      if (!this.statusOK || this.botMode) return;
      var e = {
        request_type: '1002',
        longtitude: convertLocation(this.location.longitude),
        latitude: convertLocation(this.location.latitude),
        requestid: this.genRequestId('1002'),
        platform: 0
      };
      this.sendMessage(e, '1002');
    },
    getYaolingBot: function() {
      this.botTime++;
      var e = {
        request_type: '1001',
        longtitude: convertLocation(this.location.longitude),
        latitude: convertLocation(this.location.latitude),
        requestid: this.genRequestId('1001'),
        platform: 0
      };
      this.sendMessage(e, '1001');
    },
    getYaolingById: function(id) {
      return this.yaolings.find(item => {
        return item.Id === id;
      });
    },
    getSettingFileName: function() {
      var e = {
        request_type: '1004',
        cfg_type: 1,
        requestid: this.genRequestId('10041'),
        platform: 0
      };
      this.sendMessage(e, '10041');
    },
    getBossLevelConfig: function() {
      return;
      var e = {
        request_type: '1004',
        cfg_type: 0,
        requestid: this.genRequestId('10040'),
        platform: 0
      };
      this.sendMessage(e, '10040');
    }
  },
  computed: {
    fit: function() {
      let ans = [];
      let _fit = this.settings.fit;
      if (_fit.all) {
        return ['special'];
      }

      // 根据值把key转换成FILTER_FISH这种，取常量配置中的值
      for (let _f in _fit) {
        if (_fit[_f]) {
          let _arr = FILTER[`FILTER_${_f.toLocaleUpperCase()}`];
          ans = ans.concat(_arr);
        }
      }
      return Array.from(new Set(ans));
    }
  },
  watch: {
    settings: {
      handler: function(newV, oldV) {
        console.log('settings update...');
        setSetting(this.settings);
      },
      deep: true
    }
  }
};
</script>
<style lang='less'>
</style>

