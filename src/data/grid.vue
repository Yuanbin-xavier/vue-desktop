<style>
  .d-grid {
    position: relative;
    overflow: hidden;
    box-sizing: border-box;
    width: 100%;
    max-width: 100%;
    background-color: #fff;
    border-collapse: collapse;
    border: 1px solid #e7eaec;
    font-size: 14px;
    border-radius: 3px;
  }

  .hidden-columns {
    visibility: hidden;
    position: absolute;
    z-index: -1;
  }

  .d-grid-fit {
    border-right: 0;
  }

  .d-grid th,
  .d-grid td {
    height: 20px;
    max-width: 250px;
    box-sizing: border-box;
    overflow: hidden;
    line-height: 28px;
    text-align: left;
    text-overflow: ellipsis;
    white-space: nowrap;
    vertical-align: middle;
  }

  .d-grid td {
    border-bottom: 1px solid #e7eaec;
  }

  .d-grid th {
    border-right: 1px solid #ddd;
    border-bottom: 1px solid #ddd;
  }

  .d-grid th > div {
    display: inline-block;
    padding: 2px;
  }

  .d-grid td > div {
    padding: 2px;
  }

  .d-grid .grid-fixed-header-wrapper {
    position: absolute;
    left: 0;
    top: 0;
    z-index: 3;
  }

  .d-grid .grid-fixed-body-wrapper {
    position: absolute;
    left: 0;
    top: 37px;
    overflow: hidden;
    z-index: 3;
  }

  .d-grid .grid-fixed-body-wrapper tr {
    /*background: #fff;*/
  }

  .d-grid .grid-header-wrapper,
  .d-grid .grid-body-wrapper {
    width: 100%;
  }

  .d-grid .grid-header,
  .d-grid .grid-body {
    table-layout: fixed;
  }

  .d-grid .grid-header-wrapper {
    overflow: hidden;
  }

  .d-grid .grid-body-wrapper {
    overflow: auto;
    position: relative;
  }

  .d-grid td,
  .d-grid th {
    position: relative;
    border-right: 1px solid #e7eaec;
  }

  /** TODO */
  .d-grid th.required > div:before {
    display: inline-block;
    content: "";
    width: 8px;
    height: 8px;
    border-radius: 50%;
    background: #ff4d51;
    margin-right: 5px;
    vertical-align: middle;
  }

  .d-grid th > .cell {
    position: relative;
    word-wrap: normal;
    overflow: hidden;
    text-overflow: ellipsis;
    display: inline-block;
    line-height: 20px;
    vertical-align: middle;
    width: 100%;
  }

  /** TODO */
  .d-grid .sort-caret {
    display: inline-block;
    width: 0;
    height: 0;
    margin-left: 0.3em;
    border: 0;
    content: "";
    position: absolute;
    z-index: 2;
    top: 14px;
    right: 4px;
  }

  .d-grid .ascending .sort-caret {
    vertical-align: baseline;
    border-top: none;
    border-right: 4px solid transparent;
    border-bottom: 4px solid #333;
    border-left: 4px solid transparent;
  }

  .d-grid .descending .sort-caret {
    vertical-align: super;
    border-top: 4px solid #333;
    border-right: 4px solid transparent;
    border-bottom: none;
    border-left: 4px solid transparent;
  }

  .d-grid th.gutter,
  .d-grid td.gutter {
    width: 15px;
    border-right-width: 0;
    border-bottom-width: 0;
    padding: 0;
  }

  .d-grid td.gutter {
    width: 0;
  }

  .d-grid-fit th.gutter,
  .d-grid-fit td.gutter {
    border-right-width: 1px;
  }

  .d-grid td .cell {
    box-sizing: border-box;
    overflow: hidden;
    text-overflow: ellipsis;
    min-height: 28px;
  }

  .d-grid tr {
    background-color: #fff;
  }

  .d-grid tr:nth-child(2n) {
    background: #f9f9f9;
  }

  .d-grid tr.current-row {
    background: #007fc1;
    color: #fff;
  }

  .d-grid tr.hover,
  .d-grid tr:hover {

  }

  .d-grid-column-resize-proxy {
    position: absolute;
    left: 200px;
    top: 0;
    bottom: 0;
    width: 0;
    border-left: 1px solid #999;
  }
</style>

