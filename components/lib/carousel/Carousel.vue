<template>
    <div :class="cx('root')" role="region" v-bind="ptm('root')">
        <div v-if="$slots.header" :class="cx('header')" v-bind="ptm('header')">
            <slot name="header"></slot>
        </div>
        <div :class="[cx('content'), contentClass]" v-bind="ptm('content')">
            <div :class="[cx('container'), containerClass]" :aria-live="allowAutoplay ? 'polite' : 'off'" v-bind="ptm('container')">
                <button v-if="showNavigators" v-ripple type="button" :class="cx('previousButton')" :disabled="backwardIsDisabled" :aria-label="ariaPrevButtonLabel" @click="navBackward" v-bind="{ ...prevButtonProps, ...ptm('previousButton') }">
                    <slot name="previousicon">
                        <component :is="isVertical() ? 'ChevronUpIcon' : 'ChevronLeftIcon'" :class="cx('previousButtonIcon')" v-bind="ptm('previousButtonIcon')" />
                    </slot>
                </button>

                <div :class="cx('itemsContent')" :style="[{ height: isVertical() ? verticalViewPortHeight : 'auto' }]" @touchend="onTouchEnd" @touchstart="onTouchStart" @touchmove="onTouchMove" v-bind="ptm('itemsContent')">
                    <div ref="itemsContainer" :class="cx('itemsContainer')" @transitionend="onTransitionEnd" v-bind="ptm('itemsContainer')">
                        <template v-if="isCircular()">
                            <div
                                v-for="(item, index) of value.slice(-1 * d_numVisible)"
                                :key="index + '_scloned'"
                                :class="cx('itemCloned', { index, value, totalShiftedItems, d_numVisible })"
                                v-bind="ptm('itemCloned')"
                                :data-p-carousel-item-active="totalShiftedItems * -1 === value.length + d_numVisible"
                                :data-p-carousel-item-start="index === 0"
                                :data-p-carousel-item-end="value.slice(-1 * d_numVisible).length - 1 === index"
                            >
                                <slot name="item" :data="item" :index="index"></slot>
                            </div>
                        </template>
                        <div
                            v-for="(item, index) of value"
                            :key="index"
                            :class="cx('item', { index })"
                            role="group"
                            :aria-hidden="firstIndex() > index || lastIndex() < index ? true : undefined"
                            :aria-label="ariaSlideNumber(index)"
                            :aria-roledescription="ariaSlideLabel"
                            v-bind="ptm('item')"
                            :data-p-carousel-item-active="firstIndex() <= index && lastIndex() >= index"
                            :data-p-carousel-item-start="firstIndex() === index"
                            :data-p-carousel-item-end="lastIndex() === index"
                        >
                            <slot name="item" :data="item" :index="index"></slot>
                        </div>
                        <template v-if="isCircular()">
                            <div v-for="(item, index) of value.slice(0, d_numVisible)" :key="index + '_fcloned'" :class="cx('itemCloned', { index, value, totalShiftedItems, d_numVisible })" v-bind="ptm('itemCloned')">
                                <slot name="item" :data="item" :index="index"></slot>
                            </div>
                        </template>
                    </div>
                </div>

                <button v-if="showNavigators" v-ripple type="button" :class="cx('nextButton')" :disabled="forwardIsDisabled" :aria-label="ariaNextButtonLabel" @click="navForward" v-bind="{ ...nextButtonProps, ...ptm('nextButton') }">
                    <slot name="nexticon">
                        <component :is="isVertical() ? 'ChevronDownIcon' : 'ChevronRightIcon'" :class="cx('nextButtonIcon')" v-bind="ptm('nextButtonIcon')" />
                    </slot>
                </button>
            </div>
            <ul v-if="totalIndicators >= 0 && showIndicators" ref="indicatorContent" :class="[cx('indicators'), indicatorsContentClass]" @keydown="onIndicatorKeydown" v-bind="ptm('indicators')">
                <li v-for="(indicator, i) of totalIndicators" :key="'p-carousel-indicator-' + i.toString()" :class="cx('indicator', { index: i })" v-bind="ptm('indicator', getIndicatorPTOptions(i))" :data-p-highlight="d_page === i">
                    <button
                        :class="cx('indicatorButton')"
                        type="button"
                        :tabindex="d_page === i ? '0' : '-1'"
                        :aria-label="ariaPageLabel(i + 1)"
                        :aria-current="d_page === i ? 'page' : undefined"
                        @click="onIndicatorClick($event, i)"
                        v-bind="ptm('indicatorButton', getIndicatorPTOptions(i))"
                    />
                </li>
            </ul>
        </div>
        <div v-if="$slots.footer" :class="cx('footer')" v-bind="ptm('footer')">
            <slot name="footer"></slot>
        </div>
    </div>
