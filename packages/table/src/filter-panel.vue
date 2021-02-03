<template>
  <transition name="el-zoom-in-top">
    <div
      class="el-table-filter"
      v-if="multiple"
      v-clickoutside="handleOutsideClick"
      v-show="showPopper">
      <div v-if="sortable" class="el-table-filter__sort">
        <span class="el-table-filter__sort--title">排序</span>
        <div class="el-table-filter__sort--btns">
          <el-button @click="handleSortClick('ascending')" class="el-table-filter__sort--ascending" :type="sortOrder === 'ascending' ? 'primary' : 'default'" size="small">升序</el-button>
          <el-button @click="handleSortClick('descending')" class="el-table-filter__sort--descending" :type="sortOrder === 'descending' ? 'primary' : 'default'" size="small">降序</el-button>
        </div>
      </div>
      <div class="el-table-filter__top">
        <button>{{ t('el.table.confirmFilter') }}</button>
        <button class="el-table-filter__top--reset" @click="handleReset">{{ t('el.table.resetFilter') }}</button>
      </div>
      <div class="el-table-filter__content">
        <div v-if="filterSearch" class="el-table-filter__search">
          <el-input
            ref="filterSearch"
            v-model="filterSearchValue"
            clearable
            size="small"
            @keypress.enter.native="handleFilterSearch">
            <el-button @click="handleFilterSearch" slot="append" icon="el-icon-search"></el-button>
          </el-input>
        </div>
        <template v-if="filters && filters.length">
          <el-scrollbar wrap-class="el-table-filter__wrap">
            <el-checkbox class="el-table-filter__checkbox-all" :indeterminate="isIndeterminate" v-model="checkAll" @change="handleCheckAllChange">{{ t('el.table.checkAll') }}</el-checkbox>
            <el-checkbox-group class="el-table-filter__checkbox-group" v-model="filteredValue"  @change="handleCheckedFiltersChange">
              <el-checkbox
                v-for="filter in filtersOptions"
                :key="filter.value"
                :label="filter.value">{{ filter.text }}</el-checkbox>
            </el-checkbox-group>
          </el-scrollbar>
          <div class="el-table-filter__bottom">
            <button class="el-table-filter__bottom--cancel" @click="handleOutsideClick">{{ t('el.table.cancel') }}</button>
            <button @click="handleConfirm">{{ t('el.table.confirm') }}</button>
          </div>
        </template>
      </div>
    </div>
    <div
      class="el-table-filter"
      v-else
      v-clickoutside="handleOutsideClick"
      v-show="showPopper">
      <ul class="el-table-filter__list">
        <li class="el-table-filter__list-item"
            :class="{ 'is-active': filterValue === undefined || filterValue === null }"
            @click="handleSelect(null)">{{ t('el.table.clearFilter') }}</li>
        <li class="el-table-filter__list-item"
            v-for="filter in filters"
            :label="filter.value"
            :key="filter.value"
            :class="{ 'is-active': isActive(filter) }"
            @click="handleSelect(filter.value)" >{{ filter.text }}</li>
      </ul>
    </div>
  </transition>
</template>

