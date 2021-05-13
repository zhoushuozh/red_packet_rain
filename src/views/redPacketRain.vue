<template>
  <div class="container">
    <!-- 红包雨进度条和点击个数 -->
    <div class="status-bar">
      <div class="progress-bar">
        <span>剩余时间：</span>
        <div class="progress-container">
          <div class="progress-inner"
               :style="{'width': progress + '%'}"></div>
        </div>
        <span>{{seconds}}s</span>
      </div>
      <div>获取红包：{{count}}个</div>
    </div>
    <!-- 开始前的倒计时 -->
    <div class="brefore-start-countdown"
         v-if="!started">
      <div class="trip">一大波红包即将来袭</div>
      <div class="number">{{beforeStartSeconds}}</div>
    </div>
    <!-- 红包雨 -->
    <canvas ref="rainCanvas"></canvas>
    <!-- 红包点击效果 -->
    <canvas ref="bubbleCanvas"></canvas>
  </div>
</template>

<script>
function getRandom(min, max) {
  if (min > max) {
    [min, max] = [max, min]
  }
  const random = Math.random() * (max - min) + min
  return Number(random.toFixed(2))
}

class Packet {
  constructor(options) {
    this.x = options.x // 定义红包在x轴位置
    this.y = options.y // 定义红包在y轴位置
    this.img = options.img // 用于绘制的红包对象
    this.speed = options.speed // 红包的下落速度，以像素为单位
    this.width = options.width || 80
    this.height = options.height || 80
    this.offsetX = options.offsetX || 0  // 红包下落的水平偏移
    this.isOpen = options.isOpen || false // 红包的状态
  }
}

