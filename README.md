# TransferTree
基于vue的树形穿梭选择组件

背景：Ant Design Vue 官网中有穿梭框 Transfer 组件，但提供的示例是左侧是树，右侧则是平铺列表

需求左边是树，右边也是树结构数据，且要支持搜索

自定义封装组件

TransferTree/index.vue:

```
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
import { findPathById, filterTreeArray, cloneDeep, getDeepList, getTreeKeys, treeToList, handleLeftTreeData, uniqueTree, isChecked } from './index';

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

```

组件涉及到的树形数据操作方法：TransferTree/index.js

```
/**
 * 深拷贝
 * @param data
 */
export function cloneDeep(data) {
  return JSON.parse(JSON.stringify(data));
}

/**
 * 树转数组
 * @param tree
 * @param hasChildren
 */
export function treeToList(tree = [], hasChildren = false) {
  let queen = [];
  const out = [];
  queen = queen.concat(JSON.parse(JSON.stringify(tree)));
  while (queen.length) {
    const first = queen.shift();
    if (first?.children) {
      queen = queen.concat(first.children);
      if (!hasChildren) delete first.children;
    }
    out.push(first);
  }
  return out;
}

/**
 * 数组转树
 * @param list
 * @param tree
 * @param parentId
 * @param key
 */
export function listToTree(list = [], tree = [], parentId = 0, key = 'parentId') {
  list.forEach((item) => {
    if (item[key] === parentId) {
      const child = {
        ...item,
        children: [],
      };
      listToTree(list, child.children, item.id, key);
      if (!child.children?.length) delete child.children;
      tree.push(child);
    }
  });
  return tree;
}

/**
 * 获取树节点 key 列表
 * @param treeData
 * @param key
 */
export function getTreeKeys(treeData, key = 'key') {
  const list = treeToList(treeData);
  return list.map((item) => item[key]);
}

/**
 * 循环遍历出最深层子节点，存放在一个数组中
 * @param deepList
 * @param treeData
 */
export function getDeepList(deepList, treeData) {
  treeData?.forEach((item) => {
    if (item?.children?.length) {
      getDeepList(deepList, item.children);
    } else {
      deepList.push(item.key);
    }
  });
  return deepList;
}

/**
 * 将后台返回的含有父节点的数组和第一步骤遍历的数组做比较,如果有相同值，将相同值取出来，push到一个新数组中
 * @param uniqueArr
 * @param arr
 */
export function uniqueTree(uniqueArr, arr) {
  const uniqueChild = [];
  for (const i in arr) {
    for (const k in uniqueArr) {
      if (uniqueArr[k] === arr[i]) {
        uniqueChild.push(uniqueArr[k]);
      }
    }
  }
  return uniqueChild;
}

/**
 * 是否选中
 * @param selectedKeys
 * @param eventKey
 */
export function isChecked(selectedKeys, eventKey = '') {
  return selectedKeys.indexOf(eventKey) !== -1;
}

/**
 * 处理左侧树数据
 * @param data
 * @param targetKeys
 * @param direction
 */
export function handleLeftTreeData(data, targetKeys, direction = 'right') {
  data.forEach((item) => {
    if (direction === 'right') {
      item.disabled = targetKeys.includes(item.key);
    } else if (direction === 'left') {
      if (item.disabled && targetKeys.includes(item.key)) item.disabled = false;
    }
    if (item.children) handleLeftTreeData(item.children, targetKeys, direction);
  });
  return data;
}

/**
 * 处理右侧树数据
 * @param data
 * @param targetKeys
 * @param direction
 */
export function handleRightTreeData(data, targetKeys, direction = 'right') {
  const list = treeToList(data);
  const arr = [];
  const tree = [];
  list.forEach((item) => {
    if (direction === 'right') {
      if (targetKeys.includes(item.key)) {
        const content = { ...item };
        if (content.children) delete content.children;
        arr.push({ ...content });
      }
    } else if (direction === 'left') {
      if (!targetKeys.includes(item.key)) {
        const content = { ...item };
        if (content.children) delete content.children;
        arr.push({ ...content });
      }
    }
  });
  listToTree(arr, tree, 0);
  return tree;
}

/**
 * 通过id获取树的完整路径
 * @param tree 树型数据
 * @param ids  id集合
 * @param keyName 关键字
 */
export function findPathById(tree, ids, keyName = 'id') {
  const result = [];
  for (const node of tree) {
    if (ids.includes(node[keyName])) {
      result.push(node);
    }
    if (node.children) {
      const childResults = findPathById(node.children, ids);
      if (childResults.length > 0) {
        result.push({ ...node, children: childResults });
      }
    }
  }
  return result;
}

/**
 * 通过id过滤树的id所在子节点
 * @param tree 树型数据
 * @param ids  id集合
 * @param keyName 关键字
 */
export function filterTreeArray(tree, ids, keyName = 'id') {
  return tree
    .filter((item) => {
      return ids.indexOf(item[keyName]) == -1;
    })
    .map((item) => {
      item = Object.assign({}, item);
      if (item.children) {
        item.children = filterTreeArray(item.children, ids);
      }
      return item;
    });
}
```

引入组件：

```
<TransferTree ref="transferRef" :originTreeData="originTreeData" :editKey="editKey" />
<button @click="save">保存</button>

方法：
onSave() {
  alert(this.$refs.transferRef.emitKeys);
}

数据：
 originTreeData: [
        {
          id: '1',
          parentId: '0',
          checkable: false,
          key: '0',
          title: '一级',
        },
        {
          id: '2',
          parentId: '0',
          checkable: false,
          key: '2',
          title: '一级1',
          children: [
            {
              id: '3',
              parentId: '2',
              key: '3',
              checkable: false,
              title: '二级1',
              children: [
                {
                  id: '99',
                  parentId: '3',
                  key: '99',
                  title: '三级1',
                },
                {
                  id: '98',
                  parentId: '3',
                  key: '98',
                  title: '三级11',
                },
              ],
            },
            {
              id: '4',
              parentId: '2',
              key: '4',
              checkable: false,
              title: '二级2',
              children: [
                {
                  id: '999',
                  parentId: '4',
                  key: '999',
                  title: '三级2',
                },
                {
                  id: '998',
                  parentId: '4',
                  key: '998',
                  title: '三级22',
                },
              ],
            },
          ],
        },
      ],
      editKey: ['99'],
```

效果：

![](https://pic.imgdb.cn/item/65dda20d9f345e8d0303f630.png)

![](https://pic.imgdb.cn/item/65dda3349f345e8d0306c75d.png)

