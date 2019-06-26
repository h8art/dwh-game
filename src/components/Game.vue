<template lang="pug">
.rel
  .lootbox
    .treasure-btn(@click='lootModal = true')
    .treasure-modal(v-if='lootModal')
      .title LOOTBOX
      img.loot(src='../../public/loot.png')
      .btns
        .exit(@click='lootModal = false') Cancel
        .open(@click='openLootbox') OPEN
    .treasure-weapeon(v-if='show_weapeon')
      .box
        img.animated.tada.infinite(:src='"/weapeon_"+show_weapeon+".png"')
  //- .items(@click='start')
  //-   .items-btn
  .market
    .market-btn(@click='openMarket')
    .market-modal(v-if='marketModal')
      .bag
        .bag-list
          .bag-list-item(v-for='item, i in items', v-if='i>0&&items.length>0' @click='marketSellSelected = i' :class='{"selected" : marketSellSelected == i}')
            img(:src='"weapeon_"+item[1]+".png"')
        .sell
          input.price(v-model='marketSellPrice')
          .sell-btn(@click='sellItem') SELL
      .order
        .order-list
          .order-list-item(v-for='o in orders' v-if='orders')
            img(:src='"weapeon_"+o[2]+".png"')
            .price {{o[1]}}
            .buy(@click='buyItem(o)') BUY
        .btns
          .exit(@click='marketModal = false') Cancel
  .quick
    .quick-item
      img(v-if='items.length>0' :src='"../../public/weapeon_"+items[0][1]+".png"')
    .quick-item
      img(v-if='items&&items[1]' :src='"../../public/weapeon_"+items[1][1]+".png"')
    .quick-item
      img(v-if='items&&items[2]' :src='"../../public/weapeon_"+items[2][1]+".png"')
    .quick-item
      img(v-if='items&&items[3]' :src='"../../public/weapeon_"+items[3][1]+".png"')
    .quick-item
      img(v-if='items&&items[4]' :src='"../../public/weapeon_"+items[4][1]+".png"')
  .money Cash {{money}}$
  .game(@click='shoot')
    .playerFriend(v-for='p in players' :style='{ "left": p.left + "px", "top": p.top + "px", "transform": "rotate("+p.rotate+"deg)" }')
    .player#player(:style='{"left": left + "px", "top": top + "px", "transform": rotate}')
    .enemy(v-for='e in enemies' :style='{ "left": e.left + "px", "top": e.top + "px" }')
      .hp
        .hp-inner(:style='{ "width": e.hp + "%" }')
    .bullet(v-for='b in bullets', :style='{"left": b.left + "px", "top": b.top + "px"}')
