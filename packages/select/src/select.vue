<template>
  <div
    class="el-select"
    v-clickoutside="handleClose">
    <div
      class="el-select__tags"
      v-if="multiple"
      @click.stop="toggleMenu"
      ref="tags"
      :style="{ 'max-width': inputWidth - 32 + 'px' }">
      <transition-group @after-leave="resetInputHeight">
        <el-tag
          v-for="item in selected"
          :key="item.value"
          closable
          :hit="item.hitState"
          type="primary"
          @close="deleteTag($event, item)"
          close-transition>
          {{ item.currentLabel }}
        </el-tag>
      </transition-group>

      <input
        type="text"
        class="el-select__input"
        @focus="visible = true"
        @keyup="managePlaceholder"
        @keydown="resetInputState"
        @keydown.down.prevent="navigateOptions('next')"
        @keydown.up.prevent="navigateOptions('prev')"
        @keydown.enter.prevent="selectOption"
        @keydown.esc.prevent="visible = false"
        @keydown.delete="deletePrevTag"
        v-model="query"
        :debounce="remote ? 300 : 0"
        v-if="filterable"
        :style="{ width: inputLength + 'px', 'max-width': inputWidth - 42 + 'px' }"
        ref="input">
    </div>
    <el-input
      ref="reference"
      v-model="selectedLabel"
      type="text"
      :placeholder="currentPlaceholder"
      :name="name"
      :disabled="disabled"
      :readonly="!filterable || multiple"
      @focus="toggleMenu"
      @click="handleIconClick"
      @mousedown.native="handleMouseDown"
      @keyup.native="debouncedOnInputChange"
      @keydown.native.down.prevent="navigateOptions('next')"
      @keydown.native.up.prevent="navigateOptions('prev')"
      @keydown.native.enter.prevent="selectOption"
      @keydown.native.esc.prevent="visible = false"
      @keydown.native.tab="visible = false"
      @mouseenter.native="inputHovering = true"
      @mouseleave.native="inputHovering = false"
      :icon="iconClass">
    </el-input>
    <transition name="md-fade-bottom" @after-leave="doDestroy">
      <el-select-menu
        ref="popper"
        v-show="visible && emptyText !== false">
        <ul
          class="el-select-dropdown__list"
          :class="{ 'is-empty': !allowCreate && filteredOptionsCount === 0 }"
          v-show="options.length > 0 && !loading">
          <el-option
            :value="query"
            created
            v-if="showNewOption">
          </el-option>
          <slot></slot>
        </ul>
        <p class="el-select-dropdown__empty" v-if="emptyText && !allowCreate">{{ emptyText }}</p>
      </el-select-menu>
    </transition>
  </div>
</template>

