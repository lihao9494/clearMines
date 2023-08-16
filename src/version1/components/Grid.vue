<script lang="ts" setup>
    import { ref, Ref, computed, watch } from 'vue';

    interface Props {
        msg: string,
        // flag?: boolean,
        open: boolean
    }

    // 定义可接受的 props
    // defineProps<Props>()
    
    // 给 props 设置默认值, 返回 props 对象
    const props = withDefaults(defineProps<Props>(), {
        open: false
    });
    // 判断目前的点击状态
    let actived:Ref<boolean> = ref(false);
    // 判断是否插旗
    let flag = ref(false);
    
    // 判断返回值是不是数字
    const isNum = computed(():boolean => {
        return /\d+/.test(props.msg);
    });
    // 判断是不是雷
    const isMine = computed(():boolean => {
        return !isNum.value && props.msg != '';
    });

    // 监听 open
    watch(() => props.open, (newOpen) => {
        if (!actived.value && newOpen) changeActived();
    });
    
    // template 里则写 $emit
    const emit = defineEmits(['clearAmbinetGrid', 'boom', 'noOpenFlag']);
    let mineClicked = ref(false);
    // 点击 grid 执行参数
    function changeActived() {
        // 遇旗
        if (flag.value) {
            emit('noOpenFlag');
            return;
        }

        actived.value = true;
        // 触雷
        if (isMine.value) {
            mineClicked.value = true;
            emit('boom');
            return;
        }
        // spread
        if (props.msg == '') emit('clearAmbinetGrid');
    }
    // 插旗/去旗
    function onflag() {
        if (actived.value) return;

        flag.value = !flag.value;
    }
</script>

<template>
    <div
        :class="['one-grid', {
            'on': actived,
            'mines': isMine,
            'num': isNum,
            'flag': flag,
            'trigger': mineClicked
        }]"
        @click.right="onflag"
    >
        <span class="icon">{{ msg }}</span>
    </div>
</template>

<style lang="less" scoped>
@size: 1.2rem;
.one-grid {
    min-width: 10px;
    min-height: 10px;
    width: @size;
    height: @size;
    border: 1px solid #fff;
    display: inline-block;
    padding: 0;
    margin: 0;
    position: relative;
    font-size: 0;
    line-height: 1;
    margin-left: -1px;
    margin-top: -1px;
    cursor: default;
    background-color: #eee;
    font-size: 0;
    transition: all .25s;

    &.on {
        font-size: @size * .6;
        color: #eee;
        background-color: transparent;
    }

    &.flag {
        &::before {
            content: '✿';
            display: inline-block;
            color: green;
            text-align: center;
            position: absolute;
            left: 0;
            top: 0;
            width: 100%;
            height: 100%;
            font-size: @size * .8;
            text-align: center;
            line-height: @size;
        }
    }

    &.mines {
        // color: #0000ff;
        color: #eee;

        &.on {
            background-color: red;
        }

        &.trigger {
            &::before,
            &::after {
                content: '';
                display: inline-block;
                width: 3px;
                height: 100%;
                background-color: blue;
                position: absolute;
                left: 42%;
                top: 2%;
                z-index: 1;
            }

            &::before {
                transform: rotate(-45deg);
            }

            &::after {
                transform: rotate(45deg);
            }
        }
    }
}
.icon {
    position: absolute;
    left: 50%;
    top: 50%;
    transform: translateX(-50%)translateY(-50%);
    pointer-events: none;
}
</style>