<template>
    <div
        class="dnd-grid-container"
        :style="style"
    >
        <slot></slot>
        <box
            class="placeholder"
            boxId="::placeholder::"
        ></box>
    </div>
</template>

<style>
    .dnd-grid-container {
        position: relative;
        transition: min-width ease-out 0.1s, min-height ease-out 0.1s;
    }

    .dnd-grid-container .dnd-grid-box.placeholder {
        border: 1px dashed #000;
        background: none;
        z-index: 0;
    }

    .dnd-grid-container .dnd-grid-box.placeholder {
        transition: none !important;
    }
</style>

<script>
    import Box from './Box'
    import * as utils from './utils'

    export default {
        name: 'DndGridContainer',
        components: {
            Box
        },
        props: {
            layout: {
                type: Array,
                required: true
            },
            gridSize: {
                type: Object,
                default() {
                    return {
                        w: 100,
                        h: 100
                    }
                }
            },
            margin: {
                type: Number,
                default: 5
            },
            outerMargin: {
                type: Number,
                default: 0
            },
            bubbleUp: {
                type: Boolean,
                default: false
            },
            autoAddLayoutForNewBox: {
                type: Boolean,
                required: false,
                default: true
            },
            defaultSize: {
                type: Object,
                required: false,
                default() {
                    return {
                        w: 1,
                        h: 1
                    }
                }
            },
            fixLayoutOnLoad: {
                type: Boolean,
                required: false,
                default: true
            }
        },
        watch: {
            layout (newLayout) {
                if (this.fixLayoutOnLoad) {
                    if (utils.layoutHasCollisions(newLayout)) {
                        this.updateLayout(utils.fixLayout(newLayout, this.bubbleUp))
                    }
                }
            }
        },
        data () {
            return {
                placeholder: {
                    hidden: true,
                    position: {
                        x: 0,
                        y: 0,
                        w: 1,
                        h: 1
                    }
                },
                dragging: {
                    boxLayout: null,
                    offset: {
                        x: 0,
                        y: 0
                    }
                },
                resizing: {
                    boxLayout: null,
                    offset: {
                        x: 0,
                        y: 0
                    }
                },
                isMounted: false
            }
        },
        computed: {
            style () {
                var layoutSize = utils.getLayoutSize(this.layout)
                return {
                    minWidth: (
                        (layoutSize.w * this.gridSize.w) +
                        ((layoutSize.w - 1) * this.margin) +
                        (2 * this.outerMargin)
                    ) + 'px',
                    minHeight: (
                        (layoutSize.h * this.gridSize.h) +
                        ((layoutSize.h - 1) * this.margin) +
                        (2 * this.outerMargin)
                    ) + 'px'
                }
            },
            pinnedLayout () {
                return this.layout.filter((boxLayout) => {
                    return boxLayout.pinned
                })
            }
        },
        methods: {
            getBoxLayoutById (id) {
                if (id === '::placeholder::') {
                    return this.placeholder
                }
                return this.layout.find(box => {
                    return box.id === id
                })
            },
            getPixelPositionById (id) {
                if (this.dragging.boxLayout && this.dragging.boxLayout.id === id) {
                    let pixels = utils.positionToPixels(this.dragging.boxLayout.position, this.gridSize, this.margin, this.outerMargin)
                    pixels.x += this.dragging.offset.x
                    pixels.y += this.dragging.offset.y
                    return pixels
                }

                if (this.resizing.boxLayout && this.resizing.boxLayout.id === id) {
                    let pixels = utils.positionToPixels(this.resizing.boxLayout.position, this.gridSize, this.margin, this.outerMargin)
                    pixels.w += this.resizing.offset.x
                    pixels.h += this.resizing.offset.y
                    return pixels
                }

                var boxLayout = this.getBoxLayoutById(id)
                return utils.positionToPixels(boxLayout.position, this.gridSize, this.margin, this.outerMargin)
            },
            isBoxVisible (id) {
                var boxLayout = this.getBoxLayoutById(id)
                return boxLayout && !boxLayout.hidden
            },
            getPositionByPixel (x, y) {
                return {
                    x: Math.round(x / (this.gridSize.w + this.margin)),
                    y: Math.round(y / (this.gridSize.h + this.margin))
                }
            },
            updateLayout (layout) {
                this.$emit('update:layout', layout)
            },
            registerBox (box) {
                this.enableResizing(box)
                this.enableDragging(box)
                if (this.isMounted && this.autoAddLayoutForNewBox) {
                    this.createBoxLayout(box.$props.boxId)
                }
            },
            unregisterBox (box) {
            },
            enableDragging (box) {
                var initialLayout
                var isDragging = false

                box.$on('dragStart', evt => {
                    var boxLayout = this.getBoxLayoutById(box.boxId)
                    if (boxLayout.pinned) {
                        return
                    }
                    isDragging = true

                    // find box
                    this.dragging.boxLayout = boxLayout
                    this.placeholder = {
                        ...this.dragging.boxLayout
                    }

                    // clone layout
                    initialLayout = utils.sortLayout(this.layout)
                })

                box.$on('dragUpdate', evt => {
                    if (!isDragging) {
                        return
                    }
                    this.dragging.offset.x = evt.offset.x
                    this.dragging.offset.y = evt.offset.y

                    var moveBy = this.getPositionByPixel(evt.offset.x, evt.offset.y)
                    if (!utils.isFree(this.pinnedLayout, {
                        ...this.dragging.boxLayout.position,
                        x: this.dragging.boxLayout.position.x + moveBy.x,
                        y: this.dragging.boxLayout.position.y + moveBy.y
                    })) {
                        return
                    }

                    let newX = Math.max(0, this.dragging.boxLayout.position.x + moveBy.x);
                    let newY = Math.max(0, this.dragging.boxLayout.position.y + moveBy.y);
                    // check if box has moved
                    if (this.placeholder.position.x == newX && this.placeholder.position.y == newY) {
                        return;
                    }
                    this.placeholder = utils.updateBoxPosition(this.placeholder, {
                        x: newX,
                        y: newY
                    })

                    var newLayout = [ this.placeholder ]
                    initialLayout.forEach((boxLayout) => {
                        if (boxLayout.id === this.dragging.boxLayout.id) {
                            return
                        }
                        newLayout.push(utils.moveBoxToFreePlace(newLayout, boxLayout, this.bubbleUp))
                    })

                    if (this.bubbleUp) {
                        newLayout = utils.layoutBubbleUp(newLayout)
                        this.placeholder = newLayout.find((boxLayout) => {
                            return boxLayout.id === this.dragging.boxLayout.id
                        })
                    }
                    this.updateLayout(newLayout)
                })

                box.$on('dragEnd', evt => {
                    if (!isDragging) {
                        return
                    }
                    var moveBy = this.getPositionByPixel(evt.offset.x, evt.offset.y)
                    if (utils.isFree(this.pinnedLayout, {
                        ...this.dragging.boxLayout.position,
                        x: this.dragging.boxLayout.position.x + moveBy.x,
                        y: this.dragging.boxLayout.position.y + moveBy.y
                    })) {
                        this.placeholder = utils.updateBoxPosition(this.placeholder, {
                            x: Math.max(0, this.dragging.boxLayout.position.x + moveBy.x),
                            y: Math.max(0, this.dragging.boxLayout.position.y + moveBy.y)
                        })
                    }

                    this.dragging.boxLayout = utils.updateBoxPosition(this.dragging.boxLayout, {
                        x: this.placeholder.position.x,
                        y: this.placeholder.position.y
                    })

                    var newLayout = [ this.dragging.boxLayout ]
                    initialLayout.forEach((boxPosition) => {
                        if (boxPosition.id === this.dragging.boxLayout.id) {
                            return
                        }
                        newLayout.push(utils.moveBoxToFreePlace(newLayout, boxPosition, this.bubbleUp))
                    })

                    if (this.bubbleUp) {
                        newLayout = utils.layoutBubbleUp(newLayout)
                    }
                    this.updateLayout(newLayout)

                    this.dragging.boxLayout = null
                    this.dragging.offset.x = 0
                    this.dragging.offset.y = 0

                    this.placeholder.hidden = true
                    isDragging = false
                })
            },
            enableResizing (box) {
                var initialLayout
                var isResizing = false

                box.$on('resizeStart', evt => {
                    var boxLayout = this.getBoxLayoutById(box.boxId)
                    if (boxLayout.pinned) {
                        return
                    }
                    isResizing = true

                    // find box
                    this.resizing.boxLayout = boxLayout
                    this.placeholder = {
                        ...this.resizing.boxLayout
                    }

                    // clone layout
                    initialLayout = utils.sortLayout(this.layout)
                })

                box.$on('resizeUpdate', evt => {
                    if (!isResizing) {
                        return
                    }
                    this.resizing.offset.x = evt.offset.x
                    this.resizing.offset.y = evt.offset.y

                    var resizeBy = this.getPositionByPixel(evt.offset.x, evt.offset.y)
                    if (!utils.isFree(this.pinnedLayout, {
                        ...this.resizing.boxLayout.position,
                        w: this.resizing.boxLayout.position.w + resizeBy.x,
                        h: this.resizing.boxLayout.position.h + resizeBy.y
                    })) {
                        return
                    }

                    let newW = Math.max(1, this.resizing.boxLayout.position.w + resizeBy.x)
                    let newH = Math.max(1, this.resizing.boxLayout.position.h + resizeBy.y)
                    // check if box has moved
                    if (this.placeholder.position.w == newW && this.placeholder.position.h == newH) {
                        return;
                    }
                    this.placeholder = utils.updateBoxPosition(this.placeholder, {
                        x: this.resizing.boxLayout.position.x,
                        y: this.resizing.boxLayout.position.y,
                        w: newW,
                        h: newH
                    })

                    var newLayout = [ this.placeholder ]
                    initialLayout.forEach((boxLayout) => {
                        if (boxLayout.id === this.resizing.boxLayout.id) {
                            return
                        }
                        newLayout.push(utils.moveBoxToFreePlace(newLayout, boxLayout, this.bubbleUp))
                    })

                    if (this.bubbleUp) {
                        newLayout = utils.layoutBubbleUp(newLayout)
                        this.placeholder = newLayout.find((boxLayout) => {
                            return boxLayout.id === this.resizing.boxLayout.id
                        })
                    }
                    this.updateLayout(newLayout)
                })

                box.$on('resizeEnd', evt => {
                    if (!isResizing) {
                        return
                    }
                    var resizeBy = this.getPositionByPixel(evt.offset.x, evt.offset.y)
                    if (utils.isFree(this.pinnedLayout, {
                        ...this.resizing.boxLayout.position,
                        w: this.resizing.boxLayout.position.w + resizeBy.x,
                        h: this.resizing.boxLayout.position.h + resizeBy.y
                    })) {
                        this.placeholder = utils.updateBoxPosition(this.placeholder, {
                            w: Math.max(1, this.resizing.boxLayout.position.w + resizeBy.x),
                            h: Math.max(1, this.resizing.boxLayout.position.h + resizeBy.y)
                        })
                    }

                    this.resizing.boxLayout = utils.updateBoxPosition(this.resizing.boxLayout, {
                        w: this.placeholder.position.w,
                        h: this.placeholder.position.h
                    })

                    var newLayout = [ this.resizing.boxLayout ]
                    initialLayout.forEach((boxPosition) => {
                        if (boxPosition.id === this.resizing.boxLayout.id) {
                            return
                        }
                        newLayout.push(utils.moveBoxToFreePlace(newLayout, boxPosition, this.bubbleUp))
                    })

                    if (this.bubbleUp) {
                        newLayout = utils.layoutBubbleUp(newLayout)
                    }
                    this.updateLayout(newLayout)

                    this.resizing.boxLayout = null
                    this.resizing.offset.x = 0
                    this.resizing.offset.y = 0

                    this.placeholder.hidden = true
                })
            },
            createBoxLayout (...boxIds) {
                boxIds = boxIds.filter(boxId => !this.getBoxLayoutById(boxId))

                if (boxIds.length) {
                    let newLayout = [
                        ...this.layout
                    ]
                    boxIds.forEach(boxId => {
                        newLayout.push(utils.moveBoxToFreePlace(newLayout, {
                            id: boxId,
                            hidden: false,
                            position: {
                                x: 0,
                                y: 0,
                                ...this.defaultSize
                            }
                        }, this.bubbleUp))
                    })
                    this.updateLayout(newLayout)
                }
            }
        },
        mounted () {
            this.isMounted = true
            let boxIds = this.$children.map(box => box.$props.boxId)
            this.createBoxLayout(...boxIds)
        }
    }
</script>
