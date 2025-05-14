# 2048-succinct

F12 --》 控制台  --》粘贴

```
(function () {
    // ==================== 用户配置区 ====================
    var TARGET_SCORE = 19000;      // 目标分数，达到后停止    可调整
    var SEARCH_TIME = 350;         // 每次移动间隔时间(ms)   可调整
    var EVALUATE_ONLY = false;     // 只评估不执行，用于调试

    // ==================== 搜索参数 ====================
    var SEARCH_DEPTH = 4;          // AI搜索深度，越大越强但更慢
    var RETRY_TIME = 1000;         // 重试检查间隔(ms)
    var ACCEPT_DEFEAT_VALUE = -999999; // 接受失败的评分阈值

    // ==================== 评估权重参数 ====================
    var NUM_EMPTY_WEIGHT = 5;      // 空格数量权重
    var ADJ_DIFF_WEIGHT = -0.5;    // 相邻差异权重
    var INSULATION_WEIGHT = -2;    // 隔离权重（被小值夹住的大值）
    var POSITION_WEIGHT = 0.04;    // 位置权重
    // 位置价值矩阵 - 右下角优先策略
    var POSITION_VALUE = [
        0, 0, 0, 10,
        0, 0, 0, 15,
        0, 0, -5, 20,
        10, 15, 20, 50
    ];

    // ==================== 基础参数 ====================
    var GRID_SIZE = 4;             // 网格大小
    var LOG2 = {};                 // 预计算对数值
    for (var i = 0; i < 20; i++) {
        LOG2[1 << i] = i;
    }

    // ==================== 移动方向定义 ====================
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

    // ==================== 统计数据 ====================
    var games = 0;                 // 游戏局数
    var bestScore = 0;             // 最高分
    var averageScore = 0;          // 平均分
    var bestLargestTile = 0;       // 最大数字方块
    var averageLargestTile = 0;    // 平均最大数字方块

    // 定时器ID
    var moveIntervalId = null;
    var checkIntervalId = null;

    // ==================== 程序入口 ====================
    // 如果只评估模式
    if (EVALUATE_ONLY) {
        var grid = getGrid();
        print(grid);
        evaluate(grid, true);
    } else {
        // 开始游戏循环
        moveIntervalId = setInterval(nextMove, SEARCH_TIME);
    }

    // 检查游戏是否结束
    checkIntervalId = setInterval(function() {
        var score = getScore();
        // 如果游戏结束或达到目标分数
        if (gameLost() || score >= TARGET_SCORE) {
            // 停止所有定时器
            clearInterval(moveIntervalId);
            clearInterval(checkIntervalId);

            // 更新统计数据
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
            var stopReason = gameLost() ? "游戏结束" : "分数达到" + TARGET_SCORE;
            console.log('游戏局数              ' + games + '\n' +
                '本局分数              ' + score + '\n' +
                '本局最大方块          ' + largestTile + '\n' +
                '平均分数              ' + Math.round(averageScore) + '\n' +
                '平均最大方块          ' + Math.round(averageLargestTile) + '\n' +
                '历史最高分            ' + bestScore + '\n' +
                '历史最大方块          ' + bestLargestTile + '\n' +
                stopReason + ' - AI已停止.' + '\n' +
                '\n');

            search.table = {};  // 清空搜索表
        }
    }, RETRY_TIME);

    /**
     * 选择并执行下一步移动
     */
    function nextMove() {
        // 检查游戏是否已结束或分数达到目标
        if (gameLost() || getScore() >= TARGET_SCORE) {
            return;
        }

        var grid = getGrid();
        var move = search(grid, SEARCH_DEPTH, Number.NEGATIVE_INFINITY, true);
        pressKey(move);
    }

    /**
     * 使用深度优先搜索寻找最佳移动
     * @param grid: 游戏网格的一维数组
     * @param depth: 搜索深度，叶节点深度为0
     * @param alpha: 搜索值的下限
     * @param root: 是否为根节点
     * @return 在根节点返回最佳移动方向，其他节点返回评估值
     */
    function search(grid, depth, alpha, root) {
        // 达到叶节点时直接评估
        if (depth <= 0) {
            return evaluate(grid);
        }

        // 初始化搜索缓存表
        if (!search.table) {
            search.table = {};
        }

        // 在缓存表中查找当前局面
        var key = getGridKey(grid);
        var entry = search.table[key];
        if (entry && entry.depth >= depth && (!entry.isBound || entry.value <= alpha)) {
            return root ? entry.move : entry.value;
        }

        // 尝试按最优顺序搜索移动方向
        var moves = [MOVE_RIGHT, MOVE_DOWN, MOVE_LEFT, MOVE_UP];
        // 如果有缓存的最佳移动，优先尝试
        if (entry) {
            var index = moves.indexOf(entry.move);
            var temp = moves[index];
            moves[index] = moves[0];
            moves[0] = temp;
        }

        var bestMove = undefined;
        var alphaImproved = false;

        // 尝试每个移动方向
        for (var i = 0; i < moves.length; i++) {
            var copyGrid = copy(grid);
            var move = moves[i];

            // 如果移动有效
            if (make(copyGrid, move)) {
                bestMove = bestMove || move;
                var value = Number.POSITIVE_INFINITY;

                // 尝试在每个空格中放置一个2，模拟电脑放置新方块
                for (var j = copyGrid.length - 1; j >= 0 && value > alpha; j--) {
                    if (!copyGrid[j]) {
                        copyGrid[j] = 2;
                        value = Math.min(value, search(copyGrid, depth - 1, alpha));
                        copyGrid[j] = 0;
                    }
                }

                // 更新最佳移动
                if (value > alpha) {
                    alpha = value;
                    bestMove = move;
                    alphaImproved = true;
                }
            }
        }

        // 如果没有可行移动
        if (!bestMove) {
            return root ? MOVE_LEFT : ACCEPT_DEFEAT_VALUE + evaluate(grid);
        }

        // 存储搜索结果到缓存表
        search.table[key] = {
            depth: depth,
            value: alpha,
            move: bestMove,
            isBound: !alphaImproved
        };

        return root ? bestMove : alpha;
    }

    /**
     * 评估局面价值
     * @param grid: 游戏网格
     * @param logging: 是否打印详细评估信息
     * @return 局面评估分数
     */
    function evaluate(grid, logging) {
        var value = 0;

        var positionValue = 0;     // 位置价值
        var adjDiffValue = 0;      // 相邻差异价值
        var insulationValue = 0;   // 隔离价值
        var numEmpty = 0;          // 空格数量

        // 遍历整个网格
        for (var r = 0; r < GRID_SIZE; r++) {
            for (var c = 0; c < GRID_SIZE; c++) {
                var tile = get(grid, r, c);
                // 统计空格
                if (!tile) {
                    numEmpty++;
                    continue;
                }
                
                // 计算位置价值（根据位置矩阵）
                positionValue += tile * POSITION_VALUE[r * GRID_SIZE + c];

                // 横向相邻检查
                if (c < GRID_SIZE - 1) {
                    var adjTile = get(grid, r, c + 1);
                    if (adjTile) {
                        // 横向相邻差异
                        adjDiffValue += levelDifference(tile, adjTile) * Math.log(tile + adjTile);

                        // 横向三元组检查（检测被夹住的数字）
                        if (c < GRID_SIZE - 2) {
                            var thirdTile = get(grid, r, c + 2);
                            if (thirdTile && levelDifference(tile, thirdTile) <= 1.1) {
                                var smallerTile = Math.min(tile, thirdTile);
                                insulationValue += levelDifference(smallerTile, adjTile) * Math.log(smallerTile);
                            }
                        }
                    }
                }

                // 纵向相邻检查
                if (r < GRID_SIZE - 1) {
                    adjTile = get(grid, r + 1, c);
                    if (adjTile) {
                        // 纵向相邻差异
                        adjDiffValue += levelDifference(tile, adjTile) * Math.log(tile + adjTile);

                        // 纵向三元组检查
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

        // 空格数量价值计算（使用S型曲线，空格越多越好）
        var numEmptyValue = 11.12249 + (0.05735587 - 11.12249) / (1 + Math.pow((numEmpty / 2.480941), 2.717769));

        // 组合各项评估结果
        value += POSITION_WEIGHT * positionValue;
        value += NUM_EMPTY_WEIGHT * numEmptyValue;
        value += ADJ_DIFF_WEIGHT * adjDiffValue;
        value += INSULATION_WEIGHT * insulationValue;

        // 输出详细评估信息
        if (logging) {
            console.log('综合评分      ' + value + '\n' +
                '  位置评分    ' + (POSITION_WEIGHT * positionValue) + '\n' +
                '  空格评分    ' + (NUM_EMPTY_WEIGHT * numEmptyValue) + '\n' +
                '  相邻差异    ' + (ADJ_DIFF_WEIGHT * adjDiffValue) + '\n' +
                '  隔离评分    ' + (INSULATION_WEIGHT * insulationValue) + '\n'
            );
        }

        return value;
    }

    /**
     * 计算两个数字方块之间的等级差异
     * @param tile1: 第一个方块的值
     * @param tile2: 第二个方块的值
     * @return 两个方块的等级差（对数差）
     */
    function levelDifference(tile1, tile2) {
        return tile1 > tile2 ? LOG2[tile1] - LOG2[tile2] : LOG2[tile2] - LOG2[tile1];
    }

    /**
     * 获取网格中指定位置的值
     * @param grid: 游戏网格
     * @param row: 行坐标
     * @param col: 列坐标
     * @return 该位置的方块值
     */
    function get(grid, row, col) {
        return grid[row * GRID_SIZE + col];
    }

    /**
     * 设置网格中指定位置的值
     * @param grid: 游戏网格
     * @param row: 行坐标
     * @param col: 列坐标
     * @param tile: 要设置的方块值
     */
    function set(grid, row, col, tile) {
        grid[row * GRID_SIZE + col] = tile;
    }

    /**
     * 打印网格到控制台（调试用）
     * @param grid: 游戏网格
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
     * 复制网格
     * @param grid: 原网格
     * @return 新网格副本
     */
    function copy(grid) {
        return grid.slice();
    }

    /**
     * 检查坐标是否在网格范围内
     * @param row: 行坐标
     * @param col: 列坐标
     * @return 是否在范围内
     */
    function inBounds(row, col) {
        return 0 <= row && row < GRID_SIZE && 0 <= col && col < GRID_SIZE;
    }

    /**
     * 在网格上执行移动
     * @param grid: 游戏网格
     * @param move: 移动方向
     * @return 移动是否有效（有变化）
     */
    function make(grid, move) {
        var start = move.dir * (GRID_SIZE - 1);
        var end = (1 - move.dir) * (GRID_SIZE + 1) - 1;
        var inc = 1 - 2 * move.dir;

        var anyMoved = false;  // 是否有方块移动

        // 从指定方向开始遍历网格
        for (var r = start; r != end; r += inc) {
            for (var c = start; c != end; c += inc) {
                if (get(grid, r, c)) {  // 如果当前位置有方块
                    var newr = r + move.drow;
                    var newc = c + move.dcol;
                    var oldr = r;
                    var oldc = c;

                    // 尝试移动方块直到碰到边界或其他方块
                    while (inBounds(newr, newc)) {
                        var target = get(grid, newr, newc);
                        var tile = get(grid, oldr, oldc);
                        
                        if (!target) {  // 目标位置为空，可以移动
                            set(grid, newr, newc, tile);
                            set(grid, oldr, oldc, 0);
                            anyMoved = true;
                        }
                        else if (target === tile) {  // 目标位置数字相同，可以合并
                            // 使用负值标记已合并的方块，防止连续合并
                            set(grid, newr, newc, -2 * tile);
                            set(grid, oldr, oldc, 0);
                            anyMoved = true;
                            break;
                        }
                        else {  // 目标位置有不同数字，不能再移动
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

        // 如果没有方块移动，则移动无效
        if (!anyMoved) {
            return false;
        }

        // 恢复负值标记并统计空格
        var numEmpty = 0;
        for (var i = 0; i < grid.length; i++) {
            if (grid[i] < 0) {
                grid[i] *= -1;  // 恢复负值标记
            }
            else if (!grid[i]) {
                numEmpty++;  // 统计空格
            }
        }

        // 移动后必须有空格（游戏规则）
        if (numEmpty === 0) {
            console.warn('移动后没有空格，移动无效');
            return false;
        }

        return true;
    }

    /**
     * 计算网格的哈希键（用于缓存表）
     * @param grid: 游戏网格
     * @return 哈希键
     */
    function getGridKey(grid) {
        // 初始化哈希表
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

        // 计算哈希值
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
     * 从页面中获取当前游戏网格
     * @return 当前游戏网格
     */
    function getGrid() {
        // 初始化空网格
        var grid = new Array(GRID_SIZE * GRID_SIZE);
        for (var i = 0; i < grid.length; i++) {
            grid[i] = 0;
        }

        // 获取网格容器元素
        var gridContainer = document.evaluate('/html/body/div[1]/main/div[1]/div/div/div[2]/div/div/div/div[4]', document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue;
        if (!gridContainer) return grid;

        // 获取所有方块元素
        var tileElements = gridContainer.querySelectorAll(".absolute");
        if (!tileElements || tileElements.length === 0) return grid;

        // 解析每个方块的值和位置
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

            // 计算行和列（每个方块占25%宽高）
            var row = Math.floor(top / 25);
            var col = Math.floor(left / 25);

            if (row >= 0 && row < GRID_SIZE && col >= 0 && col < GRID_SIZE) {
                set(grid, row, col, value);
            }
        }

        return grid;
    }

    /**
     * 模拟按键
     * @param move: 移动方向
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
     * 检查游戏是否结束
     * @return 游戏是否结束
     */
    function gameLost() {
        var gameOverElement = document.evaluate('/html/body/div[1]/main/div[1]/div/div/div[2]/div/div/div/div[4]/div[2]/div[1]', document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue;
        return gameOverElement && gameOverElement.textContent.includes("Game Over");
    }

    /**
     * 获取当前游戏分数
     * @return 当前分数
     */
    function getScore() {
        var scoreElement = document.evaluate('/html/body/div[1]/main/div[1]/div/div/div[2]/div/div/div/div[1]/div[2]/div[1]/div[2]', document, null, XPathResult.FIRST_ORDERED_NODE_TYPE, null).singleNodeValue;
        return scoreElement ? parseInt(scoreElement.textContent) : 0;
    }

})();

```
