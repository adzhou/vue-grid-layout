<template>
    <!--<pre>{{$data|json}}</pre>
    <br/>
    <br/>-->
    <div>
        <slot name="top"></slot>
        <div ref="item" class="vue-grid-layout" :style="mergedStyle">

            <!--<grid-item class="vue-grid-placeholder"
                       v-if="isDragging"
                       :x="placeholder.x"
                       :y="placeholder.y"
                       :w="placeholder.w"
                       :h="placeholder.h"
                       :i="placeholder.i"></grid-item>-->
        </div>
    </div>
</template>
<style>
    .vue-grid-layout {
        position: relative;
        transition: height 200ms ease;
    }
</style>
<script>
    var elementResizeDetectorMaker = require("element-resize-detector");

    import {bottom, compact, getLayoutItem, moveElement, validateLayout} from './utils';
    import GridItem from './GridItem.vue'

    export default {
        name: "GridLayout",
        components: {
            GridItem,
        },
        props: {
            // If true, the container height swells and contracts to fit contents
            autoSize: {
                type: Boolean,
                default: true
            },
            colNum: {
                type: Number,
                default: 12
            },
            rowHeight: {
                type: Number,
                default: 150
            },
            maxRows: {
                type: Number,
                default: Infinity
            },
            margin: {
                type: Array,
                default: function () {
                    return [10, 10];
                }
            },
            isDraggable: {
                type: Boolean,
                default: true
            },
            isResizable: {
                type: Boolean,
                default: true
            },
            useCssTransforms: {
                type: Boolean,
                default: true
            },
            verticalCompact: {
                type: Boolean,
                default: true
            },

            layout: {
                type: Array,
                required: true,
            },
        },
        data: function () {
            return {
                width: null,
                mergedStyle: {},
                lastLayoutLength: 0,
                isDragging: false,
                placeholder: {
                    x: 0,
                    y: 0,
                    w: 0,
                    h: 0,
                    i: 0
                },
            };
        },
        mounted: function() {
//            eventHub.$on("dragEvent", this.dragEvent);
//            eventHub.$on("resizeEvent", this.resizeEvent);
            validateLayout(this.layout);
            this.$nextTick(function() {
                var self = this;
                this.$nextTick(function () {
                    if (self.width === null) {
                        self.onWindowResize();
                        //self.width = self.$el.offsetWidth;
                        window.addEventListener('resize', self.onWindowResize);
                    }
                    compact(self.layout, self.verticalCompact);

                    self.updateHeight();
                    self.$nextTick(function () {
                        var erd = elementResizeDetectorMaker({
                            strategy: "scroll" //<- For ultra performance.
                        });
                        erd.listenTo(self.$refs.item, function (element) {
                            self.onWindowResize();
                        });
                    });
                });
                window.onload = function () {
                    if (self.width === null) {
                        self.onWindowResize();
                        //self.width = self.$el.offsetWidth;
                        window.addEventListener('resize', self.onWindowResize);
                    }
                    compact(self.layout, self.verticalCompact);

                    self.updateHeight();
                    self.$nextTick(function () {
                        var erd = elementResizeDetectorMaker({
                            strategy: "scroll" //<- For ultra performance.
                        });
                        erd.listenTo(self.$refs.item, function (element) {
                            self.onWindowResize();
                        });
                    });

                };
            });
        },
        computed: {
            layoutSize: function() {
                return this.layout.length;
            }
        },
        watch: {
            width: function () {
                var self = this;
                this.$nextTick(function () {
                    //eventHub.$emit("updateWidth", this.width);
                    this.$children.forEach(function(child) {
                        child.updateWidth(self.width);
                    });
                    self.updateHeight();
                });
            },
            layoutSize: function () {
                console.log("### layout changed! " + self.layout.length);
            },
            layout: function () {
                if (this.layout !== undefined && this.layout.length !== this.lastLayoutLength) {
                    this.lastLayoutLength = this.layout.length;
                    compact(this.layout, this.verticalCompact);

//                    eventHub.$emit("updateWidth", this.width);
                    this.$children.forEach(function(child) {
                        child.updateWidth(self.width);
                    });
                    this.updateHeight();
                }
            },
            deep: true
        },
        methods: {
            updateHeight: function () {
                this.mergedStyle = {
                    height: this.containerHeight()
                };
            },
            onWindowResize: function () {
                if (this.$refs !== null && this.$refs.item !== null) {
                    this.width = this.$refs.item.offsetWidth;
                }
            },
            containerHeight: function () {
                if (!this.autoSize) return;
                return bottom(this.layout) * (this.rowHeight + this.margin[1]) + this.margin[1] + 'px';
            },
            dragEvent: function (eventName, id, x, y, w, h) {
                var self = this;
                if (eventName == "dragmove" || eventName == "dragstart") {
                    this.isDragging = true;
                    this.placeholder.i = id;
                    this.placeholder.x = x;
                    this.placeholder.y = y;
                    this.placeholder.w = w;
                    this.placeholder.h = h;
                    //eventHub.$emit("updateWidth", this.width);
                    this.$children.forEach(function(child) {
                        child.updateWidth(self.width);
                    });

                } else {
                    this.isDragging = false;
                }
                //console.log(eventName + " id=" + id + ", x=" + x + ", y=" + y);
                var l = getLayoutItem(this.layout, id);
                // Move the element to the dragged location.
                this.layout = moveElement(this.layout, l, x, y, true);
                compact(this.layout, this.verticalCompact);
                // needed because vue can't detect changes on array element properties
                //eventHub.$emit("compact", this.layout);
                var self = this;
                this.$children.forEach(function(child) {
                    child.compact(self.layout);
                });

                this.updateHeight();
            },
            resizeEvent: function (eventName, id, x, y, h, w) {
                var self = this;
                if (eventName == "resizestart" || eventName == "resizemove") {
                    this.isDragging = true;
                    this.placeholder.i = id;
                    this.placeholder.x = x;
                    this.placeholder.y = y;
                    this.placeholder.w = w;
                    this.placeholder.h = h;
                    //eventHub.$emit("updateWidth", this.width);
                    this.$children.forEach(function(child) {
                        child.updateWidth(self.width);
                    });
                } else {
                    this.isDragging = false;
                }
                compact(this.layout, this.verticalCompact);
//                eventHub.$emit("compact", this.layout);
                var self = this;
                this.$children.forEach(function(child) {
                    child.compact(self.layout);
                });

                this.updateHeight();
            },
        },
/*
        events: {
            dragEvent: function (eventName, id, x, y, w, h) {
                if (eventName == "dragmove" || eventName == "dragstart") {
                    this.isDragging = true;
                    this.placeholder.i = id;
                    this.placeholder.x = x;
                    this.placeholder.y = y;
                    this.placeholder.w = w;
                    this.placeholder.h = h;
                    //eventHub.$emit("updateWidth", this.width);
                    this.$children.forEach(function(child) {
                        child.updateWidth(self.width);
                    });

                } else {
                    this.isDragging = false;
                }
                //console.log(eventName + " id=" + id + ", x=" + x + ", y=" + y);
                var l = getLayoutItem(this.layout, id);
                // Move the element to the dragged location.
                this.layout = moveElement(this.layout, l, x, y, true);
                compact(this.layout, this.verticalCompact);
                // needed because vue can't detect changes on array element properties
                //eventHub.$emit("compact", this.layout);
                var self = this;
                this.$children.forEach(function(child) {
                    child.compact(self.layout);
                });

                this.updateHeight();
            },
            resizeEvent: function (eventName, id, x, y, h, w) {
                if (eventName == "resizestart" || eventName == "resizemove") {
                    this.isDragging = true;
                    this.placeholder.i = id;
                    this.placeholder.x = x;
                    this.placeholder.y = y;
                    this.placeholder.w = w;
                    this.placeholder.h = h;
                    //eventHub.$emit("updateWidth", this.width);
                    this.$children.forEach(function(child) {
                        child.updateWidth(self.width);
                    });
                } else {
                    this.isDragging = false;
                }
                compact(this.layout, this.verticalCompact);
//                eventHub.$emit("compact", this.layout);
                var self = this;
                this.$children.forEach(function(child) {
                    child.compact(self.layout);
                });

                this.updateHeight();
            },
        }
*/
    }
</script>