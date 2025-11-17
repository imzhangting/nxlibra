<!-- vue3 dialog component with typescript -->
 <script setup lang="ts">
import type { Position } from '@vueuse/core';

import {
  computed,
  defineModel,
  nextTick,
  reactive,
  useTemplateRef,
  watch,
} from 'vue';

import {
  useDraggable,
  useEventListener,
  useResizeObserver,
  useWindowSize,
} from '@vueuse/core';

interface DialogProps {
  show?: boolean;
  height?: string;
  width?: string;
  title?: string;
  draggable?: boolean;
  header?: boolean;
  footer?: boolean;
  center?: boolean;
  offsetTop?: number;
  offsetLeft?: number;
  closeOnEsc?: boolean;
}

const props = withDefaults(defineProps<DialogProps>(), {
  show: false,
  height: '400px',
  width: '300px',
  title: '',
  center: true,
  offsetTop: 160,
  offsetLeft: 0,
  header: true,
  footer: false,
  draggable: false,
  closeOnEsc: false,
});

const emit = defineEmits<DialogEmits>();

// 定义事件类型
interface DialogEmits {
  close: [value: boolean];
  move: [position: Position, event: PointerEvent];
  end: [position: Position, event: PointerEvent];
}

const showModel = defineModel<boolean>('show', { default: false });
const beforeClose = () => {
  emit('close', false);
};
// 弹框大小
const dialogSize = reactive<{ height: string; width: string }>({
  width: props.width,
  height: props.height,
});

watch(
  () => [props.width, props.height],
  ([newWidth, newHeight]) => {
    dialogSize.width = newWidth ?? '';
    dialogSize.height = newHeight ?? '';
  },
);

// 计算弹框高度
const bodyHeight = computed(() => {
  if (props.header && props.footer) {
    return 'calc(100% - 40px - 40px)';
  } else if (props.header || props.footer) {
    return 'calc(100% - 40px)';
  } else {
    return '100%';
  }
});
const { width: windowWidth } = useWindowSize();
/** 拖拽方法 */
const dialogRef = useTemplateRef<HTMLElement>('dialogRef');
const dragRef = useTemplateRef<HTMLElement>('dragRef');
const initialPosition = reactive<Position>({
  x: 0,
  y: 0,
});
const useDrag = useDraggable(dragRef, {
  preventDefault: true,
  initialValue: initialPosition,
  onMove: (position: Position, event: PointerEvent) => {
    emit('move', position, event);
  },
  onEnd: (position: Position, event: PointerEvent) => {
    emit('end', position, event);
    // 记录当前位置
    initialPosition.x = position.x;
    initialPosition.y = position.y;
  },
});
/* 初始化弹框默认位置 */
nextTick(() => {
  const elWidth = dialogRef.value?.clientWidth ?? 0;
  // 方位位置
  if (props.center) {
    initialPosition.x = windowWidth.value / 2 - elWidth / 2;
    initialPosition.y = props.offsetTop;
  } else {
    initialPosition.x = props.offsetLeft;
    initialPosition.y = props.offsetTop;
  }
});

// 监听弹框大小变化
watch([showModel, dialogRef], ([newShow, el]) => {
  if (newShow && el) {
    useResizeObserver(dialogRef, (entries) => {
      const entry = entries[0];
      if (!entry) return;
      const { width, height } = entry.contentRect;
      dialogSize.width = `${width}px`;
      dialogSize.height = `${height}px`;
    });
  }
});

// 添加ESC键关闭功能
useEventListener(document, 'keydown', (e) => {
  if (props.closeOnEsc && showModel.value && e.key === 'Escape') {
    beforeClose();
  }
});
const unDragStyle = computed(
  () =>
    `
    left: ${props.offsetLeft}px;
    top: ${props.offsetTop}px;
  `,
);
</script>

<template>
  <div
    v-if="showModel"
    ref="dialogRef"
    class="tx-dialog"
    :style="[
      {
        height: dialogSize.height,
        width: dialogSize.width,
        '--dialog-body-height': bodyHeight,
      },
      draggable ? useDrag.style.value : unDragStyle,
    ]"
  >
    <div class="tx-dialog-content">
      <div
        v-if="header"
        class="tx-dialog-header"
        ref="dragRef"
        :style="[draggable ? { cursor: 'grab' } : {}]"
      >
        <div class="title">{{ title }}</div>
        <slot name="tools"></slot>
        <div class="close" @click="beforeClose"></div>
        <!-- 装饰 -->
        <div class="line"></div>
        <div class="decoration">
          <div>
            <i></i>
            <i></i>
            <i></i>
          </div>
        </div>
        <div class="decoration-bottom-right"></div>
        <div class="decoration-bottom-left"></div>
      </div>
      <div class="tx-dialog-body">
        <div class="tx-dialog-body-scroll">
          <div class="tx-dialog-body-content">
            <slot></slot>
          </div>
        </div>
      </div>
      <div v-if="footer" class="tx-dialog-footer">
        <slot name="footer"></slot>
      </div>
    </div>
    <div class="tx-dialog-skeleton">
      <div v-if="header" class="tx-dialog-skeleton-header">
        <div class="tx-dialog-skeleton-header-icon"></div>
      </div>
    </div>
  </div>
</template>