export default {
  data() {
    return {
      rainCtx: null,
      bubbleCtx: null,
      // 红包飘落速度 0到10
      speed: 2,
      // 每次生成红包的最多个数 1到5
      density: 3,
      // 每隔多久生成一次红包，单位毫秒
      timeout: 1000,
      // 判断是否点中时允许的误差
      boundary: 10,
      // 时长 单位毫秒
      duration: 15000,
      // 开始前的倒计时秒数
      beforeStartSeconds: 3,
      // 点中的红包计数
      started: false,
      count: 0,
      progress: 100,
      seconds: 0,
      clientHeight: 0,
      clientWidth: 0,
      // 红包图片
      packetImg: {
        src: require('../assets/redpacket2.png'),
        width: 80,
        height: 80
      },
      // 红包打开时的图片
      openPacketImg: require('../assets/redpacket1.png'),
      // 打开红包时的气泡效果
      bubbleImg: {
        src: require('../assets/add1.png'),
        width: 50,
        height: 50
      },
      packetImgFile: null,
      openPacketImgFile: null,
      bubbleImgFile: null,
      renderPkArr: [],
      bubbleArr: [],
      pushPackettTimer: null,
      countdownTimer: null,
      movePacketAnimation: null,
      endTimer: null
    }
  },
  mounted() {
    // 监听页面离开事件
    document.addEventListener("visibilitychange", () => {
      this.visibilityChange(!document.hidden)
    }, false)
    // 初始化
    this.init()
    // 开始
    this.beforeStartCountdown().then(() => {
      this.start()
      this.endTimer = setTimeout(() => {
        this.end()
        console.log('共点击红包' + this.count + '个')
      }, this.duration);
    })
  },
  methods: {
    async init() {
      // 设置红包雨和气泡效果canvas
      const rainCanvas = this.$refs.rainCanvas
      const bubbleCanvas = this.$refs.bubbleCanvas
      if (rainCanvas.getContext) {
        this.rainCtx = rainCanvas.getContext('2d')
        this.bubbleCtx = bubbleCanvas.getContext('2d')
      }
      rainCanvas.style.cssText = 'position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 2'
      bubbleCanvas.style.cssText = 'position: absolute; top: 0; left: 0; width: 100%; height: 100%; z-index: 1'
      this.clientWidth = document.documentElement.clientWidth
      this.clientHeight = document.documentElement.clientHeight
      this.setCanvasSize(bubbleCanvas)
      this.setCanvasSize(rainCanvas)
      // 时间读条旁显示的秒数
      this.seconds = this.duration / 1000
      // 加载图片
      this.packetImgFile = await this.loadImg(this.packetImg)
      this.openPacketImgFile = await this.loadImg(this.openPacketImg)
      this.bubbleImgFile = await this.loadImg(this.bubbleImg)
    },
    setCanvasSize(canvas) {
      canvas.width = this.clientWidth
      canvas.height = this.clientHeight
    },
    async start() {
      // 增加红包
      this.pushPacket()
      // 监听点击事件
      this.listenClick()
      // 开始掉落
      this.movePackets()
      // 开始倒计时
      this.countdown()
    },
    // 红包雨结束，清除计时器和动画，清除canvas
    end() {
      clearTimeout(this.pushPackettTimer)
      this.pushPackettTimer = null
      cancelAnimationFrame(this.movePacketAnimation)
      this.rainCtx.clearRect(0, 0, this.clientWidth, this.clientHeight)
      this.bubbleCtx.clearRect(0, 0, this.clientWidth, this.clientHeight)
    },
    pushPacket() {
      // 每次生成1到density个红包，并插入一个空数组
      const ranCount = getRandom(1, this.density)
      let arr = []
      for (let i = 0; i < ranCount; i++) {
        const packet = new Packet({
          x: getRandom(20, this.clientWidth - this.packetImg.width),
          y: getRandom(0, -200),
          img: this.packetImgFile,
          speed: (Math.random() * 10 + (10 + this.speed) * 3) / 10,
          offsetX: Math.random() / 2,
          isOpen: false
        })
        arr.push(packet)
      }
      // 将新生成的红包合并到渲染数组
      this.renderPkArr = [...this.renderPkArr, ...arr];
      // 每隔timeout生成一次
      this.pushPackettTimer = setTimeout(() => {
        this.pushPacket()
      }, this.timeout)
    },
    listenClick() {
      const canvas = this.$refs.rainCanvas
      canvas.addEventListener('click', e => {
        // 获取点击的位置
        const pos = {
          x: e.clientX,
          y: e.clientY
        };

        let clickedPackets = []
        this.renderPkArr.forEach((packet, index) => {
          // 遍历数组判断是否命中
          if (this.isIntersect(pos, packet) && !packet.isOpen) {
            clickedPackets.push({
              x: packet.x,
              y: packet.y,
              index
            })
          }
        })
        let lastIndex = clickedPackets.length - 1
        if (clickedPackets.length > 0) {
          // 命中多个时只有最上面的生效
          const clickedPkIndex = clickedPackets[lastIndex].index
          this.count += 1
          // 创建气泡对象，并插入数组，待会绘画的时候要用
          const bubble = {
            x: clickedPackets[lastIndex].x,
            y: clickedPackets[lastIndex].y,
            opacity: 1
          };
          this.bubbleArr.push(bubble);
          // 更改红包状态，将红包图片替换成打开状态的图片
          this.renderPkArr[clickedPkIndex].img = this.openPacketImgFile
          this.renderPkArr[clickedPkIndex].isOpen = true
        }
      })
    },
    // 移动红包，开始掉落，也是红包雨动画的核心
    movePackets() {
      this.rainCtx.clearRect(0, 0, this.clientWidth, this.clientHeight)
      this.drawPackets()
      this.drawBubble()
      this.movePacketAnimation = window.requestAnimationFrame(this.movePackets)
    },
    // 绘制红包
    drawPackets() {
      this.renderPkArr.forEach((packet, index) => {
        // 创建新的红包对象，更新坐标属性，其他属性不变
        const newPacket = new Packet({
          x: packet.x - packet.offsetX,
          y: packet.y + packet.speed,
          width: packet.img ? packet.img.width : this.packetImg.width,
          height: packet.img ? packet.img.height : this.packetImg.height,
          img: packet.img,
          speed: packet.speed,
          offsetX: packet.offsetX,
          isOpen: packet.isOpen
        })
        // 新的红包对象替换旧的红包对象
        this.renderPkArr.splice(index, 1, newPacket)
        // 绘制在canvas上
        this.rainCtx.drawImage(
          packet.img,
          packet.x,
          packet.y,
          packet.width,
          packet.height
        )
      })
    },
    // 绘制点击气泡
    drawBubble() {
      this.bubbleCtx.clearRect(0, 0, this.clientWidth, this.clientHeight);
      // 把opacity为0的全部清除
      this.bubbleArr.forEach((ele, index) => {
        if (ele.opacity < 0) {
          this.bubbleArr.splice(index, 1);
        }
      });
      this.bubbleArr.forEach((ele, index) => {
        if (ele.opacity > 0) {
          // 透明度渐变
          this.bubbleCtx.globalAlpha = ele.opacity;
          this.bubbleCtx.drawImage(
            this.bubbleImgFile,
            ele.x,
            ele.y,
            this.bubbleImg.width,
            this.bubbleImg.height
          )
          // 更新：每次画完就减少0.05透明度，同时位置移动
          const newEle = {
            x: ele.x + 1,
            y: ele.y - 3,
            opacity: ele.opacity - 0.05
          };
          this.bubbleArr.splice(index, 1, newEle);
        }
      });
    },
    // 红包雨开始前的倒计时
    beforeStartCountdown() {
      return new Promise((resolve) => {
        let timer = setInterval(() => {
          this.beforeStartSeconds -= 1
          if (this.beforeStartSeconds <= 0) {
            this.started = true
            // 倒计时结束清除计时器
            clearInterval(timer)
            resolve()
          }
        }, 1000)
      })
    },
    // 红包开始掉落时的倒计时，倒计时结束红包雨结束
    countdown() {
      let totalTime = Math.ceil(this.duration / 1000)
      let s = this.seconds
      this.countdownTimer = setInterval(() => {
        s -= 30 / 1000
        this.seconds = Math.ceil(s)
        // 设置进度读条
        this.progress = (s / totalTime) * 100
        if (this.seconds <= 0) {
          clearInterval(this.countdownTimer)
          this.countdownTimer = null
        }
      }, 30)
    },
    // 离开页面暂停生成红包和倒计时
    visibilityChange(isShow) {
      if (this.pushPackettTimer) {
        clearInterval(this.pushPackettTimer);
        this.pushPackettTimer = null;
      }
      if (this.endTimer) {
        clearTimeout(this.endTimer)
        this.endTimer = null
      }

      if (isShow) {
        this.pushPackettTimer = setTimeout(() => {
          this.pushPacket()
        }, this.timeout)

        this.endTimer = setTimeout(() => {
          this.end()
        }, this.seconds * 1000);
      }
    },
    // 加载图片
    loadImg(image) {
      return new Promise((resolve, reject) => {
        let img = new Image();
        img.src = typeof image === 'object' ? image.src : image
        img.width = image.width
        img.height = image.height
        img.onload = () => {
          resolve(img)
        }
        img.onerror = () => {
          reject(new Error('load image error!'))
        }
      })
    },
    isIntersect(point, packet) {
      let distanceX = point.x - packet.x
      let distanceY = point.y - packet.y
      const boundary = this.boundary // 判断是否点中时允许的误差
      let withinX = distanceX - boundary > 0 && distanceX < packet.width + boundary
      let withinY = distanceY - boundary > 0 && distanceY < packet.height + boundary
      return withinX && withinY
    }
  }
}
</script>

<style>
.container {
  position: relative;
  width: 100vw;
  height: 100vh;
}
.status-bar {
  position: fixed;
  padding: 10px;
  top: 0;
  left: 0;
  color: #fff;
  z-index: 3;
}
.progress-bar {
  height: 30px;
  display: flex;
  align-items: center;
}
.progress-container {
  height: 8px;
  width: 100px;
  background: #fff;
  border-radius: 4px;
  overflow: hidden;
  margin-right: 10px;
}
.progress-inner {
  height: 100%;
  width: 100%;
  background: red;
  border-radius: 4px;
}

.brefore-start-countdown {
  position: fixed;
  height: 100vh;
  width: 100%;
  left: 0;
  top: 0;
  z-index: 5;
  background: rgba(0, 0, 0, 0.6);
  display: flex;
  justify-content: center;
  align-items: center;
  flex-direction: column;
}
.brefore-start-countdown .trip {
  font-size: 28px;
  color: #fff;
}

.brefore-start-countdown .number {
  font-size: 120px;
  color: #fff;
  line-height: 1.5;
}
</style>