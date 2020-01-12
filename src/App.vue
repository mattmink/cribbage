<template>
  <div id="app" class="app">
    <div class="table">
      <div class="play-cards">
        <!-- play cards -->
        <card v-for="card in playCards"
            :key="`playCard_${card.id}`"
            v-bind="card"
            class="play-card" />
        <div v-if="stage === 'peg'">{{ playCardsTotal }}</div>
      </div>
      <div>
        <div class="board-container">
          <h1><span v-for="(letter, letterIndex) in title" :key="`title${letterIndex}`" :data-letter="letter">{{ letter }}</span></h1>
          <div class="board">
            <div class="face top"></div>
            <div class="face right"></div>
            <div class="face bottom"></div>
            <div class="face left"></div>
            <div class="face front">
              <div class="peg-holes" v-for="(player, playerIndex) in players" :key="`player${playerIndex + 1}PegHoles`">
                <div class="peg-hole" v-for="pegHole in 120" :key="`player${playerIndex + 1}PegHole${pegHole}`"></div>
              </div>
            </div>
            <div class="face back"></div>
          </div>
        </div>
      </div>
      <div class="turn-card">
        <card v-bind="turnCard" v-if="turnCard" />
      </div>
    </div>
    <div v-if="playing">
      <div v-if="stage === 'playerSetup'">
        <h2>Player Setup</h2>
        <div class="player-setup">
          <div class="player-setup--player" v-for="(player, playerIndex) in players" :key="`playerSetup${playerIndex}`">
            <label :for="`playerSetup${playerIndex + 1}`">Player {{ playerIndex + 1 }} Name</label>
            <input type="text" :id="`playerSetup${playerIndex + 1}`" v-model="player.name">
          </div>
        </div>
        <button class="btn btn-primary" @click="nextStage()">Go!</button>
      </div>
      <div v-else>
        <h2>{{ activePlayer.name }}'s Turn</h2>
        <p>{{ helpText }}</p>
        <!-- Cut the deck -->
        <div v-if="stage === 'cut'">
          <card v-for="card in shuffledDeck"
              :key="`deckCard_${card.id}`"
              v-bind="card"
              class="deck-card"
              :flipped="true"
              @click="selectCard(card)" />
        </div>
        <!-- Player's hand -->
        <div v-if="!turnIsOver">
          <card v-for="card in activePlayerHand"
              :key="card.id"
              v-bind="card"
              @click="selectCard(card)"
              :disabled="playCardsTotal + card.value > 31" />
        </div>
        <!-- Show hand button -->
        <div v-else>
          <button class="btn btn-primary" @click="turnIsOver = false" v-if="stage !== 'cut'">Show Hand</button>
        </div>
        <!-- Crib container -->
        <div class="crib">
          <div class="crib-label">crib</div>
          <div class="crib-cards">
            <card v-for="card in crib" :key="card.id" v-bind="card" class="crib-card" :flipped="stage !== 'count-crib'" />
          </div>
        </div>
      </div>
    </div>
    <div v-if="stage !== 'playerSetup' && ((playing && dealer === null) || (!playing && dealer !== null))" class="dealer-decide">
      <div v-for="(player, playerIndex) in players" :key="`player${playerIndex}`" class="dealer-decide-player">
        <h4>{{ player.name }}'s Cut</h4>
        <card v-if="player.hand.length > 0" v-bind="player.hand[0]" />
        <card v-else :placeholder="true"></card>
      </div>
    </div>
    <div v-if="!playing">
      <button class="btn btn-primary" @click="playing = true; turnIsOver = false" v-if="dealer === null">Start Game</button>
      <div v-else>
        <h3>{{ players[dealer].name }} is the dealer.</h3>
        <button class="btn btn-primary" @click="deal()">Deal</button>
      </div>
    </div>
  </div>
</template>

<script>
import Card from './components/Card';

