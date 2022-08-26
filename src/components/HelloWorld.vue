<template>
  <input
    v-bind="$attrs"
    class="hd-input"
    :value="realValue"
    @input="handleInput"
    @change="handleChange"
    @blur="handleBlur"
    @focus="handleFocus"
  />
</template>

<script>
export default {
  name: 'hd-input',
  inheritAttrs: false,
  components: {},
  model: {
    prop: 'value',
    event: 'input',
  },
  props: {
    value: [String, Number],
    limitType: {
      type: String,
      validator: limitType => {
        return ['int', 'float', 'decimal', 'idCard', undefined].includes(limitType);
      },
    },
    // 最大数阈值限制
    max: {
      type: Number,
    },
    // 最小数阈值限制
    min: {
      type: Number,
    },
    // 宽松的数据控制模式，允许外部组件更新数据
    isOld: {
      type: Boolean,
      default: false,
    },
    // 触发校验的时机
    triggerLimit: {
      type: [Array, String],
      default: () => ['watch', 'change'],
      validator(triggerLimit) {
        const values = ['watch', 'input', 'change', 'blur', 'focus'];
        if (Array.isArray(triggerLimit)) {
          return triggerLimit.every(item => values.includes(item));
        }
        return values.includes(triggerLimit);
      },
    },
    // 移除空白字符
    limitSpace: {
      type: Boolean,
      default: false,
    },
    // 允许一个空字符
    allowSingleEmptyChar: { type: Boolean, default: false },
    // 校验触发前回调函数
    beforeLimitFun: {
      type: Function,
    },
    // 自定义校验方法
    customLimitFun: {
      type: Function,
    },
  },
  data() {
    return {
      realValue: this.value,
    };
  },
  computed: {
    triggerLimitCal() {
      if (this.isOld) return ['blur'];
      return this.triggerLimit;
    },
    triggerLimitArrCal() {
      return Array.isArray(this.triggerLimitCal) ? this.triggerLimitCal : [this.triggerLimitCal];
    },
  },
  watch: {
    value: {
      handler(newValue, oldValue) {
        console.log(`ADI-LOG => value watch: newValue`, newValue);
        console.log(`ADI-LOG => value watch: oldValue`, oldValue);
        console.log(`ADI-LOG => value watch: this.realValue`, this.realValue);
        this.realValue = this.limitFun(newValue, 'watch');
      },
      immediate: true,
    },
    realValue(newValue) {
      console.log(`ADI-LOG => realValue watch:`, newValue);
      this.$emit('input', newValue);
    },
  },
  created() {},
  mounted() {},
  methods: {
    limitFun(newValue, triggerLimitType) {
      if (!this.triggerLimitArrCal.includes(triggerLimitType)) {
        // 允许外部组件更新数据
        if (triggerLimitType === 'watch') {
          return newValue;
        } else {
          return this.realValue;
        }
      }
      const _limitFun = this.customLimitFun ?? this.defaultLimitFun;
      return _limitFun(newValue);
    },
    handleInput(e) {
      this.realValue = this.limitFun(e.target.value, 'input');
      console.log(`ADI-LOG => handleInput`, e.target.value, '=-=', this.realValue);
      this.$emit('input', this.realValue);
    },
    handleChange(e) {
      this.realValue = this.limitFun(e.target.value, 'change');
      console.log(`ADI-LOG => handleChange`, e.target.value, '=-=', this.realValue);
      this.$emit('change', e);
    },
    handleBlur(e) {
      console.log(`ADI-LOG => handleBlur`, e.target.value, '=-=', this.realValue);
      this.realValue = this.limitFun(e.target.value, 'blur');
      console.log(`ADI-LOG => handleBlur2`, e.target.value, '=-=', this.realValue);
      // Reftesh DOM Data
      e.target.value !== this.realValue && (e.target.value = this.realValue);
      this.$emit('blur', e);
    },
    handleFocus(e) {
      this.realValue = this.limitFun(e.target.value, 'focus');
      console.log(`ADI-LOG => handleFocus`, e.target.value, '=-=', this.realValue);
      this.$emit('focus', e);
    },
    defaultLimitFun(newValue) {
      console.log(`ADI-LOG => defaultLimitFun`, newValue);
      if (this.beforeLimitFun) {
        const resValue = this.beforeLimitFun(newValue);
        if (resValue !== undefined) return resValue;
      }

      if (this.allowSingleEmptyChar && newValue === '') {
        return newValue;
      }
      let _value = String(newValue);
      // 不能输入空白
      if (this.limitSpace) {
        _value = _value.replace(/\s*/g, '');
      }
      // 最大最小值限制
      if (['int', 'float', 'decimal'].includes(this.limitType)) {
        _value = this.checkNumLimit(_value);
      }
      switch (this.limitType) {
        // 限制输入数字
        case 'int':
          _value = _value.replace(/[^\d]/g, '');
          break;
        // 限制输入（数字 + 小数点）
        case 'float':
          _value = _value.replace(/[^\d{1,}.\d{1,}|\d{1,}]/g, '');
          break;
        // 限制输入两位小数（数字 + 小数点）
        case 'decimal':
          _value = this.repalcenNum(_value);
          break;
      }
      // 还原number类型
      if (typeof newValue === 'number') {
        _value = this.toNumber(_value);
      }
      return _value;
    },
    // 转换为Number
    toNumber(val) {
      const n = parseFloat(val);
      return isNaN(n) ? 0 : n;
    },
    // 校验最大最小值
    checkNumLimit(value) {
      const numberValue = this.toNumber(value);
      console.log(`ADI-LOG => numberValue`, numberValue);
      const isString = typeof value === 'string';

      if (this.min !== undefined && numberValue < this.min) {
        return isString ? String(this.min) : this.min;
      }
      if (this.max !== undefined && numberValue > this.max) {
        return isString ? String(this.max) : this.max;
      }
      return value;
    },
    // 格式化两位小数（数字 + 小数点）
    repalcenNum(value) {
      value = value + '';
      value = value.replace(/[^\d.]/g, ''); // 清除“数字”和“.”以外的字符
      value = value.replace(/^\./g, ''); // 验证第一个字符是数字而不是.
      value = value.replace(/\.{2,}/g, '.'); // 只保留第一个.并清除多余的.
      value = value.replace('.', '$#$').replace(/\./g, '').replace('$#$', '.'); // 只允许输入一个小数点
      value = value.replace(/^(-)*(\d+)\.(\d\d).*$/, '$1$2.$3'); // 只能输入两个小数
      return value;
    },
  },
};
</script>
