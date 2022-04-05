<template>
  <div id="app">
    <div class="order-area">
      <div class="header">
        Order book
      </div>
      <div class="list">
        <div class="list-header">
          <div class="list-header__item price">Prize(USD)</div>
          <div class="list-header__item size">Size</div>
          <div class="list-header__item total">Total</div>
        </div>
        <div class="quote-list">
          <div class="quote-row sell" v-for="(row, index) in sellQuote" :key="index" :class="{'new-data': row.status}">
            <div class="quote-row__item price">{{ row.price | priceFormat }}</div>
            <div class="quote-row__item" :class="row.sizeChange">{{ row.size | priceFormat(0)}}</div>
            <div class="quote-row__item total">
              {{ row.cumulativeTotal | priceFormat(0) }}
              <div class="bar" :style="{right: Math.round((row.cumulativeTotal / maxSellTotal) * 100) + '%'}">&nbsp;</div>
            </div>
            <div class="tool-tip">
              <p>Avg Price: <span>{{(row.totalValue / row.cumulativeTotal) | priceFormat}}</span> USD</p>
              <p>Total Value: <span>{{(row.totalValue) / 1000 | priceFormat}}</span> USD</p>
            </div>
          </div>
        </div>
      </div>
      <div class="last-price">
        <div class="price" :class="lastPriceColorClass">
          {{lastPrice | priceFormat}}
          <div v-if="gain !== 0" class="status-icon">
            <svg xmlns="http://www.w3.org/2000/svg" width="24" height="24" viewBox="0 0 24 24" role="presentation" fill="none" fill-rule="nonzero" stroke="currentColor" stroke-width="4" stroke-linecap="round" stroke-linejoin="round">
              <line x1="12" y1="5" x2="12" y2="19"></line>
              <polyline points="19 12 12 19 5 12"></polyline>
            </svg>
          </div>
        </div>
      </div>
      <div class="list">
        <div class="list-header">
          <div class="list-header__item price">Prize(USD)</div>
          <div class="list-header__item size">Size</div>
          <div class="list-header__item total">Total</div>
        </div>
        <div class="quote-list ">
          <div class="quote-row buy" v-for="(row, index) in buyQuote" :key="index" :class="{'new-data': row.status}">
            <div class="quote-row__item price">{{ row.price |priceFormat }}</div>
            <div class="quote-row__item" :class="row.sizeChange">{{ row.size | priceFormat(0)}}</div>
            <div class="quote-row__item total">
              {{ row.cumulativeTotal | priceFormat(0) }}
              <div class="bar" :style="{right: Math.round((row.cumulativeTotal / maxBuyTotal) * 100) + '%'}">&nbsp;</div>
            </div>
            <div class="tool-tip">
              <p>Avg Price: <span>{{row.totalValue / row.cumulativeTotal | priceFormat}}</span> USD</p>
              <p>Total Value: <span>{{row.totalValue / 1000 | priceFormat}}</span> USD</p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
<script>
export default {
  name: 'App',
  data () {
    return {
      ws: '',
      sellQuote: [],
      buyQuote: [],
      lastSellQuote: [],
      lastPrice: 0,
      gain: 0,
      maxSellTotal: 0,
      maxBuyTotal: 0
    }
  },
  filters: {
    priceFormat (value, fixed = 1) {
      if (isNaN(value)) {
        return value
      }
      let num = Number(value)
      const re = /(-?\d+)(\d{3})/
      if (Number.prototype.toFixed) {
        num = num.toFixed(fixed)
      } else {
        num = Math.floor((num * 100)/100)
      }
      num = '' + num
      while (re.test(num)) {
        num = num.replace(re, '$1,$2')
      }
      return num
    }
  },
  computed: {
    lastPriceColorClass () {
      switch (this.gain) {
        case 1:
          return 'up'
        case -1:
          return 'down'
        default:
          return ''
      }
    }
  },
  methods: {
    setCumulativeTotal(quoteInfo, type) {
      let result = quoteInfo.slice(0, 8).map((item, index) => {
        let cumulativeTotal = 0
        let totalValue = 0
        if (type === 'buy') {
          for (let i = index; i >= 0; i--) {
            cumulativeTotal += parseInt(quoteInfo[i].size)
            totalValue += quoteInfo[i].price * quoteInfo[i].size
          }
          if (index === 7) {
            // 設定當前最大量紀錄
            this.maxBuyTotal = this.maxBuyTotal > cumulativeTotal ? this.maxBuyTotal : cumulativeTotal
          }
        } else {
          for (let i = index; i < 8; i++) {
            cumulativeTotal += parseInt(quoteInfo[i].size)
            totalValue += quoteInfo[i].price * quoteInfo[i].size
          }
          if (index === 0) {
            this.maxSellTotal = this.maxSellTotal > cumulativeTotal ? this.maxSellTotal : cumulativeTotal
          }
        }

        item.cumulativeTotal = cumulativeTotal
        item.totalValue = totalValue
        return item
      })
      return result
    },
    setNewDataStatus (value, beforeValue, quoteType) {
      value.forEach((quote, index) => {
        const checkExist = beforeValue.some(item => {
          if (item.price === quote.price) {
            return true
          }
        })
        if (!checkExist) {
          this[quoteType][index].status = true
        } else {
          // 將已存在quote的size進行比對。
          const beforeValueIndex = beforeValue.findIndex(item => item.price === quote.price)
          if (quote.size > beforeValue[beforeValueIndex].size) {
            this[quoteType][index].sizeChange = 'size-up'
          } else if (quote.size < beforeValue[beforeValueIndex].size) {
            this[quoteType][index].sizeChange = 'size-down'
          }
        }
      })
    }
  },
  watch: {
    sellQuote(value, beforeValue) {
      // 當過往資料等於0
      if (beforeValue.length) {
        this.setNewDataStatus(value, beforeValue, 'sellQuote')
      }
    },
    buyQuote (value, beforeValue) {
      if (beforeValue.length) {
        this.setNewDataStatus(value, beforeValue, 'buyQuote')
      }
    }
  },
  mounted() {
    this.ws = new WebSocket(`wss://ws.btse.com/ws/futures`)
    const that = this
    this.ws.addEventListener('open', function() {
      that.ws.send(JSON.stringify({
        "op": "subscribe",
        "args": [
          "orderBookApi:BTCPFC_0"
        ]
      }))
    })
    this.ws.onmessage = (msg) => {
      let msgData = JSON.parse(msg.data)
      if (msgData.data) {
        let newSellQuote = this.setCumulativeTotal(msgData.data.sellQuote, 'sell')
        this.sellQuote = newSellQuote || []
        let newBuyQuote = this.setCumulativeTotal(msgData.data.buyQuote, 'buy')
        this.buyQuote = newBuyQuote || []
        this.gain = msgData.data.gain
        this.lastPrice = msgData.data.lastPrice || 0
      }
    }
  }
}
</script>