<template>
  <div class="d-grid" :class="{ 'd-grid-fit': fit }" @mouseleave="hoverRowIndex = null">
    <div class="hidden-columns" v-el:hidden-columns></table><slot></slot></div>
    <div class="grid-header-wrapper">
      <table class="grid-header" cellspacing="0" cellpadding="0" border="0" :style="{ width: bodyWidth ? bodyWidth + 'px' : '' }">
        <thead></thead>
      </table>
    </div>
    <div class="grid-body-wrapper">
      <table class="grid-body" cellspacing="0" cellpadding="0" border="0" :style="{ width: bodyWidth ? bodyWidth - gutterWidth + 'px' : '' }">
        <tbody></tbody>
      </table>
    </div>
    <div class="grid-fixed-header-wrapper" v-if="fixedColumnCount > 0">
      <table class="grid-header" cellspacing="0" cellpadding="0" border="0" :style="{ width: fixedBodyWidth ? fixedBodyWidth + 'px' : '' }">
        <thead></thead>
      </table>
    </div>
    <div class="grid-fixed-body-wrapper" v-if="fixedColumnCount > 0" :style="{ top: headerHeight + 'px' }">
      <table class="grid-body" cellspacing="0" cellpadding="0" border="0" :style="{ width: fixedBodyWidth ? fixedBodyWidth + 'px' : '' }">
        <tbody></tbody>
      </table>
    </div>
    <div class="d-grid-column-resize-proxy" v-el:resize-proxy v-show="resizeProxyVisible"></div>
    <slot name="bottom"></slot>
  </div>
</template>