<style scoped lang="scss">
.tx-dialog {
  @apply pointer-events-auto fixed z-[10000]; // h-[50%] w-[50%] max-h-[70%]  max-w-[80%]

  overflow: auto;
  resize: both !important;

  .tx-dialog-content {
    @apply absolute z-[10002] h-full w-full p-1;

    .tx-dialog-header {
      @apply relative h-10;

      background: linear-gradient(
        90deg,
        rgb(36 139 235 / 100%) 10%,
        rgb(36 139 235 / 30%) 40%,
        rgb(36 139 235 / 0%) 100%
      );
      clip-path: polygon(30px 0, 100% 0%, 100% 100%, 10px 100%);

      &::before {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 1px;
        content: '';
        background: linear-gradient(
          90deg,
          rgb(206 238 255/75%) 50%,
          transparent
        );
      }

      &::after {
        position: absolute;
        bottom: 0;
        left: 0;
        width: 50%;
        height: 1px;
        content: '';
        background: linear-gradient(90deg, rgb(206 238 255) 50%, transparent);
      }

      .line {
        position: absolute;
        top: -4px;
        left: 20px;
        width: 1px;
        height: 48px;
        content: '';
        background: rgb(196 232 253);
        transform: rotate(27deg);
      }

      .title {
        position: absolute;
        top: 50%;
        margin-left: 36px;
        font-family: PangMenZhengDao !important;
        font-size: 20px;
        color: transparent;
        background: linear-gradient(180deg, #c5fefd 20%, #c5dfff 100%);
        background-clip: text;
        transform: translateY(-50%);
        -webkit-text-fill-color: transparent;
      }

      .close {
        @apply absolute right-4 top-1/2 h-4 w-4 -translate-y-1/2 cursor-pointer bg-[url('./images/close.png')] bg-cover;
      }

      .decoration-bottom-left {
        @apply absolute -bottom-1 left-4 h-1 w-32 rounded-full bg-[#3efff8] blur-[3px];

        border-radius: 120%;

        &::after {
          @apply absolute -bottom-1 left-14 h-3 w-3 rounded-full bg-[#26e6ff] blur-[3px] content-[''];
        }
      }

      .decoration-bottom-right {
        @apply absolute bottom-0 right-0 h-[3px] w-2 bg-[#3efff8];
      }

      .decoration {
        @apply absolute right-0 top-[1px] flex h-[10px] w-[120px] items-center justify-end;

        background: linear-gradient(90deg, #1a628d 50%, transparent 100%);
        clip-path: polygon(6px 0, 100% 0, 100% 100%, 0 100%);

        > div {
          @apply flex h-[7px] w-[112px] flex-row items-center justify-start gap-1 pl-2;

          background: linear-gradient(90deg, #248beb 50%, transparent 100%);
          clip-path: polygon(4px 0, 100% 0, 100% 100%, 0 100%);

          > i {
            @apply block h-[3px] w-2;

            background: linear-gradient(90deg, #b4edf1 50%, #77edef 100%);
            clip-path: polygon(
              1px 0,
              100% 0,
              100% 1px,
              calc(100% - 1px) 100%,
              0 100%
            );

            &:nth-child(2) {
              opacity: 0.6;
            }

            &:nth-child(3) {
              opacity: 0.3;
            }
          }
        }
      }
    }

    .tx-dialog-footer {
      @apply relative h-10 w-full pl-4 pr-7;
    }

    .tx-dialog-body {
      @apply relative flex h-[var(--dialog-body-height)] w-full overflow-hidden px-3 py-4;

      .tx-dialog-body-scroll {
        @apply flex flex-1 overflow-hidden overflow-y-auto;

        .tx-dialog-body-content {
          @apply w-full flex-1;
        }
      }
    }
  }
  // 背景
  .tx-dialog-skeleton {
    @apply absolute left-0 top-0 z-[-10001] h-full w-full p-[3px];

    background: linear-gradient(
      180deg,
      rgb(3 37 68 / 75%) 60%,
      rgb(34 131 221 / 50%) 100%
    );
    border: 1px solid rgb(96 167 229);
    box-shadow: inset 0 0 10px rgb(88 155 212);
    clip-path: polygon(
      20px 0,
      100% 0,
      100% calc(100% - 20px),
      calc(100% - 20px) 100%,
      0 100%,
      0 20px
    );

    .tx-dialog-skeleton-header {
      @apply relative h-10 w-[28px];

      background: rgb(36 139 235 / 50%);
      border: 1px solid rgb(196 232 253/50%);
      clip-path: polygon(20px 0, 100% 0, 5px 100%, 0% 100%, 0 20px);

      &::before {
        position: absolute;
        top: -4px;
        left: 8px;
        width: 2px;
        height: 26px;
        content: '';
        background: rgb(196 232 253/50%);
        transform: rotate(45deg);
      }

      &::after {
        position: absolute;
        top: -4px;
        left: 13px;
        width: 1px;
        height: 48px;
        content: '';
        background: rgb(196 232 253/50%);
        transform: rotate(30deg);
      }

      .tx-dialog-skeleton-header-icon {
        @apply absolute left-[4px] top-[8px] h-[24px] w-[14px];

        background: linear-gradient(180deg, #c6fffd, #49fcf7);
        clip-path: polygon(10px 0, 100% 0, 0% 100%, 0 10px);
      }
    }

    &::before {
      position: absolute;
      top: -4px;
      left: 8px;
      width: 2px;
      height: 26px;
      content: '';
      background: rgb(96 167 229);
      transform: rotate(45deg);
    }

    &::after {
      position: absolute;
      right: 8px;
      bottom: -3px;
      width: 1px;
      height: 26px;
      content: '';
      background: rgb(96 167 229);
      transform: rotate(45deg);
    }
  }
}
</style>
<style>
/* .sb-show-main.sb-main-padded {
  background: url('./images/bg.png');
} */
</style>