<style lang="scss">
$sellTxtColor: #FF5B5A;
$buyTxtColor: #00b15d;
$commonTxtColor: #FFF;
$toolTipBg: #57626E;
#app {
  font-family: Avenir, Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #2c3e50;
  .order-area {
    margin: 0 auto;
    .icon-arrow-down {
      background: url("./assets/IconArrowDown.svg") no-repeat no-repeat center center / 100%;
      display: inline-block;
      height: 24px;
      width: 24px;
      color: $commonTxtColor;
    }
    width: 273px;
    background-color: #1e2c4c;
    .header {
      text-align: left;
      font-size: 15px;
      border-bottom: 1px solid #1B2338;
      height: 33px;
      display: flex;
      align-items: center;
      padding-left: 10px;
      box-sizing: border-box;
      color: $commonTxtColor;
      font-weight: 500;
    }
    .list {
      .list-header {
        display: flex;
        justify-content: space-between;
        padding: 5px 10px 0;
        &__item {
          flex: 1 1 27%;
          color: #8698AA;
          font-size: 12px;
          text-align: right;
          &.price {
            text-align: left;
          }
          &.total {
            flex-basis: 46%;
          }
        }
      }
      .quote-list {
        padding: 4px 0 3px;
        .quote-row {
          position: relative;
          display: flex;
          justify-content: space-between;
          padding: 1px 10px;
          color: $commonTxtColor;
          &.sell {
            .price {
              color: $sellTxtColor;
            }
            .bar {
              background-color: rgba(255,90,90,.12);
            }
            &.new-data {
              animation: sellRedPop 0.3s ease;
            }
            &:hover {
              .tool-tip {
                top: -24px
              }
            }
          }
          &.buy {
            .price {
              color: $buyTxtColor;
            }
            &.new-data {
              animation: buyGreenPop 0.3s ease;
            }
            .bar {
              background-color: rgba(16,186,104,.12);
            }
            &:hover {
              .tool-tip {
                top: -5px;
              }
            }
          }
          &__item {
            position: relative;
            text-align: right;
            font-size: 13px;
            flex: 1 1 27%;
            &.price {
              text-align: left;
            }
            &.size-up {
              animation: sellRedPop 0.3s ease;
            }
            &.size-down {
              animation: buyGreenPop 0.3s ease;
            }
            &.total {
              overflow: hidden;
              flex-basis: 46%;
            }
            .bar {
              top: 0;
              right: 0;
              width: 100%;
              position: absolute;
              transform: translateX(100%);
            }
          }
          &:hover {
            .tool-tip {
              display: block;
            }
          }
          .tool-tip {
            border-radius: 5px;
            display: none;
            text-align: left;
            font-size: 12px;
            position: absolute;
            right: -5%;
            background-color: $toolTipBg;
            color: $commonTxtColor;
            transform-origin: top left;
            transform: translateX(100%);
            font-weight: 500;
            padding: 5px;
            //clip-path: polygon(5% 35%, 5% 0, 100% 0, 100% 100%, 5% 100%, 5% 65%, 0 50%);
            p {
              margin: 5px;
              span {
                font-weight: 600;
              }
            }
            &:before {
              display: block;
              position: absolute;
              content: '';
              width: 15px;
              height: 20px;
              left: -13px;
              top: 50%;
              transform-origin: center;
              transform: translateY(-50%);
              clip-path: polygon(0 50%, 100% 100%, 100% 0);
              background-color: $toolTipBg;
            }
          }
        }
      }
    }
    .last-price {
      padding: 0 10px;
      .price {
        display: flex;
        justify-content: center;
        align-items: center;
        color: $commonTxtColor;
        width: 100%;
        height: 28px;
        line-height: 28px;
        font-size: 16.8px;
        font-weight: 700;
        .status-icon {
          display: flex;
          align-items: center;
          justify-content: center;
          width: 15px;
          height: 15px;
          margin-left: 3px;
        }
        &.up {
          background-color: rgba(16, 186, 104, 0.12);
          color: $buyTxtColor;
          .status-icon {
            transform-origin: center;
            transform: rotate(180deg);
          }
        }
        &.down {
          background-color: rgba(255, 90, 90, 0.12);
          color: $sellTxtColor;
        }

      }
    }
  }
}
@keyframes sellRedPop {
  0% {background-color: transparent}
  100% {background-color: rgba(255, 91, 90, 0.5)}
}

@keyframes buyGreenPop {
  0% {background-color: transparent}
  100% {background-color: rgba(0, 177, 93, 0.5)}
}


</style>
