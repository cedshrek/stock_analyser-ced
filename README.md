<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Analyse Boursière</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            padding: 20px;
            background-color: #f4f4f4;
            margin: 0;
        }
        h1 {
            text-align: center;
            color: #333;
        }
        .input-container {
            display: flex;
            justify-content: center;
            gap: 10px;
            margin-bottom: 20px;
        }
        input, button {
            padding: 10px;
            font-size: 16px;
            border: 1px solid #ddd;
            border-radius: 5px;
        }
        button {
            background-color: #4CAF50;
            color: white;
            border: none;
            cursor: pointer;
        }
        button:hover {
            background-color: #45a049;
        }
        #output {
            background: white;
            padding: 15px;
            border-radius: 5px;
            min-height: 100px;
        }
        table {
            width: 100%;
            border-collapse: collapse;
            margin-top: 10px;
        }
        th, td {
            border: 1px solid #ddd;
            padding: 8px;
            text-align: left;
        }
        th {
            background-color: #4CAF50;
            color: white;
        }
        canvas {
            display: block;
            margin: 20px auto;
            max-width: 100%;
        }
        .error {
            color: red;
            text-align: center;
        }
    </style>
</head>
<body>
    <h1>Analyse et Prévision Boursière</h1>
    <div class="input-container">
        <input type="text" id="tickers" placeholder="Entrez les tickers (ex: AAPL,MSFT,GOOGL)">
        <button onclick="analyzeStocks()">Analyser</button>
    </div>
    <div id="output"></div>
    <canvas id="chart" width="800" height="400"></canvas>

    <script>
        let currentChart = null; // Variable pour stocker l'instance du graphique

        async function analyzeStocks() {
            const apiKey = "YOUR_ALPHA_VANTAGE_API_KEY"; // Remplace par ta clé API Alpha Vantage
            const tickers = document.getElementById("tickers").value.split(",").map(t => t.trim());
            const outputDiv = document.getElementById("output");

            if (tickers.length === 0 || tickers[0] === "") {
                outputDiv.innerHTML = '<p class="error">Veuillez entrer au moins un ticker.</p>';
                return;
            }

            if (tickers.length > 5) {
                outputDiv.innerHTML = '<p class="error">Limite de 5 tickers par analyse (limitation de l\'API). Réessayez avec moins de tickers.</p>';
                return;
            }

            outputDiv.innerHTML = "<p>Chargement en cours, veuillez patienter...</p>";
            let results = [];

            for (let ticker of tickers) {
                try {
                    const response = await fetch(`https://www.alphavantage.co/query?function=TIME_SERIES_DAILY&symbol=${ticker}&apikey=${apiKey}`);
                    const data = await response.json();

                    if (data["Time Series (Daily)"]) {
                        const prices = Object.entries(data["Time Series (Daily)"])
                            .slice(0, 90)
                            .map(([date, values]) => ({
                                date,
                                close: parseFloat(values["4. close"])
                            }))
                            .reverse(); // Inverser pour avoir les dates croissantes

                        const current_price = prices[prices.length - 1].close;

                        // Calculer les moyennes mobiles
                        const ma_7 = prices.slice(-7).reduce((sum, p) => sum + p.close, 0) / 7;
                        const ma_30 = prices.slice(-30).reduce((sum, p) => sum + p.close, 0) / 30;
                        const ma_90 = prices.reduce((sum, p) => sum + p.close, 0) / 90;

                        // Prévisions simples basées sur la tendance
                        const potential_7 = ((ma_7 - current_price) / current_price) * 100;
                        const potential_30 = ((ma_30 - current_price) / current_price) * 100;
                        const potential_90 = ((ma_90 - current_price) / current_price) * 100;

                        results.push({
                            ticker,
                            current_price,
                            potential_7,
                            potential_30,
                            potential_90,
                            historical_prices: prices.slice(-30).map(p => p.close),
                            historical_dates: prices.slice(-30).map(p => p.date)
                        });
                    } else {
                        results.push({ ticker, error: "Aucune donnée trouvée pour ce ticker" });
                    }
                } catch (error) {
                    console.error(`Erreur pour ${ticker}:`, error);
                    results.push({ ticker, error: "Erreur lors de l'analyse : " + error.message });
                }
            }

            // Trier par potentiel à 30 jours
            results.sort((a, b) => (b.potential_30 || -Infinity) - (a.potential_30 || -Infinity));
            const top10 = results.slice(0, 10);

            // Afficher les résultats
            let output = `
                <h2>Résultats de l'Analyse</h2>
                <p><strong>Source des données :</strong> Alpha Vantage API</p>
                <table>
                    <tr>
                        <th>Ticker</th>
                        <th>Prix Actuel</th>
                        <th>Potentiel 7 jours (%)</th>
                        <th>Potentiel 1 mois (%)</th>
                        <th>Potentiel 3 mois (%)</th>
                    </tr>
            `;
            for (let result of top10) {
                if (result.error) {
                    output += `<tr><td>${result.ticker}</td><td colspan="4">${result.error}</td></tr>`;
                } else {
                    output += `
                        <tr>
                            <td>${result.ticker}</td>
                            <td>${result.current_price.toFixed(2)}</td>
                            <td>${result.potential_7.toFixed(2)}</td>
                            <td>${result.potential_30.toFixed(2)}</td>
                            <td>${result.potential_90.toFixed(2)}</td>
                        </tr>
                    `;
                }
            }
            output += "</table>";
            outputDiv.innerHTML = output;

            // Détruire le graphique existant s'il y en a un
            if (currentChart) {
                currentChart.destroy();
                currentChart = null;
            }

            // Afficher un graphique pour la première action valide
            const validResult = top10.find(r => !r.error);
            if (validResult) {
                const ctx = document.getElementById("chart").getContext("2d");
                currentChart = new Chart(ctx, {
                    type: "line",
                    data: {
                        labels: validResult.historical_dates,
                        datasets: [{
                            label: `Prix de ${validResult.ticker}`,
                            data: validResult.historical_prices,
                            borderColor: "blue",
                            backgroundColor: "rgba(0, 0, 255, 0.1)",
                            fill: false
                        }]
                    },
                    options: {
                        responsive: true,
                        scales: {
                            x: { title: { display: true, text: "Date" } },
                            y: { title: { display: true, text: "Prix (USD)" } }
                        }
                    }
                });
            } else {
                outputDiv.innerHTML += '<p class="error">Aucune donnée valide pour afficher un graphique.</p>';
            }
        }
    </script>
    <script src="https://cdn.jsdelivr.net/npm/chart.js@4.4.0/dist/chart.umd.min.js"></script>
</body>
</html>
