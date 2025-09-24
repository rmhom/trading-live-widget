<!DOCTYPE html>
<html lang="zh-CN">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>交易直播小部件</title>
    <style>
        body {
            margin: 0;
            padding: 5px;
            background: transparent;
            font-family: Arial, sans-serif;
        }
        .container {
            width: 300px;
            height: 150px;
            background: rgba(0, 0, 0, 0.9);
            border-radius: 8px;
            padding: 10px;
            border: 1px solid #333;
        }
        
        /* 价格显示区域 - 更大更明显 */
        .price-section {
            text-align: center;
            margin-bottom: 15px;
        }
        
        .symbol {
            color: #ccc;
            font-size: 14px;
            margin-bottom: 5px;
        }
        
        .price {
            color: white;
            font-size: 24px;
            font-weight: bold;
            margin: 5px 0;
        }
        
        .change {
            font-size: 14px;
            font-weight: bold;
        }
        
        .positive { color: #00ff00; }
        .negative { color: #ff0000; }
        .neutral { color: #cccccc; }
        
        /* 三色灯区域 */
        .lights-section {
            text-align: center;
        }
        
        .lights-title {
            color: #ccc;
            font-size: 12px;
            margin-bottom: 8px;
        }
        
        .lights-container {
            display: flex;
            justify-content: center;
            gap: 15px;
        }
        
        .light-wrapper {
            display: flex;
            flex-direction: column;
            align-items: center;
        }
        
        .light {
            width: 30px;
            height: 30px;
            border-radius: 50%;
            cursor: pointer;
            transition: all 0.3s ease;
            border: 2px solid #555;
            display: flex;
            align-items: center;
            justify-content: center;
            font-weight: bold;
            font-size: 14px;
        }
        
        .light:hover {
            transform: scale(1.1);
            border-color: gold;
        }
        
        .green { background-color: #00ff00; }
        .red { background-color: #ff0000; }
        .white { background-color: #ffffff; }
        
        /* 颜色深度级别 */
        .green.level-1 { background-color: #00ff00; }
        .green.level-2 { background-color: #00cc00; }
        .green.level-3 { background-color: #009900; }
        .green.level-4 { background-color: #006600; }
        .green.level-5 { background-color: #004400; }
        
        .red.level-1 { background-color: #ff0000; }
        .red.level-2 { background-color: #cc0000; }
        .red.level-3 { background-color: #990000; }
        .red.level-4 { background-color: #660000; }
        .red.level-5 { background-color: #440000; }
        
        .white.level-1 { background-color: #ffffff; }
        .white.level-2 { background-color: #cccccc; }
        .white.level-3 { background-color: #999999; }
        .white.level-4 { background-color: #666666; }
        .white.level-5 { background-color: #444444; }
        
        .vote-count {
            font-size: 10px;
            color: #ccc;
            margin-top: 3px;
            font-weight: bold;
        }
        
        /* 状态指示器 */
        .status-indicator {
            width: 8px;
            height: 8px;
            border-radius: 50%;
            display: inline-block;
            margin-left: 5px;
            animation: pulse 2s infinite;
        }
        
        @keyframes pulse {
            0% { opacity: 1; }
            50% { opacity: 0.5; }
            100% { opacity: 1; }
        }
    </style>
</head>
<body>
    <div class="container">
        <!-- 价格显示区域 -->
        <div class="price-section">
            <div class="symbol">
                BTC/USDT 
                <span id="statusIndicator" class="status-indicator" style="background-color: #ccc;"></span>
            </div>
            <div class="price" id="currentPrice">--.--</div>
            <div class="change">
                <span id="priceChange">0.00%</span>
            </div>
        </div>
        
        <!-- 三色灯投票区域 -->
        <div class="lights-section">
            <div class="lights-title">观众情绪投票 ↑→↓</div>
            <div class="lights-container">
                <div class="light-wrapper">
                    <div class="light green level-1" onclick="vote('green')" title="看涨">↑</div>
                    <div class="vote-count" id="greenCount">0</div>
                </div>
                <div class="light-wrapper">
                    <div class="light white level-1" onclick="vote('white')" title="持平">→</div>
                    <div class="vote-count" id="whiteCount">0</div>
                </div>
                <div class="light-wrapper">
                    <div class="light red level-1" onclick="vote('red')" title="看跌">↓</div>
                    <div class="vote-count" id="redCount">0</div>
                </div>
            </div>
        </div>
    </div>

    <script>
        // 初始化变量
        let votes = { green: 0, white: 0, red: 0 };
        let lastPrice = 0;
        let websocket = null;

        // 页面加载完成后执行
        window.addEventListener('load', function() {
            loadVotes();
            setupPriceWebSocket();
            startPriceUpdate();
        });

        // 加载投票数据
        function loadVotes() {
            try {
                const saved = localStorage.getItem('tradingVotes');
                if (saved) {
                    votes = JSON.parse(saved);
                    updateVoteDisplay();
                }
            } catch (e) {
                console.log('加载投票数据失败:', e);
            }
        }

        // 投票功能
        function vote(color) {
            votes[color]++;
            try {
                localStorage.setItem('tradingVotes', JSON.stringify(votes));
            } catch (e) {
                console.log('保存投票数据失败:', e);
            }
            updateVoteDisplay();
            
            // 点击效果
            const light = document.querySelector(`.light.${color}`);
            light.style.boxShadow = '0 0 10px gold';
            setTimeout(() => { light.style.boxShadow = 'none'; }, 300);
        }

        // 更新投票显示
        function updateVoteDisplay() {
            ['green', 'white', 'red'].forEach(color => {
                const count = votes[color] || 0;
                const countElement = document.getElementById(`${color}Count`);
                if (countElement) {
                    countElement.textContent = count;
                }
                
                const light = document.querySelector(`.light.${color}`);
                if (light) {
                    const level = Math.min(Math.floor(count / 2) + 1, 5);
                    light.className = `light ${color} level-${level}`;
                }
            });
        }

        // WebSocket获取实时价格
        function setupPriceWebSocket() {
            try {
                websocket = new WebSocket('wss://stream.binance.com:9443/ws/btcusdt@trade');
                
                websocket.onopen = function() {
                    updateStatusIndicator('#00ff00');
                    console.log('价格连接成功');
                };
                
                websocket.onmessage = function(event) {
                    try {
                        const data = JSON.parse(event.data);
                        const currentPrice = parseFloat(data.p);
                        updatePriceDisplay(currentPrice);
                    } catch (e) {
                        console.log('价格数据解析失败:', e);
                    }
                };
                
                websocket.onerror = function(error) {
                    updateStatusIndicator('#ff0000');
                    console.log('WebSocket错误:', error);
                };
                
                websocket.onclose = function() {
                    updateStatusIndicator('#ff9900');
                    // 尝试重新连接
                    setTimeout(setupPriceWebSocket, 5000);
                };
                
            } catch (error) {
                console.log('WebSocket初始化失败:', error);
                // 使用HTTP备用方案
                startPriceUpdate();
            }
        }

        // HTTP备用价格更新
        function startPriceUpdate() {
            setInterval(() => {
                if (!websocket || websocket.readyState !== WebSocket.OPEN) {
                    fetch('https://api.binance.com/api/v3/ticker/price?symbol=BTCUSDT')
                        .then(response => response.json())
                        .then(data => {
                            const currentPrice = parseFloat(data.price);
                            updatePriceDisplay(currentPrice);
                        })
                        .catch(error => {
                            console.log('HTTP价格获取失败:', error);
                            updateStatusIndicator('#ff0000');
                        });
                }
            }, 5000);
        }

        // 更新价格显示
        function updatePriceDisplay(currentPrice) {
            const priceElement = document.getElementById('currentPrice');
            const changeElement = document.getElementById('priceChange');
            const statusIndicator = document.getElementById('statusIndicator');
            
            if (priceElement) {
                priceElement.textContent = currentPrice.toFixed(2);
            }
            
            if (lastPrice > 0) {
                const change = currentPrice - lastPrice;
                const changePercent = ((change / lastPrice) * 100).toFixed(2);
                
                if (changeElement) {
                    if (change > 0) {
                        changeElement.textContent = `+${changePercent}%`;
                        changeElement.className = 'change positive';
                        statusIndicator.style.backgroundColor = '#00ff00';
                    } else if (change < 0) {
                        changeElement.textContent = `${changePercent}%`;
                        changeElement.className = 'change negative';
                        statusIndicator.style.backgroundColor = '#ff0000';
                    } else {
                        changeElement.textContent = `0.00%`;
                        changeElement.className = 'change neutral';
                        statusIndicator.style.backgroundColor = '#cccccc';
                    }
                }
            }
            
            lastPrice = currentPrice;
            updateStatusIndicator('#00ff00');
        }

        // 更新状态指示器
        function updateStatusIndicator(color) {
            const indicator = document.getElementById('statusIndicator');
            if (indicator) {
                indicator.style.backgroundColor = color;
            }
        }

        // 重置投票（右键点击）
        document.addEventListener('contextmenu', function(e) {
            e.preventDefault();
            if (confirm('确定要重置所有投票吗？')) {
                votes = { green: 0, white: 0, red: 0 };
                try {
                    localStorage.setItem('tradingVotes', JSON.stringify(votes));
                } catch (e) {
                    console.log('重置投票数据失败:', e);
                }
                updateVoteDisplay();
            }
        });

        // 定时保存投票数据（防止数据丢失）
        setInterval(() => {
            try {
                localStorage.setItem('tradingVotes', JSON.stringify(votes));
            } catch (e) {
                console.log('定时保存失败:', e);
            }
        }, 30000);
    </script>
</body>
</html>
