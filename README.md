<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>交易直播小部件</title>
    <style>
        body {
            margin: 0;
            padding: 10px;
            background: transparent; /* 重要：透明背景 */
            font-family: Arial, sans-serif;
        }
        .container {
            width: 380px;
            height: 280px;
            background: rgba(0, 0, 0, 0.85);
            border-radius: 10px;
            padding: 10px;
            border: 2px solid #333;
        }
        
        .tradingview-widget {
            width: 100%;
            height: 180px;
            margin-bottom: 10px;
        }
        
        .lights-container {
            display: flex;
            justify-content: center;
            gap: 15px;
            margin-bottom: 10px;
        }
        
        .light {
            width: 35px;
            height: 35px;
            border-radius: 50%;
            cursor: pointer;
            transition: all 0.3s ease;
            border: 2px solid #555;
            display: flex;
            align-items: center;
            justify-content: center;
            font-weight: bold;
            font-size: 16px;
        }
        
        .light:hover {
            transform: scale(1.15);
            border-color: gold;
        }
        
        .green { background-color: #00ff00; }
        .red { background-color: #ff0000; }
        .white { background-color: #ffffff; }
        
        /* 颜色深度 */
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
        
        .price-display {
            text-align: center;
            color: white;
            font-size: 16px;
            font-weight: bold;
        }
        
        .vote-count {
            font-size: 10px;
            color: white;
            text-align: center;
            margin-top: 2px;
        }
    </style>
</head>
<body>
    <div class="container">
        <!-- TradingView图表 -->
        <div class="tradingview-widget">
            <div id="tradingview_chart"></div>
        </div>
        
        <!-- 三色灯 -->
        <div class="lights-container">
            <div class="light green level-1" onclick="vote('green')">
                ↑
                <div class="vote-count" id="greenCount">0</div>
            </div>
            <div class="light white level-1" onclick="vote('white')">
                →
                <div class="vote-count" id="whiteCount">0</div>
            </div>
            <div class="light red level-1" onclick="vote('red')">
                ↓
                <div class="vote-count" id="redCount">0</div>
            </div>
        </div>
        
        <!-- 实时价格 -->
        <div class="price-display">
            BTC/USDT: <span id="currentPrice">加载中...</span>
            <span id="priceChange"></span>
        </div>
    </div>

    <!-- TradingView脚本 -->
    <script type="text/javascript" src="https://s3.tradingview.com/tv.js"></script>
    
    <script>
        // 初始化变量
        let votes = { green: 0, white: 0, red: 0 };
        let lastPrice = 0;

        // 页面加载完成后执行
        window.addEventListener('load', function() {
            loadVotes();
            initTradingView();
            setupPriceWebSocket();
        });

        // 加载投票数据
        function loadVotes() {
            const saved = localStorage.getItem('tradingVotes');
            if (saved) {
                votes = JSON.parse(saved);
                updateVoteDisplay();
            }
        }

        // 投票功能
        function vote(color) {
            votes[color]++;
            localStorage.setItem('tradingVotes', JSON.stringify(votes));
            updateVoteDisplay();
            
            // 点击效果
            const light = document.querySelector(`.light.${color}`);
            light.style.boxShadow = '0 0 15px gold';
            setTimeout(() => { light.style.boxShadow = 'none'; }, 300);
        }

        // 更新投票显示
        function updateVoteDisplay() {
            ['green', 'white', 'red'].forEach(color => {
                const count = votes[color];
                document.getElementById(`${color}Count`).textContent = count;
                
                const light = document.querySelector(`.light.${color}`);
                const level = Math.min(Math.floor(count / 3) + 1, 5);
                light.className = `light ${color} level-${level}`;
            });
        }

        // 初始化TradingView图表
        function initTradingView() {
            if (typeof TradingView !== 'undefined') {
                new TradingView.widget({
                    "width": "100%",
                    "height": "180",
                    "symbol": "BINANCE:BTCUSDT",
                    "interval": "5",
                    "timezone": "Asia/Shanghai",
                    "theme": "dark",
                    "style": "1",
                    "locale": "zh_CN",
                    "toolbar_bg": "#f1f3f6",
                    "enable_publishing": false,
                    "hide_top_toolbar": true,
                    "hide_legend": true,
                    "save_image": false,
                    "container_id": "tradingview_chart",
                    "studies": ["volume@tv-basicstudies"]
                });
            }
        }

        // WebSocket获取实时价格
        function setupPriceWebSocket() {
            try {
                const ws = new WebSocket('wss://stream.binance.com:9443/ws/btcusdt@trade');
                
                ws.onopen = function() {
                    console.log('WebSocket连接成功');
                };
                
                ws.onmessage = function(event) {
                    const data = JSON.parse(event.data);
                    const currentPrice = parseFloat(data.p);
                    
                    document.getElementById('currentPrice').textContent = currentPrice.toFixed(2);
                    
                    if (lastPrice > 0) {
                        const change = currentPrice - lastPrice;
                        const changePercent = ((change / lastPrice) * 100).toFixed(2);
                        const changeElement = document.getElementById('priceChange');
                        
                        if (change > 0) {
                            changeElement.textContent = ` (+${changePercent}%)`;
                            changeElement.style.color = '#00ff00';
                        } else if (change < 0) {
                            changeElement.textContent = ` (${changePercent}%)`;
                            changeElement.style.color = '#ff0000';
                        } else {
                            changeElement.textContent = ` (0.00%)`;
                            changeElement.style.color = '#ffffff';
                        }
                    }
                    
                    lastPrice = currentPrice;
                };
                
                ws.onerror = function(error) {
                    console.log('WebSocket错误:', error);
                    // 备用方案：使用HTTP API
                    fetch('https://api.binance.com/api/v3/ticker/price?symbol=BTCUSDT')
                        .then(response => response.json())
                        .then(data => {
                            document.getElementById('currentPrice').textContent = parseFloat(data.price).toFixed(2);
                        });
                };
                
            } catch (error) {
                console.log('WebSocket初始化失败:', error);
            }
        }

        // 重置投票（右键点击）
        document.addEventListener('contextmenu', function(e) {
            e.preventDefault();
            if (confirm('确定要重置所有投票吗？')) {
                votes = { green: 0, white: 0, red: 0 };
                localStorage.setItem('tradingVotes', JSON.stringify(votes));
                updateVoteDisplay();
            }
        });
    </script>
</body>
</html>

