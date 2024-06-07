<script>
    // Define um objeto 'resultado' para armazenar a rota, melhor fitness, histórico e status
    let resultado = {rota: null, melhorFitness: null, historico: [], status: null};

    // Declara arrays para a população inicial e matriz de distâncias
    let populacaoInicial = [];
    let matrizDistancias = [];

    // Função para criar um atraso de execução
    function delay(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }

    // Função para calcular o fitness de um indivíduo baseado nas distâncias
    function calcularFitness(individuo, distancias) {
        let distanciaTotal = 0;
        for (let i = 0; i < individuo.length - 1; i++) {
            distanciaTotal += distancias[individuo[i]][individuo[i + 1]];
        }
        distanciaTotal += distancias[individuo[individuo.length - 1]][individuo[0]];
        return 1 / distanciaTotal;
    }

    // Função para criar a população inicial de indivíduos (caminhos)
    function criarPopulacaoInicial(tamanhoPopulacao, numCidades) {
        const populacao = new Set();
        while (populacao.size < tamanhoPopulacao) {
            const individuo = [];
            for (let j = 0; j < numCidades; j++) {
                individuo.push(j);
            }
            shuffle(individuo); // Embaralha os indivíduos para criar diferentes caminhos
            populacao.add(individuo.join(','));
        }
        return Array.from(populacao).map(individuo => individuo.split(',').map(Number));
    }

    // Função de seleção por torneio para escolher o melhor indivíduo
    function selecaoTorneio(populacao, distancias, tamanhoTorneio) {
        let vencedor = null;
        for (let i = 0; i < tamanhoTorneio; i++) {
            const competidor = populacao[Math.floor(Math.random() * populacao.length)];
            if (vencedor === null || calcularFitness(competidor, distancias) > calcularFitness(vencedor, distancias)) {
                vencedor = competidor;
            }
        }
        return vencedor;
    }

    // Função para realizar o cruzamento entre dois indivíduos
    function cruzamento(pai1, pai2, usarCorteAleatorio = true, pontoDeCorteFixo = 3) {
        let ponto;
        if (usarCorteAleatorio) {
            ponto = Math.floor(Math.random() * pai1.length);
        } else {
            ponto = pontoDeCorteFixo;
        }
        const filho = pai1.slice(0, ponto);
        for (const gene of pai2) {
            if (!filho.includes(gene)) {
                filho.push(gene);
            }
        }
        return filho;
    }

    // Função para mutar um indivíduo trocando dois pontos aleatórios
    function mutacao(individuo) {
        const ponto1 = Math.floor(Math.random() * individuo.length);
        let ponto2 = Math.floor(Math.random() * individuo.length);
        while (ponto1 === ponto2) {
            ponto2 = Math.floor(Math.random() * individuo.length);
        }
        [individuo[ponto1], individuo[ponto2]] = [individuo[ponto2], individuo[ponto1]];
        return individuo;
    }

    // Função para embaralhar um array usando o algoritmo de Fisher-Yates
    function shuffle(array) {
        for (let i = array.length - 1; i > 0; i--) {
            const j = Math.floor(Math.random() * (i + 1));
            [array[i], array[j]] = [array[j], array[i]];
        }
    }

    // Função para ler dados de um arquivo e transformar em matriz de distâncias
    function lerDadosDeArquivo(arquivo) {
        return new Promise((resolve, reject) => {
            const leitor = new FileReader();
            leitor.onload = (evento) => {
                const conteudo = evento.target.result;
                const linhas = conteudo.trim().split('\n');
                const matrizDistancias = linhas.map(linha => {
                    const distancias = linha.trim().replace('\r').split(';').map(Number);
                    return distancias;
                });
                resolve(matrizDistancias);
            };
            leitor.onerror = (evento) => {
                reject(evento.target.error);
            };
            leitor.readAsText(arquivo);
        });
    }

    // Função principal para resolver o problema do caixeiro viajante
    async function resolverCaixeiroViajante(distancias, tamanhoPopulacao, numGeracoes, taxaMutacao, fitnessIdeal) {
        const numCidades = distancias.length;
        populacaoInicial = criarPopulacaoInicial(tamanhoPopulacao, numCidades);

        for (let i = 0; i < numGeracoes; i++) {
            const novaPopulacao = [];
            const torneios = [];

            for (let j = 0; j < tamanhoPopulacao; j++) {
                const pai1 = selecaoTorneio(populacaoInicial, distancias, 2);
                const pai2 = selecaoTorneio(populacaoInicial, distancias, 2);
                let filho = cruzamento(pai1, pai2, true);

                torneios.push({
                    pai1: populacaoInicial.indexOf(pai1),
                    pai2: populacaoInicial.indexOf(pai2),
                    filho: filho.slice(),
                });

                if (Math.random() < taxaMutacao) {
                    filho = mutacao(filho);
                }
                novaPopulacao.push(filho);
            }
            populacaoInicial = novaPopulacao;

            for (const individuo of populacaoInicial) {
                const fitness = calcularFitness(individuo, distancias);
                if (fitness > resultado.melhorFitness) {
                    resultado.melhorFitness = fitness;
                    resultado.rota = individuo;
                }
            }

            resultado.historico = [...resultado.historico, {
                geracao: i + 1,
                populacao: novaPopulacao.map((ind, index) => ({
                    individuo: index + 1,
                    cidades: ind,
                    fitness: calcularFitness(ind, distancias),
                })),
                torneios: torneios,
                melhorDistancia: 1 / resultado.melhorFitness
            }];

            await delay(500);
        }
    }

    // Função para iniciar a resolução dos problemas lendo o arquivo de entrada
    async function resolverProblemas() {
        const arquivo = document.getElementById('fileInput').files[0];
        if (arquivo) {
            try {
                resultado = {rota: null, melhorFitness: null, historico: [], status: null};
                populacaoInicial = [];
                matrizDistancias = [];

                matrizDistancias = await lerDadosDeArquivo(arquivo);
                const tamanhoPopulacao = 5; // Ajuste conforme necessário
                const numGeracoes = 5; // Ajuste conforme necessário
                const taxaMutacao = 0.01; // Ajuste conforme necessário
                const fitnessIdeal = 10; // Ajuste conforme necessário

                resultado.status = 'calculando';
                await resolverCaixeiroViajante(matrizDistancias, tamanhoPopulacao, numGeracoes, taxaMutacao, fitnessIdeal);
                resultado.status = 'calculado';
            } catch (erro) {
                console.error('Erro ao ler o arquivo:', erro);
            }
        }
    }