</template>

<script>
import ChevronDownIcon from 'primevue/icons/chevrondown';
import ChevronLeftIcon from 'primevue/icons/chevronleft';
import ChevronRightIcon from 'primevue/icons/chevronright';
import ChevronUpIcon from 'primevue/icons/chevronup';
import Ripple from 'primevue/ripple';
import { DomHandler, UniqueComponentId } from 'primevue/utils';
import BaseCarousel from './BaseCarousel.vue';

export default {
    name: 'Carousel',
    extends: BaseCarousel,
    emits: ['update:page'],
    isRemainingItemsAdded: false,
    data() {
        return {
            remainingItems: 0,
            d_numVisible: this.numVisible,
            d_numScroll: this.numScroll,
            d_oldNumScroll: 0,
            d_oldNumVisible: 0,
            d_oldValue: null,
            d_page: this.page,
            totalShiftedItems: this.page * this.numScroll * -1,
            allowAutoplay: !!this.autoplayInterval,
            d_circular: this.circular || this.allowAutoplay,
            swipeThreshold: 20
        };
    },
    watch: {
        page(newValue) {
            this.d_page = newValue;
        },
        circular(newValue) {
            this.d_circular = newValue;
        },
        numVisible(newValue, oldValue) {
            this.d_numVisible = newValue;
            this.d_oldNumVisible = oldValue;
        },
        numScroll(newValue, oldValue) {
            this.d_oldNumScroll = oldValue;
            this.d_numScroll = newValue;
        },
        value(oldValue) {
            this.d_oldValue = oldValue;
        }
    },
    mounted() {
        let stateChanged = false;

        this.$el.setAttribute(this.attributeSelector, '');
        this.createStyle();
        this.calculatePosition();

        if (this.responsiveOptions) {
            this.bindDocumentListeners();
        }

        if (this.isCircular()) {
            let totalShiftedItems = this.totalShiftedItems;

            if (this.d_page === 0) {
                totalShiftedItems = -1 * this.d_numVisible;
            } else if (totalShiftedItems === 0) {
                totalShiftedItems = -1 * this.value.length;

                if (this.remainingItems > 0) {
                    this.isRemainingItemsAdded = true;
                }
            }

            if (totalShiftedItems !== this.totalShiftedItems) {
                this.totalShiftedItems = totalShiftedItems;

                stateChanged = true;
            }
        }

        if (!stateChanged && this.isAutoplay()) {
            this.startAutoplay();
        }
    },
    updated() {
        const isCircular = this.isCircular();
        let stateChanged = false;
        let totalShiftedItems = this.totalShiftedItems;

        if (this.autoplayInterval) {
            this.stopAutoplay();
        }

        if (this.d_oldNumScroll !== this.d_numScroll || this.d_oldNumVisible !== this.d_numVisible || this.d_oldValue.length !== this.value.length) {
            this.remainingItems = (this.value.length - this.d_numVisible) % this.d_numScroll;

            let page = this.d_page;

            if (this.totalIndicators !== 0 && page >= this.totalIndicators) {
                page = this.totalIndicators - 1;

                this.$emit('update:page', page);
                this.d_page = page;

                stateChanged = true;
            }

            totalShiftedItems = page * this.d_numScroll * -1;

            if (isCircular) {
                totalShiftedItems -= this.d_numVisible;
            }

            if (page === this.totalIndicators - 1 && this.remainingItems > 0) {
                totalShiftedItems += -1 * this.remainingItems + this.d_numScroll;
                this.isRemainingItemsAdded = true;
            } else {
                this.isRemainingItemsAdded = false;
            }

            if (totalShiftedItems !== this.totalShiftedItems) {
                this.totalShiftedItems = totalShiftedItems;

                stateChanged = true;
            }

            this.d_oldNumScroll = this.d_numScroll;
            this.d_oldNumVisible = this.d_numVisible;
            this.d_oldValue = this.value;

            this.$refs.itemsContainer.style.transform = this.isVertical() ? `translate3d(0, ${totalShiftedItems * (100 / this.d_numVisible)}%, 0)` : `translate3d(${totalShiftedItems * (100 / this.d_numVisible)}%, 0, 0)`;
        }

        if (isCircular) {
            if (this.d_page === 0) {
                totalShiftedItems = -1 * this.d_numVisible;
            } else if (totalShiftedItems === 0) {
                totalShiftedItems = -1 * this.value.length;

                if (this.remainingItems > 0) {
                    this.isRemainingItemsAdded = true;
                }
            }

            if (totalShiftedItems !== this.totalShiftedItems) {
                this.totalShiftedItems = totalShiftedItems;

                stateChanged = true;
            }
        }

        if (!stateChanged && this.isAutoplay()) {
            this.startAutoplay();
        }
    },
    beforeUnmount() {
        if (this.responsiveOptions) {
            this.unbindDocumentListeners();
        }

        if (this.autoplayInterval) {
            this.stopAutoplay();
        }
    },
    methods: {
        getIndicatorPTOptions(index) {
            return {
                context: {
                    highlighted: index === this.d_page
                }
            };
        },
        step(dir, page) {
            let totalShiftedItems = this.totalShiftedItems;
            const isCircular = this.isCircular();

            if (page != null) {
                totalShiftedItems = this.d_numScroll * page * -1;

                if (isCircular) {
                    totalShiftedItems -= this.d_numVisible;
                }

                this.isRemainingItemsAdded = false;
            } else {
                totalShiftedItems += this.d_numScroll * dir;

                if (this.isRemainingItemsAdded) {
                    totalShiftedItems += this.remainingItems - this.d_numScroll * dir;
                    this.isRemainingItemsAdded = false;
                }

                let originalShiftedItems = isCircular ? totalShiftedItems + this.d_numVisible : totalShiftedItems;

                page = Math.abs(Math.floor(originalShiftedItems / this.d_numScroll));
            }

            if (isCircular && this.d_page === this.totalIndicators - 1 && dir === -1) {
                totalShiftedItems = -1 * (this.value.length + this.d_numVisible);
                page = 0;
            } else if (isCircular && this.d_page === 0 && dir === 1) {
                totalShiftedItems = 0;
                page = this.totalIndicators - 1;
            } else if (page === this.totalIndicators - 1 && this.remainingItems > 0) {
                totalShiftedItems += this.remainingItems * -1 - this.d_numScroll * dir;
                this.isRemainingItemsAdded = true;
            }

            if (this.$refs.itemsContainer) {
                !this.isUnstyled && DomHandler.removeClass(this.$refs.itemsContainer, 'p-items-hidden');
                this.$refs.itemsContainer.style.transform = this.isVertical() ? `translate3d(0, ${totalShiftedItems * (100 / this.d_numVisible)}%, 0)` : `translate3d(${totalShiftedItems * (100 / this.d_numVisible)}%, 0, 0)`;
                this.$refs.itemsContainer.style.transition = 'transform 500ms ease 0s';
            }

            this.totalShiftedItems = totalShiftedItems;

            this.$emit('update:page', page);
            this.d_page = page;
        },
        calculatePosition() {
            if (this.$refs.itemsContainer && this.responsiveOptions) {
                let windowWidth = window.innerWidth;
                let matchedResponsiveOptionsData = {
                    numVisible: this.numVisible,
                    numScroll: this.numScroll
                };

                for (let i = 0; i < this.responsiveOptions.length; i++) {
                    let res = this.responsiveOptions[i];

                    if (parseInt(res.breakpoint, 10) >= windowWidth) {
                        matchedResponsiveOptionsData = res;
                    }
                }

                if (this.d_numScroll !== matchedResponsiveOptionsData.numScroll) {
                    let page = this.d_page;

                    page = parseInt((page * this.d_numScroll) / matchedResponsiveOptionsData.numScroll);

                    this.totalShiftedItems = matchedResponsiveOptionsData.numScroll * page * -1;

                    if (this.isCircular()) {
                        this.totalShiftedItems -= matchedResponsiveOptionsData.numVisible;
                    }

                    this.d_numScroll = matchedResponsiveOptionsData.numScroll;

                    this.$emit('update:page', page);
                    this.d_page = page;
                }

                if (this.d_numVisible !== matchedResponsiveOptionsData.numVisible) {
                    this.d_numVisible = matchedResponsiveOptionsData.numVisible;
                }
            }
        },
        navBackward(e, index) {
            if (this.d_circular || this.d_page !== 0) {
                this.step(1, index);
            }

            this.allowAutoplay = false;

            if (e.cancelable) {
                e.preventDefault();
            }
        },
        navForward(e, index) {
            if (this.d_circular || this.d_page < this.totalIndicators - 1) {
                this.step(-1, index);
            }

            this.allowAutoplay = false;

            if (e.cancelable) {
                e.preventDefault();
            }
        },
        onIndicatorClick(e, index) {
            let page = this.d_page;

            if (index > page) {
                this.navForward(e, index);
            } else if (index < page) {
                this.navBackward(e, index);
            }
        },
        onTransitionEnd() {
            if (this.$refs.itemsContainer) {
                !this.isUnstyled && DomHandler.addClass(this.$refs.itemsContainer, 'p-items-hidden');
                this.$refs.itemsContainer.style.transition = '';

                if ((this.d_page === 0 || this.d_page === this.totalIndicators - 1) && this.isCircular()) {
                    this.$refs.itemsContainer.style.transform = this.isVertical() ? `translate3d(0, ${this.totalShiftedItems * (100 / this.d_numVisible)}%, 0)` : `translate3d(${this.totalShiftedItems * (100 / this.d_numVisible)}%, 0, 0)`;
                }
            }
        },
        onTouchStart(e) {
            let touchobj = e.changedTouches[0];

            this.startPos = {
                x: touchobj.pageX,
                y: touchobj.pageY
            };
        },
        onTouchMove(e) {
            if (e.cancelable) {
                e.preventDefault();
            }
        },
        onTouchEnd(e) {
            let touchobj = e.changedTouches[0];

            if (this.isVertical()) {
                this.changePageOnTouch(e, touchobj.pageY - this.startPos.y);
            } else {
                this.changePageOnTouch(e, touchobj.pageX - this.startPos.x);
            }
        },
        changePageOnTouch(e, diff) {
            if (Math.abs(diff) > this.swipeThreshold) {
                if (diff < 0) {
                    // left
                    this.navForward(e);
                } else {
                    // right
                    this.navBackward(e);
                }
            }
        },
        onIndicatorKeydown(event) {
            switch (event.code) {
                case 'ArrowRight':
                    this.onRightKey();
                    break;

                case 'ArrowLeft':
                    this.onLeftKey();
                    break;

                case 'Home':
                    this.onHomeKey();
                    event.preventDefault();
                    break;

                case 'End':
                    this.onEndKey();
                    event.preventDefault();
                    break;

                case 'ArrowUp':
                case 'ArrowDown':
                    event.preventDefault();
                    break;

                case 'Tab':
                    this.onTabKey();
                    break;

                default:
                    break;
            }
        },
        onRightKey() {
            const indicators = [...DomHandler.find(this.$refs.indicatorContent, '[data-pc-section="indicator"]')];
            const activeIndex = this.findFocusedIndicatorIndex();

            this.changedFocusedIndicator(activeIndex, activeIndex + 1 === indicators.length ? indicators.length - 1 : activeIndex + 1);
        },
        onLeftKey() {
            const activeIndex = this.findFocusedIndicatorIndex();

            this.changedFocusedIndicator(activeIndex, activeIndex - 1 <= 0 ? 0 : activeIndex - 1);
        },
        onHomeKey() {
            const activeIndex = this.findFocusedIndicatorIndex();

            this.changedFocusedIndicator(activeIndex, 0);
        },
        onEndKey() {
            const indicators = [...DomHandler.find(this.$refs.indicatorContent, '[data-pc-section="indicator"]r')];
            const activeIndex = this.findFocusedIndicatorIndex();

            this.changedFocusedIndicator(activeIndex, indicators.length - 1);
        },
        onTabKey() {
            const indicators = [...DomHandler.find(this.$refs.indicatorContent, '[data-pc-section="indicator"]')];
            const highlightedIndex = indicators.findIndex((ind) => DomHandler.getAttribute(ind, 'data-p-highlight') === true);

            const activeIndicator = DomHandler.findSingle(this.$refs.indicatorContent, '[data-pc-section="indicator"] > button[tabindex="0"]');
            const activeIndex = indicators.findIndex((ind) => ind === activeIndicator.parentElement);

            indicators[activeIndex].children[0].tabIndex = '-1';
            indicators[highlightedIndex].children[0].tabIndex = '0';
        },
        findFocusedIndicatorIndex() {
            const indicators = [...DomHandler.find(this.$refs.indicatorContent, '[data-pc-section="indicator"]')];
            const activeIndicator = DomHandler.findSingle(this.$refs.indicatorContent, '[data-pc-section="indicator"] > button[tabindex="0"]');

            return indicators.findIndex((ind) => ind === activeIndicator.parentElement);
        },
        changedFocusedIndicator(prevInd, nextInd) {
            const indicators = [...DomHandler.find(this.$refs.indicatorContent, '[data-pc-section="indicator"]')];

            indicators[prevInd].children[0].tabIndex = '-1';
            indicators[nextInd].children[0].tabIndex = '0';
            indicators[nextInd].children[0].focus();
        },
        bindDocumentListeners() {
            if (!this.documentResizeListener) {
                this.documentResizeListener = (e) => {
                    this.calculatePosition(e);
                };

                window.addEventListener('resize', this.documentResizeListener);
            }
        },
        unbindDocumentListeners() {
            if (this.documentResizeListener) {
                window.removeEventListener('resize', this.documentResizeListener);
                this.documentResizeListener = null;
            }
        },
        startAutoplay() {
            this.interval = setInterval(() => {
                if (this.d_page === this.totalIndicators - 1) {
                    this.step(-1, 0);
                } else {
                    this.step(-1, this.d_page + 1);
                }
            }, this.autoplayInterval);
        },
        stopAutoplay() {
            if (this.interval) {
                clearInterval(this.interval);
            }
        },
        createStyle() {
            if (!this.carouselStyle) {
                this.carouselStyle = document.createElement('style');
                this.carouselStyle.type = 'text/css';
                DomHandler.setAttribute(this.carouselStyle, 'nonce', this.$primevue?.config?.csp?.nonce);
                document.body.appendChild(this.carouselStyle);
            }

            let innerHTML = `
                .p-carousel[${this.attributeSelector}] .p-carousel-item {
                    flex: 1 0 ${100 / this.d_numVisible}%
                }
            `;

            if (this.responsiveOptions && !this.isUnstyled) {
                let _responsiveOptions = [...this.responsiveOptions];

                _responsiveOptions.sort((data1, data2) => {
                    const value1 = data1.breakpoint;
                    const value2 = data2.breakpoint;
                    let result = null;

                    if (value1 == null && value2 != null) result = -1;
                    else if (value1 != null && value2 == null) result = 1;
                    else if (value1 == null && value2 == null) result = 0;
                    else if (typeof value1 === 'string' && typeof value2 === 'string') result = value1.localeCompare(value2, undefined, { numeric: true });
                    else result = value1 < value2 ? -1 : value1 > value2 ? 1 : 0;

                    return -1 * result;
                });

                for (let i = 0; i < _responsiveOptions.length; i++) {
                    let res = _responsiveOptions[i];

                    innerHTML += `
                        @media screen and (max-width: ${res.breakpoint}) {
                            .p-carousel[${this.attributeSelector}] .p-carousel-item {
                                flex: 1 0 ${100 / res.numVisible}%
                            }
                        }
                    `;
                }
            }

            this.carouselStyle.innerHTML = innerHTML;
        },
        isVertical() {
            return this.orientation === 'vertical';
        },
        isCircular() {
            return this.value && this.d_circular && this.value.length >= this.d_numVisible;
        },
        isAutoplay() {
            return this.autoplayInterval && this.allowAutoplay;
        },
        firstIndex() {
            return this.isCircular() ? -1 * (this.totalShiftedItems + this.d_numVisible) : this.totalShiftedItems * -1;
        },
        lastIndex() {
            return this.firstIndex() + this.d_numVisible - 1;
        },
        ariaSlideNumber(value) {
            return this.$primevue.config.locale.aria ? this.$primevue.config.locale.aria.slideNumber.replace(/{slideNumber}/g, value) : undefined;
        },
        ariaPageLabel(value) {
            return this.$primevue.config.locale.aria ? this.$primevue.config.locale.aria.pageLabel.replace(/{page}/g, value) : undefined;
        }
    },
    computed: {
        totalIndicators() {
            return this.value ? Math.max(Math.ceil((this.value.length - this.d_numVisible) / this.d_numScroll) + 1, 0) : 0;
        },
        backwardIsDisabled() {
            return this.value && (!this.circular || this.value.length < this.d_numVisible) && this.d_page === 0;
        },
        forwardIsDisabled() {
            return this.value && (!this.circular || this.value.length < this.d_numVisible) && (this.d_page === this.totalIndicators - 1 || this.totalIndicators === 0);
        },
        ariaSlideLabel() {
            return this.$primevue.config.locale.aria ? this.$primevue.config.locale.aria.slide : undefined;
        },
        ariaPrevButtonLabel() {
            return this.$primevue.config.locale.aria ? this.$primevue.config.locale.aria.prevPageLabel : undefined;
        },
        ariaNextButtonLabel() {
            return this.$primevue.config.locale.aria ? this.$primevue.config.locale.aria.nextPageLabel : undefined;
        },
        attributeSelector() {
            return UniqueComponentId();
        }
    },
    components: {
        ChevronRightIcon,
        ChevronDownIcon,
        ChevronLeftIcon,
        ChevronUpIcon
    },
    directives: {
        ripple: Ripple
    }
};
</script>
