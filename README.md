<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>HMXPANEL</title>
    <style>
        /* General Styles */
        body {
            font-family: 'Poppins', sans-serif;
            background-color: #eaffea;
            color: #333;
            margin: 0;
            padding: 20px;
            text-align: center;
        }

        /* Main Container */
        #mainApp {
            max-width: 650px;
            margin: auto;
            background: #ffffff;
            padding: 20px;
            border-radius: 10px;
            box-shadow: 0 4px 10px rgba(0, 150, 0, 0.1);
            border: 2px solid #a3dca3;
        }

        /* Title */
        h1 {
            font-size: 24px;
            color: #008800;
            text-transform: uppercase;
            letter-spacing: 1px;
            font-weight: bold;
        }

        /* Card Styles */
        .card {
            background: #f0fff0;
            padding: 15px;
            border-radius: 10px;
            margin-bottom: 15px;
            box-shadow: 0 2px 5px rgba(0, 100, 0, 0.1);
            border: 1px solid #a3dca3;
            text-align: left;
        }

        /* Section Headings */
        .card h2 {
            font-size: 18px;
            color: #007700;
            margin-bottom: 10px;
        }

        /* Statistics */
        .stat {
            display: flex;
            justify-content: space-between;
            font-size: 16px;
            font-weight: bold;
            padding: 8px;
            background: #e3ffe3;
            border-radius: 5px;
            margin-bottom: 5px;
        }

        /* History Table */
        .history-table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 10px;
            text-align: center;
        }

        .history-table th, .history-table td {
            padding: 10px;
            font-size: 14px;
            border-bottom: 1px solid #ddd;
        }

        .history-table th {
            background: #d6ffd6;
            color: #006600;
            text-transform: uppercase;
        }

        .history-table tr:hover {
            background: #f0fff0;
        }

        /* Status Colors */
        .win {
            color: green;
            font-weight: bold;
        }

        .loss {
            color: red;
            font-weight: bold;
        }

        .pending {
            color: orange;
            font-weight: bold;
        }

        /* Responsive Design */
        @media (max-width: 768px) {
            body {
                padding: 10px;
            }

            #mainApp {
                max-width: 100%;
                padding: 15px;
            }

            h1 {
                font-size: 20px;
            }

            .card h2 {
                font-size: 16px;
            }

            .stat {
                font-size: 14px;
                padding: 6px;
            }

            .history-table th, .history-table td {
                font-size: 12px;
                padding: 6px;
            }
        }
    </style>
</head>
<body>

    <div id="mainApp">
        <h1>HMXPANEL</h1>

        <div class="card">
            <h2>📅 Period: <span id="currentPeriod">-</span></h2>
            <h2>📊 Result: <span id="currentResult">-</span></h2>
        </div>

        <div class="card">
            <h2>Analysis</h2>
            <div class="stat">
                <span>Win Count</span>
                <span id="totalWins">0</span>
            </div>
            <div class="stat">
                <span>Loss Count</span>
                <span id="totalLosses">0</span>
            </div>
            <div class="stat">
                <span>Win %</span>
                <span id="accuracy">0%</span>
            </div>
            <div class="stat">
                <span>Auto-Reverse</span>
                <span>Inactive</span>
            </div>
        </div>

        <div class="card">
            <h2>History</h2>
            <table class="history-table">
                <thead>
                    <tr>
                        <th>Period</th>
                        <th>Prediction</th>
                        <th>Status</th>
                    </tr>
                </thead>
                <tbody id="historyTable">
                </tbody>
            </table>
        </div>
    </div>

    <script>
        let historyData = [];
        let totalWins = 0;
        let totalLosses = 0;
        let lastFetchedPeriod = null;

        // Function to fetch game result
        async function fetchGameResult() {
            try {
                let response = await fetch("https://api.bdg88zf.com/api/webapi/GetNoaverageEmerdList", {
                    method: "POST",
                    headers: { "Content-Type": "application/json" },
                    body: JSON.stringify({
                        pageSize: 10,
                        pageNo: 1,
                        typeId: 1,
                        language: 0,
                        random: "4a0522c6ecd8410496260e686be2a57c",
                        signature: "334B5E70A0C9B8918B0B15E517E2069C",
                        timestamp: Math.floor(Date.now() / 1000)
                    })
                });

                if (!response.ok) throw new Error(`HTTP error! Status: ${response.status}`);

                let data = await response.json();
                let latestResult = data?.data?.list?.[0];
                if (latestResult) {
                    return { period: latestResult.issueNumber, result: latestResult.number };
                } else {
                    throw new Error("No data found in API response");
                }
            } catch (error) {
                console.error("Error fetching game result:", error);
                return null;
            }
        }

        // Algorithm: Trend Analysis
        function trendAnalysis(history) {
            let bigCount = history.filter(item => item.result >= 5).length;
            let smallCount = history.filter(item => item.result < 5).length;
            return bigCount > smallCount ? "BIG" : "SMALL";
        }

        // Function to update prediction
        async function updatePrediction() {
            let apiResult = await fetchGameResult();
            if (apiResult && apiResult.period !== lastFetchedPeriod) {
                lastFetchedPeriod = apiResult.period;
                let currentPeriod = (BigInt(apiResult.period) + 1n).toString();
                let prediction = trendAnalysis(historyData);

                document.getElementById("currentPeriod").innerText = currentPeriod;
                document.getElementById("currentResult").innerText = apiResult.result >= 5 ? "BIG" : "SMALL";

                historyData.unshift({ period: currentPeriod, prediction: prediction, result: apiResult.result, status: "Pending" });
                updateHistory();
                checkWinLoss(apiResult);
            }
        }

        // Function to check win/loss
        function checkWinLoss(apiResult) {
            if (!apiResult) return;

            historyData.forEach(item => {
                if (item.period === apiResult.period) {
                    let actualResult = apiResult.result >= 5 ? "BIG" : "SMALL";
                    item.status = (item.prediction === actualResult) ? "WIN" : "LOSS";
                }
            });

            updateStats();
            updateHistory();
        }

        // Function to update stats
        function updateStats() {
            totalWins = historyData.filter(item => item.status === "WIN").length;
            totalLosses = historyData.filter(item => item.status === "LOSS").length;

            let accuracy = totalWins / (totalWins + totalLosses) * 100 || 0;
            document.getElementById("totalWins").innerText = totalWins;
            document.getElementById("totalLosses").innerText = totalLosses;
            document.getElementById("accuracy").innerText = accuracy.toFixed(2) + '%';
        }

        // Function to update history table
        function updateHistory() {
            let historyTable = document.getElementById("historyTable");
            historyTable.innerHTML = "";
            historyData.forEach(item => {
                let statusClass = item.status === "WIN" ? "win" : (item.status === "LOSS" ? "loss" : "pending");
                historyTable.innerHTML += `
                    <tr>
                        <td>${item.period}</td>
                        <td>${item.prediction}</td>
                        <td class="${statusClass}">${item.status}</td>
                    </tr>
                `;
            });
        }

        setInterval(updatePrediction, 1000);
    </script>
</body>
</html>
