<template>
    <u-card :label="$t('UAnalytics.dashboard.label')">
        <div class="layout-grid" ref="LayoutGrid">
            <grid-layout
                v-if="layout.length > 0"
                :layout="layout"
                :row-height="rowHeight"
                :margin="margin"
                :is-draggable="editable"
                :is-resizable="editable"
                @layout-updated="(n) => onLayoutUpdated(n)">

                <grid-item
                    v-for="(l, idx) of layout"
                    :x="l.x"
                    :y="l.y"
                    :w="l.w"
                    :h="l.h"
                    :i="l.i"
                    :min-w="minW"
                    @resize="(i, h, w) => onResize(i, h, w)"
                    @move="(i, x, y) => onMove(i, x, y)"
                    @moved="(i, x, y) => onMoved(i, x, y)"
                    @resized="(i, h, w, hpx, wpx) => onResized(i, h, w, hpx, wpx)"
                    drag-allow-from=".layout-grid-item-header-title"
                    drag-ignore-from=".layout-grid-item-content"
                    :key="l.i">

                    <div class="layout-grid-item" :class="{ 'layout-grid-item-border': editable }">
                        <div class="layout-grid-item-header">
                            <div class="level is-mobile">

                                <div class="level-left">
                                    <div class="level-item">
                                        <div class="layout-grid-item-header-title" :style="paddingLeft">
                                            {{ l.title || null }}
                                        </div>
                                    </div>
                                </div>

                                <div class="level-right" v-if="editable">
                                    <div class="level-item">
                                    <span v-if="!isExpanded(l.i)"
                                          class="icon">
                                        <i class="mdi mdi-arrow-expand mdi-18px"
                                           @click.stop="EXPAND_LAYOUT_ITEM(idx)"></i>
                                    </span>

                                        <span v-else class="icon">
                                        <i class="mdi mdi-arrow-collapse mdi-18px"
                                           @click.stop="COLLAPSE_LAYOUT_ITEM(l.i)"></i>
                                    </span>
                                    </div>

                                    <div class="level-item">
                                    <span class="icon">
                                        <i class="mdi mdi-pencil mdi-18px" @click.stop="onEdit(l.i)"></i>
                                    </span>
                                    </div>

                                    <div class="level-item">
                                    <span class="icon">
                                        <i class="mdi mdi-close-outline mdi-18px"
                                           @click.stop="DELETE_LAYOUT_ITEM(idx);"></i>
                                    </span>
                                    </div>
                                </div>
                            </div>
                        </div>

                        <div class="layout-grid-item-content" :style="{ height : `${l.h * rowHeight - 30}px` }">
                            <component
                                v-show="true"
                                :ref="`LayoutGridItem${l.i}`"
                                :is="canRender(l) ? l.is : 'emotion'"
                                v-bind="canRender(l) ? l.data : null">
                            </component>

                            <Table v-show="false" :data="l.data.data" v-if="Array.isArray(l.data.data)"></Table>
                        </div>

                        <span class="icon" style="position: absolute; left: 0; bottom: 0;"
                              v-if="Array.isArray(l.data.data) && editable">
                              <i class="mdi mdi-arrow-up-drop-circle-outline mdi-18px"
                                 @click.stop="(ev) => toggle(ev, l.i)">
                              </i>
                        </span>
                    </div>
                </grid-item>
            </grid-layout>

            <div class="layout-grid-empty" v-else>{{ $t('UAnalytics.dashboard.noChart') }}</div>
        </div>
    </u-card>
</template>

