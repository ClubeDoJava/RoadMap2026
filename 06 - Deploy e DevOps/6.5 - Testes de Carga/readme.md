# 6.5 — Testes de Carga com k6

Testes unitários verificam comportamento. Testes de integração verificam que as peças funcionam juntas. **Testes de carga verificam como o sistema se comporta sob pressão real** — e é onde você descobre que a query que demora 10ms com 1 usuário demora 8 segundos com 500 simultâneos.

JMH (módulo 8.3) mede a performance de um método Java isolado em microssegundos. k6 mede o comportamento de um sistema completo: throughput, latência percentil (p95, p99), taxa de erro sob carga, tempo de resposta por endpoint. São ferramentas complementares, não substitutas.

---

## k6 — Testes de Carga como Código

k6 é uma ferramenta open-source de teste de carga onde os cenários são definidos em JavaScript. O resultado pode ser visualizado em Grafana via k6 Cloud ou exportação para InfluxDB/Prometheus.

### Instalação

```bash
# Linux (Debian/Ubuntu)
sudo apt-get install k6

# macOS
brew install k6

# Via Docker (sem instalar)
docker run --rm -i grafana/k6 run - < script.js
```

### Script básico

```javascript
// tests/load/checkout.js
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Rate, Trend } from 'k6/metrics';

// Métricas customizadas
const errosTaxa = new Rate('erros');
const latenciaCheckout = new Trend('latencia_checkout', true);

export const options = {
  // Cenário de rampa de carga
  stages: [
    { duration: '30s', target: 50  },   // Sobe para 50 usuários em 30s
    { duration: '1m',  target: 50  },   // Mantém 50 usuários por 1 minuto
    { duration: '30s', target: 200 },   // Sobe para 200 usuários
    { duration: '2m',  target: 200 },   // Mantém 200 usuários por 2 minutos
    { duration: '30s', target: 0   },   // Desce para 0 (cool-down)
  ],
  // Thresholds: o teste falha se esses limites forem ultrapassados
  thresholds: {
    http_req_duration: ['p(95)<500', 'p(99)<1000'], // 95% < 500ms, 99% < 1s
    http_req_failed: ['rate<0.01'],                   // Menos de 1% de erros
    latencia_checkout: ['p(95)<800'],
  },
};

// Dados de teste
const BASE_URL = __ENV.BASE_URL || 'http://localhost:8080';
const TOKEN = __ENV.JWT_TOKEN || 'token-de-teste';

export default function () {
  const headers = {
    'Content-Type': 'application/json',
    'Authorization': `Bearer ${TOKEN}`,
  };

  // Teste 1: Listar produtos (deve ser rápido por causa do cache Redis)
  const listagem = http.get(`${BASE_URL}/api/v1/produtos?page=0&size=20`, { headers });
  check(listagem, {
    'listagem: status 200': (r) => r.status === 200,
    'listagem: tem produtos': (r) => JSON.parse(r.body).content.length > 0,
  });
  errosTaxa.add(listagem.status !== 200);

  sleep(0.5);

  // Teste 2: Checkout (endpoint crítico)
  const payload = JSON.stringify({
    itens: [{ produtoId: 1, quantidade: 2 }],
    enderecoEntrega: 'Rua Teste, 123',
  });

  const inicio = Date.now();
  const checkout = http.post(`${BASE_URL}/api/v1/pedidos`, payload, { headers });
  latenciaCheckout.add(Date.now() - inicio);

  check(checkout, {
    'checkout: status 201': (r) => r.status === 201,
    'checkout: tem pedidoId': (r) => JSON.parse(r.body).pedidoId !== undefined,
  });
  errosTaxa.add(checkout.status !== 201);

  sleep(1);
}
```

### Executar o teste

```bash
# Execução simples com output no terminal
k6 run tests/load/checkout.js

# Com variáveis de ambiente
BASE_URL=https://minha-api.com JWT_TOKEN=abc123 k6 run tests/load/checkout.js

# Com output para InfluxDB (para Grafana)
k6 run --out influxdb=http://localhost:8086/k6 tests/load/checkout.js

# Resumo em JSON para análise posterior
k6 run --out json=resultados.json tests/load/checkout.js
```

### Interpretando os resultados

```
█ THRESHOLDS

  http_req_duration........: avg=87.45ms min=12ms   med=54ms  max=2.1s  p(90)=187ms p(95)=312ms p(99)=890ms
  http_req_failed..........: 0.23%  ✓ 2314 ✗ 5
  latencia_checkout........: avg=203ms   min=45ms   med=156ms max=2.3s  p(90)=412ms p(95)=698ms

█ MÉTRICAS

  http_reqs..................: 12487 (41.6/s)
  http_req_blocked...........: avg=1.2ms
  http_req_connecting........: avg=0.8ms
  http_req_duration..........: avg=87.45ms
  http_req_receiving.........: avg=0.4ms
  http_req_sending...........: avg=0.2ms
  http_req_waiting...........: avg=86.8ms   ← tempo real no servidor
  vus........................: 200 min=0 max=200
  vus_max....................: 200
```

**O que cada métrica significa:**