</template>
<script>
import * as fluence from "fluence";
import { isNullOrUndefined } from 'util';
function randomInteger(min, max) {
  var rand = min - 0.5 + Math.random() * (max - min + 1)
  rand = Math.round(rand);
  return rand;
}
function getRandomInt(min, max) {
  return Math.floor(Math.random() * (max - min)) + min;
}
function getPlayerId() {
  var playerId = parseInt(window.localStorage.getItem("game"));
  if (isNullOrUndefined(playerId) || isNaN(playerId)) {
    playerId = getRandomInt(0, 9999999);
    window.localStorage.setItem("game", playerId);
  }
  return playerId.toString();
}
export default {
  data(){
    return{
      left: 0,
      top: 0,
      money: 1000,
      playerId: getPlayerId(),
      enemies: [],
      lootModal: false,
      marketModal: false,
      show_weapeon: null,
      marketSellSelected: null,
      marketSellPrice: null,
      rotate: 0,
      bullets: [],
      level: 1,
      items: [],
      orders: [],
      players: []
    }
  },
//   >>>CREATE TABLE weapeons(id int, type int, level int, userId int)
// table created

// >>>CREATE TABLE orders(id int, price int, type int, userId int)
// table created

//CREATE TABLE enemies(id int, hp int, posX int, posY int)
  methods: {
    getEnemies: function() {
      window.session.request(`SELECT * FROM enemies`).result().then((r)=>{
        // console.log("syncing enemies")
        try {
          var val = r.asString().split("\n").map(i => i.split(", "))
          val.splice(0, 1)
          this.enemies = []
          if(val.length <= 1){
            let id = randomInteger(500, 2000000)
            let x = Math.floor(randomInteger(50, window.innerWidth - 50))
            let y = Math.floor(randomInteger(50, window.innerHeight - 50))
            window.session.request(`INSERT INTO enemies VALUES(${id}, 100, ${x}, ${y})`).then((r) => {
              this.enemies.push({
                left: x,
                top: y,
                hp: 100,
                id
              })
            })
          }else{
            val.forEach(v=>{
              if(v){
                this.enemies.push({
                  left: v[2],
                  top: v[3],
                  hp: v[1],
                  id: v[0]
                })
              }
            })
          }
        } catch {}
        setTimeout(()=>this.getEnemies(), 1000)
      })
      
    },
    getPlayers: function() {
      // console.log("syncing players")
      window.session.request(`SELECT * FROM players`).result().then((r)=>{
        try {
          var val = r.asString().split("\n").map(i => i.split(", "))
          val.splice(0, 1)
          this.players = []
          
          val.forEach(v=>{
            function emptyPlayer(player) {
              return v[1] == 0 && v[2] == 0 && v[3] == 0;
            }
            if(v && v[0] !== this.playerId && !emptyPlayer(v)){
              let player = {
                left: v[1],
                top: v[2],
                angle: v[3]
              };

              // console.log(`player: ${v[0]} ${JSON.stringify(player)}`)

              this.players.push(player);
            }
          })
        } catch {}
        setTimeout(()=>this.getPlayers(), 500)
      })
    },
    openMarket: function() {
      this.marketModal = true
      window.session.request(`SELECT * FROM orders`).result().then((r) => {
        var val = r.asString().split("\n").map(i => i.split(", "))
        val.splice(0, 1)
        this.orders = val
      })
    },
    buyItem: function(o){
      window.session.request(`INSERT INTO weapeons VALUES(${randomInteger(500, 2000000)}, ${o[2]}, 0, ${this.playerId})`)
      
      window.session.request(`SELECT * FROM users`).result().then((r) => {
        var val = r.asString().split("\n").map(i => i.split(", "))
        val.splice(0, 1)
        window.session.request(`UPDATE users SET cash = ${parseInt(val.filter(i=>parseInt(i[0])==o[3])[0][2]) + parseInt(o[1])} WHERE id = ${o[3]}`)
        window.session.request(`UPDATE users SET cash = ${val.filter(i=>parseInt(i[0])==this.playerId)[0][2] - o[1]} WHERE id = ${this.playerId}`)
      })
      console.log(o[0])
      window.session.request(`DELETE FROM orders WHERE id = ${o[0]}`)
      this.money -= o[1]
    },
    getItems: function() {
      window.session.request(`SELECT cash FROM users WHERE id = ${this.playerId}`).result().then((r) => {
        let money = parseInt(r.asString().split("\n")[1])
        if (!isNaN(money) && !isNullOrUndefined(money)) {
          this.money = money
        } else {
          console.log(`creating new player ${this.playerId}`)
          window.session.request(`INSERT INTO users VALUES (${this.playerId}, ${this.playerId}, ${this.money})`)
          window.session.request(`INSERT INTO players VALUES (${this.playerId}, 0, 0, 0)`)
        }
        console.log(`cash for ${this.playerId} is ${this.money}`)
      })
      window.session.request(`SELECT * FROM weapeons WHERE userId = ${this.playerId}`).result().then((r) => {
        if(r.asString().split("\n")[1]){
          var val = r.asString().split("\n").map(i => i.split(", "))
          val.splice(0, 1)
          val.unshift([0,0,0,0])
          this.items = val
        }
      })
    },
    sellItem: function() {
      let iInd = this.marketSellSelected
      window.session.request(`INSERT INTO orders VALUES(${randomInteger(500, 2000000)}, ${this.marketSellPrice}, ${this.items[iInd][1]}, ${this.playerId})`)
      window.session.request(`DELETE FROM weapeons WHERE id = ${this.items[iInd][0]}`)
      this.marketSellPrice = null
      this.marketSellSelected = null
      this.items.splice(iInd, 1)
    },
    openLootbox: function(){
      if(this.money > 20){
        this.money -= 20
        let num = randomInteger(1,4)
        this.show_weapeon = num
        setTimeout(()=>this.show_weapeon = null, 3000)
        window.session.request(`INSERT INTO weapeons VALUES(${randomInteger(500, 2000000)}, ${num}, 0, ${this.playerId})`)
        this.items.push(num)
      }else {
        alert("Not enough cash. Need 20$")
      }
    },
    shoot: function(e){
      this.bullets.push({
        endY: e.y,
        endX: e.x,
        left: this.left,
        top: this.top
      })
    },
    regenerate: function () {
      if(this.bullets){
        this.bullets.forEach((e, i )=> {
          let dx = e.endX - e.left;
          let dy = e.endY - e.top;
          let angle = Math.atan2(dy, dx)
          let velocity = 5
          e.left += velocity * Math.cos(angle)
          e.top += velocity * Math.sin(angle)
          this.enemies.forEach((v, j)=> {
            if(Math.abs(v.left - e.left) < 30 && Math.abs(v.top - e.top) < 30){ 
              this.bullets.splice(i, 1)
              if(v.hp - 10 <= 0){
                this.enemies.splice(j, 1)
                window.session.request(`DELETE FROM ENEMIES WHERE id = ${v.id}`)
                this.money += 10
                window.session.request(`UPDATE users SET cash = ${this.money} WHERE id = ${this.playerId}`)

                if (this.money >= this.level * 100 + 1000) {
                  if (this.level < 5) {
                    this.level += 1;
                    console.log("Level INCREASED => " + this.level)
                  }
                }
                
                // Don't always generate enemies
                if (randomInteger(0, 10) % (this.level + 1) == 0) {
                  console.log(`generating ${this.level} enemies`)
                  for(let l=0; l<this.level; l++){
                    let id = randomInteger(500, 2000000)
                    let x = Math.floor(randomInteger(50, window.innerWidth - 50))
                    let y = Math.floor(randomInteger(50, window.innerHeight - 50))
                    window.session.request(`INSERT INTO enemies VALUES(${id}, 100, ${x}, ${y})`)
                    this.enemies.push({
                      left: x,
                      top: y,
                      hp: 100,
                      id
                    })
                  }
                }
              }else {
                v.hp -= 10
                window.session.request(`UPDATE enemies SET hp = ${v.hp - 10} WHERE id = ${v.id}`)
              }
            }
          })
          if(Math.abs(dx) < 10 && Math.abs(dy) < 10){ 
            this.bullets.splice(i, 1)
          }
        });
      }
      
    }
  },
  mounted(){
    let privateKey = "569ae4fed4b0485848d3cf9bbe3723f5783aadd0d5f6fd83e18b45ac22496859"; // Authorization private key
    let contract = "0xeFF91455de6D4CF57C141bD8bF819E5f873c1A01";                         // Fluence contract address
    let appId = 267;                                                                      // Deployed database id
    let ethereumUrl = "http://geth.fluence.one:8545";                                    // Ethereum light node URL

    fluence.connect(contract, appId, ethereumUrl, privateKey).then((s) => {
        console.log("Session created");
        window.session = s;
        this.getItems()
    });
    setTimeout(()=>{
     var arrow = document.getElementById("player");
    var arrowRects = arrow.getBoundingClientRect();
    var arrowX = arrowRects.left + arrowRects.width / 2;
    var arrowY = arrowRects.top + arrowRects.height / 2;
    this.rotate = "rotate(" + Math.abs(Math.atan2(0 - arrowY, 0 - arrowX)) + "rad)";
    addEventListener("mousemove", (e) => {
      let eyes = document.getElementById("player");
      let mouseX = (eyes.getBoundingClientRect().left); 
      let mouseY = (eyes.getBoundingClientRect().top);
      let radianDegrees = Math.atan2(e.pageX - mouseX, e.pageY - mouseY);
      let rotationDegrees = (radianDegrees * (180/ Math.PI) * -1) + 180;
      eyes.style.transform = `rotate(${rotationDegrees}deg)`
    });
    addEventListener("keydown", (e) => {
      let pRad = 0//Math.atan2(event.clientY - arrowY, event.clientX - arrowX) //TODO: where to get event.clientY & event.clientX?
      arrow.style.transform = "rotate(" + pRad + "rad)";
      if (e.keyCode == '87') {
        // up arrow
        if(this.top - 50 >= 0) {
          this.top -= 50
        }
      }
      else if (e.keyCode == '83') {
        // down arrow
        if(this.top + 50 < window.innerHeight) {
          this.top += 50
        }
      }
      else if (e.keyCode == '65') {
        // left arrow
        if(this.left - 50 >= 0) {
          this.left -= 50
        }
      }
      else if (e.keyCode == '68') {
        // right arrow
        if(this.left + 50 < window.innerWidth) {
          this.left += 50
        }
      }
      //CREATE TABLE players(id int, posX int, posY int, angle int)
      window.session.request(`UPDATE players SET posX = ${this.left}, posY = ${this.top}, angle = ${pRad} WHERE id = ${this.playerId}`)
    })
    this.getEnemies()
    this.getPlayers()
    var self = this;
		setInterval(function(){
      try { self.regenerate(); } catch {}
    }, 10);
    }, 1000)
  }
}
</script>

