<script setup lang="ts">
    import { reactive, ref, onUpdated, onMounted } from 'vue';
    import Grid from './components/Grid.vue';

    const mineICON = '●';
    let firstClick = ref(true);
    let GameOver = ref('');
    // 矩阵【行，列】
    const numSquare:[number, number] = [10, 10];
    const minesNum:number = Math.floor(numSquare[0] * numSquare[1] * 0.1);
    const minesArray:string[] = [];
    const square:string[][] = new Array(numSquare[0]);
    for (let i = 0; i < numSquare[0]; i++) {
        square[i] = new Array(numSquare[1]).fill('');
    }
    // 初始化游戏
    function initGame(eRow:number, eCol:number):void {
        firstClick.value = false;
        // console.log('initGame');
        // 随机放雷
        for (let mineIndex = 0; mineIndex < minesNum;) {
            const row = Math.floor(Math.random() * numSquare[0]);
            const col = Math.floor(Math.random() * numSquare[1]);

            if (row == eRow && col == eCol) continue;

            if (!square[row][col]) {
                square[row][col] = mineICON; // 放雷
                mineIndex++;
            }
        }
        // 计算雷四周的数字提示
        square.forEach((colData:string[], row:number):void => {
            colData.forEach((data:string, col:number) => {
                // console.log('<row, col>', row, col);
                if (data == mineICON) return;
                // 不是雷的情况下，去计算周围的布局
                let mines = 0;
                computedNine(row, col, (iRow:number, iCol:number) => {
                    if (
                        square[iRow][iCol] == mineICON
                        &&
                        !(iRow == row && iCol == col)
                    ) {
                        mines++;
                        minesArray.push(`${iRow},${iCol}`);
                    }
                })
                if (mines) square[row][col] = String(mines);
                // console.log('square[' + row + '][' + col + ']', mines)
            });
        });
        // console.log(square);
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

    // 遇到空格展示大片
    const openGird:Set<string> = reactive(new Set<string>());
    function clearAmbinetGrid(row:number, col:number):void {
        computedNine(row, col, (iRow:number, iCol:number) => {
            const key = `${iRow},${iCol}`;
            if (square[iRow][iCol] != mineICON && !openGird.has(key))
                openGird.add(key);
        });
    }

    // 触发雷区
    function showAllMines():void {
        GameOver.value = 'LOSE!';
        minesArray.forEach((grid) => {
            openGird.add(grid)
        });
    }

    // 判赢
    const winNum = numSquare[0] * numSquare[1] - minesNum;
    async function isWin(row:number, col:number) {
        if (GameOver.value.includes('LOSE')) return;
        openGird.add(`${row},${col}`);
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
    });

    // 展开周围九格
    function openAmbient (rowS: string | number, colS: string | number) {
        computedNine(Number(rowS), Number(colS), (iRow:number, iCol:number) => {
            const key = `${iRow},${iCol}`;
            // console.log('Gamebox-mouse-key', key, openGird);
            if (!openGird.has(key)) openGird.add(key);
        });
    }

    // 清 openGird
    function clearOpenGrid (iRow:number, iCol:number) {
        const key = `${iRow},${iCol}`;
        // console.log('clearOpenGrid', key, openGird.has(key), openGird);
        if (openGird.has(key)) openGird.delete(key);
    }
</script>

<template>
    <p class="Upon" v-if="GameOver">{{ GameOver }}</p>
    <div class="game-box" ref="Gamebox">
        <div v-for="row in square.length" class="grid-line">
            <template v-for="(item, col) in square[row - 1]" :key="`${row - 1},${col}`">
                <Grid
                    :msg="item"
                    :data-label="`${row - 1},${col}`"
                    :open="openGird.has(`${row - 1},${col}`)"
                    @click.capture="firstClick ? initGame(row - 1, col) : ''"
                    @click="isWin(row - 1, col)"
                    @clearAmbinetGrid="clearAmbinetGrid(row - 1, col)"
                    @boom="showAllMines"
                    @noOpenFlag="clearOpenGrid(row - 1, col)"
                />
            </template>
        </div>
    </div>
</template>

<style lang="less">
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
}
body {
    background-color: #111;
}
</style>