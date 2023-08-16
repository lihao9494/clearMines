<script setup lang="ts">
    import { reactive, ref, onUpdated, onMounted } from 'vue';

    const mineICON = '●';
    let firstClick = ref(true);
    let GameOver = ref('');
    let minesNum = ref(0);
    let flagNum = ref(0);

    let touchAllX:number = 0;

    // 矩阵【行，列】
    const numSquare:[number, number] = [12, 12];
    minesNum.value = Math.floor(numSquare[0] * numSquare[1] * 0.15);
    const minesArray:string[] = [];
    interface squareT {
        msg: string,
        open: boolean,
        actived: boolean,
        flag: boolean,
        mineClicked: boolean,
        mines?: boolean,
        num?: boolean
    };
    const square:squareT[][] = reactive(new Array(numSquare[0]));

    function newGame () {
        console.log('newGame');
        firstClick.value = true;
        GameOver.value = '';
        flagNum.value = 0;
        for (let i = 0; i < numSquare[0]; i++) {
            const obj = {
                msg: '',
                open: false,
                actived: false,
                flag: false,
                mineClicked: false
            };
            square[i] = new Array(numSquare[1]).fill('').map(() => {
                return Object.assign({}, obj);
            });
        }
    }
    newGame();
    
    // 初始化游戏
    function initGame(eRow:number, eCol:number):void {
        firstClick.value = false;
        console.log('initGame');
        // 随机放雷
        for (let mineIndex = 0; mineIndex < minesNum.value;) {
            const row = Math.floor(Math.random() * numSquare[0]);
            const col = Math.floor(Math.random() * numSquare[1]);

            if (row == eRow && col == eCol) continue;

            if (!square[row][col].msg) {
                square[row][col].msg = mineICON; // 放雷
                square[row][col].mines = true; // 放雷
                mineIndex++;
                minesArray.push(`${row},${col}`);
            }
        }
        // 计算雷四周的数字提示
        square.forEach((colData:squareT[], row:number):void => {
            colData.forEach((data:squareT, col:number) => {
                // console.log('<row, col>', row, col);
                if (data.msg == mineICON) return;
                // 不是雷的情况下，去计算周围的布局
                let mines = 0;
                computedNine(row, col, (iRow:number, iCol:number) => {
                    if (
                        square[iRow][iCol].msg == mineICON
                        &&
                        !(iRow == row && iCol == col)
                    ) {
                        mines++;
                    }
                })
                if (mines) {
                    square[row][col].msg = String(mines);
                    square[row][col].num = true;
                }
                // console.log('square[' + row + '][' + col + ']', mines)
            });
        });
        console.log(square);
    }

    interface Frontier {
        left: number,
        right: number,
        top: number,
        bottom: number
    }
    // 确定排布边界
    function getFrontier(row:number, col:number):Frontier {
        return {
            left: col - 1 > 0 ? col - 1 : 0,
            right: col + 1 > numSquare[1] - 1 ? numSquare[1] - 1 : col + 1,
            top: row - 1 > 0 ? row - 1 : 0,
            bottom: row + 1 > numSquare[0] - 1 ? numSquare[0] - 1 : row + 1
        };
    }
    // 九宫格计算
    function computedNine (row:number, col:number, cb:Function):void {
        const { left, right, top, bottom } = getFrontier(row, col);
        for (let iRow:number = top; iRow <= bottom; iRow++) {
            for (let iCol:number = left; iCol <= right; iCol++) {
                cb && cb(iRow, iCol);
            }
        }
    }
    // 上下左右格计算
    // function computedFour (row:number, col:number, cb:Function):void {
    //     const { left, right, top, bottom } = getFrontier(row, col);
    //     const fourGrid = new Set([`${top},${col}`, `${row},${left}`, `${bottom},${col}`, `${row},${right}`]);
    //     fourGrid.forEach((val) => {
    //         const [iRow, iCol] = val.split(',');
    //         cb && cb(iRow, iCol);
    //     });
    // }
    
    // 确认展开
    function openGridFunc (row:number, col:number) {
        const target = square[row][col];

        if (target.flag) return;

        const key = `${row},${col}`;
        openGird.add(key);
        
        target.actived = true;
        // spread
        if (target.msg == '') clearAmbinetGrid(row, col);
    }

    // 遇到空格展示大片
    const openGird:Set<string> = reactive(new Set<string>());
    function clearAmbinetGrid(row:number, col:number):void {
        computedNine(row, col, (iRow:number, iCol:number) => {
            const key = `${iRow},${iCol}`;
            if (square[iRow][iCol].msg != mineICON && !openGird.has(key))
            openGridFunc(iRow, iCol);
        });
    }

    // 触发雷区
    function showAllMines():void {
        GameOver.value = 'LOSE!';
        minesArray.forEach((grid) => {
            const [row, col] = grid.split(',') as [string, string];
            openGridFunc(Number(row), Number(col));
        });
    }

    // 判赢
    const winNum = numSquare[0] * numSquare[1] - minesNum.value;
    async function isWin(row:number, col:number) {
        if (GameOver.value.includes('LOSE')) return;
        changeActived(row, col);
    }

    // 禁用鼠标右键触发的菜单
    window.document.oncontextmenu = function(){ return false; }

    onUpdated(() => {
        if (openGird.size == winNum && GameOver.value == '') GameOver.value = 'WIN!';
    });
    
    const Gamebox = ref<HTMLDivElement | null>(null);
    onMounted(() => {
        // console.log('Gamebox.value', Gamebox.value);
        
        // 左右键功能
        let mousedownStartTime:Date | null = null;
        let button:number | null = null;
        
        Gamebox.value?.addEventListener('mousedown', (e: MouseEvent) => {
            const element:HTMLElement  = e?.target as HTMLElement;
            const bottonP = e?.button;
            if ([0, 2].includes(bottonP)) {
                if (button == null) {
                    button = bottonP;
                    mousedownStartTime = new Date();
                } else if ((button == 0 && bottonP == 2) || (button == 2 && bottonP == 0)) {
                    let mousedownEndTime:Date = new Date();
                    let diff = mousedownEndTime.getTime() - (mousedownStartTime?.getTime() || 0);
                    if (diff < 1000) {
                        const label:string | null = element?.getAttribute('data-label');
                        const [ rowS, colS ] = label?.split(',') as string[];
                        openAmbient(rowS, colS);
                    }
                }
            }
        });
        Gamebox.value?.addEventListener('mouseup', () => {
            button = null;
            mousedownStartTime = null;
        });

        let width = document.querySelector('.one-grid')?.getBoundingClientRect().width;
        touchAllX = Number(width) / 5;
    });

    // 展开周围九格
    function openAmbient (rowS: string | number, colS: string | number) {
        computedNine(Number(rowS), Number(colS), (iRow:number, iCol:number) => {
            const key = `${iRow},${iCol}`;
            // console.log('Gamebox-mouse-key', key, openGird);
            if (!openGird.has(key)) openGridFunc(iRow, iCol);
        });
    }

    // 插旗/去旗
    function onflag(item:squareT) {
        if (item.actived || GameOver.value) return;

        item.flag = !item.flag;

        if (item.flag) {
            flagNum.value++;
        } else {
            flagNum.value--;
        }
    }
    // 点击 grid 执行参数
    function changeActived(row:number, col:number) {
        const item = square[row][col];
        console.log('changeActived', item, row, col);

        // 触雷
        if (item.msg == mineICON) {
            item.actived = true;
            item.mineClicked = true;
            showAllMines();
            return;
        }

        openGridFunc(row, col);
    }

    // touch 事件触发置旗
    function getTouchX (e:TouchEvent) {
       return e.changedTouches[0]?.clientX;
    }
    let changeTouchX = 0;
    function touchNeedFlag (e:TouchEvent) {
        console.log('touchNeedFlag', e);
        changeTouchX = getTouchX(e);
    }
    function touchSetFlag (e:TouchEvent, item:squareT) {
        console.log('touchSetFlag', e);
        const touchC = getTouchX(e);
        if (touchAllX == 0) {
            throw new Error('touchAllX 为 0');
        }
        console.log('touchC - changeTouchX > Number(touchAllX) / 2', touchC, changeTouchX, Number(touchAllX) / 2);
        if (Math.abs(touchC - changeTouchX) > touchAllX) {
            onflag(item);
        }

        changeTouchX = 0;
    }
