<template>
    <div :class="['playing-card', playingCardColor, { disabled, flipped, 'playing-card-placeholder': placeholder, selectable: $listeners.click && !disabled }]"
            :data-card="id"
            @click="doClick">
        <span v-if="!flipped">{{ id }}</span>
    </div>
</template>

<script>
export default {
    name: 'Card',
    props: {
        flipped: {
            type: Boolean,
            default: false,
        },
        placeholder: {
            type: Boolean,
            default: false,
        },
        id: {
            type: String,
            required: false
        },
        color: {
            type: String,
            required: false
        },
        disabled: {
            type: Boolean,
            default: false
        }
    },
    computed: {
        playingCardColor() {
            if (this.color) {
                return `playing-card-${this.color}`;
            }
            return '';
        }
    },
    methods: {
        doClick($event) {
            if (!this.disabled) {
                this.$emit('click', $event);
            }
        }
    }
}
</script>

<style>

.playing-card {
    display: inline-flex;
    height: 120px;
    width: 85px;
    align-items: center;
    justify-content: center;
    font-size: 36px;
    line-height: 1;
    border: 1px solid #ddd;
    box-shadow: 0 1px 2px 1px rgba(0,0,0,.25);
    margin: 10px;
    border-radius: 5px;
    cursor: default;
    transition: all linear .2s;
    position: relative;
    overflow: hidden;
    background: #fff;
    color: #000;
    box-sizing: border-box;
}
.playing-card:before,
.playing-card:after {
    content: attr(data-card);
    color: inherit;
    display: block;
    position: absolute;
    font-size: .5em;
    line-height: 1;
}
.playing-card:before {
    top: 5px;
    left: 5px;
}
.playing-card:after {
    bottom: 5px;
    right: 5px;
    transform: rotate(180deg);
}
.playing-card.flipped {
    transform: rotate3d(0, 1, 0, 180deg);
    color: #3355aa;
    background: #3355aa;
    border-width: 5px;
    border-color: #fff;
}
.playing-card.flipped span {
    display: none;
}
.playing-card-red {
  color: red;
}
.playing-card.playing-card-placeholder {
  border-color: #fff;
  border-style: dashed;
  box-shadow: 0 2px 2px rgba(0,0,0,.1) inset;
  background: rgba(255, 255, 255, .5);
}
.playing-card.selectable {
  cursor: pointer;
}
.playing-card.selectable:hover {
  transform: translate3d(0, -10px, 0);
}
.playing-card.selectable.flipped:hover {
  transform: translate3d(0, -10px, 0) rotate3d(0, 1, 0, 180deg);
}
.playing-card.disabled {
    opacity: .5;
}
</style>