<script type="text/babel">
  import Popper from 'eb-element/src/utils/vue-popper';
  import { PopupManager } from 'eb-element/src/utils/popup';
  import Locale from 'eb-element/src/mixins/locale';
  import Clickoutside from 'eb-element/src/utils/clickoutside';
  import Dropdown from './dropdown';
  import ElCheckbox from 'eb-element/packages/checkbox';
  import ElCheckboxGroup from 'eb-element/packages/checkbox-group';
  import ElScrollbar from 'eb-element/packages/scrollbar';
  import ElInput from 'eb-element/packages/input';

  export default {
    name: 'ElTableFilterPanel',

    mixins: [Popper, Locale],

    directives: {
      Clickoutside
    },

    components: {
      ElCheckbox,
      ElCheckboxGroup,
      ElScrollbar,
      ElInput
    },

    props: {
      placement: {
        type: String,
        default: 'bottom-start'
      }
    },

    methods: {
      isActive(filter) {
        return filter.value === this.filterValue;
      },

      handleOutsideClick() {
        setTimeout(() => {
          this.showPopper = false;
        }, 16);
      },

      handleConfirm() {
        this.confirmFilter(this.filteredValue);
        this.handleOutsideClick();
      },

      handleReset() {
        this.filterSearchValue = '';
        this.filteredValue = [];
        this.confirmFilter(this.filteredValue);
        this.handleOutsideClick();
      },

      handleSelect(filterValue) {
        this.filterValue = filterValue;

        if ((typeof filterValue !== 'undefined') && (filterValue !== null)) {
          this.confirmFilter(this.filteredValue);
        } else {
          this.confirmFilter([]);
        }

        this.handleOutsideClick();
      },

      confirmFilter(filteredValue) {
        this.table.store.commit('filterChange', {
          column: this.column,
          values: filteredValue
        });
        this.table.store.updateAllSelected();
      },

      handleFilterSearch() {
        // 输入框确认分两种情况，一种是直接搜索，一种是搜索筛选项
        if (this.filters) {
          // 多选+搜索模式
          return;
        }
        // 搜索模式
        if (this.filteredValue) {
          const value = this.filterSearchValue;
          if ((typeof value !== 'undefined') && (value !== '')) {
            this.filteredValue = [value];
          } else {
            this.filteredValue = [];
          }
        }
        this.handleConfirm();
      },

      handleCheckAllChange(val) {
        this.filteredValue = val ? this.filters.map(filter => filter.value) : [];
      },

      handleCheckedFiltersChange(val) {
        const checkedCount = val.length;
        this.checkAll = checkedCount === this.filters.length;
      },

      handleSortClick(givenOrder) {
        const sortOrder = this.column.order === givenOrder ? null : givenOrder;
        const states = this.table.store.states;
        const sortingColumn = states.sortingColumn;

        if (sortingColumn !== this.column) {
          if (sortingColumn) {
            sortingColumn.order = null;
          }
          states.sortingColumn = this.column;
        }

        this.column.order = sortOrder;

        states.sortProp = this.column.property;
        states.sortOrder = sortOrder;

        this.table.store.commit('changeSortCondition');
      }
    },

    data() {
      return {
        table: null,
        cell: null,
        column: null,
        checkAll: false
      };
    },

    computed: {
      filters() {
        return this.column && this.column.filters;
      },

      filtersOptions() {
        if (this.filterSearch) {
          return this.filters.filter(filter => [filter.value, filter.text].some(val => val.includes(this.filterSearchValue)));
        } else {
          return this.filters;
        }
      },

      filterValue: {
        get() {
          return (this.column.filteredValue || [])[0];
        },
        set(value) {
          if (this.filteredValue) {
            if ((typeof value !== 'undefined') && (value !== null)) {
              this.filteredValue.splice(0, 1, value);
            } else {
              this.filteredValue.splice(0, 1);
            }
          }
        }
      },

      filteredValue: {
        get() {
          if (this.column) {
            return this.column.filteredValue || [];
          }
          return [];
        },
        set(value) {
          if (this.column) {
            this.column.filteredValue = value;
          }
        }
      },

      filterSearch() {
        return this.column && this.column.filterSearch;
      },

      filterSearchValue: {
        get() {
          if (this.column) {
            return this.column.filterSearchValue || '';
          }
          return '';
        },
        set(value) {
          if (this.column) {
            this.column.filterSearchValue = value;
          }
          // 输入框赋值分两种情况，一种是只赋值输入框用于过滤筛选项，一种是同步到筛选值
          if (this.filters) {
            // 多选+搜索模式
            return;
          }
          // 搜索模式
          if (this.filteredValue) {
            if ((typeof value !== 'undefined') && (value !== null)) {
              this.filteredValue.splice(0, 1, value);
            } else {
              this.filteredValue.splice(0, 1);
            }
          }
        }
      },

      multiple() {
        if (this.column) {
          return this.column.filterMultiple;
        }
        return true;
      },

      disabledConfirm() {
        if (this.filterSearch) {
          return !this.filterSearchValue.length;
        }
        return this.filteredValue.length === 0;
      },

      sortable() {
        return this.column.sortable;
      },

      sortOrder() {
        let states = this.table.store.states;
        if (states.sortingColumn === this.column) {
          return states.sortOrder;
        }
        return null;
      },

      isIndeterminate() {
        return (this.filteredValue.length !== 0) && (this.filteredValue.length !== this.filters.length);
      }
    },

    mounted() {
      this.popperElm = this.$el;
      this.referenceElm = this.cell;
      this.table.bodyWrapper.addEventListener('scroll', () => {
        this.updatePopper();
      });

      this.$watch('showPopper', (value) => {
        if (this.column) this.column.filterOpened = value;
        if (value) {
          Dropdown.open(this);
        } else {
          Dropdown.close(this);
        }
      });
    },
    watch: {
      showPopper(val) {
        if (val === true && parseInt(this.popperJS._popper.style.zIndex, 10) < PopupManager.zIndex) {
          this.popperJS._popper.style.zIndex = PopupManager.nextZIndex();
        }
        if (val && this.filterSearch && this.$refs.filterSearch) {
          this.$nextTick(() => {
            this.$refs.filterSearch.focus();
          });
        }
      }
    }
  };
</script>
