<template>
    <div class="knob-control" :style="style">
        <svg :width="computedSize.width" :height="computedSize.height" viewBox="0 0 100 100"
            @click="onClick"
            @mousedown="onMouseDown"
            @mouseup="onMouseUp"
            @touchstart="onTouchStart"
            @touchend="onTouchEnd">
            <path
              :d="rangePath"
              :stroke-width="strokeWidth"
              :stroke="secondaryColor"
              class="knob-control__range">
            </path>
            <path
              v-if="showValue"
              :d="valuePath"
              :stroke-width="strokeWidth"
              :stroke="primaryColor"
              class="knob-control__value">
            </path>
            <text
              v-if="showValue"
              :x="50"
              :y="57"
              text-anchor="middle"
              :fill="textColor"
              class="knob-control__text-display">
              {{valueDisplay}}
            </text>
        </svg>
    </div>
</template>

<script>
    const RADIUS = 40;
    const MID_X = 50;
    const MID_Y = 50;
    const MIN_RADIANS = 4 * Math.PI / 3;
    const MAX_RADIANS = -Math.PI / 3;

    // map a value (x) from one range (in min/max) onto another (out min/max)
    const mapRange = (x, inMin, inMax, outMin, outMax) => {
        return (x - inMin) * (outMax - outMin) / (inMax - inMin) + outMin;
    };

    export default {
        data () {
            return {}
        },
        props: {
            'value': {
                type: Number,
                required: true
            },
            'max': {
                type: Number,
                default: 100
            },
            'min': {
                type: Number,
                default: 0
            },
            'stepSize': {
                type: Number,
                default: 1
            },
            'disabled': {
                type: Boolean,
                default: false
            },
            'size': {
                type: Number,
                default: 100
            },
            'responsive': {
                type: Boolean,
                default: false
            },
            'allowOverSize': {
                type: Boolean,
                default: false
            },
            'primaryColor': {
                type: String,
                default: '#409eff'
            },
            'secondaryColor': {
                type: String,
                default: '#dcdfe6'
            },
            'textColor': {
                type: String,
                default: '#000000'
            },
            'strokeWidth': {
                type: Number,
                default: 17
            },
            'valueDisplayFunction': {
                type: Function,
                default: (v) => v
            },
        },
        computed: {
            style () {
                return {
                    height: this.responsive ? this.size + '%' : this.size - 5 + 'px'
                };
            },
            computedSize () {
                // size check if responsive is true, greater than 100 and NOT absolutely wanted (allowOverSize === true)
                if (this.responsive && this.size > 100 && !this.allowOverSize) {
                    return 100 + '%';
                }

                if (this.responsive) {
                    return {
                        width: this.size + '%',
                        height: 'auto'
                    }
                } else {
                    return {
                        width: this.size,
                        height: this.size
                    }
                }
            },
            rangePath () {
                return `M ${this.minX} ${this.minY} A ${RADIUS} ${RADIUS} 0 1 1 ${this.maxX} ${this.maxY}`;
            },
            valuePath () {
                return `M ${this.zeroX} ${this.zeroY} A ${RADIUS} ${RADIUS} 0 ${this.largeArc} ${this.sweep} ${this.valueX} ${this.valueY}`;
            },
            showValue () {
                return (this.value >= this.min && this.value <= this.max) && !this.disabled;
            },
            zeroRadians () {
                /* this weird little bit of logic below is to handle the fact that usually we
                    want the value arc to start drawing from the 'zero' point, but, in the case
                    that the minimum and maximum values are both above zero, we set the 'zero point'
                    at the supplied minimum, so the value arc renders as the user would expect */
                if (this.min > 0 && this.max > 0)
                    return mapRange(this.min, this.min, this.max, MIN_RADIANS, MAX_RADIANS);
                else
                    return mapRange(0, this.min, this.max, MIN_RADIANS, MAX_RADIANS);
            },
            valueRadians () {
                return mapRange(this.value, this.min, this.max, MIN_RADIANS, MAX_RADIANS);
            },
            minX () {
                return MID_X + Math.cos(MIN_RADIANS) * RADIUS;
            },
            minY () {
                return MID_Y - Math.sin(MIN_RADIANS) * RADIUS;
            },
            maxX () {
                return MID_X + Math.cos(MAX_RADIANS) * RADIUS;
            },
            maxY () {
                return MID_Y - Math.sin(MAX_RADIANS) * RADIUS;
            },
            zeroX () {
                return MID_X + Math.cos(this.zeroRadians) * RADIUS;
            },
            zeroY () {
                return MID_Y - Math.sin(this.zeroRadians) * RADIUS;
            },
            valueX () {
                return MID_X + Math.cos(this.valueRadians) * RADIUS;
            },
            valueY () {
                return MID_Y - Math.sin(this.valueRadians) * RADIUS;
            },
            largeArc () {
                return Math.abs(this.zeroRadians - this.valueRadians) < Math.PI ? 0 : 1;
            },
            sweep () {
                return this.valueRadians > this.zeroRadians ? 0 : 1;
            },
            valueDisplay () {
                return this.valueDisplayFunction(this.value);
            },
        },
        methods: {
            updatePosition (offsetX, offsetY) {
                const dx = offsetX - this.size / 2;
                const dy =  this.size / 2 - offsetY;
                const angle = Math.atan2(dy, dx);
                let v;
                /* bit of weird looking logic to map the angles returned by Math.atan2() onto
                    our own unconventional coordinate system */
                const start = -Math.PI / 2 - Math.PI / 6;
                if (angle > MAX_RADIANS) {
                    v = mapRange(angle, MIN_RADIANS, MAX_RADIANS, this.min, this.max);
                } else if (angle < start) {
                    v = mapRange(angle + 2 * Math.PI, MIN_RADIANS, MAX_RADIANS, this.min, this.max);
                } else {
                    return;
                }
                this.$emit('input', Math.round((v - this.min) / this.stepSize) * this.stepSize + this.min);
            },
            onClick (e) {
                if (!this.disabled) {
                    this.updatePosition(e.offsetX, e.offsetY);
                }
            },
            onMouseDown (e) {
                if (!this.disabled) {
                    e.preventDefault();
                    window.addEventListener('mousemove', this.onMouseMove);
                    window.addEventListener('mouseup', this.onMouseUp);
                }
            },
            onMouseUp (e) {
                if (!this.disabled) {
                    e.preventDefault();
                    window.removeEventListener('mousemove', this.onMouseMove);
                    window.removeEventListener('mouseup', this.onMouseUp);
                }
            },
            onTouchStart (e) {
                if (!this.disabled) {
                    e.preventDefault();
                    window.addEventListener('touchmove', this.onTouchMove);
                    window.addEventListener('touchend', this.onTouchEnd);
                }
            },
            onTouchEnd (e) {
                if (!this.disabled) {
                    e.preventDefault();
                    window.removeEventListener('touchmove', this.onTouchMove);
                    window.removeEventListener('touchend', this.onTouchEnd);
                }
            },
            onMouseMove (e) {
                if (!this.disabled) {
                    e.preventDefault();
                    this.updatePosition(e.offsetX, e.offsetY);
                }
            },
            onTouchMove (e) {
                if (!this.disabled && e.touches.length == 1) {
                    const boundingClientRect = this.$el.getBoundingClientRect();
                    const touch = e.targetTouches.item(0);
                    const offsetX = touch.clientX - boundingClientRect.left;
                    const offsetY = touch.clientY - boundingClientRect.top;
                    this.updatePosition(offsetX, offsetY);
                }
            },
        }
    };
</script>

<style>
    .knob-control__range {
        fill: none;
        transition: stroke .1s ease-in;
    }
    .knob-control__value {
        fill: none;
    }
    .knob-control__text-display {
        font-size: 1.3rem;
        text-align: center;
    }
</style>
