<script lang="ts">
import { Component, Inject, Prop, Watch } from 'vue-property-decorator';
import Draggable from './Draggable.vue';
import { EventEmitter } from 'events';
import { PieceType, UIData } from '@/types/ui-data';

@Component({
    created(this: Piece) {
        this.$on('draggedTo', (coords: { x: number, y: number }) => {
            [this.currentX, this.currentY] = [coords.x, coords.y];

            this.$nextTick(() => {
                this.communicator.emit('draggedPosChanged', this);
            });
        });
    },
    mounted(this: Piece) {
        this.mounted = true;
    },
    beforeDestroy(this: Piece) {
        this.onTransitionEnd();
    }
})
export default class Piece extends Draggable {
    @Prop({ default: () => ({ x: 20, y: 20, rotate: 0 }) })
    targetState!: {
        x: number,
        y: number,
        rotate: number
    };

    @Prop()
    pieceId!: string;

    @Inject()
    readonly communicator!: EventEmitter;

    @Inject()
    readonly ui!: UIData;

    public pieceType!: PieceType;

    currentX = 0;
    currentY = 0;
    rotate = 0;
    mounted = false;
    transitioning = false;
    transitionCount = 0;

    moving = false;

    startTransitioning() {
        if (this.transitioning) {
            return;
        }

        this.ui.waitingAnimations += 1;
        this.transitioning = true;
        this.transitionCount += 1;

        const count = this.transitionCount;

        // Safeguard to make sure even if we don't catch the transition end event, we stop the transition
        setTimeout(() => {
            if (count === this.transitionCount) {
                this.onTransitionEnd();
            }
        }, 1000);
    }

    @Watch('dragging')
    @Watch('targetState', { immediate: true })
    onTargetChanged(newVal: boolean, oldVal: boolean) {
        if (this.dragging) {
            return;
        }

        if (this.targetState.x === this.currentX && this.targetState.y === this.currentY) {
            return;
        }

        if (!this.transitioning && this.mounted) {
            if (!(oldVal && !newVal)) {
                this.startTransitioning();
            } else {
                this.moving = true;
                setTimeout(() => this.moving = false, 800);
            }
        }

        if (!this.mounted) {
            this.currentX = this.targetState.x;
            this.currentY = this.targetState.y;
            this.rotate = this.targetState.rotate;
        } else {
            // Make sure the animation plays by waiting until the class "dragging" is removed for sure from the element
            setTimeout(() => {
                this.currentX = this.targetState.x;
                this.currentY = this.targetState.y;
                this.rotate = this.targetState.rotate;
            });
        }
    }

    @Watch('dragging')
    onDraggingChanged() {
        if (this.dragging) {
            this.ui.dragged = this;
        } else {
            this.ui.dragged = null;
            this.communicator.emit('draggedPosChanged', this);
        }
    }

    onTransitionEnd() {
        // console.log("on transition end", this.transitioning);
        if (this.transitioning) {
            this.transitioning = false;
            this.ui.waitingAnimations = Math.max(this.ui.waitingAnimations - 1, 0);
        }
    }

    get elId() {
        return this.dragging ? 'dragged' : (this.transitioning || this.moving ? 'moving' : undefined);
    }
}

</script>