| Métrica | O que mede | O que procurar |
|---|---|---|
| `p(95)` | 95% das requests foram mais rápidas que este valor | < 500ms para APIs |
| `p(99)` | 99% das requests foram mais rápidas | < 1s para APIs |
| `http_req_failed` | Taxa de erros (5xx, timeouts) | < 1% em carga normal |
| `http_req_waiting` | Tempo efetivo de processamento no servidor | Comparar com/sem carga |
| `vus` | Virtual Users ativos no momento | Correlacionar com latência |

---

## Cenários de teste com k6

### Teste de smoke (verificação rápida)

```javascript
export const options = {
  vus: 1,
  duration: '30s',
};
```

Roda com 1 usuário por 30 segundos. Confirma que o script funciona e o sistema responde corretamente antes de executar testes pesados.

### Teste de carga (carga esperada)

```javascript
export const options = {
  stages: [
    { duration: '5m', target: 100 },   // Rampa de subida
    { duration: '30m', target: 100 },  // Carga sustentada
    { duration: '5m', target: 0 },     // Rampa de descida
  ],
};
```

Simula a carga típica do sistema. Baseline para comparações futuras.

### Teste de estresse (acima do esperado)

```javascript
export const options = {
  stages: [
    { duration: '2m', target: 100 },
    { duration: '5m', target: 100 },
    { duration: '2m', target: 200 },
    { duration: '5m', target: 200 },
    { duration: '2m', target: 400 },   // Dobra a carga
    { duration: '5m', target: 400 },
    { duration: '10m', target: 0 },
  ],
};
```

Descobre o ponto de saturação: a partir de quantos usuários o sistema começa a degradar.

### Teste de spike (pico repentino — ex: Black Friday)

```javascript
export const options = {
  stages: [
    { duration: '10s', target: 100 },   // Carga normal
    { duration: '1m', target: 100 },
    { duration: '10s', target: 1400 },  // Spike de 14x em 10 segundos
    { duration: '3m', target: 1400 },
    { duration: '10s', target: 100 },   // Volta ao normal
    { duration: '3m', target: 100 },
    { duration: '10s', target: 0 },
  ],
};
```

---

## Visualizando resultados no Grafana

### Docker Compose com k6 + InfluxDB + Grafana

```yaml
# docker-compose.k6.yml
services:
  influxdb:
    image: influxdb:1.8
    environment:
      INFLUXDB_DB: k6
    ports:
      - "8086:8086"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      GF_AUTH_ANONYMOUS_ENABLED: "true"
      GF_AUTH_ANONYMOUS_ORG_ROLE: Admin
    volumes:
      - grafana_data:/var/lib/grafana
    depends_on:
      - influxdb

volumes:
  grafana_data:
```

```bash
# Subir a stack de monitoramento
docker compose -f docker-compose.k6.yml up -d

# Rodar o teste enviando para InfluxDB
k6 run --out influxdb=http://localhost:8086/k6 tests/load/checkout.js

# Abrir Grafana: http://localhost:3001
# Dashboard k6: import ID 2587 do Grafana marketplace
```

---

## Integrando no CI/CD (GitHub Actions)

```yaml
# Executar testes de carga em ambiente de staging após deploy
load-test:
  name: Testes de Carga
  runs-on: ubuntu-latest
  needs: deploy-staging
  if: github.ref == 'refs/heads/main'

  steps:
    - uses: actions/checkout@v4

    - name: Instalar k6
      run: |
        sudo gpg -k
        sudo gpg --no-default-keyring --keyring /usr/share/keyrings/k6-archive-keyring.gpg \
          --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys C5AD17C747E3415A3642D57D77C6C491D6AC1D69
        echo "deb [signed-by=/usr/share/keyrings/k6-archive-keyring.gpg] \
          https://dl.k6.io/deb stable main" | sudo tee /etc/apt/sources.list.d/k6.list
        sudo apt-get update && sudo apt-get install k6

    - name: Executar testes de carga em staging
      run: |
        k6 run \
          -e BASE_URL=${{ secrets.STAGING_URL }} \
          -e JWT_TOKEN=${{ secrets.STAGING_TEST_TOKEN }} \
          --out json=k6-results.json \
          tests/load/checkout.js

    - name: Upload dos resultados
      if: always()
      uses: actions/upload-artifact@v4
      with:
        name: k6-load-test-results
        path: k6-results.json
```

> **Cuidado:** testes de carga em produção só com aprovação explícita (`workflow_dispatch` com input de confirmação), nunca automáticos. O ambiente de staging deve ser dimensionado próximo ao de produção para que os resultados sejam representativos.

---

## Projeto prático

Construa o seguinte relatório para a API do módulo 5:

1. **Baseline:** rode o teste de carga com Spring MVC padrão (pool de threads bloqueantes). Documente p50, p95, p99, throughput máximo e taxa de erro.
2. **Virtual Threads:** habilite `spring.threads.virtual.enabled=true`. Rode o mesmo teste. Compare os resultados.
3. **Cache:** identifique o endpoint de listagem. Meça com e sem `@Cacheable`. Documente a diferença de p95.
4. **Gargalo:** use o VisualVM (módulo 8.3) durante o teste de estresse para identificar se o gargalo é CPU, I/O ou banco de dados.

Documente tudo no `decisoes.md`: o que você mediu, o que melhorou, o que não mudou e por quê.
