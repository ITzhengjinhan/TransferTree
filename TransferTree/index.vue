<template>
  <a-transfer
    class="tree-transfer"
    :data-source="dataSource"
    checkStrictly
    :target-keys="targetKeys"
    :render="(item) => item.title"
    :show-select-all="false"
    @change="onChange"
    :titles="['源数据', '已添加数据']"
  >
    <template slot="children" slot-scope="{ props: { direction, selectedKeys }, on: { itemSelect } }">
      <template v-if="direction === 'left'">
        <a-input-search style="margin-bottom: 8px" placeholder="搜索" @change="onLeftSearch" />
        <a-tree
          v-if="leftTreeData.length"
          blockNode
          checkable
          :auto-expand-parent="leftAutoExpandParent"
          :expanded-keys="leftExpandedKeys"
          @expand="onLeftExpand"
          :tree-data="leftTreeData"
          :checked-keys="leftCheckedKey"
          @check="
            (_, props) => {
              handleLeftChecked(_, props, [...selectedKeys, ...targetKeys], itemSelect);
            }
          "
        >
          <template slot="title" slot-scope="{ title }">
            <span v-if="title.indexOf(leftSearchValue) > -1">
              {{ title.substr(0, title.indexOf(leftSearchValue)) }}
              <span style="color: #f50">{{ leftSearchValue }}</span>
              {{ title.substr(title.indexOf(leftSearchValue) + leftSearchValue.length) }}
            </span>
            <span v-else>{{ title }}</span>
          </template>
        </a-tree>

        <a-empty v-else>
          <template #description>暂无数据</template>
        </a-empty>
      </template>
      <template v-else-if="direction === 'right'">
        <a-input-search style="margin-bottom: 8px" placeholder="搜索" @change="onRightSearch" />
        <a-tree
          v-if="rightTreeData.length"
          blockNode
          checkable
          :auto-expand-parent="rightAutoExpandParent"
          :expanded-keys="rightExpandedKeys"
          @expand="onRightExpand"
          :tree-data="rightTreeData"
          :checked-keys="rightCheckedKey"
          @check="
            (_, props) => {
              handleRightChecked(_, props, [...selectedKeys, ...targetKeys], itemSelect);
            }
          "
        >
          <template slot="title" slot-scope="{ title }">
            <span v-if="title.indexOf(rightSearchValue) > -1">
              {{ title.substr(0, title.indexOf(rightSearchValue)) }}
              <span style="color: #f50">{{ rightSearchValue }}</span>
              {{ title.substr(title.indexOf(rightSearchValue) + rightSearchValue.length) }}
            </span>
            <span v-else>{{ title }}</span>
          </template>
        </a-tree>
        <a-empty v-else>
          <template #description>暂无数据</template>
        </a-empty>
      </template>
    </template>
  </a-transfer>
</template>

<script>
import { cloneDeep, getDeepList, getTreeKeys, treeToList, handleLeftTreeData, uniqueTree, isChecked } from './index';
import { findPathById, filterTreeArray } from '@/utils/treeUtils.js';

