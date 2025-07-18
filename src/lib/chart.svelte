<!-- Chart.svelte cleaned up for Svelte component usage -->

<link rel="stylesheet" type="text/css" href="https://code.highcharts.com/css/stocktools/gui.css">
<link rel="stylesheet" type="text/css" href="https://code.highcharts.com/css/annotations/popup.css">

<script>
    import { onMount } from 'svelte';
    let chartRef;
    const smaState = {
        9: false,
        14: false,
        50: false,
        100: false
    };

    // List of Highcharts script URLs to load
    const highchartsScripts = [
        'https://code.highcharts.com/stock/highstock.js',
        'https://code.highcharts.com/stock/indicators/indicators-all.js',
        'https://code.highcharts.com/stock/modules/drag-panes.js',
        'https://code.highcharts.com/modules/annotations-advanced.js',
        'https://code.highcharts.com/modules/price-indicator.js',
        'https://code.highcharts.com/modules/full-screen.js',
        'https://code.highcharts.com/modules/stock-tools.js',
        'https://code.highcharts.com/stock/modules/heikinashi.js',
        'https://code.highcharts.com/stock/modules/hollowcandlestick.js',
        'https://code.highcharts.com/modules/exporting.js',
        'https://code.highcharts.com/modules/accessibility.js',
        'https://code.highcharts.com/connectors/morningstar/connectors-morningstar.js',
        'https://code.highcharts.com/themes/adaptive.js'
    ];

    function loadScript(src) {
        return new Promise((resolve, reject) => {
            if (document.querySelector(`script[src="${src}"]`)) {
                resolve();
                return;
            }
            const script = document.createElement('script');
            script.src = src;
            script.async = false;
            script.onload = resolve;
            script.onerror = reject;
            document.body.appendChild(script);
        });
    }

    function generateRandomOHLCData(days = 500) {
        const ohlc = [], volumeData = [];
        const oneDay = 24 * 3600 * 1000;
        let time = Date.now() - days * oneDay;
        let price = 100;
        let bias = Math.random() < 0.5 ? -1 : 1;

        for (let i = 0; i < days; i++) {
            const volatility = Math.random() * 5 + 1;
            const open = price;
            const change = (Math.random() - 0.5 + 0.1 * bias) * volatility;
            const close = open + change;
            const high = Math.max(open, close) + Math.random() * 2;
            const low = Math.min(open, close) - Math.random() * 2;
            const volume = Math.floor(Math.random() * 9000 + 1000);
            ohlc.push([time, +open.toFixed(2), +high.toFixed(2), +low.toFixed(2), +close.toFixed(2)]);
            volumeData.push([time, volume]);
            price = close;
            time += oneDay;
            if (i % 30 === 0) bias *= -1;
        }

        return { ohlc, volumeData };
    }

    function loadChartForTicker(ticker) {
        const { ohlc, volumeData } = generateRandomOHLCData();
        // Reset SMA state
        Object.keys(smaState).forEach(k => smaState[k] = false);

        chartRef = window.Highcharts.stockChart('container', {
            title: { text: `Stock Chart: ${ticker}` },
            yAxis: [{
                labels: { align: 'left' },
                height: '80%',
                resize: { enabled: true }
            }, {
                labels: { align: 'left' },
                top: '80%',
                height: '20%',
                offset: 0
            }],
            rangeSelector: { selected: 2 },
            tooltip: {
                shape: 'square',
                headerShape: 'callout',
                borderWidth: 0,
                shadow: false,
                fixed: true
            },
            series: [{
                type: 'candlestick',
                id: 'price-series',
                name: `${ticker} Price`,
                data: ohlc
            }, {
                type: 'column',
                name: `${ticker} Volume`,
                data: volumeData,
                yAxis: 1
            }]
        });
    }

    function toggleSMA(period) {
        if (!chartRef) return;
        const id = `sma-${period}`;
        if (!smaState[period]) {
            chartRef.addSeries({
                type: 'sma',
                linkedTo: 'price-series',
                params: { period },
                id,
                name: `SMA ${period}`
            });
        } else {
            const smaSeries = chartRef.get(id);
            if (smaSeries) smaSeries.remove();
        }
        smaState[period] = !smaState[period];
    }

    onMount(async () => {
        // Load all Highcharts scripts sequentially
        for (const src of highchartsScripts) {
            await loadScript(src);
        }
        // DOM event listeners
        document.getElementById('ticker-form').addEventListener('submit', function (e) {
            e.preventDefault();
            const ticker = document.getElementById('ticker-input').value.trim();
            if (ticker) loadChartForTicker(ticker);
        });
        document.getElementById('toggle-sma-9-btn').addEventListener('click', () => toggleSMA(9));
        document.getElementById('toggle-sma-14-btn').addEventListener('click', () => toggleSMA(14));
        document.getElementById('toggle-sma-50-btn').addEventListener('click', () => toggleSMA(50));
        document.getElementById('toggle-sma-100-btn').addEventListener('click', () => toggleSMA(100));
        // Initial load
        loadChartForTicker('NVIDIA');
    });
</script>

<div class="top-bar">
    <form id="ticker-form">
        <label for="ticker-input">Enter ticker:</label>
        <input type="text" id="ticker-input" placeholder="e.g. ibm" required>
        <button type="submit">Load</button>
    </form>

    <div class="sma-buttons">
        <button id="toggle-sma-9-btn" class="sma-button">Toggle SMA 9</button>
        <button id="toggle-sma-14-btn" class="sma-button">Toggle SMA 14</button>
        <button id="toggle-sma-50-btn" class="sma-button">Toggle SMA 50</button>
        <button id="toggle-sma-100-btn" class="sma-button">Toggle SMA 100</button>
    </div>
</div>

<div id="container" class="chart"></div>

<style>
    .top-bar {
        display: flex;
        align-items: center;
        justify-content: space-between;
        gap: 20px;
        flex-wrap: wrap;
        margin-bottom: 20px;
    }

    #ticker-form {
        display: flex;
        align-items: center;
        gap: 10px;
        flex-shrink: 0;
    }

    #ticker-form label {
        font-size: 14px;
        font-weight: bold;
    }

    #ticker-input {
        padding: 8px 12px;
        border: 1px solid #ccc;
        border-radius: 8px;
        font-size: 14px;
        width: 150px;
    }

    #ticker-form button {
        padding: 8px 16px;
        background-color: #007bff;
        border: none;
        border-radius: 8px;
        color: white;
        font-size: 14px;
        cursor: pointer;
        transition: background-color 0.3s ease;
    }

    #ticker-form button:hover {
        background-color: #0056b3;
    }

    .sma-buttons {
        display: flex;
        gap: 10px;
        flex-wrap: wrap;
    }

    .sma-button {
        padding: 8px 14px;
        background-color: #6c757d;
        color: white;
        border: none;
        border-radius: 8px;
        font-size: 14px;
        cursor: pointer;
        transition: background-color 0.3s ease;
    }

    .sma-button:hover {
        background-color: #5a6268;
    }

    #container {
        border: 1px solid #ddd;
        border-radius: 8px;
        background: white;
        height: 80vh;
        min-height: 300px;
    }

    @media screen and (max-width: 600px) {
        #container {
            height: 50vh;
        }
    }
</style>