const getDeck = () => {
  const suits = ['♥', '♦', '♠', '♣'];
  const values = { A: 1, J: 10, Q: 10, K: 10 };
  const symbols = ['A', ...[...Array(9)].map((_, i) => i + 2), 'J', 'Q', 'K'];
  return suits.flatMap((suit, suitIndex) => symbols.map(symbol => ({
    suit,
    color: suitIndex < 2 ? 'red' : 'black',
    symbol: `${symbol}`,
    value: values[symbol] || symbol,
    id: `${symbol}${suit}`,
  })));
};

const randomShuffle = (arr = []) => {
  const array = arr.slice();
  for (let i = array.length - 1; i > 0; i--) {
    const j = Math.floor(Math.random() * (i + 1));
    [array[i], array[j]] = [array[j], array[i]];
  }
  return array;
}

const faroShuffle = (arr = []) => {
  const splitAt = Math.round(arr.length / 2);
  return arr.slice(0, splitAt).flatMap((card, i) => {
    const matchingCard = arr[splitAt + i];
    if (!matchingCard) return card;
    return [card, matchingCard];
  });
};


export default {
  name: 'app',
  data() {
    return {
      title: 'Cribbage',
      dealer: null,
      playing: false,
      turnIsOver: true,
      playerTurn: 0,
      stages: ['cut', 'crib', 'cut', 'peg', 'count', 'cribCount'],
      stage: 'playerSetup',
      shuffledDeck: null,
      players: [
        {
          hand: [],
          score: 0,
          name: 'Player 1',
        },
        {
          hand: [],
          score: 0,
          name: 'Player 2',
        },
      ],
      turnCard: null,
      crib: [],
      playCards: [],
    }
  },
  components: { Card },
  computed: {
    activePlayer() {
      return this.players[this.playerTurn];
    },
    activePlayerHand() {
      return this.activePlayer.hand.filter(card => !card.isPlayed);
    },
    helpText() {
      if (this.stage === 'crib') {
        const remaining = this.activePlayer.hand.length - 4;
        const cardsText = remaining === 1 ? 'card' : 'cards';
        return `Throw ${remaining} ${cardsText} to the crib.`;
      }
      if (this.stage === 'cut') {
        if (this.dealer === null) {
          return 'Cut the deck to determine the dealer.';
        }
        return 'Cut the deck to select a turn card.';
      }
      return '';
    },
    playCardsTotal() {
      return this.playCards.reduce((total, { value }) => total + value, 0);
    }
  },
  methods: {
    selectNextPlayer() {
      this.playerTurn = (this.playerTurn + 1) % this.players.length;
    },
    buildDeck() {
      this.shuffledDeck = randomShuffle(faroShuffle(randomShuffle(faroShuffle(getDeck()))));
    },
    deal() {
      this.players.forEach(player => {
        player.hand = [];
      });
      this.crib = [];
      this.buildDeck();
      let i = this.playerTurn;
      while (this.players[this.dealer].hand.length < 6) {
        this.players[i % this.players.length].hand.push(this.shuffledDeck.shift());
        i += 1;
      }
      this.nextStage();
      this.playing = true;
    },
    determineDealer() {
      this.playing = false;
      this.turnIsOver = true;
      const dealer = this.players.reduce((dealerIndex, player, playerIndex) => {
        const isPlayerGreater = player.hand[0].value >= this.players[dealerIndex].hand[0].value;
        if (isPlayerGreater) return playerIndex;
        return dealerIndex;
      }, 0);
      this.dealer = dealer;
      this.playerTurn = (this.dealer + 1) % this.players.length;
    },
    nextStage() {
      this.stage = this.stages.shift();
    },
    sendToCrib(card) {
      this.activePlayer.hand.splice(this.getPlayerCardIndex(card), 1);
      this.crib.push(card);
      if (this.activePlayer.hand.length === 4) {
        this.turnIsOver = true;
        this.selectNextPlayer();
      }
      if (this.turnIsOver && this.playerTurn !== this.dealer) {
        this.nextStage();
      }
    },
    getPlayerCardIndex(card) {
      return this.activePlayerHand.findIndex(({ id }) => id === card.id);
    },
    getCardIndex(card) {
      return this.shuffledDeck.findIndex(({ id }) => id === card.id);
    },
    sendToHand(card) {
      this.activePlayer.hand.push(this.shuffledDeck.splice(this.getCardIndex(card), 1)[0]);
    },
    sendToPlayCards(card) {
      this.playCards.push({ ...card });
      this.$set(card, 'isPlayed', true);
    },
    selectTurnCard(card) {
      this.turnCard = this.shuffledDeck.splice(this.getCardIndex(card), 1)[0];
      this.nextStage();
    },
    selectCard(card) {
      if (this.stage === 'cut') {
        if (this.dealer === null) {
          this.sendToHand(card);
          if (this.playerTurn === 0) {
            this.selectNextPlayer();
          } else {
            this.determineDealer();
          }
        } else {
          this.selectTurnCard(card);
        }
      } else if (this.stage === 'crib') {
        if (this.activePlayer.hand.length === 4) return;
        this.sendToCrib(card);
      } else if (this.stage === 'peg') {
        this.sendToPlayCards(card);
        this.turnIsOver = true;
        this.selectNextPlayer();
      }
    }
  },
  created() {
    this.buildDeck();
  }
};
</script>