</script>

<!-- Interface do usuário para selecionar o arquivo e iniciar a resolução -->
<div class="tw-flex tw-flex-row tw-gap-4 tw-justify-center tw-items-center tw-py-8">
    <input type="file" id="fileInput" accept=".txt,.csv">
    <button on:click={resolverProblemas} class="tw-px-4 tw-py-2 !tw-h-fit">Resolver Problemas</button>
</div>

<!-- Exibição dos resultados e histórico -->

<div>
    {#if resultado.status}
        <div class="tw-border tw-rounded-md tw-p-4">
            <span>Status: {resultado.status.toUpperCase()}</span>
            {#if resultado.status === 'calculado'}
                <h1>Resultado:</h1>
                <div>
                    <p>Melhor rota:</p>
                    <table class="tw-border tw-rounded tw-text-center">
                        <tr>
                            {#each resultado.rota as gene}
                                <td>{gene}</td>
                            {/each}
                            <td>{resultado.melhorFitness}</td>
                        </tr>
                    </table>
                </div>
            {/if}
        </div>
        <details class="pico-background-zinc-150 tw-mt-4">
            <summary>População inicial:</summary>
            <div>
                <p>População:</p>
                <table class="tw-border tw-rounded tw-text-center">
                    {#each populacaoInicial as populacao}
                        <tr>
                            {#each populacao as gene}
                                <td>
                                    {gene}
                                </td>
                            {/each}
                        </tr>
                    {/each}
                </table>
            </div>
        </details>
        {#each resultado?.historico as historico}
            <details class="pico-background-zinc-150 tw-mt-4">
                <summary>Geração: {historico.geracao}</summary>
                <div>
                    <p>População:</p>
                    <table class="tw-border tw-rounded tw-text-center">
                        {#each historico.populacao as individuo}
                            <tr>
                                {#each individuo.cidades as cidade}
                                    <td>{cidade}</td>
                                {/each}
                                <td>
                                    {individuo.fitness.toFixed(4)}
                                </td>
                            </tr>
                        {/each}
                    </table>
                </div>
                <div>
                    <p>Torneios:</p>
                    <table class="tw-border tw-rounded tw-text-center">
                        {#each historico.torneios as individuo}
                            <tr>
                                <td>Pai 1</td>
                                <td>Pai 2</td>
                                <td>Filho:</td>
                            </tr>
                            <tr>
                                <td>{individuo.pai1}</td>
                                <td>{individuo.pai2}</td>
                                <td> ></td>
                                {#each individuo.filho as gene}
                                    <td>{gene}</td>
                                {/each}
                            </tr>

                        {/each}
                    </table>
                </div>
            </details>
        {/each}
    {/if}

</div>