export default {
  name: 'OptionsTransfer',
  props: {
    /** 源树数据 */
    originTreeData: {
      type: Array,
      default: () => [],
    },
    /** 编辑 key */
    editKey: {
      type: Array,
      default: () => [],
    },
  },
  data() {
    return {
      targetKeys: [], // 显示在右侧框数据的 key 集合
      dataSource: [], // 数据源，其中的数据将会被渲染到左边一栏
      leftCheckedKey: [], // 左侧树选中 key 集合
      leftHalfCheckedKeys: [], // 左侧半选集合
      leftCheckedAllKey: [], // 左侧树选中的 key 集合，包括半选与全选
      leftTreeData: [], // 左侧树
      rightCheckedKey: [], // 右侧树选中 key 集合
      rightCheckedAllKey: [], // 右侧树选中集合，包括半选与全选
      rightExpandedKey: [], // 右侧展开数集合
      rightTreeData: [], // 右侧树
      emitKeys: [], // 往父级组件传递的数据
      deepList: [], // 深层列表
      leftSearchValue: '', // 左侧树搜索值
      leftAutoExpandParent: false, // 左侧树是否展开父节点
      leftExpandedKeys: [], // 左侧树扩展的key
      rightSearchValue: '', // 右侧树搜索值
      rightAutoExpandParent: false, // 右侧树是否展开父节点
      rightExpandedKeys: [], // 右侧树扩展的key
    };
  },
  watch: {
    originTreeData: {
      deep: true,
      handler(val) {
        this.processTreeData();
      },
    },
    editKey: {
      deep: true,
      handler(val) {
        this.processTreeData();
      },
    },
  },
  created() {
    this.processTreeData();
  },
  methods: {
    // 处理树数据
    processTreeData() {
      this.dataSource = treeToList(cloneDeep(this.originTreeData), true);
      if (this.editKey.length) {
        this.processEditData();
      } else {
        this.leftTreeData = handleLeftTreeData(cloneDeep(this.originTreeData), this.leftCheckedKey);
      }
    },
    // 处理编辑数据
    processEditData() {
      this.leftCheckedAllKey = this.editKey;
      this.rightExpandedKeys = this.editKey;
      this.targetKeys = this.editKey;
      this.rightTreeData = findPathById(cloneDeep(this.originTreeData), this.editKey);

      getDeepList(this.deepList, this.originTreeData);

      this.leftCheckedKey = uniqueTree(this.editKey, this.deepList);
      this.leftHalfCheckedKeys = this.leftCheckedAllKey.filter((item) => this.leftCheckedKey.indexOf(item) === -1);
      this.leftTreeData = handleLeftTreeData(cloneDeep(this.originTreeData), this.leftCheckedKey);

      this.emitKeys = this.rightExpandedKeys;
    },
    // 穿梭更改
    onChange(_, direction) {
      if (direction === 'right') {
        this.targetKeys = this.leftCheckedAllKey;
        this.rightCheckedKey = [];
        this.rightTreeData = findPathById(cloneDeep(this.originTreeData), this.leftCheckedAllKey);
        this.leftTreeData = handleLeftTreeData(cloneDeep(this.originTreeData), this.leftCheckedKey, 'right');
      } else if (direction === 'left') {
        this.rightTreeData = filterTreeArray(this.rightTreeData, this.rightCheckedKey, 'id');
        this.leftTreeData = handleLeftTreeData(this.leftTreeData, this.rightCheckedKey, 'left');
        this.leftCheckedKey = this.leftCheckedKey.filter((item) => this.rightCheckedKey.indexOf(item) === -1);
        this.targetKeys = this.targetKeys.filter((item) => this.rightCheckedKey.indexOf(item) === -1);
        this.leftHalfCheckedKeys = this.leftHalfCheckedKeys.filter((item) => this.rightCheckedKey.indexOf(item) === -1);
        this.rightCheckedKey = [];
      }
      this.rightExpandedKeys = getTreeKeys(this.rightTreeData);
      this.emitKeys = this.rightExpandedKeys;
    },
    // 左侧选择
    handleLeftChecked(_, { node, halfCheckedKeys }, checkedKeys, itemSelect) {
      this.leftCheckedKey = _;
      this.leftHalfCheckedKeys = [...new Set([...this.leftHalfCheckedKeys, ...halfCheckedKeys])];
      this.leftCheckedAllKey = [...new Set([...this.leftHalfCheckedKeys, ...halfCheckedKeys, ..._])];
      const { eventKey } = node;
      itemSelect(eventKey, !isChecked(checkedKeys, eventKey));
    },
    // 右侧选择
    handleRightChecked(_, { node, halfCheckedKeys }, checkedKeys, itemSelect) {
      this.rightCheckedKey = _;
      this.rightCheckedAllKey = [...halfCheckedKeys, ..._];
      const { eventKey } = node;
      itemSelect(eventKey, isChecked(_, eventKey));
    },
    getParentKey(key, tree) {
      let parentKey;
      for (let i = 0; i < tree.length; i++) {
        const node = tree[i];
        if (node.children) {
          if (node.children.some((item) => item.key === key)) {
            parentKey = node.key;
          } else if (this.getParentKey(key, node.children)) {
            parentKey = this.getParentKey(key, node.children);
          }
        }
      }
      return parentKey;
    },
    onLeftExpand(leftExpandedKeys) {
      this.leftExpandedKeys = leftExpandedKeys;
      this.leftAutoExpandParent = false;
    },
    onLeftSearch(e) {
      const value = e.target.value;
      const leftExpandedKeys = treeToList(this.originTreeData, true)
        .map((item) => {
          if (item.title.indexOf(value) > -1) {
            return this.getParentKey(item.key, this.originTreeData);
          }
          return null;
        })
        .filter((item, i, self) => item && self.indexOf(item) === i);
      Object.assign(this, {
        leftExpandedKeys,
        leftSearchValue: value,
        leftAutoExpandParent: true,
      });
    },
    onRightExpand(rightExpandedKeys) {
      this.rightExpandedKeys = rightExpandedKeys;
      this.rightAutoExpandParent = false;
    },
    onRightSearch(e) {
      const value = e.target.value;
      const rightExpandedKeys = treeToList(this.rightTreeData, true)
        .map((item) => {
          if (item.title.indexOf(value) > -1) {
            return this.getParentKey(item.key, this.rightTreeData);
          }
          return null;
        })
        .filter((item, i, self) => item && self.indexOf(item) === i);
      Object.assign(this, {
        rightExpandedKeys,
        rightSearchValue: value,
        rightAutoExpandParent: true,
      });
    },
  },
};
</script>

<style lang="less">
.ant-transfer-list {
  width: 800px;
}
.ant-transfer-list-header-selected {
  span {
    display: none;
  }
}
.ant-transfer-list-header-title {
  display: contents !important;
}
</style>
