<template>
<transition name="city-picker-transition">
  <div class="city-picker-container" @click="handleWrapper" v-show="visible">
    <transition
      v-on:after-enter="afterEnterAnimation"
      v-on:before-leave="beforeLeaveAnimation"
      name="citypicker-slide">
      <div class="wrapper"
        :style="wrapperHeight"
        v-show="visible">
        <div class="head">
          所在地区
          <span class="close" @click="close">X</span>
        </div>
        <div class="con">
          <div class="tab">
            <span
              @click="handleTabs(1)"
              :class="curTabs === 1 ? 'tabCur' : ''">
              {{selected.province.name || '请选择'}}
            </span>
            <span
              @click="handleTabs(2)"
              v-show="Object.keys(Citys).length > 0"
              :class="curTabs === 2 ? 'tabCur' : ''">
              {{selected.city.name || '请选择'}}
            </span>
            <span
              @click="handleTabs(3)"
              v-show="Object.keys(Areas).length > 0"
              :class="curTabs === 3 ? 'tabCur' : ''">
              {{selected.area.name || '请选择'}}
            </span>
          </div>
          <div class="tab-con">
            <div class="province" v-show="curTabs === 1">
              <span
                v-for="(value, key) in Province"
                :key="key"
                @click="handleProvince(key, value)"
                :class="key === selected.province.code ? 'selected' : ''">
              {{value}}
              </span>
            </div>
            <div class="city" v-show="curTabs === 2">
              <span
                v-for="(value, key) in Citys"
                :key="key"
                @click="handleCity(key, value)"
                :class="key === selected.city.code ? 'selected' : ''">
              {{value}}
              </span>
            </div>
            <div class="area" v-show="curTabs === 3">
              <span
                v-for="(value, key) in Areas"
                :key="key"
                @click="handleArea(key, value)"
                :class="key === selected.area.code ? 'selected' : ''">
              {{value}}
              </span>
            </div>
          </div>
        </div>
      </div>
    </transition>
  </div>
</transition>
</template>

<script>
  import locationData from './list';

  const Province = {}; // 省（自治区、直辖市、特别行政区）
  const locationCode = Object.keys(locationData);
  locationCode.forEach((key) => {
    // 省
    if (key.match(/^\d{2}0000$/)) {
      Province[key] = locationData[key];
    }
  });
  // const Area = {}; // 县（市辖区、县级市、旗)
  function getCitys(provinceCode) {
    const citys = {};
    const area = {};
    let isMunicipality = true; // 是否是自治区、直辖市、特别行政区
    locationCode.forEach((key) => {
      const cityCode = key - provinceCode;
      if (cityCode > 0 && cityCode < 1e4) {
        // 存在 xxxx00 的为非直辖市
        if (/00$/.test(cityCode)) {
          isMunicipality = false;
          citys[key] = locationData[key];
        } else {
          area[key] = locationData[key];
        }
      }
    });
    return {
      isMunicipality,
      values: isMunicipality ? area : citys,
    };
  }
  function getAreas(cityCode) {
    const area = {};
    locationCode.forEach((key) => {
      const areaCode = key - cityCode;
      if (areaCode > 0 && areaCode < 1e2) {
        area[key] = locationData[key];
      }
    });
    return area;
  }

  export default {
    name: 'cityPicker',
    props: {
      value: {
        type: Boolean,
        default: false,
      },
      defaultData: {
        type: Array,
        default: () => [],
      },
    },
    data() {
      return {
        selected: {
          province: {
            code: '',
            name: '',
          },
          city: {
            code: '',
            name: '',
            isMunicipality: false,
          },
          area: {
            code: '',
            name: '',
          },
        },
        Province,
        Citys: {},
        Areas: {},
        curTabs: 1, // province/citys/area
        wrapperHeight: {},
      };
    },
    computed: {
      visible() {
        return this.value;
      },
    },
    watch: {
      value(val) {
        if (val && this.defaultData.length > 0) {
          this.setDefaultData(this.defaultData);
        }
      },
    },
    methods: {
      close() {
        this.Citys = {};
        this.Areas = {};
        this.selected.province.code = '';
        this.selected.province.name = '';
        this.selected.city.code = '';
        this.selected.city.name = '';
        this.selected.area.code = '';
        this.selected.area.name = '';
        this.curTabs = 1;
        this.$emit('input', false);
      },
      handleWrapper(e) {
        // console.log(e.target)
        if (e.target.className === 'city-picker-container') {
          this.close();
        }
      },
      handleProvince(key) {
        let citys = {};
        if (this.selected.provinceCode !== key) {
          citys = getCitys(key);
          // 城市数据
          this.Citys = citys.values;
          // 是否是直辖市
          this.selected.city.isMunicipality = citys.isMunicipality;
          // 重置市
          this.selected.city.code = '';
          this.selected.city.name = '';
          // 重置区
          this.selected.area.name = '';
          this.selected.area.code = '';
          this.Areas = '';
        }
        this.selected.province.code = key;
        this.selected.province.name = locationData[key];
        // 设置当前tab
        if (Object.keys(citys.values).length > 0) {
          this.curTabs = 2;
        } else {
          this.finish();
        }
      },
      handleCity(key) {
        let areas = {};
        if (!this.selected.city.isMunicipality
          && this.selected.city.code !== key
        ) {
          areas = getAreas(key);
          if (Object.keys(areas).length > 0) {
            this.Areas = areas;
            this.selected.area.code = '';
            this.selected.area.name = '';
            this.curTabs = 3;
          }
        }
        this.selected.city.code = key;
        this.selected.city.name = locationData[key];
        if (this.selected.city.isMunicipality || Object.keys(areas).length === 0) {
          this.finish();
        }
      },
      handleArea(key) {
        this.selected.area.code = key;
        this.selected.area.name = locationData[key];
        this.finish();
      },
      handleTabs(index) {
        this.curTabs = index;
      },
      finish() {
        const temp = {};
        Object.keys(this.selected).forEach((key) => {
          temp[key] = {};
          temp[key].code = this.selected[key].code;
          temp[key].name = this.selected[key].name;
        });
        this.close();
        this.$emit('on-finish', temp);
      },
      setDefaultData(data) {
        let curTabs = 1;
        const temp = [];
        data.forEach((item) => {
          if (item !== '' && item !== undefined) {
            temp.push(item.toString());
          }
        });
        if (temp.length === 0) {
          return false;
        }
        // 设置省份
        this.selected.province.code = temp[0];
        this.selected.province.name = locationData[temp[0]];
        // 设置市
        const citys = getCitys(temp[0]);
        if (Object.keys(citys.values).length > 0) {
          this.selected.city.code = temp[1];
          this.selected.city.name = locationData[temp[1]];
          this.Citys = citys.values;
          curTabs = 2;
          // 设置地区
          if (temp[2]) {
            const areas = getAreas(temp[1]);
            // debugger;
            if (Object.keys(areas).length > 0) {
              this.selected.area.code = temp[2];
              this.selected.area.name = locationData[temp[2]];
              this.Areas = areas;
              curTabs = 3;
            }
          }
        }
        this.curTabs = curTabs;
        return true;
      },
      afterEnterAnimation() {
        this.wrapperHeight = {
          height: '60%',
        };
      },
      beforeLeaveAnimation() {
        this.wrapperHeight = {};
      },
    },
  };
