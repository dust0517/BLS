<template>
  <div>
      <el-row :gutter="10">
        <el-col :span="24" style="margin-bottom:10px;">
          <el-alert type="warning" title="【警告】" :closable="false">此处的日志将触发界面重绘，非常影响性能，建议使用<a @click="jmpToAbout" style="color:#6494ff;text-decoration:underline;font-weight:bold;">开发者工具</a></el-alert>
        </el-col>
        <el-col :span="8">
            <el-checkbox label="错误" border size="small" v-model="control.error"></el-checkbox>
            <el-checkbox label="成功" border size="small" v-model="control.success"></el-checkbox>
            <el-checkbox label="信息" border size="small" v-model="control.info"></el-checkbox>
        </el-col>
        <el-col :span="8">
          <el-switch v-model="allowLog" active-text="允许日志"></el-switch>
        </el-col>
        <el-col :span="4">
            <el-button type="primary" size="mini" plain @click="clear">清空</el-button>
        </el-col>
        <el-col :span="4">
            <el-tag type="warning" size="medium">日志数量:{{msg.length}}</el-tag>          
        </el-col>
        <el-col :span="24">
          <div class="info">
            <el-alert 
            v-if="control[m.type]" 
            v-for="m in msg" 
            :key="m.msg+m.time+Math.random()" 
            :type="m.type" 
            :title="m.time" 
            :closable="false"
            > {{m.msg}}</el-alert>
          </div>
        </el-col>
      </el-row>
  </div>
</template>
<script>


export default {
  name: "ResponseCenter",
  data() {
    return {
      allowLog:false,
      msg: [],
      control: {
        error: true,
        success: true,
        info:false,
      },
      giftconfig:{
        status:false,
        giftdata:[],
      }
    };
  },
  mounted() {
    this.AddListener();
    this.initGiftConfig();
    // this.KitMsg("初始化完毕");
  },
  methods: {
    jmpToAbout(){
      this.$eve.emit("selectTab","关于");
    },
    async initGiftConfig(){
      try{
        let rq = await this.$api.origin({
          uri:"https://api.live.bilibili.com/gift/v3/live/gift_config",
          method:"get",
          headers:{
            Origin: "https://live.bilibili.com",
            Referer: "https://live.bilibili.com/3"
          }
        });
        this.giftconfig.giftdata = rq.data;
        this.giftconfig.status = true;
      }catch(e){
        this.$eve.emit("error",`礼物列表初始化异常:${e.message}`);
      }
      
      
    },
    getRealGiftType(msg){
      for(let gift of this.giftconfig.giftdata){
        if(msg.indexOf(gift.name)>=0 && gift.broadcast ){ // 只判断可广播礼物
          return gift.name;
        }
      }
      return "小电视";
    },
    AddListener() {
      this.$eve.on("error", message => {
        this.KitMsg(message, "error");
      });
      this.$eve.on("info", message => {
        this.KitMsg(message);
      });
      this.$eve.on("giftCount", (user,name,number,type) => {
        this.KitMsg(`${user} 从${type}获取了 ${number} 个 ${name}`);
      });
      this.$eve.on("success", message => {
        this.KitMsg(message, "success");
      });
      this.$eve.on("dm_raffle",data=>{
        this.KitMsg(`房间 [${data.roomid}] 开启抽奖`);
      });
      this.$eve.on("dm_SmallTv",data=>{
        let type = this.getRealGiftType(data.msg);
        this.KitMsg(`房间 [${data.roomid}] ${type}`);
      });
      this.$eve.on("user_validate",user=>{
        this.revalidate(user);
      });
    },
    async revalidate(user){
      try{
        await user.RefreshCookie();
        this.$eve.emit("success",`${user.name} cookies更新成功`)
      }catch(e){
        try{
          await user.RefreshToken();
          this.$eve.emit("success",`${user.name} Token更新成功`)
        }
        catch(e){
          user.isLogin=false;
          this.$eve.emit("error",`Token更新失败: ${e.message}`);
        }
      }
      
    },
    clear(){
      this.msg=  [];
    },
    KitMsg(msg, type = "info") {
      
      let time = this.formatTime(new Date(),"HH:mm:ss");
      
      console.log(`${time} :　${msg}`);
      if(!this.allowLog){ //判断是否允许日志
        return ;
      }
      this.msg.unshift({
        time,
        type,
        msg
      });
      if(this.msg.length>1000){
        /* 只保留最近1k条日志 */
        this.msg.pop();
      }
    },
    formatTime(date = new Date(), fmt = "YYYY-MM-DD HH:mm:ss") {
      date = typeof date === "number" ? new Date(date) : date;
      var o = {
        "M+": date.getMonth() + 1,
        "D+": date.getDate(),
        "h+": date.getHours() % 12 === 0 ? 12 : date.getHours() % 12,
        "H+": date.getHours(),
        "m+": date.getMinutes(),
        "s+": date.getSeconds(),
        "q+": Math.floor((date.getMonth() + 3) / 3),
        S: date.getMilliseconds()
      };
      var week = {
        "0": "\u65e5",
        "1": "\u4e00",
        "2": "\u4e8c",
        "3": "\u4e09",
        "4": "\u56db",
        "5": "\u4e94",
        "6": "\u516d"
      };
      if (/(Y+)/.test(fmt)) {
        fmt = fmt.replace(
          RegExp.$1,
          (date.getFullYear() + "").substr(4 - RegExp.$1.length)
        );
      }
      if (/(E+)/.test(fmt)) {
        fmt = fmt.replace(
          RegExp.$1,
          (RegExp.$1.length > 1
            ? RegExp.$1.length > 2 ? "\u661f\u671f" : "\u5468"
            : "") + week[date.getDay() + ""]
        );
      }
      for (var k in o) {
        if (new RegExp("(" + k + ")").test(fmt)) {
          fmt = fmt.replace(
            RegExp.$1,
            RegExp.$1.length === 1
              ? o[k]
              : ("00" + o[k]).substr(("" + o[k]).length)
          );
        }
      }
      return fmt;
    }
  } //因为这个大括号放错位置，调试了三个小时 TAT
};
</script>
<style scoped>
.info{
  margin-top: 20px;
  line-height: 20px;
  font-size: 15px;
  font-family: "微软雅黑","Ping Fang SC",sans-serif;
}
.info .el-alert{
  margin:5px 0;
}
</style>
