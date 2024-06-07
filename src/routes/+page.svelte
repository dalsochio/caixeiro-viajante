<script>
    let resultado = {rota: null, melhorFitness: null, historico: [], status: null};
    let populacaoInicial = [];
    let matrizDistancias = [];

    function delay(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }

    function calcularFitness(individuo, distancias) {
        let distanciaTotal = 0;
        for (let i = 0; i < individuo.length - 1; i++) {
            distanciaTotal += distancias[individuo[i]][individuo[i + 1]];
        }
        distanciaTotal += distancias[individuo[individuo.length - 1]][individuo[0]];
        return 1 / distanciaTotal;
    }

    function criarPopulacaoInicial(tamanhoPopulacao, numCidades) {
        const populacao = new Set();
        while (populacao.size < tamanhoPopulacao) {
            const individuo = [];
            for (let j = 0; j < numCidades; j++) {
                individuo.push(j);
            }
            shuffle(individuo);
            populacao.add(individuo.join(','));
        }
        return Array.from(populacao).map(individuo => individuo.split(',').map(Number));
    }


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

    function cruzamento(pai1, pai2, usarCorteAleatorio = true, pontoDeCorteFixo = 3) {
        let ponto;

        // Define o ponto de corte
        if (usarCorteAleatorio) {
            ponto = Math.floor(Math.random() * pai1.length);
        } else {
            ponto = pontoDeCorteFixo;
        }

        // Cria o filho pegando os genes do pai1 desde o início até o ponto de corte
        const filho = pai1.slice(0, ponto);

        // Itera através dos genes do pai2
        for (const gene of pai2) {
            // Se o gene do pai2 não está ainda no filho, adiciona-o ao filho
            if (!filho.includes(gene)) {
                filho.push(gene);
            }
        }

        // Retorna o novo indivíduo (filho) resultante do cruzamento
        return filho;
    }


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
        // Itera do final do array até o início
        for (let i = array.length - 1; i > 0; i--) {
            // Seleciona um índice aleatório entre 0 e i (inclusivo)
            const j = Math.floor(Math.random() * (i + 1));

            // Troca os elementos nos índices i e j
            [array[i], array[j]] = [array[j], array[i]];
        }
    }


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

    async function resolverCaixeiroViajante(distancias, tamanhoPopulacao, numGeracoes, taxaMutacao, fitnessIdeal) {
        const numCidades = distancias.length;
        populacaoInicial = criarPopulacaoInicial(tamanhoPopulacao, numCidades);

        for (let i = 0; i < numGeracoes; i++) {
            const novaPopulacao = [];
            const torneios = [];

            for (let j = 0; j < tamanhoPopulacao; j++) {
                const pai1 = selecaoTorneio(populacaoInicial, distancias, 2);
                const pai2 = selecaoTorneio(populacaoInicial, distancias, 2);
                let filho = cruzamento(pai1, pai2, true); // Assumindo que usarCorteAleatorio é true

                torneios.push({
                    pai1: populacaoInicial.indexOf(pai1), // Índice +1 para identificar o indivíduo
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

            // Armazenar o estado da geração atual no histórico
            resultado.historico = [...resultado.historico, {
                geracao: i + 1,
                populacao: novaPopulacao.map((ind, index) => ({
                    individuo: index + 1,
                    cidades: ind, // Adicionando 1 para representar a cidade corretamente
                    fitness: calcularFitness(ind, distancias),
                })),
                torneios: torneios,
                // melhorRota: rota.slice(),
                melhorDistancia: 1 / resultado.melhorFitness
            }]

            // // Verificar se o fitness ideal foi atingido
            // if (1 / resultado.melhorFitness <= fitnessIdeal) {
            //     console.log(`Parou na geração ${i + 1} com fitness ${1 / resultado.melhorFitness}`);
            //     return {rota: rota, distancia: 1 / melhorFitness, historico: historico};
            // }

            // Esperar 500 milissegundos antes de iniciar a próxima iteração
            await delay(500);
        }
    }


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

<div class="tw-flex tw-flex-row tw-gap-4 tw-justify-center tw-items-center tw-py-8">
    <input type="file" id="fileInput" accept=".txt,.csv">
    <button on:click={resolverProblemas} class="tw-px-4 tw-py-2 !tw-h-fit">Resolver Problemas</button>
</div>

<div>
    {#if resultado.status}
        <div class="tw-border tw-rounded-md">
            <span>Status: {resultado.status.toUpperCase()}</span>
            <h1>Resultado:</h1>
            <div>
                <p>Melhor rota:</p>
                <table>
                    <tr>
                        {#each resultado.rota as gene}
                            <td>{gene}</td>
                        {/each}
                        <td>{resultado.melhorFitness}</td>
                    </tr>
                </table>
            </div>
        </div>
        <details class="pico-background-zinc-150">
            <summary>População inicial:</summary>
            <div>
                <p>População:</p>
                <table>
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
            <details class="pico-background-zinc-150">
                <summary>Geração: {historico.geracao}</summary>
                <div>
                    <p>População:</p>
                    {#each historico.populacao as individuo}
                        <table>
                            <tr>
                                {#each individuo.cidades as cidade}
                                    <td>{cidade}</td>
                                {/each}
                                <td>
                                    {individuo.fitness.toFixed(4)}
                                </td>
                            </tr>

                        </table>
                    {/each}
                </div>
                <div>
                    <p>Torneios:</p>
                    {#each historico.torneios as individuo}
                        <table>
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

                        </table>
                    {/each}
                </div>
            </details>
        {/each}
    {/if}

</div>
