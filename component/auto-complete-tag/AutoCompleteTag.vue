<template>
  <a-popover
    :placement="placement"
    :trigger="trigger"
  >
    <template slot="content">
      <a-input-search class="mb-sm" size="small" v-if="apiConfig.allowSearch" placeholder="请输入关键词搜索" @input="getDataSource($event.target.value)" @search="getDataSource" />

      <a-spin :spinning="loading">
        <div :style="{ 'width': width }" @click="focusComponent">
          <h5>{{ dataTitle }}</h5>
          <div v-if="dataSource.length > 0">
            <div class="my-xs" v-for="(item, index) in dataSource" :key="index">
              <a-tag class="pointer" @click="addValue(item, index)">{{ item }}</a-tag>
            </div>
          </div>
          <div v-if="showTime">
            <h5>常用时间</h5>
            <div class="my-xs" v-for="(item, index) in commonTime" :key="index">
              <a-tag class="pointer" v-for="(val, index) in item" :key="index" @click="addValue(val, null)">{{ val }}</a-tag>
            </div>
          </div>
          <div v-if="!showTime && !loading && dataSource.length === 0">
            <a-empty></a-empty>
          </div>
        </div>
      </a-spin>
    </template>
    <a-input
      ref="input"
      v-if="inputType === 'input'"
      v-model="dataVal"
      :class="inputClass"
      :placeholder="placeholder"
      :allowClear="allowClear"
      :disabled="disabled"
      @change="valueChange"
      @click="openPopover"
    ></a-input>
    <a-textarea
      ref="text"
      v-if="inputType === 'text'"
      v-model="dataVal"
      :class="inputClass"
      :placeholder="placeholder"
      :rows="rows"
      :allowClear="allowClear"
      :disabled="disabled"
      @change="valueChange"
      @click="openPopover"
    ></a-textarea>
  </a-popover>
</template>

<script>
import { COMMON_TIME } from '@comp/common/auto-complete-tag/auto-complete-tag.const'
import { getAction, postAction } from '@api/manage'