</script>

<style lang="less" scoped>
  .city-picker-container{
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0, 0, 0, 0.5);
    -webkit-transform: translateZ(0);
    transform: translateZ(0);
    z-index: 999;
    .wrapper{
      position:absolute;
      left:0;
      right:0;
      bottom: 0;
      background: #fff;
      display: flex;
      flex-direction: column;
      overflow: hidden;
      font-size: 14px;
      -webkit-transform: translateZ(0);
      transform: translateZ(0);
      .head{
        flex: 0 0 41px;
        height: 41px;
        line-height: 41px;
        text-align: center;
        border-bottom: 1px solid #e3e3e3;
        .close{
          float: right;
          color:#ccc;
          display: inline-block;
          padding: 0 10px;
        }
      }
      .con{
        flex: 1 1 100%;
        width: 100%;
        position:relative;
        .tab{
          width: 100%;
          height: 31px;
          line-height: 31px;
          padding: 0 5px;
          border-bottom: 1px solid #e3e3e3;
          position: relative;
          z-index: 10;
          white-space: nowrap;
          overflow-x: auto;
          overflow-y: hidden;
          span{
            display: inline-block;
            padding: 0 5px;
            line-height: 30px;
            margin-right: 10px;
          }
          .tabCur{
            color: #ff3131;
            // border-bottom: 2px solid #ff3131;
          }
        }
        .tab-con{
          position: absolute;
          width: 100%;
          height: 100%;
          top:0;
          padding-top: 31px;
          height: 100%;
          box-sizing: border-box;
          overflow:hidden;
        }
        .province,.city,.area{
          width: 100%;
          height:100%;
          padding: 0 10px 10px 10px;
          box-sizing: border-box;
          overflow-y: scroll;
          -webkit-overflow-scrolling: touch;
          span{
            display: block;
            line-height: 30px;
          }
          .selected{
            color: #ff3131;
            &:after{
              // font-family: 'iconfont';
              content: '';
              margin-left: 4px;
              font-size: 12px;
              display: inline-block;
              width: 8px;
              height: 14px;
              border-right: 1px solid #ff3131;
              border-bottom: 1px solid #ff3131;
              transform: rotate(45deg);
              // position: relative;
              // top: 1px;
            }
          }
        }
      }
    }
  }
</style>

<style class="less">
.city-picker-transition-enter-active, .city-picker-transition-leave-active{
  transition: opacity 0.3s linear;
}
.city-picker-transition-enter, .city-picker-transition-leave-to{
  opacity: 0;
}
.citypicker-slide-enter-active, .citypicker-slide-leave-active {
  height: 60%;
  transition: all 0.3s linear;
}
.citypicker-slide-enter, .citypicker-slide-leave-to{
  height: 0%;
}
</style>