<script>
    /* eslint-disable */
    import { mapMutations, mapState } from 'vuex';
    import Emotion from './Emotion';
    import Table from './Table';
    import GridItem from '../GridItem';
    import GridLayout from '../GridLayout';
    import UCard from '../UCard';

    const isObject            = o   => String(o) === '[object Object]';
    const isArrayAndHasLength = arr => Array.isArray(arr) && arr.length > 0;
    const isEmpty             = s   => s === '' || s === undefined || s === null;
    const isFunction          = f   => typeof f === 'function';

    const rootEmitter         = (vm, event, payload)  => {
        if (vm.$root !== vm) {
            vm.$root.$emit(event, payload)
        }
    };

    const isDisplay           = el => el.style.display === '' || el.style.display === 'block';

    const toggleVisibility    = el => {
        const DISPLAY = 'block',
              NONE    = 'none';

        el.style.display = isDisplay(el)
            ? NONE
            : DISPLAY;
    };

    const toggleVisibilityBy   = (el, ele) => {
        const DISPLAY = 'block',
              NONE    = 'none';

        el.style.display = isDisplay(ele)
            ? NONE
            : DISPLAY;
    };

    const isGeoJsonFeatureCollectionAndHasFeatures  =  (data) => {
        return isObject(data) && (data['type'] === 'FeatureCollection') && isArrayAndHasLength(data['features']) ;
    };

    const rotateIcon = ev => {
        const icon = ev.target,
            arrowUp = 'mdi-arrow-up-drop-circle-outline',
            arrowDown = 'mdi-arrow-down-drop-circle-outline';

        if (icon.classList.contains(arrowUp)) {
           icon.classList.remove(arrowUp);
           icon.classList.add(arrowDown);
        }

        else {
            icon.classList.remove(arrowDown);
            icon.classList.add(arrowUp);
        }
    };


    export default {
        name: 'layout-grid',
        props: {
            editable: {
                type: Boolean,
                default: false,
            },
            margin: {
                type: Array,
                default: () => [0, 0]
            },
            rowHeight: {
                type: Number,
                default: 100,
            },
            minW: {
                type: Number,
                default: 1
            }
        },
        methods: {
            ...mapMutations('LayoutGrid', [
                'EXPAND_LAYOUT_ITEM',
                'DELETE_LAYOUT_ITEM',
                'COLLAPSE_LAYOUT_ITEM',
                'DELETE_LAYOUT_ITEM_IN_CACHE'
            ]),
            isExpanded(i) {
               return this.layoutCache.findIndex(x => x.i === i) !== -1 ;
            },
            isIndoorMapComponent(vm) {
                const INDOOR_MAP_COMPONENTS = [
                    'd3-l-choropleth',
                    'd3-l-heat'
                ];

                return INDOOR_MAP_COMPONENTS.includes(vm.$options.name);
            },
            getComponentById(i) {
                const ref = `LayoutGridItem${i}`;
                const [component] = this.$refs[ref];

                return component;
            },
            getLayoutGridItem(ev) {
                return ev.target
                    .parentNode
                    .parentNode
                    .querySelector('.layout-grid-item-content')
                    .childNodes[0];
            },
            toggle(ev, i) {
                const el = this.getLayoutGridItem(ev),
                    table = el.nextSibling.nextSibling,
                    component = this.getComponentById(i);

                toggleVisibility(el);
                toggleVisibilityBy(table, el);

                rotateIcon(ev);

                if (isFunction(component.safeDraw) && isDisplay(el)) component.safeDraw();
            },
            onMove(i, x, y) {
                this.$emit('move', i, x, y);
            },
            onMoved(i, x, y) {
                this.$emit('moved', i, x, y);

                this.DELETE_LAYOUT_ITEM_IN_CACHE(i);
            },
            onResize(i, h, w) {
                this.$emit('resize', i, h, w);
            },
            onResized(i, h, w, hpx, wpx) {
                this.$emit('resized', i, h, w, hpx, wpx);

                this.DELETE_LAYOUT_ITEM_IN_CACHE(i);

                const component = this.getComponentById(i);
                if (isFunction(component.safeDraw) && isDisplay(component.$el)) this.$nextTick(() => component.safeDraw());
            },
            onLayoutUpdated(n) {
                this.$emit('updated', n);
            },
            onEdit(i) {
                this.$emit('edit', i);

                const event = 'layout-item-edit',
                    payload = { i, payload: null };

                rootEmitter(this, event, payload);
            },
            canRender(l) {
                if (!isObject(l.data) || isEmpty(l.is)) return false;

                switch (l.is) {
                    case 'd3-pie':
                    case 'd3-horizontal-bar':
                    case 'd3-vertical-bar':
                    case 'd3-line':
                    case 'd3-timeline':
                    case 'd3-timelion':
                    case 'd3-multi-line':
                    case 'd3-grouped-area':
                    case 'd3-word-cloud':
                    case 'd3-area':
                        return isArrayAndHasLength(l.data.data);
                    case 'd3-sankey-circular':
                        return isArrayAndHasLength(l.data.nodes) && isArrayAndHasLength(l.data.links);
                    case 'd3-l-heat':
                        return isArrayAndHasLength(l.data.data);
                    case 'd3-l-choropleth':
                        return isGeoJsonFeatureCollectionAndHasFeatures(l.data.data);
                    case 'd3-metric':
                    case 'd3-circle':
                        return !isEmpty(l.data.data);
                }
            }
        },
        computed: {
            ...mapState('LayoutGrid', [
                'layout',
                'layoutCache'
            ]),
            paddingLeft() {
                return {
                    left: this.editable ? '2px' : '16px'
                }
            }
        },
        components: {
            UCard,
            GridLayout,
            GridItem,
            Table,
            Emotion
        }
    }
</script>

<style scoped>
    .layout-grid-item {
        display: flex;
        display: -webkit-flex;
        flex-direction: column;
        justify-content: space-between;

        overflow: hidden;

        position: relative;
    }

    .layout-grid-item-border {
        /*border: 2px dashed #888;*/

        /*https://codepen.io/Hawkun/pen/rsIEp*/
        box-shadow: 2px 0 0 0 #e4e4e4,
        0 2px 0 0 #e4e4e4,
        2px 2px 0 0 #e4e4e4,
        2px 0 0 0 #e4e4e4 inset,
        0 2px 0 0 #e4e4e4 inset;
    }

    .layout-grid-item-header {
        height: 30px;
    }

    .layout-grid-item-header-title {
        font-weight: 600;
        font-family: inherit;
        font-size: 1.1rem;

        opacity: .5;

        word-break: break-all;
        word-wrap: break-word;

        position: relative;
    }

    .layout-grid-item-content {
        align-self: center;

        padding: 0px 18px 18px 18px;

        /*for emotion*/
        display: flex;
        display: -webkit-flex;
        justify-content: center;
        align-items: center;

        width: 100%;

        overflow-x: hidden;
        overflow-y: hidden;
    }

    .layout-grid-empty {
        width: 100%;
        height: 40px;
        border: 1px solid #dbdbdb;
        line-height: 40px;
        text-align: center;
    }

    .icon {
        transition: .3s opacity;
        opacity: .4;
        cursor: pointer;
    }

    .icon:hover {
        opacity: 1;
    }
</style>

<style>
    .layout-grid {
        background-color: #FFFFFF;
    }

    .vue-grid-item.vue-grid-placeholder {
        display: none;
    }

    .vue-grid-item > .vue-resizable-handle {
        background-position: unset;
    }
</style>
