<template>
  <div
    class="wat-flex"
    @touchstart="touchstart($event)"
    @touchmove="touchmove($event)"
    @touchend="touchend($event)"
    @mousedown="touchstart($event)"
    v-if="aLists.length > 0"
  >
    <div
      class="wat-picker wat-flex-item"
      v-for="(list, index) in aLists"
      :key="index"
    >
      <div class="wat-picker_mask" :data-index="index"></div>
      <div class="wat-picker_line" :data-index="index"></div>
      <div class="wat-picker_slider wat-flex">
        <ul
          class="wat-flex-item"
          v-if="aPickerData[index]"
          :style="aPickerData[index].oPosition"
        >
          <li v-for="(item, index) in list" :key="index">
            {{ item.name || item }}
          </li>
        </ul>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: "picker",
  props: {
    // 初始化传进来的值
    lists: {
      type: Array,
      required: true
    },
    // 限制列数
    columns: {
      type: String
    },
    value: {
      type: Array
    }
  },
  data() {
    return {
      bIsLinkage: false,
      nWaitTime: 150,
      aLists: [], // copy from lists
      aPickerData: [],
      aResult: []
    };
  },
  created() {
    // copy data
    if (
      this.columns === undefined ||
      (this.columns && !isNaN(Number(this.columns)))
    ) {
      this.aLists = this.lists;
    }
    if (this.columns) {
      if (isNaN(Number(this.columns))) {
        console.error(`[WAT error] columns 必须是一个数字`);
        return;
      }
      this.bIsLinkage = true;
      let aLinkageList = [[]];
      this.aLists.forEach(val => {
        if (val.parent === "0") {
          aLinkageList[0].push(val);
        }
      });
      this.aLists = this.resolveLinkageData(aLinkageList);
    }
  },
  mounted() {
    if (this.columns && isNaN(Number(this.columns))) return;
    this.aPickerData = this.aLists.map((oData, index) => {
      let nFindIndex;
      // 初始化定位
      if (this.value.length > 0) {
        nFindIndex = oData.findIndex((val, i) => {
          let type = typeof val;
          let objectType = Object.prototype.toString.call(val);
          if (
            objectType === "[object Object]" ||
            type === "number" ||
            type === "string"
          ) {
            if (objectType === "[object Object]") {
              return String(val.value) === String(this.value[index]);
            } else {
              return String(val) === String(this.value[index]);
            }
          } else {
            console.error(
              `[WAT error] lists元素内不能包含除对象、字符串、数字之外的数据类型，lists[${index}][${i}]`
            );
          }
        });
        nFindIndex = nFindIndex < 0 ? 0 : nFindIndex;
      } else {
        nFindIndex = 0;
      }
      return {
        nStartY: 0,
        nEndPosition: (3 - nFindIndex) * 34,
        oPosition: {
          transform: `translate3d(0, ${(3 - nFindIndex) * 34}px, 0)`
        }
      };
    });
  },
  watch: {
    aResult(newData, oldData) {
      if (JSON.stringify(newData) !== JSON.stringify(oldData)) {
        setTimeout(
          () => {
            this.$emit("on-change", newData);
          },
          this.bIsLinkage ? 50 : this.nWaitTime
        );
      }
      console.log("hhh");
    },
    aLists(newData, oldData) {
      // 数据变化校验是否超出距离，这里主要为了每月的总日数不一致做矫正
      newData.map((val, i) => {
        if (val.length - 1 < 3 - this.aPickerData[i].nEndPosition / 34) {
          let maxVal = (3 - val.length + 1) * 34;
          this.$set(this.aPickerData, i, {
            nEndPosition: maxVal,
            oPosition: {
              transform: `translate3d(0, ${maxVal}px, 0)`,
              transition: "all .3s ease-out"
            }
          });
        }
      });
    }
  },
  methods: {
    touchstart(e) {
      e.preventDefault();
      let clientY = e.touches ? e.touches[0].pageY : e.pageY;
      let nIndex = e.touches
        ? +e.touches[0].target.dataset.index
        : +e.target.dataset.index;
      this.aPickerData[nIndex].nStartY = clientY;
      this.nStartTime = new Date().getTime();

      // pc滑动兼容
      if ("onmousemove" in window) {
        document.addEventListener("mousemove", this.touchmove, false);
        document.addEventListener("mouseup", this.touchend, false);
      }
    },

    touchmove(e) {
      e.preventDefault();
      let clientY = e.touches ? e.touches[0].pageY : e.pageY;
      let nIndex = e.touches
        ? +e.touches[0].target.dataset.index
        : +e.target.dataset.index;
      let nScrollY = parseInt(
        this.aPickerData[nIndex].nEndPosition +
          clientY -
          this.aPickerData[nIndex].nStartY
      );
      this.aPickerData[nIndex].oPosition = {
        transform: `translate3d(0, ${nScrollY}px, 0)`
      };
    },
    touchend(e) {
      e.preventDefault();

      let clientY = e.changedTouches ? e.changedTouches[0].pageY : e.pageY;
      let nIndex = e.changedTouches
        ? +e.changedTouches[0].target.dataset.index
        : +e.target.dataset.index;

      // 加速度
      let nSpeed = clientY - this.aPickerData[nIndex].nStartY;
      let sDirection = nSpeed > 0 ? "up" : "down";
      if (nSpeed === 0) return;
      let nTime = new Date().getTime() - this.nStartTime;
      if (nTime < 300) {
        nSpeed = nSpeed * 3;
        this.nWaitTime = 500;
      }

      let nScrollY = parseInt(this.aPickerData[nIndex].nEndPosition + nSpeed);
      let nMaxLen = this.aLists[nIndex].length - 4;

      // 处理边界问题
      if (Math.abs(nScrollY) > nMaxLen * 34 && sDirection === "down") {
        nScrollY = -nMaxLen * 34;
      } else if (nScrollY > 102 && sDirection === "up") {
        nScrollY = 102;
      } else if (nScrollY % 34 !== 0) {
        nScrollY = Math.round(nScrollY / 34) * 34;
      }
      this.aPickerData[nIndex].oPosition = {
        transform: `translate3d(0, ${nScrollY}px, 0)`,
        transition: "all .5s ease-out"
      };
      this.aPickerData[nIndex].nEndPosition = nScrollY;

      // if linkage
      if (this.bIsLinkage && +this.columns !== nIndex + 1) {
        this.aLists = this.resolveLinkageData(
          this.aLists,
          nIndex,
          3 - nScrollY / 34
        );
        this.aPickerData.forEach((oData, i) => {
          if (i > nIndex) {
            oData.nEndPosition = 102;
            oData.oPosition = {
              transform: `translate3d(0, 102px, 0)`,
              transition: "all .3s ease-out"
            };
          }
        });
      }

      // return result
      let aResult = this.aPickerData.map((val, nIndex) => {
        let result = this.aLists[nIndex][3 - val.nEndPosition / 34];
        return typeof result === "object"
          ? String(result.value) || ""
          : String(result);
      });
      this.aResult = aResult;

      // 移除滑动和抬起时间
      if ("onmousemove" in window) {
        document.removeEventListener("mousemove", this.touchmove, false);
        document.removeEventListener("mouseup", this.touchend, false);
      }
    },
    /**
     * @description 解析联动数据
     * @param {Array} [aColumns] 当前列数据，需要找下列数
     * @param {Number} [nStartIdx=0] 当前处理的列下标
     * @param {Number} [nFristIdx=0] 列表里的第几位数
     */
    resolveLinkageData(aColumns, nStartIdx = 0, nFristIdx = 0) {
      if (+this.columns === 1) return aColumns;
      let len = aColumns.length - 1;
      let bIsNext = false;
      let value = aColumns[nStartIdx][nFristIdx].value;
      // 如果当前滑动的列表存在，则重置
      if (aColumns[nStartIdx + 1]) {
        aColumns[nStartIdx + 1] = undefined;
      }
      // 循环查找关联的下一级
      for (let i = 0; i < this.lists.length; i++) {
        if (this.lists[i].parent === value) {
          if (!aColumns[nStartIdx + 1]) {
            aColumns[nStartIdx + 1] = [this.lists[i]];
          } else {
            aColumns[nStartIdx + 1].push(this.lists[i]);
          }
          bIsNext = true;
        }
      }
      // 如果没有下一级或者不需要那么多列的情况下（滑动了后续需要联动变化，直接判断aColumns.length === +this.columns会有问题，
      // 因为这里aColumns不会清空，需要判断当前更新数据的下标）结束递归
      if (
        !bIsNext ||
        (aColumns.length === +this.columns && nStartIdx + 2 === +this.columns)
      ) {
        return aColumns;
      }
      return this.resolveLinkageData(aColumns, nStartIdx + 1, 0);
    }
  }
};
</script>

<style scoped lang="less">
@import "./index.less";
</style>