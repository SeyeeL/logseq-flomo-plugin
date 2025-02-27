<template>
  <h3 class="customise-header">
    STEP 2：定制设置
    <a-button type="link" @click="toggle">{{ collapsed ? '显示' : '隐藏' }}</a-button>
  </h3>
  <div v-if="!collapsed">
    <a-row class="basic-row">
      <a-col :span="6">
        <div class="item-label">
          <label>标题</label>
          <a-tooltip placement="bottom">
            <template #title>
              <p>Page中flomo节点的标题，默认为flomo。</p>
            </template>
            <question-circle-outlined style="margin-left: 4px" />
          </a-tooltip>
        </div>
      </a-col>
      <a-col :span="16">
        <a-input v-model:value="title" @blur="saveTitle" />
      </a-col>
    </a-row>
    <a-row class="basic-row">
      <a-col :span="6">
        <div class="item-label">
          <label>同步模式</label>
          <a-tooltip placement="bottom">
            <template #title>
              <span>
                - 标签模式：每一个 tag 作为独立的 page<br />
                - 日记模式：指定时间范围内，把 memos 写入相应的 journal note 中<br />
                - 单页模式：所有的 memos 放一个 page 中<br />
                <!-- - 仅导出数据：仅导出flomo笔记，不带有logseq块属性，无法持续更新，用其他markdown工具打开无块属性污染，适用于备份数据。 -->
              </span>
            </template>
            <question-circle-outlined style="margin-left: 4px" />
          </a-tooltip>
        </div>
      </a-col>
      <a-col :span="16">
        <a-select
          ref="select"
          v-model:value="syncMode"
          style="width: 160px"
          @change="saveConfig($event, 'syncMode')"
        >
          <a-select-option value="1">标签模式</a-select-option>
          <a-select-option value="2">日记模式</a-select-option>
          <a-select-option value="3">单页模式</a-select-option>
        </a-select>
        <a-space style="margin-left: 50px"> </a-space>
      </a-col>
    </a-row>
    <a-row class="basic-row" v-if="syncMode === '1'">
      <a-col :span="6">
        <div class="item-label">标签范围</div>
      </a-col>
      <a-col :span="6">
        <a-select
          ref="select"
          v-model:value="rangeType"
          style="width: 100px"
          @change="saveConfig($event, 'rangeType')"
        >
          <a-select-option value="1">不限</a-select-option>
          <a-select-option value="2">排除标签</a-select-option>
          <a-select-option value="3">仅同步标签</a-select-option>
        </a-select>
      </a-col>
      <a-col :span="10" v-if="rangeType !== '1'">
        <a-select
          ref="select"
          mode="multiple"
          v-model:value="tagRange"
          style="width: 195px"
          :max-tag-count="1"
          :maxTagTextLength="4"
          @change="saveConfig($event, 'tagRange')"
        >
          <a-select-option v-for="item in tags" :key="item.name" :value="item.name">{{
            item.name
          }}</a-select-option>
        </a-select>
      </a-col>
    </a-row>
    <a-row class="basic-row" v-if="syncMode === '2'">
      <a-col :span="6">
        <div class="item-label">时间范围</div>
      </a-col>
      <a-col :span="16">
        <a-range-picker v-model:value="syncRange" @change="saveConfig($event, 'syncRange')" />
      </a-col>
    </a-row>
    <div>
      <a-switch
        size="small"
        v-model:checked="exportMode"
        @change="saveConfig($event, 'exportMode')"
        default-checked
      />
      <span>仅导出数据</span>
    </div>
    <div class="tips">
      导出内容不含 logseq 块属性，无法持续更新，用其他 markdown 工具打开无块属性污染，适用于备份数据
    </div>
    <div>
      <a-switch
        size="small"
        v-model:checked="addTime"
        @change="saveConfig($event, 'addTime')"
        default-checked
      />
      <span>memo 前加上时间</span>
      <div class="tips">如 12：00 这是一条 memo，日记模式只加时间，其他格式会加上日期</div>
    </div>
    <div></div>
  </div>
</template>

<style lang="less">
.customise-header {
  display: flex;
  justify-content: space-between;
}
.ant-picker.ant-picker-range {
  width: 100%;
}
.ant-switch-small {
  margin: -3px 10px 0 0;
}
.tips {
  color: #c5c5c5;
  font-size: 12px;
  padding: 0 35px 0 38px;
  &:not(:last-child) {
    margin-bottom: 10px;
  }
}
</style>

<script>
import { defineComponent, ref, toRefs, reactive, onMounted } from 'vue';
import { QuestionCircleOutlined } from '@ant-design/icons-vue';
import { fetchAllTags } from '../utils';
import dayjs from 'dayjs';
export default defineComponent({
  props: {
    title: {
      type: String,
    },
    maxCount: {
      type: Number,
      default: 0,
    },
    syncRange: {
      type: Array,
    },
  },
  components: {
    QuestionCircleOutlined,
  },
  setup(props, content) {
    const s = logseq.settings || {};
    const logseqSettings = reactive({
      exportMode: ref(s.exportMode),
      syncRange: ref(props.syncRange),
      tags: ref(s.tags),
      syncMode: ref(s.syncMode),
      tagRange: ref(s.tagRange),
      rangeType: ref('1'),
      addTime: ref(s.addTime),
      maxCount: ref(s.maxCount),
      title: ref(s.title),
      collapsed: ref(false),
    });
    const toggle = () => {
      logseqSettings.collapsed = !logseqSettings.collapsed;
    };
    const saveConfig = (e, name) => {
      let value = e;
      if (name === 'rangeType') {
        onfetchAllTags();
      } else if (name === 'tagRange') {
        content.emit('changeTagRange', e);
        return;
      } else if (name === 'syncRange') {
        content.emit('syncRangeChange', value);
        logseq.updateSettings({
          [name]: [dayjs(value[0]).format('YYYY-MM-DD'), dayjs(value[1]).format('YYYY-MM-DD')],
        });
        return;
      }
      console.log('saveConfig', name, value);
      logseq.updateSettings({ [name]: value });
    };
    const onfetchAllTags = async () => {
      const { cookie, token, server } = s;
      const { tags } = await fetchAllTags({ cookie, token, server });
      console.log(tags);
      const newTags = tags.filter(item => {
        return item.name.indexOf('/') === -1;
      });
      logseqSettings.tags = newTags;
    };
    const logseqSettingsRef = toRefs(logseqSettings);
    return {
      toggle,
      saveConfig,
      fetchAllTags,
      ...logseqSettingsRef,
    };
  },
});
</script>