<style>
html {
  font-size: 16px;
  height: 100%;
}

body {
  padding: 0;
  margin: 0;
  min-height: 100%;
  background-color: #ccc;
  background-image: url(https://3.bp.blogspot.com/-sBe9-hyLTNg/Vkzoi9hLp8I/AAAAAAAAIfI/kB6tQjz9P_E/w1200-h630-p-k-no-nu/Grass%2Bchoppy%2Bgreen%2Bseamless%2Btexture.jpg);
  background-size: 20%;
  background-blend-mode: overlay;
}

body:before {
  position: absolute;
  content: "";
  display: block;
  top: 0;
  bottom: 0;
  left: 0;
  right: 0;
  background: rgba(0, 150, 0, .5);
  background-image: radial-gradient(circle at center 20%, rgba(199, 241, 199, .15) 0%, rgba(15, 37, 15, .75) 100%);
}

h2, h3, h4, h5, p, label {
  text-shadow: 0 1px 2px rgba(0, 0, 0, .5);
}

p, label {
  font-weight: 500;
}

#app {
  font-family: "Avenir", Helvetica, Arial, sans-serif;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
  text-align: center;
  color: #fff;
  position: relative;
  z-index: 1;
}

input[type="text"] {
  font: inherit;
  padding: .25em .5em;
}

.table {
  display: grid;
  grid-template-columns: 2fr 600px 1fr 1fr;
  padding-top: 2rem;
}

.turn-card {
    text-align: left;
}

.btn {
  background: #0099CC;
  border: 1px solid rgba(255, 255, 255, .25);
  border-top-color: rgba(255, 255, 255, .4);
  border-bottom-color: rgba(0, 0, 0, .15);
  color: #fff;
  border-radius: 4px;
  font-size: 1.5rem;
  font-weight: normal;
  padding: .5em 1em;
  cursor: pointer;
}

.crib {
  position: absolute;
  top: 10px;
  right: 10px;
  transform: scale3d(.75, .75, 1);
}

.play-cards {
  text-align: right;
}
.crib-card + .crib-card,
.deck-card + .deck-card {
  margin-left: -80px;
}
.play-card + .play-card {
  margin-left: -60px;
}

.board-container {
  perspective: 1100px;
  perspective-origin: 50% 50%;
  margin: 1rem 0;
  position: relative;
  overflow: hidden;
}