</script>

<template>
    <p class="Upon" v-if="GameOver">{{ GameOver }}</p>
    <p class="Upon tip" v-else>未标记炸弹数：{{ minesNum - flagNum }}</p>
    <div class="game-box" ref="Gamebox">
        <div v-for="row in square.length" class="grid-line">
            <template v-for="(item, col) in square[row - 1]" :key="`${row - 1},${col}`">
                <div
                    :class="['one-grid',
                        'one-grid-' + (row - 1) + '-' + col,
                    {
                        'on': item.actived,
                        'mines': item.mines,
                        'num': item.num,
                        'flag': item.flag,
                        'trigger': item.mineClicked
                    }]"
                    :data-label="`${row - 1},${col}`"
                    @click.right="onflag(item)"
                    @click.capture="firstClick ? initGame(row - 1, col) : ''"
                    @click="isWin(row - 1, col)"
                    @touchstart="touchNeedFlag"
                    @touchend="touchSetFlag($event, item)"
                    @dblclick="openAmbient(row - 1, col);"
                >
                    <span class="icon">{{ item.msg }}</span>
                </div>
            </template>
        </div>
    </div>
    <p class="Btn" @click="newGame()">New Game</p>
</template>

<style lang="less">
@size: 1.5rem;
.grid-line {
    line-height: 1;
    font-size: 0;
    white-space: nowrap;
}
.Upon {
    position: absolute;
    top: 1rem;
    left: 0;
    width: 100%;
    text-align: center;

    &.tip {
        font-size: @size * .7;
    }
}
body {
    background-color: #111;
    color: #fff;
}

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
    line-height: @size;
    margin-left: -1px;
    margin-top: -1px;
    cursor: default;
    background-color: #eee;
    font-size: 0;
    transition: all .15s;
    overflow: hidden;

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
        line-height: @size;

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
    left: 0;
    top: 0;
    pointer-events: none;
    width: 100%;
    height: 100%;

    &.iflag {
        color: green;
        font-size: @size * .8;
        text-align: center;
        line-height: @size;
        left: 0;
        transform: translateX(-100%);
    }
}

.Btn {
    position: relative;
    top: calc(100% - @size);
    left: 0;
    pointer-events: none;
    width: @size * 3;
    height: @size * 1.1;
    line-height: @size * 1.1;
    padding: 0 @size * .2;
    margin: 40px auto;
    border-radius: 4px;
    border: 1px solid;
    font-size: @size * .5;
    z-index: 1000;
    display: block;
}
</style>