<script type="text/ecmascript-6">
  var Vue = require('vue');

  import { throttle, debounce, getScrollbarWidth } from '../util';
  import { default as SchemaStore } from '../schema/store';
  import { hasClass, addClass, removeClass } from 'wind-dom';

  var gridIdSeed = 1;
  var GUTTER_WIDTH;

  var getPath = function(object, sortKey) {
    if (!object || !sortKey) return null;
    var path = sortKey.split('.');
    if (path.length === 1) return object[sortKey];

    var target = object;
    for (var i = 0, j = path.length; i < j; i++) {
      var key = path[i];
      var value = target[key];
      if (i === j - 1) {
        return value;
      }
      if (value === null || value === undefined) return null;
      target = value;
    }
  };

  var isObject = function(obj) {
    return obj !== null && typeof obj === 'object'
  };

  export default {
    props: {
      data: {
        type: Array,
        default: function() {
          return [];
        }
      },

      schema: {},

      width: {
        type: String
      },

      height: {
        type: String
      },

      fit: {
        type: Boolean,
        default: true
      },

      fixedColumnCount: {
        type: Number,
        default: 0
      },

      selectionMode: {
        type: String,
        default: 'single'
      },

      selection: {},

      gutterWidth: {
        default: 0
      }
    },

    events: {
      onresize() {
        var grid = this;

        Vue.nextTick(function() {
          if (grid.$el.getAttribute('flex') !== null) {
            grid.height = grid.$el.offsetHeight;
          }
          grid.$calcColumns();
        });
      }
    },

    methods: {
      handleDataChange(data) {
        data = data || [];

        if (this.selectionMode === 'single') {
          var selected = this.selected;
          if (selected === null) {
            this.selected = data[0];
            if (this.selected) {
              this.$emit('selection-change', this.selected);
            }
          } else if (data.indexOf(selected) === -1) {
            this.selected = data[0];
            if (this.selected) {
              this.$emit('selection-change', this.selected);
            }
          }
        }
      },

      toggleSelection(event, row) {
        var target = event.target;
        Vue.set(row, '$selected', target.checked);
        if (this.selectionMode === 'multiple') {
          this.$emit('selection-change', this.selection);
        }
      },

      toggleAllSelection(event) {
        var target = event.target;
        var checked = target.checked;
        this.data.forEach(function(item) {
          Vue.set(item, '$selected', checked);
        });
        if (this.selectionMode === 'multiple') {
          this.$emit('selection-change', this.selection);
        }
      },

      $calcColumns() {
        var fit = this.fit;
        var columns = this.columns;

        var bodyWidth = this.$el.clientWidth;
        var bodyMinWidth = 0;

        if (fit) {
          var flexColumns = [];
          var definedWidthColumnsWidth = 0;
          var definedMinWidthSum = 0;

          columns.forEach((column) => {
            definedMinWidthSum += column.minWidth || 80;
            bodyMinWidth += column.width || column.minWidth || 80;

            if (typeof column.width === 'number') {
              definedWidthColumnsWidth += column.width;
            } else {
              flexColumns.push(column);
            }
          });

          if (bodyMinWidth < bodyWidth - GUTTER_WIDTH) { // do not have scroll bar.
            var flexWidthTotal = bodyWidth - GUTTER_WIDTH - columns.length - bodyMinWidth;
            var flexWidthPerColumn = Math.floor(flexWidthTotal / flexColumns.length);
            var flexWidthFirstColumn = flexWidthTotal - flexWidthPerColumn * flexColumns.length + flexWidthPerColumn;

            flexColumns.forEach(function(column, index) {
              if (index === 0) {
                column.realWidth = (column.minWidth || 80) + flexWidthFirstColumn;
              } else {
                column.realWidth = (column.minWidth || 80) + flexWidthPerColumn;
              }
            });
          } else { // need horizontal scroll bar.
            this.showHoriScrollbar = true;
            flexColumns.forEach(function(column) {
              column.realWidth = column.minWidth;
            });
          }
        } else {
          columns.forEach((column) => {
            if (!column.width && !column.minWidth) {
              column.realWidth = 80;
            }

            bodyMinWidth += column.realWidth;
          });
          this.showHoriScrollbar = bodyMinWidth > bodyWidth;
        }

        if (this.styleNode) {
          var styleSheet = this.styleNode.sheet;

          for (var i = 0, j = styleSheet.cssRules.length; i < j; i++) {
            styleSheet.deleteRule(0);
          }

          columns.forEach(function(column) {
            styleSheet.insertRule(`.${column.id}, .${column.id} > div { width: ${column.realWidth}px; }`, styleSheet.cssRules.length);
          });
        }

        if (this.fixedColumnCount > 0) {
          var fixedBodyWidth = 0;
          var fixedColumnCount = this.fixedColumnCount;

          columns.forEach(function(column, index) {
            if (index < fixedColumnCount) {
              fixedBodyWidth += column.realWidth;
            }
          });

          this.fixedBodyWidth = fixedBodyWidth;
        }

        this.bodyWidth = bodyWidth;
        this.headerHeight = this.$el.querySelector('.grid-header-wrapper').offsetHeight;
      },

      $calcHeight(height) {
        if (typeof height === 'string') {
          if (/^\d+$/.test(height)) {
            height = Number(height);
          }
        }

        if (!isNaN(height) && this.$el) {
          var headerHeight = this.headerHeight = this.$el.querySelector('.grid-header-wrapper').offsetHeight;
          var bodyHeight = (height - headerHeight);
          var gridWrapper = this.$el.querySelector('.grid-body-wrapper');
          gridWrapper.style.height = bodyHeight + 'px';

          this.$el.style.height = height + 'px';

          var fixedBodyWrapper = this.$el.querySelector('.grid-fixed-body-wrapper');
          if (fixedBodyWrapper) {
            fixedBodyWrapper.style.height = (this.showHoriScrollbar ? gridWrapper.offsetHeight - this.gutterWidth : gridWrapper.offsetHeight) + 'px';
          }
        }
      },

      handleHeaderClick(column, event) {
        var target = event.target;
        while (target && target.tagName !== 'TH') {
          target = target.parentNode;
        }

        if (target && target.tagName === 'TH') {
          if (hasClass(target, 'noclick')) {
            removeClass(target, 'noclick');
            return;
          }
        }

        if (!column.sortable) return;

        if (this.sortingColumn !== column) {
          if (this.sortingColumn) {
            this.sortingColumn.direction = '';
          }
          this.sortingColumn = column;
          this.sortingProperty = column.property;
        }

        if (!column.direction) {
          column.direction = 'ascending';
        } else if (column.direction === 'ascending') {
          column.direction = 'descending';
        } else {
          column.direction = '';
          this.sortingColumn = null;
          this.sortingProperty = null;
        }
        this.sortingDirection = column.direction === 'descending' ? -1 : 1;
      },

      reRender() {
        if (this.$body) {
          this.$body.$destroy();
        }

        if (this.$header) {
          this.$header.$destroy();
        }

        this.doRender();
      },

      doRender() {
        var bodyWrapper = this.$el.querySelector('.grid-body-wrapper');
        var headerWrapper = this.$el.querySelector('.grid-header-wrapper');
        var fixedBodyWrapper = this.$el.querySelector('.grid-fixed-body-wrapper');

        if (!this.$ready) {
          bodyWrapper.addEventListener('scroll', function () {
            headerWrapper.scrollLeft = this.scrollLeft;
            if (fixedBodyWrapper) {
              fixedBodyWrapper.scrollTop = this.scrollTop;
            }
          });
        }

        this.$calcColumns();

        if (!this.$ready && this.fit) {
          this.windowResizeListener = throttle(() => {
            this.$calcColumns();
          }, 200);
          window.addEventListener('resize', this.windowResizeListener);
        }

        this.$renderHeader();
        this.$renderBody();
        this.$renderHeader(true);
        this.$renderBody(true);

        Vue.nextTick(() => {
          if (this.height) {
            this.$calcHeight(this.height);
          }
        });
      },

      $renderHeader(fixed) {
        var columns = fixed ? this.fixedColumns : this.columns;
        if (fixed) {
          if (columns.length === 0) return;
        }
        var rowTemplate = '';

        columns.forEach(function (column, index) {
          var columnTemplate = column.headerTemplate || `{{ columns[${index}].label }}`;
          rowTemplate += `<th @mousemove="handleMouseMove($event, columns[${index}])" @mouseout="handleMouseOut" @mousedown="handleMouseDown($event, columns[${index}])" @click="$parent.handleHeaderClick(columns[${index}], $event)" class="{{ columns[${index}].id }} {{ columns[${index}].direction }}" ><div>${ columnTemplate }</div><i class="sort-caret"></i></th>`;
        });

        if (!fixed) {
          rowTemplate += `<th class="gutter" style="width: ${GUTTER_WIDTH}px">&nbsp;</th>`;
        }

        var repeatTemplate = '<tr>' + rowTemplate + '</tr>';

        var headerTable = this.$el.querySelector(fixed ? '.grid-fixed-header-wrapper thead' : '.grid-header thead');

        this.$header = new Vue({
          parent: this,
          el: headerTable,
          template: repeatTemplate,
          replace: false,
          methods: {
            handleMouseDown(event, column) {
              if (this.dragReadyColumn) {
                this.dragging = true;

                this.$parent.resizeProxyVisible = true;

                var gridEl = this.$parent.$el;
                var gridLeft = gridEl.getBoundingClientRect().left;
                var columnEl = this.$el.querySelector(`th.${column.id}`);
                var columnRect = columnEl.getBoundingClientRect();
                var minLeft = columnRect.left - gridLeft + 30;

                addClass(columnEl, 'noclick');

                this.dragState = {
                  startMouseLeft: event.clientX,
                  startLeft: columnRect.right - gridLeft,
                  startColumnLeft: columnRect.left - gridLeft,
                  gridLeft: gridLeft
                };

                var resizeProxy = this.$parent.$els.resizeProxy;
                resizeProxy.style.left = this.dragState.startLeft + 'px';

                document.onselectstart = function() { return false; };
                document.ondragstart = function() { return false; };

                var mousemove = (event) => {
                  var deltaLeft = event.clientX - this.dragState.startMouseLeft;
                  var proxyLeft = this.dragState.startLeft + deltaLeft;

                  resizeProxy.style.left = Math.max(minLeft, proxyLeft) + 'px';
                };

                var mouseup = () => {
                  if (this.dragging) {
                    var finalLeft = parseInt(resizeProxy.style.left, 10);
                    var columnWidth = finalLeft - this.dragState.startColumnLeft;
                    column.width = column.realWidth = columnWidth;

                    Vue.nextTick(() => {
                      this.$parent.$calcColumns();
                    });

                    document.body.style.cursor = '';
                    this.dragging = false;
                    this.dragReadyColumn = null;
                    this.dragState = {};

                    this.$parent.resizeProxyVisible = false;
                  }

                  document.removeEventListener('mousemove', mousemove);
                  document.removeEventListener('mouseup', mouseup);
                  document.onselectstart = null;
                  document.ondragstart = null;

                  setTimeout(function() {
                    removeClass(columnEl, 'noclick');
                  }, 0);
                };

                document.addEventListener('mousemove', mousemove);
                document.addEventListener('mouseup', mouseup);
              }
            },

            handleMouseMove(event, column) {
              var target = event.target;
              if (!column || !column.resizable) return;

              if (!this.dragging) {
                var rect = target.getBoundingClientRect();

                if (rect.right - event.pageX < 8) {
                  document.body.style.cursor = 'e-resize';
                  this.dragReadyColumn = column;
                } else if (!this.dragging) {
                  document.body.style.cursor = '';
                  this.dragReadyColumn = null;
                }
              }
            },

            handleMouseOut() {
              document.body.style.cursor = '';
            }
          },

          data() {
            return {
              dragReadyColumn: false,
              dragging: false,
              dragState: {},
              columns: columns.slice(0)
            };
          }
        });
      },

      $renderBody(fixed) {
        var columns = fixed ? this.fixedColumns : this.columns;
        if (fixed && columns.length === 0) return;
        var rowTemplate = '';

        columns.forEach(function (column) {
          var columnTemplate = column.template || '';
          rowTemplate += `<td class="${column.id}"><div class="cell">${columnTemplate}</div></td>`;
        });

        if (!fixed) {
          rowTemplate += '<td class="gutter"></td>';
        }

        var repeatTemplate = '<tr v-for="row in $parent.data | orderBy $parent.sortingProperty $parent.sortingDirection" @click="handleClick(row)" @mouseenter="$parent.$parent.hoverRowIndex = $index" :class="{ \'current-row\': row === $parent.$parent.selected, hover: $parent.$parent.hoverRowIndex === $index }">' + rowTemplate + '</tr>';

        var bodyTable = this.$el.querySelector(fixed ? '.grid-fixed-body-wrapper tbody' : '.grid-body tbody');

        var grid = this;

        this.$body = new Vue({
          parent: this,
          inherit: true,
          el: bodyTable,
          template: repeatTemplate,
          replace: false,
          filters: {
            orderBy(array, sortKey, reverse) {
              if (!sortKey) {
                return array;
              }
              var order = (reverse && reverse < 0) ? -1 : 1;

              // sort on a copy to avoid mutating original array
              return array.slice().sort(function (a, b) {
                if (sortKey !== '$key') {
                  if (isObject(a) && '$value' in a) a = a.$value;
                  if (isObject(b) && '$value' in b) b = b.$value;
                }
                a = isObject(a) ? getPath(a, sortKey) : a;
                b = isObject(b) ? getPath(b, sortKey) : b;
                return a === b ? 0 : a > b ? order : -order;
              })
            }
          },
          methods: {
            handleClick: function(row) {
              // TODO add selection change.
              if (grid.selectionMode === 'single') {
                grid.selected = row;
                grid.$emit('selection-change', row);
              }
              grid.$emit('row-click', row);
            },
            $getPropertyText: function(row, property, columnId) {
              var column = null;
              grid.columns.forEach(function(item) {
                if (item.id === columnId) {
                  column = item;
                }
              });

              if (column && column.formatter) {
                return column.formatter(row, column);
              }

              var schema = grid.gridSchema;
              if (schema) {
                var mapping = schema.getPropertyMapping(property);
                if (mapping) {
                  return schema.translateProperty(property, row[property]);
                }
                return schema.getPropertyText(row, property);
              }

              return row[property];
            }
          }
        });
      }
    },

    created() {
      this.gridId = 'grid_' + gridIdSeed + '_';

      if (GUTTER_WIDTH === undefined) {
        GUTTER_WIDTH = getScrollbarWidth();
      }
      this.gutterWidth = GUTTER_WIDTH;

      this.debouncedReRender = debounce(() => {
        Vue.nextTick(() => {
          this.reRender();
        });
      }, 200);
    },

    computed: {
      selection() {
        if (this.selectionMode === 'multiple') {
          var data = this.data || [];
          return data.filter(function(item) {
            return item.$selected === true;
          });
        } else if (this.selectionMode === 'single') {
          return this.selected;
        } else {
          return null;
        }
      },

      fixedColumns() {
        var columns = this.columns || [];
        var fixedColumnCount = this.fixedColumnCount;
        return columns.filter(function(item, index) {
          return index < fixedColumnCount;
        });
      },

      gridSchema() {
        var schema = this.schema;

        if (typeof schema === 'string') {
          schema = SchemaStore.getSchema(schema);
        }

        return typeof schema === 'string' ? null : schema;
      }
    },

    watch: {
      height(value) {
        this.$calcHeight(value);
      },

      data(newVal) {
        this.handleDataChange(newVal);
      }
    },

    beforeCompile() {
      var styleNode = document.createElement('style');
      styleNode.type = 'text/css';
      styleNode.rel = 'stylesheet';
      styleNode.title = 'Grid Column Style';
      document.getElementsByTagName('head')[0].appendChild(styleNode);

      this.styleNode = styleNode;
    },

    destroyed() {
      if (this.styleNode) {
        this.styleNode.parentNode.removeChild(this.styleNode);
      }
      if (this.windowResizeListener) {
        window.removeEventListener('resize', this.windowResizeListener);
      }
    },

    ready() {
      this.doRender();

      this.$ready = true;
      if (this.data) {
        this.handleDataChange(this.data);
      }
    },

    data() {
      return {
        showHoriScrollbar: false,
        hoverRowIndex: null,
        headerHeight: 35,
        selected: null,
        columns: [],
        resizeProxyVisible: false,
        bodyWidth: '',
        fixedBodyWidth: '',
        sortingColumn: null,
        sortingProperty: null,
        sortingDirection: 1
      };
    }
  };
</script>