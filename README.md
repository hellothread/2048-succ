# 2048-succinct

F12 --》 控制台  --》粘贴

```
(function () {
    // 配置项
    var EVALUATE_ONLY = false; // 只评估不执行动作

    // 搜索相关常量
    var SEARCH_DEPTH = 4; // 搜索深度
    var SEARCH_TIME = 350; // 滑动时间(ms)
    var RETRY_TIME = 1000; // 重试检查间隔(ms)
    var ACCEPT_DEFEAT_VALUE = -999999; // 接受失败的值

    // 评估常量
    var NUM_EMPTY_WEIGHT = 5; // 空格权重
    var ADJ_DIFF_WEIGHT = -0.5; // 相邻差异权重
    var INSULATION_WEIGHT = -2; // 隔离权重
    var POSITION_WEIGHT = 0.04; // 位置权重
    var POSITION_VALUE = [
        0, 0, 0, 10,
        0, 0, 0, 15,
        0, 0, -5, 20,
        10, 15, 20, 50
    ];
    var LOG2 = {};
    for (var i = 0; i < 20; i++) {
        LOG2[1 << i] = i;
    }

    // 游戏常量
    var GRID_SIZE = 4;

    // 移动常量
    var MOVE_UP = {
        drow: -1,
        dcol: 0,
        dir: 0,
        keyCode: 38,
        key: 'ArrowUp'
    };
    var MOVE_DOWN = {
        drow: 1,
        dcol: 0,
        dir: 1,
        keyCode: 40,
        key: 'ArrowDown'
    };
    var MOVE_LEFT = {
        drow: 0,
        dcol: -1,
        dir: 0,
        keyCode: 37,
        key: 'ArrowLeft'
    };
    var MOVE_RIGHT = {
        drow: 0,
        dcol: 1,
        dir: 1,
        keyCode: 39,
        key: 'ArrowRight'
    };

    // 游戏统计
    var games = 0;
    var bestScore = 0;
    var averageScore = 0;
    var bestLargestTile = 0;
    var averageLargestTile = 0;

    // 用于存储定时器ID，以便在游戏结束时清除
    var moveIntervalId = null;
    var checkIntervalId = null;

    // 如果只评估不执行，则打印评估结果
    if (EVALUATE_ONLY) {
        var grid = getGrid();
        print(grid);
        evaluate(grid, true);
    } else {
        // 开始游戏循环
        moveIntervalId = setInterval(nextMove, SEARCH_TIME);
    }

    // 检查游戏是否结束或分数达到20000，如果满足条件则停止所有操作
    checkIntervalId = setInterval(function() {
        var score = getScore();
        if (gameLost() || score >= 20000) {
            // 停止所有定时器
            clearInterval(moveIntervalId);
            clearInterval(checkIntervalId);

            bestScore = Math.max(bestScore, score);

            var grid = getGrid();
            var largestTile = 0;
            for (var i = 0; i < grid.length; i++) {
                largestTile = Math.max(largestTile, grid[i]);
            }
            bestLargestTile = Math.max(bestLargestTile, largestTile);

            averageScore = (averageScore * games + score) / (games + 1);
            averageLargestTile = (averageLargestTile * games + largestTile) / (games + 1);
            games++;

            // 判断终止原因
            var stopReason = gameLost() ? "Game Over" : "分数达到20000";
            console.log('Game                   ' + games + '\n' +
                'Score                  ' + score + '\n' +
                'Largest tile           ' + largestTile + '\n' +
                'Average score          ' + Math.round(averageScore) + '\n' +
                'Average largest tile   ' + Math.round(averageLargestTile) + '\n' +
                'Best score             ' + bestScore + '\n' +
                'Best largest tile      ' + bestLargestTile + '\n' +
                stopReason + ' - Bot stopped.' + '\n' +
                '\n');

            search.table = {};

            // 游戏结束后不再尝试重新开始
            // 如果需要重新开始，用户需要手动点击"Try Again"按钮
        }
    }, RETRY_TIME);

    /**
     * 选择并执行下一步移动
     */
    function nextMove() {
        // 检查游戏是否已结束或分数是否已达到要求，如果满足则不执行移动
        if (gameLost() || getScore() >= 20000) {
            return;
        }

        var grid = getGrid();
        var move = search(grid, SEARCH_DEPTH, Number.NEGATIVE_INFINITY, true);
        pressKey(move);
    }

    /**
     * 使用深度优先搜索寻找最佳移动
     * @param grid: 游戏网格的扁平数组表示
     * @param depth: 搜索树深度，叶节点深度为0
     * @param alpha: 搜索值的下限
     * @param root: 是否为根节点
     * @return 在根节点返回最佳移动，在其他节点返回最佳移动的值
     */
    function search(grid, depth, alpha, root) {
        if (depth <= 0) {
            return evaluate(grid);
        }

        if (!search.table) {
            search.table = {};
        }

        // 在置换表中查找游戏网格
        var key = getGridKey(grid);
        var entry = search.table[key];
        if (entry && entry.depth >= depth && (!entry.isBound || entry.value <= alpha)) {
            return root ? entry.move : entry.value;
        }

        // 如果有置换表条目但无法使用其值，则至少将其最佳移动移至当前移动列表的前面
        var moves = [MOVE_RIGHT, MOVE_DOWN, MOVE_LEFT, MOVE_UP];
        if (entry) {
            var index = moves.indexOf(entry.move);
            var temp = moves[index];
            moves[index] = moves[0];
            moves[0] = temp;
        }

        var bestMove = undefined;
        var alphaImproved = false;

        for (var i = 0; i < moves.length; i++) {
            var copyGrid = copy(grid);
            var move = moves[i];

            if (make(copyGrid, move)) {
                bestMove = bestMove || move;
                var value = Number.POSITIVE_INFINITY;

                // 尝试在每个空格中放置一个2，从右下角开始迭代
                for (var j = copyGrid.length - 1; j >= 0 && value > alpha; j--) {
                    if (!copyGrid[j]) {
                        copyGrid[j] = 2;
                        value = Math.min(value, search(copyGrid, depth - 1, alpha));
                        copyGrid[j] = 0;
                    }
                }

                if (value > alpha) {
                    alpha = value;
                    bestMove = move;
                    alphaImproved = true;
                }
            }
        }

        if (!bestMove) {
            return root ? MOVE_LEFT : ACCEPT_DEFEAT_VALUE + evaluate(grid);
        }

        // 将搜索结果存储在置换表中
        search.table[key] = {
            depth: depth,
            value: alpha,
            move: bestMove,
            isBound: !alphaImproved
        };

        return root ? bestMove : alpha;
    }

    /**
     * 评估给定的网格状态
     * @param grid: 游戏网格的扁平数组表示
     * @param logging: 是否记录评估计算过程
     * @return 网格状态的估计值
     */
    function evaluate(grid, logging) {
        var value = 0;

        var positionValue = 0;
        var adjDiffValue = 0;
        var insulationValue = 0;
        var numEmpty = 0;

        for (var r = 0; r < GRID_SIZE; r++) {
            for (var c = 0; c < GRID_SIZE; c++) {
                var tile = get(grid, r, c);
                if (!tile) {
                    numEmpty++;
                    continue;
                }
                positionValue += tile * POSITION_VALUE[r * GRID_SIZE + c];

                // 执行成对比较
                if (c < GRID_SIZE - 1) {
                    var adjTile = get(grid, r, c + 1);
                    if (adjTile) {
                        adjDiffValue += levelDifference(tile, adjTile) * Math.log(tile + adjTile);

                        // 执行三元组比较
                        if (c < GRID_SIZE - 2) {
                            var thirdTile = get(grid, r, c + 2);
                            if (thirdTile && levelDifference(tile, thirdTile) <= 1.1) {
                                var smallerTile = Math.min(tile, thirdTile);
                                insulationValue += levelDifference(smallerTile, adjTile) * Math.log(smallerTile);
                            }
                        }
                    }
                }

                // 执行成对比较
                if (r < GRID_SIZE - 1) {
                    adjTile = get(grid, r + 1, c);
                    if (adjTile) {
                        adjDiffValue += levelDifference(tile, adjTile) * Math.log(tile + adjTile);

                        // 执行三元组比较
                        if (r < GRID_SIZE - 2) {
                            var thirdTile = get(grid, r + 2, c);
                            if (thirdTile && levelDifference(tile, thirdTile) <= 1.1) {
                                var smallerTile = Math.min(tile, thirdTile);
                                insulationValue += levelDifference(smallerTile, adjTile) * Math.log(smallerTile);
                            }
                        }
                    }
                }
            }
        }

        // 类似对数的曲线方程，从0开始，在numEmpty=5时迅速上升到10，之后基本趋于平稳
        var numEmptyValue = 11.12249 + (0.05735587 - 11.12249) / (1 + Math.pow((numEmpty / 2.480941), 2.717769));

        value += POSITION_WEIGHT * positionValue;
        value += NUM_EMPTY_WEIGHT * numEmptyValue;
        value += ADJ_DIFF_WEIGHT * adjDiffValue;
        value += INSULATION_WEIGHT * insulationValue;

        if (logging) {
            console.log('EVALUATION     ' + value + '\n' +
                '  position     ' + (POSITION_WEIGHT * positionValue) + '\n' +
                '  numEmpty     ' + (NUM_EMPTY_WEIGHT * numEmptyValue) + '\n' +
                '  adjDiff      ' + (ADJ_DIFF_WEIGHT * adjDiffValue) + '\n' +
                '  insulation   ' + (INSULATION_WEIGHT * insulationValue) + '\n'
            );
        }

        return value;
    }

    /**
     * 计算两个瓦片之间的堆栈级别差异
     * @param tile1: 第一个瓦片值
     * @param tile2: 第二个瓦片值
     * @return 两个给定瓦片之间的堆栈级别差异
     */
    function levelDifference(tile1, tile2) {
        return tile1 > tile2 ? LOG2[tile1] - LOG2[tile2] : LOG2[tile2] - LOG2[tile1];
    }

    /**
     * 返回网格中给定位置的瓦片值
     * @param grid: 游戏网格的扁平数组表示
     * @param row: 位置行
     * @param col: 位置列
     * @return 给定位置的瓦片值
     */
    function get(grid, row, col) {
        return grid[row * GRID_SIZE + col];
    }

    /**
     * 设置网格中给定位置的瓦片值
     * @param grid: 游戏网格的扁平数组表示
     * @param row: 位置行
     * @param col: 位置列
     * @param tile: 要分配的新瓦片值
     */
    function set(grid, row, col, tile) {
        grid[row * GRID_SIZE + col] = tile;
    }

    /**
     * 将给定的网格打印到控制台
     * @param grid: 游戏网格的扁平数组表示
     */
    function print(grid) {
        function pad(str, len) {
            len -= str.length;
            while (len-- > 0)
                str = ' ' + str;
            return str;
        }

        var result = '';
        for (var r = 0; r < GRID_SIZE; r++) {
            for (var c = 0; c < GRID_SIZE; c++) {
                var tile = get(grid, r, c);
                result += tile ? pad(tile + '', 5) : '    .';
            }
            result += '\n';
        }
        console.log(result);
    }

    /**
     * 复制给定的网格
     * @param grid: 游戏网格的扁平数组表示
     * @return 给定网格的副本
     */
    function copy(grid) {
        return grid.slice();
    }

    /**
     * 确定给定位置是否在网格范围内
     * @param row: 位置行
     * @param col: 位置列
     * @return 给定位置是否在网格范围内
     */
    function inBounds(row, col) {
        return 0 <= row && row < GRID_SIZE && 0 <= col && col < GRID_SIZE;
    }

    /**
     * 在网格上执行给定的移动，不插入新瓦片
     * @param grid: 游戏网格的扁平数组表示
     * @param move: 包含移动向量的对象
     * @return 移动是否成功执行
     */
    function make(grid, move) {
        var start = move.dir * (GRID_SIZE - 1);
        var end = (1 - move.dir) * (GRID_SIZE + 1) - 1;
        var inc = 1 - 2 * move.dir;

        var anyMoved = false;

        for (var r = start; r != end; r += inc) {
            for (var c = start; c != end; c += inc) {
                if (get(grid, r, c)) {
                    var newr = r + move.drow;
                    var newc = c + move.dcol;
                    var oldr = r;
                    var oldc = c;

                    while (inBounds(newr, newc)) {
                        var target = get(grid, newr, newc);
                        var tile = get(grid, oldr, oldc);
                        if (!target) {
                            set(grid, newr, newc, tile);
                            set(grid, oldr, oldc, 0);
                            anyMoved = true;
                        }
                        else if (target === tile) {
                            // 负值防止额外合并
                            set(grid, newr, newc, -2 * tile);
                            set(grid, oldr, oldc, 0);
                            anyMoved = true;
                            break;
                        }
                        else {
                            break;
                        }
                        oldr = newr;
                        oldc = newc;
                        newr += move.drow;
                        newc += move.dcol;
                    }
                }
            }
        }

        if (!anyMoved) {
            return false;
        }

        var numEmpty = 0;
        for (var i = 0; i < grid.length; i++) {
            if (grid[i] < 0) {
                grid[i] *= -1;
            }
            else if (!grid[i]) {
                numEmpty++;
            }
        }

        if (numEmpty === 0) {
            console.warn('No empty squares after making move.');
            return false;
        }

        return true;
    }

    /**
     * 计算给定游戏网格的哈希键
     * @param grid: 游戏网格的扁平数组表示
     * @return 给定游戏网格的哈希键
     */
    function getGridKey(grid) {
        if (!getGridKey.table1) {
            getGridKey.table1 = {};
            getGridKey.table2 = {};

            for (var i = 0; i < grid.length; i++) {
                for (var t = 2; t <= 8192; t *= 2) {
                    var key = t * grid.length + i;
                    getGridKey.table1[key] = Math.round(0xffffffff * Math.random());
                    getGridKey.table2[key] = Math.round(0xffffffff * Math.random());
                }
            }
        }

        var value1 = 0;
        var value2 = 0;
        for (var i = 0; i < grid.length; i++) {
            var tile = grid[i];
            if (tile) {
                var key = tile * grid.length + i;
                value1 ^= getGridKey.table1[key];
                value2 ^= getGridKey.table2[key];
            }
        }

        return value1 + '' + value2;
    }

    /**
     * 从DOM构造当前游戏网格
     * @return 游戏网格的扁平数组表示
     */
    function getGrid() {
        var grid = new Array(GRID_SIZE * GRID_SIZE);
        for (var i = 0; i < grid.length; i++) {
            grid[i] = 0;
        }

        // 获取网格容器
        var gridContainer = document.evaluate('/html/body/div[1]/main/div[1]/div/div/div[2]/div/div/div/div[4]', document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue;
        if (!gridContainer) return grid;

        // 获取所有瓦片
        var tileElements = gridContainer.querySelectorAll(".absolute");
        if (!tileElements || tileElements.length === 0) return grid;

        for (var i = 0; i < tileElements.length; i++) {
            var tile = tileElements[i];
            var value = parseInt(tile.textContent);
            if (isNaN(value)) continue;

            // 从样式中提取位置
            var style = tile.getAttribute('style');
            if (!style) continue;

            // 解析top和left值
            var topMatch = style.match(/top:\s*calc\((\d+)%/);
            var leftMatch = style.match(/left:\s*calc\((\d+)%/);

            if (!topMatch || !leftMatch) continue;

            var top = parseInt(topMatch[1]);
            var left = parseInt(leftMatch[1]);

            // 计算行和列
            var row = Math.floor(top / 25);
            var col = Math.floor(left / 25);

            if (row >= 0 && row < GRID_SIZE && col >= 0 && col < GRID_SIZE) {
                set(grid, row, col, value);
            }
        }

        return grid;
    }

    /**
     * 模拟给定移动的按键事件
     * @param move: 包含按键信息的对象
     */
    function pressKey(move) {
        var event = new KeyboardEvent('keydown', {
            bubbles: true,
            cancelable: true,
            key: move.key,
            keyCode: move.keyCode,
            which: move.keyCode
        });

        document.dispatchEvent(event);
    }

    /**
     * 从DOM确定当前游戏是否已结束
     * @return 当前游戏是否已结束
     */
    function gameLost() {
        var gameOverElement = document.evaluate('/html/body/div[1]/main/div[1]/div/div/div[2]/div/div/div/div[4]/div[2]/div[1]', document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue;
        return gameOverElement && gameOverElement.textContent.includes("Game Over");
    }

    /**
     * 获取当前分数
     * @return 游戏当前分数
     */
    function getScore() {
        var scoreElement = document.evaluate('/html/body/div[1]/main/div[1]/div/div/div[2]/div/div/div/div[1]/div[2]/div[1]/div[2]', document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue;
        return scoreElement ? parseInt(scoreElement.textContent) : 0;
    }

})();

```
