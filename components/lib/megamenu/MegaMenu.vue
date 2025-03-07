<template>
    <div :ref="containerRef" :id="id" :class="cx('root')" v-bind="ptm('root')" data-pc-name="megamenu">
        <div v-if="$slots.start" :class="cx('start')" v-bind="ptm('start')">
            <slot name="start"></slot>
        </div>
        <MegaMenuSub
            :ref="menubarRef"
            :id="id + '_list'"
            :tabindex="!disabled ? tabindex : -1"
            role="menubar"
            :aria-label="ariaLabel"
            :aria-labelledby="ariaLabelledby"
            :aria-disabled="disabled || undefined"
            :aria-orientation="orientation"
            :aria-activedescendant="focused ? focusedItemId : undefined"
            :menuId="id"
            :focusedItemId="focused ? focusedItemId : undefined"
            :items="processedItems"
            :horizontal="horizontal"
            :templates="$slots"
            :activeItem="activeItem"
            :exact="exact"
            :level="0"
            :pt="pt"
            @focus="onFocus"
            @blur="onBlur"
            @keydown="onKeyDown"
            @item-click="onItemClick"
            @item-mouseenter="onItemMouseEnter"
        />
        <div v-if="$slots.end" :class="cx('end')" v-bind="ptm('end')">
            <slot name="end"></slot>
        </div>
    </div>
</template>

<script>
import { DomHandler, ObjectUtils, UniqueComponentId } from 'primevue/utils';
import BaseMegaMenu from './BaseMegaMenu.vue';
import MegaMenuSub from './MegaMenuSub.vue';