export default {
  name: 'AutoCompleteTag',
  props: {
    // 输入类型, 支持文本域及输入框
    inputType: {
      type: 'text' | 'input',
      default: 'text',
    },
    // 输入框样式, 可自定义
    inputClass: {
      type: Object,
      default: () => {
        return {};
      }
    },
    // 数据源, 提示内容
    data: {
      type: Array,
      default: () => []
    },
    // 数据源标题, 默认为 `快速输入`
    dataTitle: {
      type: String,
      default: '快速输入'
    },
    // 提示框位置, 建议使用上方 `topLeft` 或者下方 `bottomLeft`
    placement: {
      type: 'top' | 'left' | 'right' | 'bottom' | 'topLeft' | 'topRight' | 'bottomLeft' | 'bottomRight' | 'leftTop' | 'leftBottom' | 'rightTop' | 'rightBottom',
      default: 'bottomLeft',
    },
    // 提示语, 默认 `请输入`
    placeholder: {
      type: String,
      default: '请输入',
    },
    // 文本域所占行数, 默认 `4` 行, inputType 为 text 生效
    rows: {
      type: Number,
      default: 4
    },
    // 是否允许调整大小, inputType 为 text 生效
    resize: {
      type: Boolean,
      default: true,
    },
    // 是否允许清空, 为 true 时, 显示可清空按钮
    allowClear: {
      type: Boolean,
      default: true
    },
    // 触发方式, 默认 `click`
    trigger: {
      type: 'hover' | 'focus' | 'click',
      default: 'click'
    },
    // 提示面板宽度, 默认 600px
    width: {
      type: String,
      default: '600px',
    },
    // 是否禁用组件
    disabled: {
      type: Boolean,
      default: false
    },
    // 是否显示常用时间
    showTime: {
      type: Boolean,
      default: true
    },
    // 文本分割符号, 默认为 `,`
    separator: {
      type: String,
      default: '，'
    },
    value: {
      type: String,
    },
    triggerChange: {
      type: Boolean,
      required: false,
      default: false
    },
    /**
     * 配置接口信息
     *
     * @param apiConfig 包含的字段：
     * apiConfig.method 接口请求方式, 支持 `get` | `post`
     * apiConfig.url 请求的接口地址, 若设置则开启搜索框, 支持远程搜索
     * apiConfig.params 额外请求参数, 用于数据过滤
     * apiConfig.textField 接口返回字段, 用于显示文本
     * apiConfig.searchField 搜索字段
     */
    apiConfig: {
      type: Object,
      default: () => {
        return {}
      }
    }
  },
  model: {
    prop: 'value',
    event: 'input'
  },
  watch: {
    value(val) {
      if (!val) {
        this.dataVal = ''
      } else {
        this.dataVal = val
      }
    },
    resize: {
      immediate: true,
      handler(val) {
        this.inputClass['no-resize'] = !val;
      }
    },
    data(val) {
      if (!val) {
        this.dataSource = [];
      } else {
        try {
          this.dataSource = [...val];
        } catch (e) {
          console.error('[AutoCompleteTag] data 不是合法的数组, 请检查 data 值');
        }
      }
    }
  },
  computed: {},
  data() {
    return {
      dataSource: [],  // 数据源, 提示文本列表
      dataVal: null,
      commonTime: COMMON_TIME,
      visible: false,  // 是否显示提示框
      loading: true,  // 是否正在加载
      searchVal: '',  // 搜索框绑定变量
    }
  },
  created() {
    if (this.apiConfig.url) {
      this.getDataSource();
    } else {
      this.loading = false;
    }
  },
  methods: {
    openPopover() {
      this.visible = true;
    },
    // 监听数据发生改变时触发
    valueChange(index) {
      this.$emit('input', this.dataVal)
      // 默认返回当前值, 若是从后台返回的数据, 可以返回整个对象, 便于做其他处理, 故抛出 valueChange 事件
      let item = this.dataVal;
      if (index !== null) {
        item = this.dataSource[index];
      }
      // 后期调整可抛出算个对象
      this.$emit('valueChange', item);
    },
    // 点击标签追加值
    addValue(val, index) {
      // 若dataVal为null 默认为空字符串, 否则自加会追加null字符串
      if (!this.dataVal) {
        this.dataVal = '';
      } else {
        this.dataVal += '，';
      }
      this.dataVal += `${val}`;
      this.valueChange(index);
      this.focusComponent();
    },
    focusComponent() {
      this.$refs[this.inputType].focus();
    },
    getDataSource(value) {
      if (this.apiConfig.searchField) {
        this.apiConfig.params[this.apiConfig.searchField] = value ? `*${value}*` : '';
      }
      this.loading = true;
      let httpAction;
      if (this.apiConfig.method === 'post') {
        httpAction = postAction(this.apiConfig.url, this.apiConfig.params, this.apiConfig.method)
      } else {
        httpAction = getAction(this.apiConfig.url, this.apiConfig.params, this.apiConfig.method)
      }
      // 发送接口请求, 获取数据并处理
      httpAction.then(res => {
        if (res && res.success) {
          const result = [...res.result];
          try {
            // 若配置了文本字段, 表示返回的是一个对象数组, 显示只需显示对应文本, 否则为字符串数组
            if (this.apiConfig.textField) {
              this.dataSource = result.map(m => m[this.apiConfig.textField]);
            } else {
              this.dataSource = [...result];
            }
          } catch (e) {
            console.warn('值转换失败, 请检查接口返回数据或者textField字段')
          }
          this.loading = false;
        }
      }, () => {
        console.error('接口调用失败, 请检查接口状态')
      })
    }
  }
}
</script>

<style scoped>
.no-resize /deep/ textarea {
  resize: none !important;
}
/**隐藏小箭头*/
body >>> .ant-popover-arrow {
  display: none !important;
}
</style>