<style lang="sass" scoped>
.rel
  font-family: Roboto
  position: relative
  width: 100%
  height: 100%
  .quick
    z-index: 10
    position: absolute
    width: 350px
    bottom: 0
    left: calc(50% - 175px)
    height: 70px
    display: flex
    border-left: 5px solid rgb(233,30,99)
    border-right: 5px solid rgb(233,30,99)
    border-top: 5px solid rgb(233,30,99)
    border-top-left-radius: 25px
    border-top-right-radius: 25px
    overflow: hidden
    &-item
      width: 70px
      height: 70px
      background: rgba(233,30,99, .4)
      border-left: 1px solid rgb(233,30,99)
      border-right: 1px solid rgb(233,30,99)
      display: flex
      align-items: center
      justify-content: center
      img
        max-width: 90%
  .treasure
    &-weapeon
      position: fixed
      top: 0
      left: 0
      right: 0
      bottom: 0
      background: rgba(0,0,0,.8)
      z-index: 2000
      display: flex
      align-items: center
      justify-content: center
      .box
        width: 300px
        height: 300px
        display: flex
        align-items: center
        justify-content: center
        border-radius: 50%
        box-shadow: inset 0 0 50px #fff, inset 20px 0 80px #f0f, inset -20px 0 80px #0ff, inset 20px 0 300px #f0f, inset -20px 0 300px #0ff, 0 0 50px #fff, -10px 0 80px #f0f, 10px 0 80px #0ff
    &-modal
      background: rgba(233,30,99, .4)
      border-radius: 25px
      position: fixed
      width: 650px
      height: 450px
      z-index: 1000
      border: 5px solid rgb(233,30,99)
      top: calc(50% - 225px)
      left: calc(50% - 325px)
      display: flex
      flex-direction: column
      align-items: center
      .title
        text-align: center
        color: #fff
        font-size: 50px
        margin-top: 30px
        font-weight: bold
      .loot
        width: 250px
        height: 150px
        margin-top: 50px
      .btns
        display: flex
        align-items: flex-end
        margin-top: 50px
        margin-right: 80px
        .open
          font-size: 32px
          font-weight: bold
          color: #fff
          padding: 15px 30px
          background: rgb(233,30,99)
          border-radius: 25px
          &:hover
            cursor: pointer
        .exit
          font-size: 18px
          color: #fff
          margin-right: 15px
          &:hover
            cursor: pointer
    &-btn
      z-index: 10
      position: absolute
      right: 0px
      top: 200px
      width: 90px
      height: 90px
      background-repeat: no-repeat
      background: center center rgba(233,30,99, .4) url('../../public/treasure.png') no-repeat
      border-top-left-radius: 25px
      border-bottom-left-radius: 25px
      border-left: 5px solid rgb(233,30,99)
      border-top: 5px solid rgb(233,30,99)
      border-bottom: 5px solid rgb(233,30,99)
      background-size: 64px 64px
      &:hover
        cursor: pointer
  .market
    &-modal
      background: rgba(233,30,99, .4)
      border-radius: 25px
      position: fixed
      width: 950px
      height: 750px
      z-index: 1000
      border: 5px solid rgb(233,30,99)
      top: calc(50% - 375px)
      left: calc(50% - 475px)
      display: flex
      padding: 24px
      box-sizing: border-box
      .bag
        width: 350px
        display: flex
        flex-direction: column
        height: 100%
        .sell
          display: flex
          .price
            border: 0
            background: transparent
            border-bottom: 3px solid rgb(233,30,99)
            outline: none
            color: #fff
            font-size: 18px
          &-btn
            color: #fff
            font-size: 17px
            padding: 10px 20px
            border-radius: 25px
            margin-left: 20px
            background: rgb(233,30,99)
            &:hover
              cursor: pointer
        &-list
          flex: 1
          display: flex
          flex-wrap: wrap
          .selected
            background: rgba(233,30,99, .4)
          &-item
            width: 100px
            height: 100px
            border: 2px solid rgb(233,30,99)
            display: flex
            align-items: center
            justify-content: center
            margin: 0 5px
            &:hover
              cursor: pointer
              background: rgba(233,30,99, .2)
            img
              max-width: 90%
      .order
        flex: 1
        display: flex
        flex-direction: column
        &-list
          flex: 1
          &-item
            display: flex
            align-items: center
            justify-content: space-between
            margin: 0 25px
            margin-bottom: 15px
            img
              max-height: 50px
            .price
              color: #fff
            .buy
              color: #fff
              font-size: 14px
              padding: 10px 15px
              border-radius: 15px
              background: rgb(233,30,99)
              &:hover
                cursor: pointer
        .btns
          display: flex
          justify-content: flex-end
          .buy
            color: #fff
            font-size: 17px
            padding: 10px 20px
            border-radius: 25px
            margin-left: 20px
            background: rgb(233,30,99)
            &:hover
              cursor: pointer
          .exit
            color: #fff
            align-self: flex-start
            &:hover
              cursor: pointer
    &-btn
      z-index: 10
      position: absolute
      right: 0px
      top: 420px
      width: 90px
      height: 90px
      background-repeat: no-repeat
      background: center center rgba(233,30,99, .4) url('../../public/auction.png') no-repeat
      border-top-left-radius: 25px
      border-bottom-left-radius: 25px
      border-left: 5px solid rgb(233,30,99)
      border-top: 5px solid rgb(233,30,99)
      border-bottom: 5px solid rgb(233,30,99)
      background-size: 64px 64px
      &:hover
        cursor: pointer
  .items
    &-btn
      z-index: 10
      position: absolute
      right: 0px
      top: 310px
      width: 90px
      height: 90px
      background-repeat: no-repeat
      background: center center rgba(233,30,99, .4) url('../../public/rucksack.png') no-repeat
      border-top-left-radius: 25px
      border-bottom-left-radius: 25px
      border-left: 5px solid rgb(233,30,99)
      border-top: 5px solid rgb(233,30,99)
      border-bottom: 5px solid rgb(233,30,99)
      background-size: 64px 64px
      &:hover
        cursor: pointer
  .money
    font-size: 25px
    font-family: Roboto
    color: #fff
    position: absolute
    right: 15px
    top: 15px
  .game
    background: url('../../public/stone2_b.jpg')
    background-size: 42px 32px
    width: 100vw
    height: 100vh
    cursor: url('../../public/crosshair.png'), auto;
    .enemy
      width: 40px
      height: 40px
      position: absolute
      background-image: url('../../public/enemy.png')
      background-size: 100% 100%
      .hp
        background: red
        height: 5px
        width: 100%
        &-inner
          height: 5px
          background: green
    .playerFriend
      width: 40px
      height: 40px
      position: absolute
      background-image: url('../../public/player.png')
      background-size: 100% 100%
      transition: top .3s, left .3s
    .player
      width: 40px
      height: 40px
      position: absolute
      background-image: url('../../public/player.png')
      background-size: 100% 100%
      transition: top .3s, left .3s
    .bullet
      width: 5px
      height: 5px
      background: red
      position: absolute
</style>