export default {
    name: 'MegaMenu',
    extends: BaseMegaMenu,
    emits: ['focus', 'blur'],
    props: {
        model: {
            type: Array,
            default: null
        },
        orientation: {
            type: String,
            default: 'horizontal'
        },
        exact: {
            type: Boolean,
            default: true
        },
        disabled: {
            type: Boolean,
            default: false
        },
        tabindex: {
            type: Number,
            default: 0
        },
        'aria-labelledby': {
            type: String,
            default: null
        },
        'aria-label': {
            type: String,
            default: null
        }
    },
    outsideClickListener: null,
    resizeListener: null,
    container: null,
    menubar: null,
    searchTimeout: null,
    searchValue: null,
    data() {
        return {
            id: this.$attrs.id,
            focused: false,
            focusedItemInfo: { index: -1, key: '', parentKey: '' },
            activeItem: null,
            dirty: false
        };
    },
    watch: {
        '$attrs.id': function (newValue) {
            this.id = newValue || UniqueComponentId();
        },
        activeItem(newItem) {
            if (ObjectUtils.isNotEmpty(newItem)) {
                this.bindOutsideClickListener();
                this.bindResizeListener();
            } else {
                this.unbindOutsideClickListener();
                this.unbindResizeListener();
            }
        }
    },
    mounted() {
        this.id = this.id || UniqueComponentId();
    },
    beforeUnmount() {
        this.unbindOutsideClickListener();
        this.unbindResizeListener();
    },
    methods: {
        getItemProp(item, name) {
            return item ? ObjectUtils.getItemValue(item[name]) : undefined;
        },
        getItemLabel(item) {
            return this.getItemProp(item, 'label');
        },
        isItemDisabled(item) {
            return this.getItemProp(item, 'disabled');
        },
        isItemGroup(item) {
            return ObjectUtils.isNotEmpty(this.getItemProp(item, 'items'));
        },
        isItemSeparator(item) {
            return this.getItemProp(item, 'separator');
        },
        getProccessedItemLabel(processedItem) {
            return processedItem ? this.getItemLabel(processedItem.item) : undefined;
        },
        isProccessedItemGroup(processedItem) {
            return processedItem && ObjectUtils.isNotEmpty(processedItem.items);
        },
        hide(event, isFocus) {
            this.activeItem = null;
            this.focusedItemInfo = { index: -1, key: '', parentKey: '' };

            isFocus && DomHandler.focus(this.menubar);
            this.dirty = false;
        },
        onFocus(event) {
            this.focused = true;

            if (this.focusedItemInfo.index === -1) {
                const index = this.findFirstFocusedItemIndex();
                const processedItem = this.findVisibleItem(index);

                this.focusedItemInfo = { index, key: processedItem.key, parentKey: processedItem.parentKey };
            }

            this.$emit('focus', event);
        },
        onBlur(event) {
            this.focused = false;
            this.focusedItemInfo = { index: -1, key: '', parentKey: '' };
            this.searchValue = '';
            this.dirty = false;
            this.$emit('blur', event);
        },
        onKeyDown(event) {
            if (this.disabled) {
                event.preventDefault();

                return;
            }

            const metaKey = event.metaKey || event.ctrlKey;

            switch (event.code) {
                case 'ArrowDown':
                    this.onArrowDownKey(event);
                    break;

                case 'ArrowUp':
                    this.onArrowUpKey(event);
                    break;

                case 'ArrowLeft':
                    this.onArrowLeftKey(event);
                    break;

                case 'ArrowRight':
                    this.onArrowRightKey(event);
                    break;

                case 'Home':
                    this.onHomeKey(event);
                    break;

                case 'End':
                    this.onEndKey(event);
                    break;

                case 'Space':
                    this.onSpaceKey(event);
                    break;

                case 'Enter':
                    this.onEnterKey(event);
                    break;

                case 'Escape':
                    this.onEscapeKey(event);
                    break;

                case 'Tab':
                    this.onTabKey(event);
                    break;

                case 'PageDown':
                case 'PageUp':
                case 'Backspace':
                case 'ShiftLeft':
                case 'ShiftRight':
                    //NOOP
                    break;

                default:
                    if (!metaKey && ObjectUtils.isPrintableCharacter(event.key)) {
                        this.searchItems(event, event.key);
                    }

                    break;
            }
        },
        onItemChange(event) {
            const { processedItem, isFocus } = event;

            if (ObjectUtils.isEmpty(processedItem)) return;

            const { index, key, parentKey, items } = processedItem;
            const grouped = ObjectUtils.isNotEmpty(items);

            grouped && (this.activeItem = processedItem);
            this.focusedItemInfo = { index, key, parentKey };

            grouped && (this.dirty = true);
            isFocus && DomHandler.focus(this.menubar);
        },
        onItemClick(event) {
            const { originalEvent, processedItem } = event;
            const grouped = this.isProccessedItemGroup(processedItem);
            const root = ObjectUtils.isEmpty(processedItem.parent);
            const selected = this.isSelected(processedItem);

            if (selected) {
                const { index, key, parentKey } = processedItem;

                this.activeItem = null;
                this.focusedItemInfo = { index, key, parentKey };

                this.dirty = !root;
                DomHandler.focus(this.menubar);
            } else {
                if (grouped) {
                    this.onItemChange(event);
                } else {
                    const rootProcessedItem = root ? processedItem : this.activeItem;

                    this.hide(originalEvent);
                    this.changeFocusedItemInfo(originalEvent, rootProcessedItem ? rootProcessedItem.index : -1);

                    DomHandler.focus(this.menubar);
                }
            }
        },
        onItemMouseEnter(event) {
            if (this.dirty) {
                this.onItemChange(event);
            }
        },
        onArrowDownKey(event) {
            if (this.horizontal) {
                if (ObjectUtils.isNotEmpty(this.activeItem) && this.activeItem.key === this.focusedItemInfo.key) {
                    this.focusedItemInfo = { index: -1, key: '', parentKey: this.activeItem.key };
                } else {
                    const processedItem = this.findVisibleItem(this.focusedItemInfo.index);
                    const grouped = this.isProccessedItemGroup(processedItem);

                    if (grouped) {
                        this.onItemChange({ originalEvent: event, processedItem });
                        this.focusedItemInfo = { index: -1, key: processedItem.key, parentKey: processedItem.parentKey };
                        this.searchValue = '';
                    }
                }
            }

            const itemIndex = this.focusedItemInfo.index !== -1 ? this.findNextItemIndex(this.focusedItemInfo.index) : this.findFirstFocusedItemIndex();

            this.changeFocusedItemInfo(event, itemIndex);
            event.preventDefault();
        },
        onArrowUpKey(event) {
            if (event.altKey && this.horizontal) {
                if (this.focusedItemInfo.index !== -1) {
                    const processedItem = this.findVisibleItem(this.focusedItemInfo.index);
                    const grouped = this.isProccessedItemGroup(processedItem);

                    if (!grouped && ObjectUtils.isNotEmpty(this.activeItem)) {
                        if (this.focusedItemInfo.index === 0) {
                            this.focusedItemInfo = { index: this.activeItem.index, key: this.activeItem.key, parentKey: this.activeItem.parentKey };
                            this.activeItem = null;
                        } else {
                            this.changeFocusedItemInfo(event, this.findFirstItemIndex());
                        }
                    }
                }

                event.preventDefault();
            } else {
                const itemIndex = this.focusedItemInfo.index !== -1 ? this.findPrevItemIndex(this.focusedItemInfo.index) : this.findLastFocusedItemIndex();

                this.changeFocusedItemInfo(event, itemIndex);
                event.preventDefault();
            }
        },
        onArrowLeftKey(event) {
            const processedItem = this.findVisibleItem(this.focusedItemInfo.index);
            const grouped = this.isProccessedItemGroup(processedItem);

            if (grouped) {
                if (this.horizontal) {
                    const itemIndex = this.focusedItemInfo.index !== -1 ? this.findPrevItemIndex(this.focusedItemInfo.index) : this.findLastFocusedItemIndex();

                    this.changeFocusedItemInfo(event, itemIndex);
                }
            } else {
                if (this.vertical && ObjectUtils.isNotEmpty(this.activeItem)) {
                    if (processedItem.columnIndex === 0) {
                        this.focusedItemInfo = { index: this.activeItem.index, key: this.activeItem.key, parentKey: this.activeItem.parentKey };
                        this.activeItem = null;
                    }
                }

                const columnIndex = processedItem.columnIndex - 1;
                const itemIndex = this.visibleItems.findIndex((item) => item.columnIndex === columnIndex);

                itemIndex !== -1 && this.changeFocusedItemInfo(event, itemIndex);
            }

            event.preventDefault();
        },
        onArrowRightKey(event) {
            const processedItem = this.findVisibleItem(this.focusedItemInfo.index);
            const grouped = this.isProccessedItemGroup(processedItem);

            if (grouped) {
                if (this.vertical) {
                    if (ObjectUtils.isNotEmpty(this.activeItem) && this.activeItem.key === processedItem.key) {
                        this.focusedItemInfo = { index: -1, key: '', parentKey: this.activeItem.key };
                    } else {
                        const processedItem = this.findVisibleItem(this.focusedItemInfo.index);
                        const grouped = this.isProccessedItemGroup(processedItem);

                        if (grouped) {
                            this.onItemChange({ originalEvent: event, processedItem });
                            this.focusedItemInfo = { index: -1, key: processedItem.key, parentKey: processedItem.parentKey };
                            this.searchValue = '';
                        }
                    }
                }

                const itemIndex = this.focusedItemInfo.index !== -1 ? this.findNextItemIndex(this.focusedItemInfo.index) : this.findFirstFocusedItemIndex();

                this.changeFocusedItemInfo(event, itemIndex);
            } else {
                const columnIndex = processedItem.columnIndex + 1;
                const itemIndex = this.visibleItems.findIndex((item) => item.columnIndex === columnIndex);

                itemIndex !== -1 && this.changeFocusedItemInfo(event, itemIndex);
            }

            event.preventDefault();
        },
        onHomeKey(event) {
            this.changeFocusedItemInfo(event, this.findFirstItemIndex());
            event.preventDefault();
        },
        onEndKey(event) {
            this.changeFocusedItemInfo(event, this.findLastItemIndex());
            event.preventDefault();
        },
        onEnterKey(event) {
            if (this.focusedItemInfo.index !== -1) {
                const element = DomHandler.findSingle(this.menubar, `li[id="${`${this.focusedItemId}`}"]`);
                const anchorElement = element && DomHandler.findSingle(element, 'a[data-pc-section="action"]');

                anchorElement ? anchorElement.click() : element && element.click();

                const processedItem = this.visibleItems[this.focusedItemInfo.index];
                const grouped = this.isProccessedItemGroup(processedItem);

                !grouped && this.changeFocusedItemInfo(event, this.findFirstFocusedItemIndex());
            }

            event.preventDefault();
        },
        onSpaceKey(event) {
            this.onEnterKey(event);
        },
        onEscapeKey(event) {
            if (ObjectUtils.isNotEmpty(this.activeItem)) {
                this.focusedItemInfo = { index: this.activeItem.index, key: this.activeItem.key };
                this.activeItem = null;
            }

            event.preventDefault();
        },
        onTabKey(event) {
            if (this.focusedItemInfo.index !== -1) {
                const processedItem = this.findVisibleItem(this.focusedItemInfo.index);
                const grouped = this.isProccessedItemGroup(processedItem);

                !grouped && this.onItemChange({ originalEvent: event, processedItem });
            }

            this.hide();
        },
        bindOutsideClickListener() {
            if (!this.outsideClickListener) {
                this.outsideClickListener = (event) => {
                    const isOutsideContainer = this.container && !this.container.contains(event.target);
                    const isOutsideTarget = this.popup ? !(this.target && (this.target === event.target || this.target.contains(event.target))) : true;

                    if (isOutsideContainer && isOutsideTarget) {
                        this.hide();
                    }
                };

                document.addEventListener('click', this.outsideClickListener);
            }
        },
        unbindOutsideClickListener() {
            if (this.outsideClickListener) {
                document.removeEventListener('click', this.outsideClickListener);
                this.outsideClickListener = null;
            }
        },
        bindResizeListener() {
            if (!this.resizeListener) {
                this.resizeListener = (event) => {
                    if (!DomHandler.isTouchDevice()) {
                        this.hide(event, true);
                    }
                };

                window.addEventListener('resize', this.resizeListener);
            }
        },
        unbindResizeListener() {
            if (this.resizeListener) {
                window.removeEventListener('resize', this.resizeListener);
                this.resizeListener = null;
            }
        },
        isItemMatched(processedItem) {
            return this.isValidItem(processedItem) && this.getProccessedItemLabel(processedItem).toLocaleLowerCase().startsWith(this.searchValue.toLocaleLowerCase());
        },
        isValidItem(processedItem) {
            return !!processedItem && !this.isItemDisabled(processedItem.item) && !this.isItemSeparator(processedItem.item);
        },
        isValidSelectedItem(processedItem) {
            return this.isValidItem(processedItem) && this.isSelected(processedItem);
        },
        isSelected(processedItem) {
            return ObjectUtils.isNotEmpty(this.activeItem) ? this.activeItem.key === processedItem.key : false;
        },
        findFirstItemIndex() {
            return this.visibleItems.findIndex((processedItem) => this.isValidItem(processedItem));
        },
        findLastItemIndex() {
            return ObjectUtils.findLastIndex(this.visibleItems, (processedItem) => this.isValidItem(processedItem));
        },
        findNextItemIndex(index) {
            const matchedItemIndex = index < this.visibleItems.length - 1 ? this.visibleItems.slice(index + 1).findIndex((processedItem) => this.isValidItem(processedItem)) : -1;

            return matchedItemIndex > -1 ? matchedItemIndex + index + 1 : index;
        },
        findPrevItemIndex(index) {
            const matchedItemIndex = index > 0 ? ObjectUtils.findLastIndex(this.visibleItems.slice(0, index), (processedItem) => this.isValidItem(processedItem)) : -1;

            return matchedItemIndex > -1 ? matchedItemIndex : index;
        },
        findSelectedItemIndex() {
            return this.visibleItems.findIndex((processedItem) => this.isValidSelectedItem(processedItem));
        },
        findFirstFocusedItemIndex() {
            const selectedIndex = this.findSelectedItemIndex();

            return selectedIndex < 0 ? this.findFirstItemIndex() : selectedIndex;
        },
        findLastFocusedItemIndex() {
            const selectedIndex = this.findSelectedItemIndex();

            return selectedIndex < 0 ? this.findLastItemIndex() : selectedIndex;
        },
        findVisibleItem(index) {
            return ObjectUtils.isNotEmpty(this.visibleItems) ? this.visibleItems[index] : null;
        },
        searchItems(event, char) {
            this.searchValue = (this.searchValue || '') + char;

            let itemIndex = -1;
            let matched = false;

            if (this.focusedItemInfo.index !== -1) {
                itemIndex = this.visibleItems.slice(this.focusedItemInfo.index).findIndex((processedItem) => this.isItemMatched(processedItem));
                itemIndex = itemIndex === -1 ? this.visibleItems.slice(0, this.focusedItemInfo.index).findIndex((processedItem) => this.isItemMatched(processedItem)) : itemIndex + this.focusedItemInfo.index;
            } else {
                itemIndex = this.visibleItems.findIndex((processedItem) => this.isItemMatched(processedItem));
            }

            if (itemIndex !== -1) {
                matched = true;
            }

            if (itemIndex === -1 && this.focusedItemInfo.index === -1) {
                itemIndex = this.findFirstFocusedItemIndex();
            }

            if (itemIndex !== -1) {
                this.changeFocusedItemInfo(event, itemIndex);
            }

            if (this.searchTimeout) {
                clearTimeout(this.searchTimeout);
            }

            this.searchTimeout = setTimeout(() => {
                this.searchValue = '';
                this.searchTimeout = null;
            }, 500);

            return matched;
        },
        changeFocusedItemInfo(event, index) {
            const processedItem = this.findVisibleItem(index);

            this.focusedItemInfo.index = index;
            this.focusedItemInfo.key = ObjectUtils.isNotEmpty(processedItem) ? processedItem.key : '';
            this.scrollInView();
        },
        scrollInView(index = -1) {
            const id = index !== -1 ? `${this.id}_${index}` : this.focusedItemId;
            const element = DomHandler.findSingle(this.menubar, `li[id="${id}"]`);

            if (element) {
                element.scrollIntoView && element.scrollIntoView({ block: 'nearest', inline: 'start' });
            }
        },
        createProcessedItems(items, level = 0, parent = {}, parentKey = '', columnIndex) {
            const processedItems = [];

            items &&
                items.forEach((item, index) => {
                    const key = (parentKey !== '' ? parentKey + '_' : '') + (columnIndex !== undefined ? columnIndex + '_' : '') + index;
                    const newItem = {
                        item,
                        index,
                        level,
                        key,
                        parent,
                        parentKey,
                        columnIndex: columnIndex !== undefined ? columnIndex : parent.columnIndex !== undefined ? parent.columnIndex : index
                    };

                    newItem['items'] =
                        level === 0 && item.items && item.items.length > 0 ? item.items.map((_items, _index) => this.createProcessedItems(_items, level + 1, newItem, key, _index)) : this.createProcessedItems(item.items, level + 1, newItem, key);
                    processedItems.push(newItem);
                });

            return processedItems;
        },
        containerRef(el) {
            this.container = el;
        },
        menubarRef(el) {
            this.menubar = el ? el.$el : undefined;
        }
    },
    computed: {
        processedItems() {
            return this.createProcessedItems(this.model || []);
        },
        visibleItems() {
            const processedItem = ObjectUtils.isNotEmpty(this.activeItem) ? this.activeItem : null;

            return processedItem && processedItem.key === this.focusedItemInfo.parentKey
                ? processedItem.items.reduce((items, col) => {
                      col.forEach((submenu) => {
                          submenu.items.forEach((a) => {
                              items.push(a);
                          });
                      });

                      return items;
                  }, [])
                : this.processedItems;
        },
        horizontal() {
            return this.orientation === 'horizontal';
        },
        vertical() {
            return this.orientation === 'vertical';
        },
        focusedItemId() {
            return ObjectUtils.isNotEmpty(this.focusedItemInfo.key) ? `${this.id}_${this.focusedItemInfo.key}` : null;
        }
    },
    components: {
        MegaMenuSub: MegaMenuSub
    }
};
</script>
