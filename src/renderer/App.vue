<template style="margin: 0px;">
    <el-container style="height: 100%; border: 1px solid #eee">


        <el-aside width="210px" v-show="showPlayList">
            <el-menu class="el-menu-vertical-demo" :unique-opened="true">
                <h3 style="margin-left: 25px;" @click="reloadList(null)">节目列表</h3>
                <el-submenu :index="(i+1)+''" v-for="(item,i) in playerList">
                    <template slot="title">
                        <i class="el-icon-location"></i>
                        <span slot="title">{{item.groupName}}</span>
                    </template>
                    <el-menu-item-group>
                        <span slot="title"></span>
                        <el-menu-item  style="padding-left: 30px;" :index="i+'-'+j" @click="playOne(jtem.url)"
                                       :uu="jtem.url"
                                       @mouseup.native="itemMouseUp"
                                       v-for="(jtem,j) in item.list"><i class="el-icon-caret-right"></i>{{jtem.name}}
                        </el-menu-item>
                    </el-menu-item-group>
                </el-submenu>
            </el-menu>
        </el-aside>


        <el-container>
            <el-dialog
                    style="z-index:99999"
                    title="源文本"
                    :visible="dialogVisible"
                    width="70%"
                    :append-to-body="true"
                    center>
                <el-input
                        type="textarea"
                        :autosize="{ minRows: 8, maxRows: 10}"
                        placeholder="输入源(节目,链接)"
                        v-model="textContent">
                </el-input>

                <span slot="footer" class="dialog-footer">
                <el-button @click="dialogVisible = false">取 消</el-button>
                <el-button type="primary" @click="processM3u8Text">确 定</el-button>
          </span>
            </el-dialog>
            <el-main v-loading="loading" style="background-color: #000000">
                <section>
                    <div id="wrapper" style="width: 100%;height: 100%;">
                    </div>
                </section>
            </el-main>
        </el-container>
    </el-container>

</template>

<script>

  import Chimee from 'chimee';
  import hls from 'chimee-kernel-hls';

  const ipcRenderer = require("electron").ipcRenderer
  const fs = require("fs")
  var request = require("request")

  var starting = false


  export default {
    components: {

    },
    props: {
      source: {
        type: String,
        default: ""
      }
    },
    data() {
      return {
        showPlayList:true,

        textContent: "",
        dialogVisible: false,
        chimee:null,
        options: {
          video: {
            url: ""
          },
          contextmenu: [
            {}
          ]
        },
        loading: false,
        hls: "",
        playerList: [
          {
            groupName: "央视",
            list: [
              { name: "CCTV1", url: "http://144.48.240.144:2019/SunTv.php?name=CCTV1HD" },
              { name: "CCTV2", url: "http://144.48.240.144:2019/SunTv.php?name=CCTV2HD" }
            ]
          }
        ]


      }
    },
    mounted() {
      this.reloadList(null)

      this.chimee = new Chimee({
        wrapper: '#wrapper',
        controls: true,
        autoplay: true,
        autoload:false,
        kernels: {
          hls
        }
      });
      //全屏事件
      this.chimee.$watch('isFullscreen', (newVal, oldVal) => {
            var isFull = !oldVal;
            console.log(isFull);
        ipcRenderer.send("fullEvent",isFull);
      }, {deep: true});
      //视频右键
      this.chimee.on('mousedown', evt => {
            if(evt.button == 2){
                console.log(evt,thiz.showPlayList);
                ipcRenderer.send("clickRightEvent", { menuHideStatus:!thiz.showPlayList });

            }
      }, { target: 'container' });

      const thiz = this;
      //切换列表状态
      ipcRenderer.on("togglePlayListShow", function(event, message) {
        thiz.showPlayList = !thiz.showPlayList;
      })
      //触发输入框
      ipcRenderer.on("selectedFile", function(event, message) {
        const path = message.path[0]
        const data = fs.readFileSync(path, "utf-8")
        thiz.loadData(data)
      })

      ipcRenderer.on("inputText", function(event, message) {
        thiz.dialogVisible = true
      })
      ipcRenderer.on("toInputUrl", function(event, message) {
        thiz.$prompt("请输入在线源", "提示", {
          confirmButtonText: "确定",
          cancelButtonText: "取消"
        }).then(({ value }) => {
          if (!value.startsWith("http")) {
            return
          }
          thiz.reloadList(value)

        }).catch(() => {

        })
      })
      ipcRenderer.on("inputM3U8", function(event, message) {

        thiz.$prompt("请输入M3U8", "m3u8", {
          confirmButtonText: "确定",
          cancelButtonText: "取消"
        }).then(({ value }) => {
          if (!value.startsWith("http")) {
            return
          }
          thiz.playOne(value)

        }).catch(() => {

        })
      })
    },
    methods: {
      processM3u8Text() {
        this.dialogVisible = false
        var content = this.textContent.trim()
        if (!content || content.length == 0) {
          return
        }
        let list = content.split("\n")
        let line = list[0]
        if (!line.includes("#genre#")) {
          content = "未分类,#genre#\n" + content

        }
        this.loadData(content)

      },
      reloadList(url) {
        let thiz = this
        if (url == null) {
          url = "http://dd.laigc.com:10080/linsongze/tv/raw/branch/master/m.txt?xxx=" + Math.random()
        } else {
          if (url.includes("?")) {
            url = url + "&xxx=" + Math.random()
          } else {
            url = url + "?xxx=" + Math.random()
          }

        }
        this.$axios.get(url).then(res => {
          let content = res.data
          thiz.loadData(content)
        })
      },
      loadData(content) {
        let list = content.split("\n")
        list = list.filter(s => s.trim().length > 0)
        let rs = []
        let groupName = null
        let group = {}
        for (let line of list) {
          line = line.trim()
          if (line.includes("#genre#")) {//新分组
            groupName = line.split(",")[0]
            group = { groupName: groupName, list: [] }
            rs.push(group)
          } else {
            if (groupName == null) continue
            if (!line.includes(",")) {
              continue
            }

            let ss = line.split(",")
            group.list.push({ name: ss[0], url: ss[1] })

          }
        }
        this.playerList = rs
      },
      itemMouseUp(a){
        var e = event || window.event;
        let nType = e.button;

        if (2 === nType) {

          ipcRenderer.send("clickRightEvent",{uu:a.srcElement.getAttribute("uu")})
        }
        e.stopPropagation();

      },
      playOne(url) {
        if (starting) {
          return;
        }
        starting = true;//控制重重复点击
        var thiz = this;
        var r = request.get(url, function(err, res, body) {//获取最终重定向的地址再进行播放
          console.log(url = res.request.uri.href)
          if (err == null) {
            url = res.request.uri.href
            thiz.getStream(url)
          }else{
            alert("资源有问题无法打开");
          }
          starting = false;
        });

      },
      getStream(source) {
        this.chimee.load(source);
        this.chimee.play(); // play!!
      }

    }
  }
</script>

<style>
    .el-header {
        background-color: #B3C0D1;
        color: #333;
        line-height: 50px;
    }

    .el-main {
        padding: 0px;
    }

    .el-menu-vertical-demo:not(.el-menu--collapse) {
        width: 200px;
        min-height: 400px;
        height: 100%;
    }

    .el-aside {
        color: #333;
    }

    html, body {
        margin: 0;
        padding: 0;
        height: 100%;
    }



    container {
        position: relative;
        display: block;
        width: 100%;
        height: 100%;
    }
    video {
        width: 100%;
        height: 100%;
        display: block;
        background-color: #000;
    }
    video:focus,
    video:active {
        outline: none;
    }
</style>
