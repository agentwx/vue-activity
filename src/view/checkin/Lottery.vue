<template>
  <div class="container">
    <div class="title"></div>
    <div class="tip">签到抽奖 把好礼带回家</div>
    <div class="lucky-wheel">
      <div class="wheel-main">
        <div class="wheel-pointer-box">
          <div class="wheel-pointer"
               @click="rotateHandle()"
               :style="{transform:rotateAnglePointer,transition:rotateTransitionPointer}">
            <div class="lottery-start">开始</div>
            <div class="lottery-ticket">(您有{{ lotteryTicket }}次机会)</div>
          </div>
        </div>
        <div class="wheel-bg"
             :style="{transform:rotateAngle,transition:rotateTransition}">
          <div class="prize-list">
            <div class="prize-item"
                 v-for="(item,index) in prizeList"
                 :key="index">
              <div class="prize-type">
                {{item.prizeName}}
              </div>
              <div class="prize-pic">
                <img :src="item.prizeImg"
                     :alt="item.prizeName">
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
    <!-- 抽奖信息弹框 -->
    <info-dialog :show="toastControl"
                 :key="toastControl"
                 :info="recordInfo"
                 @close="closeToast"></info-dialog>
  </div>
</template>
<script>
import InfoDialog from './children/Dialog'
export default {
  components: { InfoDialog },
  data () {
    return {
      lotteryTicket: 0, //抽奖次数
      prizeList: [], //奖品列表
      toastControl: false, //抽奖结果弹出框控制器
      hasPrize: false, //是否中奖
      startRotatingDegree: 0, //初始旋转角度
      rotateAngle: 0, //将要旋转的角度
      startRotatingDegreePointer: 0, //指针初始旋转角度
      rotateAnglePointer: 0, //指针将要旋转的度数
      rotateTransition: 'transform 6s ease-in-out', //初始化选中的过度属性控制
      rotateTransitionPointer: 'transform 12s ease-in-out', //初始化指针过度属性控制
      clickFlag: true, //是否可以旋转抽奖
      index: '', // 指定每次旋转到的奖品下标
      recordInfo: {} // 弹框中奖信息
    }
  },
  created () {
    this.init()
  },
  methods: {
    // 初始化
    init () {
      this.getPrizeList()
      this.getLotteryTimes()
    },

    // 获取奖品列表
    getPrizeList (callBack) {
      this.$api.checkin.getPrize()
        .then(res => {
          if (res && res.length > 0) {
            let arr = []
            for (let item of res) {
              arr.push({
                id: item.id,
                prizeName: item.prize_name,
                prizeImg: item.prize_image,
                selected: item.selected,
                winning: item.winning
              })
            }
            this.prizeList = arr
            if (callBack) {
              callBack()
            }
          }
        })
    },

    // 获取该天的抽奖次数
    getLotteryTimes () {
      const date = this.$route.query.date
      this.$api.checkin.getCheckin()
        .then(res => {
          if (res && res.length > 0) {
            for (let item of res) {
              let time = this.dateFormatter(item.check_in_time)
              if (date === time) {
                // 该天已连续签到七天, 并且为待抽奖状态, 则可以抽奖一次
                this.lotteryTicket = (item.check_in_times === 7 && item.status === 1) ? 1 : 0
              }
            }
          }
        })
    },

    // 时间格式转化 例如: 2019-06-08 => 201968
    dateFormatter (date) {
      let arr = date.split('-')
      let str = '' + arr[0] + Number(arr[1]) + Number(arr[2])
      return str
    },

    // 点击开始
    rotateHandle () {
      if (!this.clickFlag) return

      if (this.lotteryTicket <= 0) {
        this.$toast('您的抽奖次数用完了哦!')
        return false
      }

      let len = this.prizeList.length
      if (len === 0) {
        this.$toast('奖品不存在,无法抽奖!请联系管理员')
        return false
      }

      this.$api.checkin.winPrize()
        .then(res => {
          for (let i = 0; i < len; i++) {
            if (this.prizeList[i].id === res.prize_id) {
              this.index = i
              break
            }
          }
          if (this.index !== '') {
            this.rotating()
          } else {
            this.$toast('请稍后再试')
          }
        })
    },

    rotating () {
      var type = 0 // 默认为 0  转盘转动 1 箭头和转盘都转动(暂且遗留)
      var duringTime = 5 // 默认为1s
      // var random = Math.floor(Math.random() * 7)
      var resultIndex = this.index // 最终要旋转到哪一块，对应 prizeList 的下标
      var resultAngle = [3, 300, 240, 180, 120, 60] //最终会旋转到下标的位置所需要的度数
      var randCircle = 6 // 附加多转几圈，2-3
      this.clickFlag = false // 旋转结束前，不允许再次触发
      if (type == 0) {
        // 转动盘子
        var rotateAngle =
          this.startRotatingDegree +
          randCircle * 360 +
          resultAngle[resultIndex] -
          this.startRotatingDegree % 360
        this.startRotatingDegree = rotateAngle
        this.rotateAngle = 'rotate(' + rotateAngle + 'deg)'
        // // //转动指针
        // this.rotateAnglePointer = 'rotate(' + this.startRotatingDegreePointer + 360 * randCircle + 'deg)'
        // this.startRotatingDegreePointer = 360 * randCircle
        var that = this
        // 旋转结束后，允许再次触发
        setTimeout(function () {
          that.clickFlag = true
          that.gameOver()
        }, duringTime * 1000 + 1500) // 延时，保证转盘转完
      } else {
        //
      }
    },

    // 结束
    gameOver () {
      const data = this.prizeList[this.index]
      this.recordInfo = {
        prizeName: data.prizeName,
        hasPrize: data.winning,
        infoImg: data.winning ? data.prizeImg : '',
        buttonText: data.winning ? '立即领取' : '返回首页'
      }
      this.lotteryTicket--
      this.toastControl = true
      if (!data.winning) {
        // 提交未中奖结果
        const params = {
          prize_name: data.prizeName,
          prize_img: data.prizeImg,
          prize_status: 0,
          contact_name: '',
          contact_phone: '',
          province: '',
          city: '',
          district: '',
          address: ''
        }
        this.$api.checkin.postWinPrize(params)
          .then(res => {
            if (res.errorCode && res.errorCode !== 0) {
              this.$toast(res.message)
            }
          })
      }

    },

    //关闭弹窗
    closeToast () {
      this.toastControl = false
    }
  }
};
</script>
<style lang="less" scoped>
.container {
  position: absolute;
  top: 86px;
  left: 0;
  right: 0;
  bottom: 0;
  overflow: hidden;
  .bg-img("~@images/checkin/pic_checkin_background2");
  .title {
    width: 594px;
    height: 124px;
    margin: 44px auto 0;
    .bg-img("~@images/checkin/pic_lottery_font");
  }
  .tip {
    height: 60px;
    font-size: 42px;
    font-weight: 500;
    color: rgba(248, 250, 181, 1);
    line-height: 60px;
    text-align: center;
    margin-top: 16px;
  }
}
.lucky-wheel {
  width: 100%;
  padding-top: 66px;
}
.wheel-main {
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
}
.wheel-bg {
  width: 750px;
  height: 750px;
  .bg-img("~@images/checkin/pic_checkin_round");
  color: #fff;
  font-weight: 500;
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-content: center;
  transition: transform 3s ease;
}
.wheel-pointer-box {
  position: absolute;
  top: 50%;
  left: 50%;
  z-index: 100;
  transform: translate(-50%, -50%);
  width: 232px;
  height: 232px;
}
.wheel-pointer {
  width: 200px;
  height: 232px;
  margin: 0 auto;
  .bg-img("~@images/checkin/pic_checkin_start");
  transform-origin: center 60%;
}
.lottery-start {
  height: 62px;
  font-size: 44px;
  font-weight: 600;
  color: rgba(255, 255, 255, 1);
  line-height: 62px;
  text-align: center;
  padding: 86px 0 5px;
}
.lottery-ticket {
  height: 30px;
  font-size: 22px;
  font-weight: 400;
  color: rgba(251, 224, 174, 1);
  line-height: 30px;
  text-align: center;
  letter-spacing: 1px;
}
.wheel-bg div {
  text-align: center;
}
.prize-list {
  width: 100%;
  height: 100%;
  position: relative;
}
.prize-list .prize-item {
  position: absolute;
  top: 0;
  left: 0;
  z-index: 2;
  transform-origin: bottom center;
}
.prize-list .prize-item:first-child {
  top: 15px;
  left: 214px;
  transform: rotate(0deg);
}
.prize-list .prize-item:nth-child(2) {
  top: 15px;
  left: 214px;
  transform: rotate(60deg);
}
.prize-list .prize-item:nth-child(3) {
  top: 15px;
  left: 214px;
  transform: rotate(120deg);
}
.prize-list .prize-item:nth-child(4) {
  top: 15px;
  left: 214px;
  transform: rotate(180deg);
}
.prize-list .prize-item:nth-child(5) {
  top: 15px;
  left: 214px;
  transform: rotate(240deg);
}
.prize-list .prize-item:nth-child(6) {
  top: 15px;
  left: 214px;
  transform: rotate(300deg);
}
.prize-item {
  width: 300px;
  height: 375px;
}
.prize-pic {
  width: 100%;
  padding-top: 20px;
}
.prize-pic img {
  max-width: 4.0625rem;
  max-height: 2.5rem;
}
.prize-type {
  padding-top: 130px;
  font-size: 26px;
  color: #fb695a;
}
.toast-mask {
  position: fixed;
  top: 0;
  left: 0;
  background: rgba(0, 0, 0, 0.6);
  z-index: 10000;
  width: 100%;
  height: 100%;
}
.toast {
  position: fixed;
  top: 50%;
  left: 50%;
  z-index: 20000;
  transform: translate(-50%, -50%);
  width: 15.4375rem;
  background: #fff;
  border-radius: 0.3125rem;
  padding: 0.3125rem;
  background-color: rgb(252, 244, 224);
}
.toast-container {
  position: relative;
  width: 100%;
  height: 100%;
  border: 1px dotted #fccc6e;
}
.toast-picture {
  position: absolute;
  top: -6.5rem;
  left: -1.5rem;
  width: 18.75rem;
  height: 8.5625rem;
}
.toast-pictrue-change {
  position: absolute;
  top: -1.5rem;
  left: -1.375rem;
  width: 17.5rem;
  height: 3.125rem;
}
.toast-title {
  padding: 2.8125rem 0;
  font-size: 18px;
  color: #fc7939;
  text-align: center;
}
.toast-btn {
  display: flex;
  align-items: center;
  justify-content: center;
  margin-bottom: 0.9375rem;
}
.toast-btn div {
  background-image: -moz-linear-gradient(
    -18deg,
    rgb(242, 148, 85) 0%,
    rgb(252, 124, 88) 51%,
    rgb(252, 124, 88) 99%
  );

  background-image: -ms-linear-gradient(
    -18deg,
    rgb(242, 148, 85) 0%,
    rgb(252, 124, 88) 51%,
    rgb(252, 124, 88) 99%
  );
  background-image: -webkit-linear-gradient(
    -18deg,
    rgb(242, 148, 85) 0%,
    rgb(252, 124, 88) 51%,
    rgb(252, 124, 88) 99%
  );
  box-shadow: 0px 4px 0px 0px rgba(174, 34, 5, 0.7);
  width: 4.6875rem;
  height: 1.875rem;
  border-radius: 1.875rem;
  text-align: center;
  line-height: 1.875rem;
  color: #fff;
}
.close {
  position: absolute;
  top: -0.9375rem;
  right: -0.9375rem;
  width: 2rem;
  height: 2rem;
  background-size: 100%;
}
</style>

