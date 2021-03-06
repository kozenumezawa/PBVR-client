<template lang="pug">
.title Opacity
canvas#opacity(width="430" height="200" @mousemove="onMouseMove" @mousedown="onMouseDown"
              @mouseup="onMouseUp" @mouseleave="onMouseUp")
.row
  .column.column-50.min-value: small {{ minValue }}
  .column.column-60.max-value: small.float-right {{ maxValue }}
</template>

<script>
import _ from 'lodash';
import spline from 'cardinal-spline-js';

import helper from '../helper';

const initialOpacity = [[0, 100], [215, 100], [430, 100]];

export default {
  data: () => ({
    currentOpacity: initialOpacity,
    pIndex: -1,
    isDown: false,
  }),
  computed: {
    minValue() {
      return this.$parent.minValue;
    },
    maxValue() {
      return this.$parent.maxValue;
    },
    opacity() {
      return document.getElementById('opacity');
    },
    context() {
      return document.getElementById('opacity').getContext('2d');
    },
    getCurvePoints() {
      return _.chunk(spline.getCurvePoints(_.flatten(this.currentOpacity), 0.2, 200), 2);
    }
  },
  ready() {
    this.initGraph();

    this.$on('reset', () => {
      this.currentOpacity = initialOpacity;
      this.initGraph();
      this.$parent.emit('updateTransferFunction');
    });
  },
  methods: {
    initGraph() {
      this.context.clearRect(0, 0, this.opacity.width, this.opacity.height);
      this.drawGrid();
      this.drawLine();
      this.drawHandles();

      // normalize opacity range
      this.$parent.opacity = _.range(100).map(i => 1 - this.getCurvePoints.find(pt => {
        return Math.floor(pt[0] / this.opacity.width * 100) === i;
      })[1] / this.opacity.height);
    },
    drawGrid() {
      this.context.lineWidth = 0.3;
      this.context.strokeStyle = '#202020';
      this.context.beginPath();
      this.context.moveTo(0, this.opacity.height / 2);
      this.context.lineTo(this.opacity.width, this.opacity.height / 2);
      this.context.stroke();
      this.context.beginPath();
      this.context.moveTo(this.opacity.width / 3, 0);
      this.context.lineTo(this.opacity.width / 3, this.opacity.height);
      this.context.stroke();
      this.context.beginPath();
      this.context.moveTo(this.opacity.width / 3 * 2, 0);
      this.context.lineTo(this.opacity.width / 3 * 2, this.opacity.height);
      this.context.stroke();
      this.context.strokeStyle = '#000000';
      this.context.lineWidth = 1.0;
    },
    drawLine() {
      const s = this.getCurvePoints;
      this.context.beginPath();
      for(let i=0; i < s.length - 1; i++) {
        this.context.lineTo(s[i][0], s[i][1]);
      }
      this.context.stroke();
    },
    drawHandles() {
      this.context.strokeStyle = 'black';
      this.context.beginPath();
      for(let i=1; i < this.currentOpacity.length - 1; i++){
        this.context.rect(this.currentOpacity[i][0] - 3, this.currentOpacity[i][1] - 3, 6, 6);
      }
      this.context.stroke();
    },
    onMouseDown(e) {
      const pos = helper.getClickedPoint(e);

      this.pIndex = -1;
      this.isDown = false;

      for(let i=0; i < this.currentOpacity.length; i++) {
        if(Math.abs(pos.x - this.currentOpacity[i][0]) < 10 && Math.abs(pos.y - this.currentOpacity[i][1] < 10)){
          this.pIndex = i;
          this.isDown = true;
          return;
        }
      }

      const s = this.getCurvePoints;
      const newHandlePos = s.find(pt => Math.abs(pt[0] - pos.x) < 10 && Math.abs(pt[1] - pos.y) < 10);

      if(!newHandlePos) return;

      this.currentOpacity = this.currentOpacity.concat([newHandlePos]).sort((a, b) => a[0] - b[0]);

      this.onMouseDown(e);
      this.initGraph();
    },
    onMouseMove(e) {
      if(!this.isDown || this.pIndex === 0 || this.pIndex === this.currentOpacity.length - 1) return;

      const pos = helper.getClickedPoint(e);
			this.currentOpacity[this.pIndex] = [pos.x, pos.y];

      // Remove the point if its position is overrapped with another point
      this.currentOpacity = _.uniqWith(this.currentOpacity.sort((a, b) => a[0] - b[0]), (a, b) => Math.abs(a[0] - b[0]) < 1);

      this.initGraph();
    },
    onMouseUp() {
      this.isDown = false;
      this.$parent.emit('updateTransferFunction');
    }
  }
};
</script>

<style scoped>
#opacity {
  border: 1px solid #aaa;
  cursor: crosshair;
}
.max-value {
  padding-right: 40px;
}
</style>