h1 {
  position: absolute;
  font-family: 'Poller One', cursive;
  font-size: 3rem;
  text-align: center;
  width: 100%;
  z-index: 1;
  color: rgb(0,0,0, 0.5);
  opacity: .75;
  font-weight: normal;
  margin: 0;
  top: calc(50% - 0.75em);
  line-height: 1;
  transform: translateZ(230px) rotateX(50deg);
}
h1 span {
  margin: 0 0.075rem;
  display: inline-block;
  position: relative;
  position: relative;
}
h1 span:before, h1 span:after {
  content: attr(data-letter);
  color: rgba(255, 233, 202, 0.25);
  position: absolute;
  display: block;
  height: 1em;
}
h1 span:before {
  top: 1px;
  left: 1px;
  text-shadow: -1px -1px 2px rgba(53, 46, 36,.1);
}
h1 span:after  {
  top: 2px;
  left: 2px;
  text-shadow: -1px -1px 2px rgba(255, 233, 202,.1);
}
h1 span:nth-child(1) {
    transform: rotate(-24deg) translate(-.125em, .6em);
}
h1 span:nth-child(2) {
    transform: rotate(-16deg) translate(-.05em, .3em);
}
h1 span:nth-child(3) {
    transform: rotate(-12deg) translate(-.0125em, .125em);
}
h1 span:nth-child(4) {
    transform: rotate(-5deg);
}
h1 span:nth-child(5) {
    transform: rotate(5deg);
}
h1 span:nth-child(6) {
    transform: rotate(12deg) translate(.0125em, .125em);
}
h1 span:nth-child(7) {
    transform: rotate(16deg) translate(.05em, .3em);
}
h1 span:nth-child(8) {
    transform: rotate(24deg) translate(.125em, .6em);
}

.board {
  height: 120px;
  width: 420px;
  position: relative;
  transform-style: preserve-3d;
  transform: translateZ(230px) rotateX(50deg);
  margin: 0 auto;
}

.face {
  position: absolute;
  overflow: hidden;
}

.board .top,
.board .bottom,
.board .front,
.board .back {
  width: 420px;
}

.board .right,
.board .left,
.board .front,
.board .back {
  height: 120px;
}

.board .top,
.board .bottom {
  height: 10px;
  top: calc(50% - 5px);
}

.board .right,
.board .left {
  width: 10px;
  left: calc(50% - 5px);
}

.board .front  {
  transform: rotateY(  0deg) translateZ( 5px);
  background-color: rgb(247, 201, 142);
}
.board .front,
.board .bottom {
  background-image: url(https://media.istockphoto.com/photos/light-wooden-texture-picture-id511198824?k=6&m=511198824&s=612x612&w=0&h=-qEF05Bwc8rMmCV9PmMerqelrbGKaQrxVBg-9Cxc9Q4=);
  background-repeat: no-repeat;
  background-size: cover;
}

.board .back   {
  transform: rotateY(180deg) translateZ( 5px);
  box-shadow: 0 0 5px 1px rgba(0,0,0,.2), 2px 4px 7px 1px rgba(0, 0, 0, .25), -2px 4px 10px 1px rgba(0, 0, 0, .25);
}

.board .right  { transform: rotateY( 90deg) translateZ(210px); }
.board .left   { transform: rotateY(-90deg) translateZ(210px); }

.board .top    { transform: rotateX( 90deg) translateZ(60px); }
.board .bottom {
  transform: rotateX(-90deg) translateZ(60px);
  background-color: rgb(182, 155, 117);
  box-shadow: 0 0 50px inset rgba(0, 0, 0, .33);
  background-position: bottom;
}
.dealer-decide {
  text-align: center;
  margin-top: 1em;
}
.dealer-decide-player {
  display: inline-block;
  vertical-align: top;
  margin: 0 1rem;
}
.dealer-decide-player h4 {
  margin: 0;
}
.player-setup {
  text-align: center;
}
.player-setup--player {
  text-align: left;
  display: inline-block;
  vertical-align: top;
  padding: 0 .5rem;
  margin-bottom: 1rem;
}
.player-setup--player label {
  display: block;
  font-size: .75rem;
}
</style>
