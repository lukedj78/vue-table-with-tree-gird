<template lang="html">
  <div
    v-if="columns.length > 0"
    ref="table"
    :class="[prefixCls, tableClass]">
    <div
      v-show="showHeader"
      ref="header-wrapper"
      :class="`${prefixCls}__header-wrapper`"
      @mousewheel="handleEvent('header', $event)">
      <table-header
        ref="table-header">
      </table-header>
    </div>
    <div
      ref="body-wrapper"
      :style="bodyWrapperStyle"
      :class="`${prefixCls}__body-wrapper`"
      @scroll="handleEvent('body', $event)">
      <table-body
        ref="table-body"
        :class="bodyClass">
      </table-body>
    </div>
    <div
      v-show="showSummary && data.length > 0"
      ref="footer-wrapper"
      :class="`${prefixCls}__footer-wrapper`"
      @mousewheel="handleEvent('footer', $event)">
      <table-footer
        ref="table-footer">
      </table-footer>
    </div>
  </div>
</template>

<script>
  import TableHeader from './TableHeader';
  import TableBody from './TableBody';
  import TableFooter from './TableFooter';
  import { mixins, scrollBarWidth as getSbw } from './utils';

  /* eslint-disable no-underscore-dangle */
  /* eslint-disable no-param-reassign */

  function getBodyData(data, isTreeType, childrenProp, isFold, level = 1) {
    let bodyData = [];
    data.forEach((row, index) => {
      const children = row[childrenProp];
      const childrenLen = Object.prototype.toString.call(children).slice(8, -1) === 'Array' ? children.length : 0;
      bodyData.push({
        _isHover: false,
        _isExpanded: false,
        _isChecked: false,
        _level: level,
        _isHide: isFold ? level !== 1 : false,
        _isFold: isFold,
        _childrenLen: childrenLen,
        _normalIndex: index + 1,
        ...row,
      });
      if (isTreeType) {
        if (childrenLen > 0) {
          bodyData = bodyData.concat(getBodyData(children, true, childrenProp, isFold, level + 1));
        }
      }
    });
    return bodyData;
  }

  function initialState(table) {
    return {
      bodyHeight: 'auto',
      firstProp: table.columns[0].prop,
      bodyData: getBodyData(table.data, table.treeType, table.childrenProp, table.isFold),
    };
  }

  function initialColumns(table, clientWidth) {
    let columnsWidth = 0;
    const minWidthColumns = [];
    const otherColumns = [];
    const columns = table.columns.concat();
    if (table.expandType) {
      columns.unshift({
        width: '50',
      });
    }
    if (table.selectionType) {
      columns.unshift({
        width: '50',
      });
    }
    if (table.showIndex) {
      columns.unshift({
        width: '50px',
        prop: '_normalIndex',
        label: table.indexText,
      });
    }
    columns.forEach((column, index) => {
      let width = '';
      let minWidth = '';
      if (!column.width) {
        if (column.minWidth) {
          minWidth = typeof column.minWidth === 'number' ? column.minWidth : parseInt(column.minWidth, 10);
        } else {
          minWidth = 80;
        }
        minWidthColumns.push({
          ...column,
          minWidth,
          _index: index,
        });
      } else {
        width = typeof column.width === 'number' ? column.width : parseInt(column.width, 10);
        otherColumns.push({
          ...column,
          width,
          _index: index,
        });
      }
      columnsWidth += minWidth || width;
    });
    const scrollBarWidth = getSbw();
    const totalWidth = columnsWidth + scrollBarWidth;
    const isScrollX = totalWidth > clientWidth;
    if (!isScrollX) {
      const extraWidth = clientWidth - totalWidth;
      const averageExtraWidth = Math.floor(extraWidth / minWidthColumns.length);
      minWidthColumns.forEach((column) => {
        column.computedWidth = column.minWidth + averageExtraWidth;
      });
    }
    const tableColumns = otherColumns.concat(minWidthColumns);
    tableColumns.sort((a, b) => a._index - b._index);
    return tableColumns;
  }

  export default {
    name: 'zk-table',
    mixins: [mixins],
    components: {
      TableHeader,
      TableBody,
      TableFooter,
    },
    props: {
      data: {
        type: Array,
        default: () => [],
      },
      columns: {
        type: Array,
        default: () => [],
      },
      maxHeight: {
        type: [String, Number],
        default: 'auto',
      },
      stripe: {
        type: Boolean,
        default: false,
      },
      border: {
        type: Boolean,
        default: false,
      },
      treeType: {
        type: Boolean,
        default: true,
      },
      childrenProp: {
        type: String,
        default: 'children',
      },
      isFold: {
        type: Boolean,
        default: true,
      },
      expandType: {
        type: Boolean,
        default: true,
      },
      selectionType: {
        type: Boolean,
        default: true,
      },
      emptyText: {
        type: String,
        default: '暂无数据',
      },
      showHeader: {
        type: Boolean,
        default: true,
      },
      showIndex: {
        type: Boolean,
        default: false,
      },
      indexText: {
        type: String,
        default: '序号',
      },
      showSummary: {
        type: Boolean,
        default: false,
      },
      sumText: {
        type: String,
        default: '合计',
      },
      summaryMethod: Function,
      showRowHover: {
        type: Boolean,
        default: true,
      },
      rowKey: Function,
      rowClassName: [String, Function],
      cellClassName: [String, Function],
      rowStyle: [Object, Function],
      cellStyle: [Object, Function],
    },
    data() {
      return {
        computedWidth: '',
        computedHeight: '',
        tableColumns: [],
        ...initialState(this),
      };
    },
    computed: {
      bodyWrapperStyle() {
        return {
          height: this.bodyHeight,
        };
      },
      tableClass() {
        return {
          [`${this.prefixCls}--border`]: this.border,
        };
      },
      bodyClass() {
        return {
          [`${this.prefixCls}--stripe`]: this.stripe,
        };
      },
    },
    methods: {
      handleEvent(type, $event) {
        this.validateType(type, ['header', 'body', 'footer'], 'handleEvent');
        const eventType = $event.type;
        if (eventType === 'scroll') {
          this.$refs['header-wrapper'].scrollLeft = $event.target.scrollLeft;
          this.$refs['footer-wrapper'].scrollLeft = $event.target.scrollLeft;
        }
        if (eventType === 'mousewheel') {
          const deltaX = $event.deltaX;
          const $body = this.$refs['body-wrapper'];
          if (deltaX > 0) {
            $body.scrollLeft += 10;
          } else {
            $body.scrollLeft -= 10;
          }
        }
        return this.$emit(`${type}-${eventType}`, $event);
      },
      // computedWidth, computedHeight, tableColumns
      measure() {
        this.$nextTick(() => {
          const { clientWidth, clientHeight } = this.$el;
          this.computedWidth = clientWidth + 2;
          this.computedHeight = clientHeight + 2;

          const maxHeight = parseInt(this.maxHeight, 10);
          if (this.maxHeight !== 'auto' && this.computedHeight > maxHeight) {
            this.bodyHeight = `${maxHeight - 83}px`;
          }
          this.tableColumns = initialColumns(this, clientWidth);
        });
      },
      getCheckedProp(prop = 'index') {
        if (!this.selectionType) {
          return [];
        }
        const checkedIndexs = [];
        this.bodyData.forEach((item, index) => {
          if (item._isChecked) {
            if (prop === 'index') {
              checkedIndexs.push(index);
            } else {
              checkedIndexs.push(item[prop]);
            }
          }
        });
        return checkedIndexs;
      },
    },
    watch: {
      $props: {
        deep: true,
        handler() {
          Object.assign(this.$data, initialState(this));
        },
      },
    },
    updated() {
      this.measure();
    },
    mounted() {
      this.measure();
      window.addEventListener('resize', this.measure);
    },
    beforeDestroy() {
      window.removeEventListener('resize', this.measure);
    },
  };