<script type="text/babel">
  import Emitter from 'element-ui/src/mixins/emitter';
  import Locale from 'element-ui/src/mixins/locale';
  import ElInput from 'element-ui/packages/input';
  import ElSelectMenu from './select-dropdown.vue';
  import ElOption from './option.vue';
  import ElTag from 'element-ui/packages/tag';
  import debounce from 'throttle-debounce/debounce';
  import Clickoutside from 'element-ui/src/utils/clickoutside';
  import { addClass, removeClass, hasClass } from 'wind-dom/src/class';
  import { addResizeListener, removeResizeListener } from 'element-ui/src/utils/resize-event';
  import { t } from 'element-ui/src/locale';

  export default {
    mixins: [Emitter, Locale],

    name: 'ElSelect',

    componentName: 'ElSelect',

    computed: {
      iconClass() {
        return this.showCloseIcon ? 'circle-close' : (this.remote && this.filterable ? '' : 'caret-top');
      },

      debounce() {
        return this.remote ? 300 : 0;
      },

      showCloseIcon() {
        let criteria = this.clearable &&
          this.inputHovering &&
          !this.multiple &&
          this.value !== undefined &&
          this.value !== '';
        if (!this.$el) return false;
        this.$nextTick(() => {
          let icon = this.$el.querySelector('.el-input__icon');
          if (icon) {
            criteria ? addClass(icon, 'is-show-close') : removeClass(icon, 'is-show-close');
          }
        });
        return criteria;
      },

      emptyText() {
        if (this.loading) {
          return this.t('el.select.loading');
        } else {
          if (this.voidRemoteQuery) {
            this.voidRemoteQuery = false;
            return false;
          }
          if (this.filterable && this.filteredOptionsCount === 0) {
            return this.t('el.select.noMatch');
          }
          if (this.options.length === 0) {
            return this.t('el.select.noData');
          }
        }
        return null;
      },

      showNewOption() {
        let hasExistingOption = this.options.filter(option => !option.created)
          .some(option => option.currentLabel === this.query);
        return this.filterable && this.allowCreate && this.query !== '' && !hasExistingOption;
      }
    },

    components: {
      ElInput,
      ElSelectMenu,
      ElOption,
      ElTag
    },

    directives: { Clickoutside },

    props: {
      name: String,
      value: {},
      disabled: Boolean,
      clearable: Boolean,
      filterable: Boolean,
      allowCreate: Boolean,
      loading: Boolean,
      remote: Boolean,
      remoteMethod: Function,
      filterMethod: Function,
      multiple: Boolean,
      multipleLimit: {
        type: Number,
        default: 0
      },
      placeholder: {
        type: String,
        default() {
          return t('el.select.placeholder');
        }
      }
    },

    data() {
      return {
        options: [],
        selected: this.multiple ? [] : {},
        isSelect: true,
        inputLength: 20,
        inputWidth: 0,
        cachedPlaceHolder: '',
        optionsCount: 0,
        filteredOptionsCount: 0,
        dropdownUl: null,
        visible: false,
        selectedLabel: '',
        hoverIndex: -1,
        query: '',
        voidRemoteQuery: false,
        bottomOverflowBeforeHidden: 0,
        topOverflowBeforeHidden: 0,
        optionsAllDisabled: false,
        inputHovering: false,
        currentPlaceholder: ''
      };
    },

    watch: {
      placeholder(val) {
        this.currentPlaceholder = val;
      },

      value(val) {
        if (this.multiple) {
          this.resetInputHeight();
          if (val.length > 0) {
            this.currentPlaceholder = '';
          } else {
            this.currentPlaceholder = this.cachedPlaceHolder;
          }
        }
        this.selected = this.getSelected();
        if (this.filterable && !this.multiple) {
          this.inputLength = 20;
        }
        this.$emit('change', val);
        this.dispatch('ElFormItem', 'el.form.change', val);
      },

      query(val) {
        this.broadcast('ElSelectDropdown', 'updatePopper');
        this.hoverIndex = -1;
        if (this.multiple && this.filterable) {
          this.resetInputHeight();
        }
        if (this.remote && typeof this.remoteMethod === 'function') {
          this.hoverIndex = -1;
          this.remoteMethod(val);
          this.voidRemoteQuery = val === '';
          this.broadcast('ElOption', 'resetIndex');
        } else if (typeof this.filterMethod === 'function') {
          this.filterMethod(val);
        } else {
          this.filteredOptionsCount = this.optionsCount;
          this.broadcast('ElOption', 'queryChange', val);
        }
      },

      visible(val) {
        if (!val) {
          this.$refs.reference.$el.querySelector('input').blur();
          this.handleIconHide();
          this.broadcast('ElSelectDropdown', 'destroyPopper');
          if (this.$refs.input) {
            this.$refs.input.blur();
          }
          this.query = '';
          this.selectedLabel = '';
          this.resetHoverIndex();
          if (!this.multiple) {
            this.getOverflows();
            if (this.selected && this.selected.value) {
              this.selectedLabel = this.selected.currentLabel;
            }
          }
        } else {
          this.handleIconShow();
          this.broadcast('ElSelectDropdown', 'updatePopper');
          if (this.filterable) {
            this.query = this.selectedLabel;
            if (this.multiple) {
              this.$refs.input.focus();
            } else {
              this.broadcast('ElInput', 'inputSelect');
            }
          }
          if (!this.dropdownUl) {
            let dropdownChildNodes = this.$refs.popper.$el.childNodes;
            this.dropdownUl = [].filter.call(dropdownChildNodes, item => item.tagName === 'UL')[0];
          }
          if (!this.multiple && this.dropdownUl) {
            this.setOverflow();
          }
        }
      },

      options(val) {
        this.optionsAllDisabled = val.length === val.filter(item => item.disabled === true).length;
        if (this.multiple) {
          this.resetInputHeight();
        }
      }
    },

    methods: {
      handleIconHide() {
        let icon = this.$el.querySelector('.el-input__icon');
        if (icon) {
          removeClass(icon, 'is-reverse');
        }
      },

      handleIconShow() {
        let icon = this.$el.querySelector('.el-input__icon');
        if (icon && !hasClass(icon, 'el-icon-circle-close')) {
          addClass(icon, 'is-reverse');
        }
      },

      getOverflows() {
        if (this.dropdownUl && this.selected && this.selected.$el) {
          let selectedRect = this.selected.$el.getBoundingClientRect();
          let popperRect = this.$refs.popper.$el.getBoundingClientRect();
          this.bottomOverflowBeforeHidden = selectedRect.bottom - popperRect.bottom;
          this.topOverflowBeforeHidden = selectedRect.top - popperRect.top;
        }
      },

      setOverflow() {
        if (this.bottomOverflowBeforeHidden > 0) {
          this.$nextTick(() => {
            this.dropdownUl.scrollTop += this.bottomOverflowBeforeHidden;
          });
        } else if (this.topOverflowBeforeHidden < 0) {
          this.$nextTick(() => {
            this.dropdownUl.scrollTop += this.topOverflowBeforeHidden;
          });
        }
      },

      getSelected() {
        if (!this.multiple) {
          let option = this.options.filter(option => option.value === this.value)[0] ||
              { value: this.value, currentLabel: this.value };
          this.selectedLabel = option.currentLabel;
          return option;
        }
        let result = [];
        if (Array.isArray(this.value)) {
          this.value.forEach(value => {
            let option = this.options.filter(option => option.value === value)[0];
            if (option) {
              result.push(option);
            } else {
              result.push({
                value: this.value,
                currentLabel: value,
                hitState: false
              });
            }
          });
        }
        return result;
      },

      handleIconClick(event) {
        if (this.iconClass === 'circle-close') {
          this.deleteSelected(event);
        } else {
          this.toggleMenu();
        }
      },

      handleMouseDown(event) {
        if (event.target.tagName !== 'INPUT') return;
        if (this.visible) {
          this.handleClose();
          event.preventDefault();
        }
      },

      doDestroy() {
        this.$refs.popper.doDestroy();
      },

      handleClose() {
        this.visible = false;
      },

      toggleLastOptionHitState(hit) {
        if (!Array.isArray(this.selected)) return;
        const option = this.selected[this.selected.length - 1];
        if (!option) return;

        if (hit === true || hit === false) {
          option.hitState = hit;
          return hit;
        }

        option.hitState = !option.hitState;
        return option.hitState;
      },

      deletePrevTag(e) {
        if (e.target.value.length <= 0 && !this.toggleLastOptionHitState()) {
          this.value.pop();
        }
      },

      managePlaceholder() {
        if (this.currentPlaceholder !== '') {
          this.currentPlaceholder = this.$refs.input.value ? '' : this.cachedPlaceHolder;
        }
      },

      resetInputState(e) {
        if (e.keyCode !== 8) this.toggleLastOptionHitState(false);
        this.inputLength = this.$refs.input.value.length * 15 + 20;
      },

      resetInputHeight() {
        this.$nextTick(() => {
          let inputChildNodes = this.$refs.reference.$el.childNodes;
          let input = [].filter.call(inputChildNodes, item => item.tagName === 'INPUT')[0];
          input.style.height = Math.max(this.$refs.tags.clientHeight + 6, this.size === 'small' ? 28 : 36) + 'px';
          this.broadcast('ElSelectDropdown', 'updatePopper');
        });
      },

      resetHoverIndex() {
        setTimeout(() => {
          if (!this.multiple) {
            this.hoverIndex = this.options.indexOf(this.selected);
          } else {
            if (this.selected.length > 0) {
              this.hoverIndex = Math.min.apply(null, this.selected.map(item => this.options.indexOf(item)));
            } else {
              this.hoverIndex = -1;
            }
          }
        }, 300);
      },

      handleOptionSelect(option) {
        if (!this.multiple) {
          this.$emit('input', option.value);
          this.visible = false;
        } else {
          let optionIndex = -1;
          this.value.forEach((item, index) => {
            if (item === option.value) {
              optionIndex = index;
            }
          });
          if (optionIndex > -1) {
            this.value.splice(optionIndex, 1);
          } else if (this.multipleLimit <= 0 || this.value.length < this.multipleLimit) {
            this.value.push(option.value);
          }
          if (option.created) {
            this.query = '';
            this.inputLength = 20;
            this.$refs.input.focus();
          }
        }
      },

      toggleMenu() {
        if (this.filterable && this.query === '' && this.visible) {
          return;
        }
        if (!this.disabled) {
          this.visible = !this.visible;
        }
      },

      navigateOptions(direction) {
        if (!this.visible) {
          this.visible = true;
          return;
        }
        if (this.options.length === 0 || this.filteredOptionsCount === 0) return;
        if (!this.optionsAllDisabled) {
          if (direction === 'next') {
            this.hoverIndex++;
            if (this.hoverIndex === this.options.length) {
              this.hoverIndex = 0;
            }
            this.resetScrollTop();
            if (this.options[this.hoverIndex].disabled === true ||
              this.options[this.hoverIndex].groupDisabled === true ||
              !this.options[this.hoverIndex].visible) {
              this.navigateOptions('next');
            }
          }
          if (direction === 'prev') {
            this.hoverIndex--;
            if (this.hoverIndex < 0) {
              this.hoverIndex = this.options.length - 1;
            }
            this.resetScrollTop();
            if (this.options[this.hoverIndex].disabled === true ||
              this.options[this.hoverIndex].groupDisabled === true ||
              !this.options[this.hoverIndex].visible) {
              this.navigateOptions('prev');
            }
          }
        }
      },

      resetScrollTop() {
        let bottomOverflowDistance = this.options[this.hoverIndex].$el.getBoundingClientRect().bottom -
          this.$refs.popper.$el.getBoundingClientRect().bottom;
        let topOverflowDistance = this.options[this.hoverIndex].$el.getBoundingClientRect().top -
          this.$refs.popper.$el.getBoundingClientRect().top;
        if (bottomOverflowDistance > 0) {
          this.dropdownUl.scrollTop += bottomOverflowDistance;
        }
        if (topOverflowDistance < 0) {
          this.dropdownUl.scrollTop += topOverflowDistance;
        }
      },

      selectOption() {
        if (this.options[this.hoverIndex]) {
          this.handleOptionSelect(this.options[this.hoverIndex]);
        }
      },

      deleteSelected(event) {
        event.stopPropagation();
        this.$emit('input', '');
        this.$emit('change', '');
        this.visible = false;
      },

      deleteTag(event, tag) {
        let index = this.selected.indexOf(tag);
        if (index > -1) {
          this.value.splice(index, 1);
        }
        event.stopPropagation();
      },

      onInputChange() {
        if (this.filterable) {
          this.query = this.selectedLabel;
        }
      },

      onOptionDestroy(option) {
        this.optionsCount--;
        this.filteredOptionsCount--;
        let index = this.options.indexOf(option);
        if (index > -1) {
          this.options.splice(index, 1);
        }
        this.broadcast('ElOption', 'resetIndex');
      },

      resetInputWidth() {
        this.inputWidth = this.$refs.reference.$el.getBoundingClientRect().width;
      }
    },

    created() {
      this.cachedPlaceHolder = this.currentPlaceholder = this.placeholder;
      if (this.multiple && !Array.isArray(this.value)) {
        this.$emit('input', []);
      }
      if (!this.multiple && (!this.value || Array.isArray(this.value))) {
        this.$emit('input', '');
      }
      if (this.remote) {
        this.voidRemoteQuery = true;
      }

      this.debouncedOnInputChange = debounce(this.debounce, () => {
        this.onInputChange();
      });

      this.$on('handleOptionClick', this.handleOptionSelect);
      this.$on('onOptionDestroy', this.onOptionDestroy);
    },

    mounted() {
      if (this.multiple && Array.isArray(this.value) && this.value.length > 0) {
        this.currentPlaceholder = '';
      }
      addResizeListener(this.$el, this.resetInputWidth);
      this.selected = this.getSelected();
      if (this.remote && this.multiple) {
        this.resetInputHeight();
      }
      this.$nextTick(() => {
        if (this.$refs.reference.$el) {
          this.inputWidth = this.$refs.reference.$el.getBoundingClientRect().width;
        }
      });
    },

    destroyed() {
      if (this.resetInputWidth) removeResizeListener(this.$el, this.resetInputWidth);
    }
  };
</script>
