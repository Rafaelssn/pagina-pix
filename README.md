<!DOCTYPE html>
<html lang="pt-BR">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>IA TetoCheck</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #e0f7e9;
            color: #333;
            display: flex;
            flex-direction: column;
            align-items: center;
        }
        h1 {
            color: #2e7d32;
        }
        #upload-section {
            margin-top: 20px;
            padding: 20px;
            border: 1px solid #2e7d32;
            background-color: #ffffff;
            border-radius: 8px;
            text-align: center;
        }
        input[type="file"], button {
            padding: 10px;
            margin: 10px;
            border-radius: 5px;
            border: none;
            color: white;
            background-color: #2e7d32;
            cursor: pointer;
        }
        #result {
            margin-top: 20px;
            width: 90%;
            max-width: 600px;
            border: 1px solid #2e7d32;
            padding: 15px;
            border-radius: 8px;
            background-color: #ffffff;
        }
        .hidden {
            display: none;
        }
        #progress-bar {
            width: 100%;
            background-color: #e0f2f1;
            border-radius: 5px;
            margin-top: 10px;
            position: relative;
        }
        #progress-bar div {
            height: 20px;
            width: 0;
            background-color: #2e7d32;
            border-radius: 5px;
            text-align: center;
            color: white;
            line-height: 20px;
            font-size: 14px;
        }
    </style>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/xlsx/0.16.9/xlsx.full.min.js"></script>
</head>
<body>

    <h1>IA TetoCheck</h1>

    <div id="upload-section">
        <input type="file" id="file-input" accept=".xlsx" />
        <button onclick="analyzeFile()">Analisar</button>
        <div id="progress-bar" class="hidden"><div>0%</div></div>
    </div>

    <div id="result" class="hidden">
        <h2>Resultados da Análise</h2>
        <p id="result-text"></p>
        <button onclick="window.print()">Imprimir Resultados</button>
    </div>

    <script>
        function updateProgressBar(percentage) {
            const progressBar = document.getElementById('progress-bar');
            const progress = progressBar.children[0];
            progress.style.width = percentage + '%';
            progress.textContent = percentage + '%';
        }

        function analyzeFile() {
            const fileInput = document.getElementById('file-input');
            const progressBar = document.getElementById('progress-bar');
            const result = document.getElementById('result');
            const resultText = document.getElementById('result-text');

            if (!fileInput.files.length) {
                alert('Por favor, selecione um arquivo.');
                return;
            }

            result.classList.add('hidden');
            progressBar.classList.remove('hidden');
            updateProgressBar(0);

            const reader = new FileReader();
            reader.onload = function(e) {
                const data = new Uint8Array(e.target.result);
                const workbook = XLSX.read(data, {type: 'array'});

                const tetosSheet = workbook.Sheets["TETOS"];
                const atendimentosSheet = workbook.Sheets["ATENDIMENTOS"];

                const tetos = XLSX.utils.sheet_to_json(tetosSheet);
                const atendimentos = XLSX.utils.sheet_to_json(atendimentosSheet);

                const excedidos = [];
                atendimentos.forEach((atendimento, index) => {
                    const teto = tetos.find(t => 
                        t["CÓDIGO SERVIÇO"] === atendimento["CÓDIGO SERVIÇO"] &&
                        t["CÓDIGO PRESTADOR"] === atendimento["CÓDIGO PRESTADOR"]
                    );
                    if (teto) {
                        const tetoMensal = parseInt(teto["TETO MENSAL DE ATENDIMENTO"], 10);
                        const quantidade = parseInt(atendimento["QUANTIDADE"], 10);

                        if (quantidade > tetoMensal) {
                            excedidos.push({
                                codigo: atendimento["CÓDIGO PRESTADOR"],
                                nome: atendimento["NOME PRESTADOR"],
                                servico: atendimento["SERVIÇO"],
                                teto_mensal: tetoMensal,
                                excedente: quantidade - tetoMensal
                            });
                        }
                    }
                    // Atualiza a barra de progresso
                    const percentage = Math.round(((index + 1) / atendimentos.length) * 100);
                    updateProgressBar(percentage);
                });

                if (excedidos.length > 0) {
                    let output = '<ul>';
                    excedidos.forEach(excedido => {
                        output += `<li>Prestador: ${excedido.nome} (Código: ${excedido.codigo})<br>Serviço: ${excedido.servico}<br>Teto Mensal: ${excedido.teto_mensal}<br>Excedente: ${excedido.excedente}</li><br>`;
                    });
                    output += '</ul>';
                    resultText.innerHTML = output;
                } else {
                    resultText.textContent = 'Nenhum prestador ultrapassou o teto.';
                }

                updateProgressBar(100);
                setTimeout(() => {
                    progressBar.classList.add('hidden');
                    result.classList.remove('hidden');
                }, 500);
            };
            reader.readAsArrayBuffer(fileInput.files[0]);
        }
    </script>
</body>
</html>