</script>

<style>

@font-face {
  font-family: "iconfont";
  src: url('font/iconfont.eot?t=1505310522875');
  /* IE9*/
  src: url('font/iconfont.eot?t=1505310522875#iefix') format('embedded-opentype'), /* IE6-IE8 */ url('data:application/x-font-woff;charset=utf-8;base64,d09GRgABAAAAAAW0AAsAAAAACOQAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAABHU1VCAAABCAAAADMAAABCsP6z7U9TLzIAAAE8AAAARAAAAFZW7kggY21hcAAAAYAAAABuAAABojLtBtFnbHlmAAAB8AAAAa8AAAKA73SzT2hlYWQAAAOgAAAALwAAADYO3fRqaGhlYQAAA9AAAAAcAAAAJAfeA4ZobXR4AAAD7AAAABMAAAAUE+kAAGxvY2EAAAQAAAAADAAAAAwBbAHYbWF4cAAABAwAAAAeAAAAIAEUAF1uYW1lAAAELAAAAUUAAAJtPlT+fXBvc3QAAAV0AAAAQAAAAFryy5h0eJxjYGRgYOBikGPQYWB0cfMJYeBgYGGAAJAMY05meiJQDMoDyrGAaQ4gZoOIAgCKIwNPAHicY2Bk/s04gYGVgYOpk+kMAwNDP4RmfM1gxMjBwMDEwMrMgBUEpLmmMDgwVDwzZm7438AQw9zA0AAUZgTJAQAoHgyieJzFkcENwyAUQ98HyqHKKDmEZoEOklOnYOK/RmI+uXSCWDLG/pZAALyALK5iAfthDBxKLfLMO/LCJl+lRqL7fp7y3VuoKprV0KxO0qbyGOy5o/+xxPq9nV6YflNX9DY5fsA/k6Pj+yTpAn3jEO8AAHiclVDNitNQFD7n3slNE9vE5N7kpOn0J0mbKB3DGDMZRGw3bhQXA2LB5TyAbmfjohvBhQvfYEAEoc8wr+EDiK4KPkITU0EcXDmHw3fOgfN9fHygATTf+BUPQMIduA9P4AwAxRxjiw0xysqczdGLNI+UxbMki/QkzvljpFgov6jKlIQubLRwhA+iospyluFJuWCPsPCHiP1B+MKdHbr8I5pBNnpXP2Of0Bsnh/biXv30aKmKiexcdF2377ofOkLTOowd2Ba+Jt/QDFPUnzU79K7Gd9kYu/0sfP6qNxm45+/LN8MZGYjrNcrBxPqydEKn7behL92+frvXCcJeMlV48eNWILvD9Du0hXtgl+wSHIBZnJZLzNKyKgh9paPAdVca260hAxPBMBowz9t1uzUDuT8CswEDDtq8Gr7mADaM4QgetrKRhbozQooWeOrkKJVIojg9ccqqjcT3uBJ6lGNZnUYjnBU+ORbGaeYskM93m+kx4vGUrX5PQXK3kUSSrSS9JFWvFJHCjaIGJCFSugcOLeM6c7f6wyFZf/37+PO6AgD/x/vNnN/A7H8bhF86tmcbAHicY2BkYGAAYl/p/jnx/DZfGbhZGEDg6v0IKwT9/yELA7MEkMvBwAQSBQAZYgnZAHicY2BkYGBu+N/AEMPCAAJAkpEBFbACAEcLAm54nGNhYGBgfsnAwMKAwAAOmwD9AAAAAAAAdgCYAPYBQHicY2BkYGBgZQgEYhBgAmIuIGRg+A/mMwAAES0BcgAAeJxlj01OwzAQhV/6B6QSqqhgh+QFYgEo/RGrblhUavdddN+mTpsqiSPHrdQDcB6OwAk4AtyAO/BIJ5s2lsffvHljTwDc4Acejt8t95E9XDI7cg0XuBeuU38QbpBfhJto41W4Rf1N2MczpsJtdGF5g9e4YvaEd2EPHXwI13CNT+E69S/hBvlbuIk7/Aq30PHqwj7mXle4jUcv9sdWL5xeqeVBxaHJIpM5v4KZXu+Sha3S6pxrW8QmU4OgX0lTnWlb3VPs10PnIhVZk6oJqzpJjMqt2erQBRvn8lGvF4kehCblWGP+tsYCjnEFhSUOjDFCGGSIyujoO1Vm9K+xQ8Jee1Y9zed0WxTU/3OFAQL0z1xTurLSeTpPgT1fG1J1dCtuy56UNJFezUkSskJe1rZUQuoBNmVXjhF6XNGJPyhnSP8ACVpuyAAAAHicY2BigAAuBuyAlZGJkZmRhZGVkY2BsYI7MS89J1W3KDM9o4S3IKe0WLe4sDSxKFU3ny83Mw+Jy8AAAHSWD8A=') format('woff'), url('font/iconfont.ttf?t=1505310522875') format('truetype'), /* chrome, firefox, opera, Safari, Android, iOS 4.2+*/ url('font/iconfont.svg?t=1505310522875#iconfont') format('svg');
  /* iOS 4.1- */
}
.zk-icon {
  font-family: "iconfont" !important;
  font-size: 14px;
  font-style: normal;
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
.zk-icon-plus-square-o:before {
  content: "\e631";
}
.zk-icon-minus-square-o:before {
  content: "\e632";
}
.zk-icon-angle-right:before {
  content: "\e633";
}
.zk-table {
  position: relative;
  width: 100%;
  -webkit-box-sizing: border-box;
          box-sizing: border-box;
  background-color: #ffffff;
  border: 1px solid #e9eaec;
  font-size: 12px;
  line-height: 26px;
  color: #1F2D3D;
  overflow: hidden;
}
.zk-table__header-wrapper,
.zk-table__footer-wrapper {
  overflow: hidden;
}
.zk-table__body-wrapper {
  overflow: auto;
}
.zk-table__header,
.zk-table__body,
.zk-table__footer {
  width: 100%;
  table-layout: fixed;
  border-collapse: collapse;
  border-spacing: 0;
}
.zk-table__header-row {
  height: 40px;
  -webkit-box-sizing: border-box;
          box-sizing: border-box;
  background-color: #f8f8f9;
  border-bottom: 1px solid #e9eaec;
}
.zk-table__footer-row {
  height: 40px;
  -webkit-box-sizing: border-box;
          box-sizing: border-box;
  background-color: #ffffff;
  border-top: 1px solid #e9eaec;
}
.zk-table__body-row {
  height: 48px;
  -webkit-box-sizing: border-box;
          box-sizing: border-box;
}
.zk-table__body-row:not(:first-of-type) {
  border-top: 1px solid #e9eaec;
}
.zk-table__header-cell,
.zk-table__body-cell,
.zk-table__footer-cell {
  -webkit-box-sizing: border-box;
          box-sizing: border-box;
  text-align: left;
  vertical-align: middle;
  word-break: break-all;
  overflow: hidden;
}
.zk-table__header-cell {
  font-weight: bold;
}
.zk-table__cell-inner {
  padding: 6px 12px;
}
.zk-table--firstProp-header-inner {
  padding-left: 32px;
}
.zk-table--empty-row {
  height: 80px;
}
.zk-table--empty-content {
  text-align: center;
}
.zk-table--center-cell {
  text-align: center;
}
.zk-table--right-cell {
  text-align: right;
}
.zk-table--stripe-row {
  background-color: #f8f8f9;
}
.zk-table--row-hover {
  background-color: #ebf7ff;
}
.zk-table--border-cell:not(:last-of-type) {
  border-right: 1px solid #e9eaec;
}
.zk-table--tree-icon {
  margin-right: 6px;
  cursor: pointer;
}
.zk-table--expand-inner {
  text-align: center;
  cursor: pointer;
  -webkit-transition: -webkit-transform 0.2s ease-in-out;
  transition: -webkit-transform 0.2s ease-in-out;
  transition: transform 0.2s ease-in-out;
  transition: transform 0.2s ease-in-out, -webkit-transform 0.2s ease-in-out;
}
.zk-table--expanded-inner {
  -webkit-transform: rotate(90deg);
          transform: rotate(90deg);
}
.zk-table--expand-content {
  padding: 20px;
}

</style>
