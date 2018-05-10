<template>
  <div class="el-form-item" :class="[{
      'el-form-item--feedback': elForm && elForm.statusIcon,
      'is-error': validateState === 'error',
      'is-validating': validateState === 'validating',
      'is-success': validateState === 'success',
      'is-required': isRequired || required
    },
    sizeClass ? 'el-form-item--' + sizeClass : ''
  ]">
    <label :for="labelFor" class="el-form-item__label" v-bind:style="labelStyle" v-if="label || $slots.label">
      <label v-bind:style="labelLabelStyle"><slot name="label">{{label + form.labelSuffix}}</slot></label>
      <span>{{labelSuffix}}</span>
    </label>
    <div class="el-form-item__content" v-bind:style="contentStyle">
      <slot></slot>
      <transition name="el-zoom-in-top">
        <div
          v-if="validateState === 'error' && showMessage && form.showMessage"
          class="el-form-item__error"
          :class="{
            'el-form-item__error--inline': typeof inlineMessage === 'boolean'
              ? inlineMessage
              : (elForm && elForm.inlineMessage || false)
          }"
        >
          {{validateMessage}}
        </div>
      </transition>
    </div>
  </div>
</template>
<script>
  import AsyncValidator from '../../../local_modules/async-validator/lib/index';
  import emitter from 'element-ui/src/mixins/emitter';
  import { noop, getPropByPath } from 'element-ui/src/utils/util';

  export default {
    name: 'ElFormItem',

    componentName: 'ElFormItem',

    mixins: [emitter],

    provide() {
      return {
        elFormItem: this
      };
    },

    inject: ['elForm'],

    props: {
      label: String,
      labelSuffix: String,
      labelWidth: String,
      labelPosition: String,
      prop: String,
      required: {
        type: Boolean,
        default: undefined
      },
      requiredCount: String,
      requiredMessage: String,
      requiredChar: String,
      rules: [Object, Array],
      error: String,
      validateStatus: String,
      for: String,
      inlineMessage: {
        type: [String, Boolean],
        default: ''
      },
      showMessage: {
        type: Boolean,
        default: true
      },
      size: String
    },
    watch: {
      error(value) {
        this.validateMessage = value;
        this.validateState = value ? 'error' : '';
      },
      validateStatus(value) {
        this.validateState = value;
      }
    },
    computed: {
      labelFor() {
        return this.for || this.prop;
      },
      labelLabelStyle() {
        var ret = {};
        var labelWidth = this.labelWidth || this.form.labelWidth;
        if (labelWidth) {
          ret.display = 'inline-block';
          ret.width = labelWidth;
        }
        return ret;
      },
      labelStyle() {
        var ret = {};
        if (this.form.labelPosition === 'top') {
          if (this.labelPosition) {
            if (this.labelPosition === 'left') {
              ret.float = 'left';
              ret.paddingBottom = '0px';
            }
          }
          return ret;
        }
        return ret;
      },
      contentStyle() {
        var ret = {};
        const label = this.label;
        if (this.form.labelPosition === 'top' || this.form.inline) return ret;
        if (!label && !this.labelWidth && this.isNested) return ret;
        var labelWidth = this.labelWidth || this.form.labelWidth;
        if (labelWidth) {
          ret.marginLeft = labelWidth;
        }
        return ret;
      },
      form() {
        let parent = this.$parent;
        let parentName = parent.$options.componentName;
        while (parentName !== 'ElForm') {
          if (parentName === 'ElFormItem') {
            this.isNested = true;
          }
          parent = parent.$parent;
          parentName = parent.$options.componentName;
        }
        return parent;
      },
      fieldValue: {
        cache: false,
        get() {
          var model = this.form.model;
          if (!model || !this.prop) { return; }

          var path = this.prop;
          if (path.indexOf(':') !== -1) {
            path = path.replace(/:/, '.');
          }

          return getPropByPath(model, path, true).v;
        }
      },
      isRequired() {
        let rules = this.getRules();
        let isRequired = false;

        if (rules && rules.length) {
          rules.every(rule => {
            if (rule.required) {
              isRequired = true;
              return false;
            }
            return true;
          });
        }
        return isRequired;
      },
      _formSize() {
        return this.elForm.size;
      },
      elFormItemSize() {
        return this.size || this._formSize;
      },
      sizeClass() {
        return (this.$ELEMENT || {}).size || this.elFormItemSize;
      }
    },
    data() {
      return {
        validateState: '',
        validateMessage: '',
        validateDisabled: false,
        validator: {},
        isNested: false
      };
    },
    methods: {
      validate(trigger, callback = noop) {
        this.validateDisabled = false;
        var rules = this.getFilteredRule(trigger);
        if ((!rules || rules.length === 0) && this.required === undefined) {
          callback();
          return true;
        }
        this.validateState = 'validating';

        var descriptor = {};
        if (rules && rules.length > 0) {
          rules.forEach(rule => {
            delete rule.trigger;
          });
        }
        if (this.requiredCount !== undefined) {
          rules.push({ validator: (rule, value, callback) => {
              if (value.length > this.requiredCount) {
                return callback(new Error('最多输入' + this.requiredCount + '个字'));
              } else {
                return callback();
              }
            }
          });
        }
        // requiredChar
        // if (this.requiredChar !== undefined) {
        //   rules.push({ validator: (rule, value, callback) => {
        //       if (/[`~!@#$%^&*()_+<>?:"{},.\/;'[\]]/im.test(value)) {
        //         return callback(new Error('仅支持汉字、字母、数字、标点输入'));
        //       } else {
        //         return callback();
        //       }
        //     }
        //   });
        // }


        descriptor[this.prop] = rules;
        var label = this.label || '该字段';
        var validator = new AsyncValidator(descriptor, label);
        var model = {};

        model[this.prop] = this.fieldValue;

        validator.validate(model, { firstFields: true }, (errors, fields) => {
          this.validateState = !errors ? 'success' : 'error';
          this.validateMessage = errors ? errors[0].message : '';

          callback(this.validateMessage);
        });
      },
      clearValidate() {
        this.validateState = '';
        this.validateMessage = '';
        this.validateDisabled = false;
      },
      resetField() {
        this.validateState = '';
        this.validateMessage = '';

        let model = this.form.model;
        let value = this.fieldValue;
        let path = this.prop;
        if (path.indexOf(':') !== -1) {
          path = path.replace(/:/, '.');
        }

        let prop = getPropByPath(model, path, true);

        if (Array.isArray(value)) {
          this.validateDisabled = true;
          prop.o[prop.k] = [].concat(this.initialValue);
        } else {
          this.validateDisabled = true;
          prop.o[prop.k] = this.initialValue;
        }
      },
      getRules() {
        var formRules = this.form.rules;
        var selfRules = this.rules;
        var requiredRule = this.required !== undefined ? { required: !!this.required } : [];
        if (this.requiredMessage) {
          requiredRule = { validator: (rule, value, callback) => {
              if (!value) {
                return callback(new Error(this.requiredMessage));
              } else {
                return callback();
              }
            }
          };
        }
        formRules = formRules ? formRules[this.prop] : [];

        return [].concat(selfRules || formRules || []).concat(requiredRule);
      },
      getFilteredRule(trigger) {
        var rules = this.getRules();

        return rules.filter(rule => {
          return !rule.trigger || rule.trigger.indexOf(trigger) !== -1;
        });
      },
      onFieldBlur() {
        this.validate('blur');
      },
      onFieldChange() {
        if (this.validateDisabled) {
          this.validateDisabled = false;
          return;
        }

        this.validate('change');
      }
    },
    mounted() {
      if (this.prop) {
        this.dispatch('ElForm', 'el.form.addField', [this]);

        let initialValue = this.fieldValue;
        if (Array.isArray(initialValue)) {
          initialValue = [].concat(initialValue);
        }
        Object.defineProperty(this, 'initialValue', {
          value: initialValue
        });

        let rules = this.getRules();

        if (rules.length || this.required !== undefined) {
          this.$on('el.form.blur', this.onFieldBlur);
          this.$on('el.form.change', this.onFieldChange);
        }
      }
    },
    beforeDestroy() {
      this.dispatch('ElForm', 'el.form.removeField', [this]);
    }
  };
</script>
